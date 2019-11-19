---
layout: post
title:  "Kubebuilder"
date:   2019-11-19 22:30:00 +0800
categories: Terraform
---

# Kubebuilder

Kubebuilder是一个使用自定义资源定义（[CRD](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)）构建Kubernetes API的框架。

## Quick Start

### Install
```bash
git clone https://github.com/kubernetes-sigs/kubebuilder
cd kubebuilder
make install
```

### Create Project
```bash
kubebuilder init --domain oudishen.net
```

### Create API
```bash
kubebuilder create api --group sample --version v1alpha1 --kind PodSet
```

### Edit the API definition and the reconciliation business logic

api/v1alpha1/podset_types.go
```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// PodSetSpec defines the desired state of PodSet
type PodSetSpec struct {
	Replicas int32 `json:"replicas"`
}

// PodSetStatus defines the observed state of PodSet
type PodSetStatus struct {
	AvailableReplicas int32 `json:"availableReplicas"`
}

// +kubebuilder:object:root=true

// PodSet is the Schema for the podsets API
type PodSet struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   PodSetSpec   `json:"spec,omitempty"`
	Status PodSetStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// PodSetList contains a list of PodSet
type PodSetList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []PodSet `json:"items"`
}

func init() {
	SchemeBuilder.Register(&PodSet{}, &PodSetList{})
}
```

controller/podset_controller.go
```go

package controllers

import (
	"context"

	"github.com/pkg/errors"
	corev1 "k8s.io/api/core/v1"
	apierr "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/source"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/util/rand"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	"oudishen.net/podset/api/v1alpha1"
)

// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
	client.Client
	Log    logr.Logger
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=sample.oudishen.net,resources=podsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=sample.oudishen.net,resources=podsets/status,verbs=get;update;patch

func (r *PodSetReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	logger := r.Log.WithValues("podset", req.NamespacedName)
	logger.Info("Reconciling")

	// Fetch the PodSet instance
	instance := &v1alpha1.PodSet{}
	err := r.Get(context.TODO(), req.NamespacedName, instance)
	if err != nil {
		if apierr.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, errors.Wrap(err, "r.Get")
	}

	pods := &corev1.PodList{}
	err = r.List(
		context.TODO(),
		pods,
		client.InNamespace(req.Namespace),
		&client.MatchingLabels{
			"app": req.Name,
		},
	)
	replicas := int32(len(pods.Items))

	logger.Info("", "Update Status.AvailableReplicas to ", replicas)
	instance.Status.AvailableReplicas = replicas
	if err := r.Update(context.TODO(), instance); err != nil {
		logger.Info("", "Failed to update Status.AvailableReplicas to ", replicas)
		return reconcile.Result{}, errors.Wrap(err, "r.Status().Update")
	}

	n := int(instance.Spec.Replicas - replicas)
	if n > 0 {
		for i := 0; i < n; i++ {
			// Define a new Pod object
			pod := newPodForCR(instance)

			// Set PodSet instance as the owner and controller
			if err := controllerutil.SetControllerReference(instance, pod, r.Scheme); err != nil {
				return reconcile.Result{}, errors.Wrap(err, "controllerutil.SetControllerReference")
			}

			logger.Info("Creating a Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
			err = r.Create(context.TODO(), pod)
			if err != nil {
				return reconcile.Result{}, errors.Wrap(err, "r.Create pod")
			}
		}

		// Pod created successfully - don't requeue
		return reconcile.Result{}, nil
	}

	if n < 0 {
		for i := 0; i < -n; i++ {
			pod := pods.Items[i].DeepCopy()
			logger.Info("Deleting a Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
			err = r.Delete(context.TODO(), pod)
			if err != nil {
				return reconcile.Result{}, errors.Wrap(err, "r.Delete pod")
			}
		}

		// Pod delete successfully - don't requeue
		return reconcile.Result{}, nil
	}

	// Pod already exists - don't requeue
	logger.Info("Skip reconcile: replicas already adjust")
	return reconcile.Result{}, nil
}

// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *v1alpha1.PodSet) *corev1.Pod {
	labels := map[string]string{
		"app": cr.Name,
	}
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cr.Name + "-pod-" + rand.String(5),
			Namespace: cr.Namespace,
			Labels:    labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "busybox",
					Image:   "busybox:1",
					Command: []string{"sleep", "3600"},
				},
			},
		},
	}
}

func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&v1alpha1.PodSet{}).
		Watches(
			&source.Kind{Type: &corev1.Pod{}},
			&handler.EnqueueRequestForOwner{
				IsController: true,
				OwnerType:    &v1alpha1.PodSet{},
			},
		).
		Complete(r)
}
```

### Generate manifests
```bash
make manifests
```

### Generate code
```bash
make generate
```

### Install CRDs into cluster
```bash
make install
```

### Build manager binary
```bash
make manager
```

### Build the docker image
```bash
make docker-build
make docker-push
```

### Deploy controller into cluster
```bash
make deploy
```

### Create PodSet

config/samples/sample_v1alpha1_podset.yaml
```yaml
apiVersion: sample.oudishen.net/v1alpha1
kind: PodSet
metadata:
  name: podset-sample
spec:
  replicas: 3
```

```bash
kubectl apply -f config/samples/sample_v1alpha1_podset.yaml
```

kubectl get podset podset-sample -oyaml
```yaml
apiVersion: sample.oudishen.net/v1alpha1
kind: PodSet
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"sample.oudishen.net/v1alpha1","kind":"PodSet","metadata":{"annotations":{},"name":"podset-sample","namespace":"default"},"spec":{"replicas":3}}
  creationTimestamp: "2019-11-19T16:26:20Z"
  generation: 3
  name: podset-sample
  namespace: default
  resourceVersion: "8558160"
  selfLink: /apis/sample.oudishen.net/v1alpha1/namespaces/default/podsets/podset-sample
  uid: 5210a92a-0ae9-11ea-bb67-00163e097f91
spec:
  replicas: 3
status:
  availableReplicas: 3
```

kubectl get pod | grep podset
```
podset-sample-pod-6b56t                           1/1     Running            0          73s
podset-sample-pod-d8pwn                           1/1     Running            0          72s
podset-sample-pod-wwp8p                           1/1     Running            0          72s
```

