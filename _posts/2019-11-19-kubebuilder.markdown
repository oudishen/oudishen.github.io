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
/*

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"

	"github.com/go-logr/logr"
	"github.com/pkg/errors"
	corev1 "k8s.io/api/core/v1"
	apierr "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/util/rand"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	"oudishen.net/podset/api/v1alpha1"
)

const (
	PodSetFinalizer = "podset.sample.oudishen.net"
)

func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&v1alpha1.PodSet{}).
		Owns(&corev1.Pod{}).
		Complete(r)
}

// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
	client.Client
	Log      logr.Logger
	Scheme   *runtime.Scheme
	Recorder record.EventRecorder
}

// +kubebuilder:rbac:groups=sample.oudishen.net,resources=podsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=sample.oudishen.net,resources=podsets/status,verbs=get;update;patch

func (r *PodSetReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	logger := r.Log.WithValues("podset", req.NamespacedName)
	logger.Info("Reconciling")

	// Fetch the PodSet instance
	podSet := &v1alpha1.PodSet{}
	err := r.Get(context.TODO(), req.NamespacedName, podSet)
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
			"podset": req.Name,
		},
	)

	scope := &PosSetScope{
		Client:   r.Client,
		ctx:      context.TODO(),
		logger:   logger.WithName("PodSetScope"),
		scheme:   r.Scheme,
		podSet:   podSet,
		pods:     pods,
		recorder: r.Recorder,
	}

	defer func() {
		if err := scope.Close(); err != nil {
			logger.Info("scope.Close error: " + err.Error())
		}
	}()

	// Handle deleted clusters
	if !podSet.DeletionTimestamp.IsZero() {
		return scope.ReconcileDelete()
	}

	// Handle non-deleted clusters
	return scope.ReconcileNormal()
}

type PosSetScope struct {
	client.Client
	ctx      context.Context
	logger   logr.Logger
	scheme   *runtime.Scheme
	podSet   *v1alpha1.PodSet
	pods     *corev1.PodList
	recorder record.EventRecorder
}

func (s *PosSetScope) Close() error {
	if err := s.Client.Update(context.TODO(), s.podSet); err != nil {
		return errors.Wrap(err, "s.client.Update")
	}
	if len(s.podSet.Finalizers) == 0 {
		return nil
	}
	if err := s.Client.Status().Update(context.TODO(), s.podSet); err != nil {
		return errors.Wrap(err, "s.Client.Status().Update")
	}
	return nil
}

func (s *PosSetScope) ReconcileDelete() (reconcile.Result, error) {
	s.logger.Info("ReconcileDelete")
	s.recorder.Event(s.podSet, "Normal", "Delete", "Deleting")

	if len(s.pods.Items) == 0 {
		controllerutil.RemoveFinalizer(s.podSet, PodSetFinalizer)
		return reconcile.Result{}, nil
	}

	for _, pod := range s.pods.Items {
		if pod.DeletionTimestamp != nil {
			continue
		}
		s.logger.Info("Deleting a Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
		if err := s.Delete(context.TODO(), pod.DeepCopy()); err != nil {
			if !apierr.IsNotFound(err) {
				return reconcile.Result{}, errors.Wrap(err, "r.Delete pod")
			}
		}
	}

	return reconcile.Result{}, nil
}

func (s *PosSetScope) ReconcileNormal() (reconcile.Result, error) {
	controllerutil.AddFinalizer(s.podSet, PodSetFinalizer)

	replicas := int32(len(s.pods.Items))
	s.podSet.Status.AvailableReplicas = replicas
	s.recorder.Eventf(s.podSet, "Normal", "Reconcile", "Set AvailableReplicas to %d", replicas)
	s.logger.Info("", "Update Status.AvailableReplicas to ", replicas)
	if err := s.Status().Update(context.TODO(), s.podSet); err != nil {
		s.logger.Info("", "Failed to update Status.AvailableReplicas to ", replicas)
		return reconcile.Result{}, errors.Wrap(err, "s.Status().Update")
	}

	n := int(s.podSet.Spec.Replicas - replicas)
	if n > 0 {
		for i := 0; i < n; i++ {
			// Define a new Pod object
			pod := newPodForCR(s.podSet)

			// Set PodSet instance as the owner and controller
			if err := controllerutil.SetControllerReference(s.podSet, pod, s.scheme); err != nil {
				return reconcile.Result{}, errors.Wrap(err, "controllerutil.SetControllerReference")
			}

			s.logger.Info("Creating a Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
			if err := s.Create(context.TODO(), pod); err != nil {
				return reconcile.Result{}, errors.Wrap(err, "s.Create pod")
			}
		}

		// Pod created successfully - don't requeue
		return reconcile.Result{}, nil
	}

	if n < 0 {
		for i := 0; i < -n; i++ {
			pod := s.pods.Items[i].DeepCopy()
			if pod.DeletionTimestamp != nil {
				continue
			}

			s.logger.Info("Deleting a Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
			if err := s.Delete(context.TODO(), pod); err != nil {
				if !apierr.IsNotFound(err) {
					return reconcile.Result{}, errors.Wrap(err, "s.Delete pod")
				}
			}
		}

		// Pod delete successfully - don't requeue
		return reconcile.Result{}, nil
	}

	s.recorder.Eventf(s.podSet, "Normal", "Reconcile", "Reconciled")
	// Pod already exists - don't requeue
	s.logger.Info("Skip reconcile: replicas already adjust")
	return reconcile.Result{}, nil
}

// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *v1alpha1.PodSet) *corev1.Pod {
	labels := map[string]string{
		"podset": cr.Name,
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
```

main.go
```
	...

	if err = (&controllers.PodSetReconciler{
		Client:   mgr.GetClient(),
		Log:      ctrl.Log.WithName("controllers").WithName("PodSet"),
		Scheme:   mgr.GetScheme(),
		Recorder: mgr.GetEventRecorderFor("PodSet"),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "PodSet")
		os.Exit(1)
	}

	...
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

