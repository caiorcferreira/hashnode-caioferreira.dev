---
title: "The Kubernetes dynamic client"
datePublished: Fri May 28 2021 22:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clf8ylsz6079ixnnv4i2j6g1j
slug: the-kubernetes-dynamic-client
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678841031510/f867d383-5d8a-4174-b150-7844f462d4c5.png
tags: go, kubernetes, kubernetes-operators

---

---

## Introduction

Kubernetes won the battle for the cloud-native platform and the characteristic that makes me enjoy the most working with it is its extensibility. By providing an open model through the `kube-apiserver`, without splitting an internal and external interface, we can interact with the cluster and any other system to integrate both from the same application (Controller) and even use custom resources to describe our unique operations, known as the [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

Although one could use any HTTP Client to interact with the API Server, this is no simple task. There are many resources with different response structures and possible operations if we only consider the core resources on Kubernetes. Hence, Kubernetes itself provides a set of clients for easier integration through the [`k8s.io/client-go` project](https://github.com/kubernetes/client-go).

The most used client provided by this project is the `k8s.io/client-go/kubernets.ClientSet`, which is a typed client. What that means is that this interface provides exclusive methods for each resource on Kubernetes (think of Pods, Deployments, Services, everything!) and operation (Create, Get, List, Watch, Update, Patch and Delete). It is obvious why you should, whenever possible, prefer to use this client.

However, there are situations where this can be limiting. It is when `k8s.io/client-go/dynamic.Interface`, the dynamic client, will enter the game. This client has a twofold purpose:

First, it allows working with custom resources while avoiding strong dependencies. If you want to build some automation or workflow using another Operator as the building block, like ExternalDNS, CertManager, or Prometheus (Operator) usually you would need to add these projects as dependencies to use their Go types and register them on your client instance. This obviously introduces a lot of burdens as you now need to manage their ever-evolving versions and try to keep the version you have installed on the cluster matching the version on your `go.mod`.

Secondly, you can work with multiple or unknown resources. When your operator implements a generic logic that can interact with any common Kubernetes resource (from RBAC to Pods) and even custom resources, the dynamic client may be your only solution. A few examples are the garbage collection controller relies heavily on it and if you would want to add support for an arbitrary custom resource on a project like [KubeWatch](https://github.com/bitnami-labs/kubewatch).

Therefore, let’s dive into this resourceful (*pun intended*) component of the [`k8s.io/client-go`](http://k8s.io/client-go) project and see how we can leverage it.

## Basic operations with the dynamic client

> The code below assumes to be running inside a Kubernetes cluster.

Many operations with the dynamic client is similar to the typed client, like creating a new instance can be done by providing the config to its constructor:

```go
func newClient() (dynamic.Interface, error) {
	config, err := rest.InClusterConfig()
	if err != nil {
		return nil, err
	}

	dynClient, err := dynamic.NewForConfig(config)
	if err != nil {
		return nil, err
	}

	return dynClient, nil
}
```

Since the dynamic client has no knowledge about the resource you want to consume, it does not provide helper methods like `CoreV1().Pod` . Instead, you need to first provide a `schema.GroupVersionResource`, which is a Golang type that provides the necessary information to construct an HTTP request to the cluster API Server.

For example, if you want a function to list all MongoDB resources from the MongoDB Community Operator:

```go
var monboDBResource = schema.GroupVersionResource{Group: "mongodbcommunity.mongodb.com", Version: "v1", Resource: "mongodbcommunity"}

func ListMongoDB(ctx context.Context, client dynamic.Interface, namespace string) ([]unstructured.Unstructured, error) {
	// GET /apis/mongodbcommunity.mongodb.com/v1/namespaces/{namespace}/mongodbcommunity/
	list, err := client.Resource(monboDBResource).Namespace(namespace).List(ctx, metav1.ListOptions{})
	if err != nil {
		return nil, err
	}

	return list.Items, nil
}
```

Note that if you are dealing with a namespaced resource then `.Namespace(namespace)` is obligatory, even if you will use an empty string to list on all namespaces.

In this snippet, we can see the main companion of the dynamic client: `unstructured.Unstructured`. This is a special type that encapsulates an arbitrary JSON while also complying with standard Kubernetes interfaces like `runtime.Object` , but most importantly it provides a set of helpers on the `unstructured` package to manipulate this data.

Expanding our example, if we would scale a MongoDB by an proportion we could do so like:

```go
// ScaleMongoDB changes the number of members by the given proportion,
// which should be 0 =< proportion < 1.
func ScaleMongoDB(ctx context.Context, client dynamic.Interface, name string, namespace string, proportion uint) error {
	if proportion > 1 {
		return fmt.Errorf("proportion should be between 0 =< proportion < 1")
	}

	mongoDBClient := client.Resource(monboDBResource).Namespace(namespace)
	mdb, err := mongoDBClient.Get(ctx, name, metav1.GetOptions{})
	if err != nil {
		return err
	}

	members, found, err := unstructured.NestedInt64(mdb.UnstructuredContent(), "spec", "members")
	if err != nil {
		return err
	}

	if !found {
		return fmt.Errorf("members field not found on MongoDB spec")
	}

	scaled := int(members) * (1 + int(proportion))

	patch := []interface{}{
		map[string]interface{}{
			"op": "replace",
			"path": "/spec/members",
			"value": scaled,
		},
	}

	payload, err := json.Marshal(patch)
	if err != nil {
		return err
	}

	_, err = mongoDBClient.Patch(ctx, name, types.JSONPatchType, payload, metav1.PatchOptions{})
	if err != nil {
		return err
	}

	return nil
}
```

Here we leverage `unstructured.NestedInt64` to access only the field that we are interested in, keeping our coupling to the MongoDB CRD to a minimum while also being able to manipulate the resource data with type safety.

The `unstructured` package has lots of helpers like this, not only for reading but also for writing to any field on the resource.

Performing all the usual operations on Kubernetes (get, list, watch, create, patch, and delete) follow the same approach: provide the `scheme.GroupVersionResource` and handle the `unstructured.Unstructured` result.

## Controller with a dynamic client

More advanced but frequent use of a Kubernetes client is to build a controller that reacts to changes on the actual cluster state to bring it to the desired state.

Usually, we leverage an Informer, a component provided by `k8s.io/client-go`, that runs a handler when changes are detected, created from a typed client. Luckily the `dynamic` package also provides an Informer component that we can use.

For example, if we want to capture when a MongoDB is deleted to clean the associated `PersistentVolumeClaims`:

```go
package main

import (
	"fmt"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/util/workqueue"
	"time"
)

const maxRetries = 3

var monboDBResource = schema.GroupVersionResource{Group: "mongodbcommunity.mongodb.com", Version: "v1", Resource: "mongodbcommunity"}

type MongoDBController struct {
	informer cache.SharedIndexInformer
	stopper chan struct{}
	queue workqueue.RateLimitingInterface
}

func NewMongoDBController(client dynamic.Interface) (*MongoDBController, error) {
	dynInformer := dynamicinformer.NewDynamicSharedInformerFactory(client, 0)
	informer := dynInformer.ForResource(monboDBResource).Informer()
	stopper := make(chan struct{})

	queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: func(obj interface{}) {
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
	})

	return &MongoDBController{
		informer: informer,
		queue: queue,
		stopper: stopper,
	}, nil
}

func (m *MongoDBController) Stop() {
	close(m.stopper)
}

func (m *MongoDBController) Run() {
	defer utilruntime.HandleCrash()

	defer m.queue.ShutDown()

	go m.informer.Run(m.stopper)

	// wait for the caches to synchronize before starting the worker
	if !cache.WaitForCacheSync(m.stopper, m.informer.HasSynced) {
		utilruntime.HandleError(fmt.Errorf("timed out waiting for caches to sync"))
		return
	}

	// runWorker will loop until some problem happens. The wait.Until will then restart the worker after one second
	wait.Until(m.runWorker, time.Second, m.stopper)
}

func (m *MongoDBController) runWorker() {
	for {
		key, quit := m.queue.Get()
		if quit {
			return
		}

		err := m.processItem(key.(string))

		if err == nil {
			m.queue.Forget(key)
		} else if m.queue.NumRequeues(key) < maxRetries {
			m.queue.AddRateLimited(key)
		} else {
			m.queue.Forget(key)
			utilruntime.HandleError(err)
		}

		m.queue.Done(key)
	}
}

func (m *MongoDBController) processItem(mongodb string) error {
	// Clean up PVCs
	return nil
}
```

Most of this code is standard, like the work queue, informer event handlers, and item processing, of a controller using the typed client.

Hence, leveraging the decoupling provided by the dynamic client really comes with a low overhead in terms of complexity.

## Testing with the dynamic client

If we were to scale the use of the dynamic client it is paramount that it is as easy to test as the typed client.

As in the controller case, the `dynamic` package provides an equivalent fake client that allows for stubbing objects and asserting actions performed using it.

```go
package main

import (
	"context"
	"k8s.io/apimachinery/pkg/runtime"
	dynamicfake "k8s.io/client-go/dynamic/fake"
)

func TestDynamicClient(t *testing.T) {
	// Setup an Object as mock on the client
	// Write it like its YAML manifest
	mdb := &unstructured.Unstructured{}
	mdb.SetUnstructuredContent(map[string]interface{}{
		"apiVersion": "mongodbcommunity.mongodb.com/v1",
		"kind": "MongoDBCommunity",
		"metadata": map[string]interface{} {
			"name": "mongodb-test",
			"namespace": "default",
		},
		"spec": map[string]interface{}{
			"members": 3,
		},
	})

	dynamicClient := dynamicfake.NewSimpleDynamicClient(runtime.NewScheme(), mdb)

	// Run any logic that depend on the dynamic client
  NotifyMongoDBs(context.Background(), dynamicClient)

	AssertActions(t, dynamicClient.Actions(), []ExpectedAction{
		{
			Verb: "list",
			Namespace: "default",
			Resource: "mongodbcommunity",
		},
	})

}
```

Using the `unestructured.Unestructured` type we can create stub Kubernetes objects using the same syntax as in YAML, but with maps.

After performing the tested logic we can use `dynamicClient.Actions()` to see all operations that were performed by our code. However, manually asserting these actions on every test often lead to unreadable code and brittle assertions.

Hence, I often use a special assertion function `AssertActions` that verify if every expected action can be found in the performed actions. An important note is that this function does not perform an exact list match, i.e. if a delete operation was performed using the client the test would not break, the only condition for the `AssertActions` to fail is if the list operation provided on the expected list isn’t found. One could change the asserting function or make a sibling function that validates only if the expected actions were performed.

Although the current implementation is verbose, this function has the benefit of working both with the dynamic and the typed client.

```go
type ExpectedAction struct {
	Verb string
	Name string
	Namespace string
	Resource string

	// Patch action
	PatchType types.PatchType
	PatchPayload []map[string]interface{}
}

func AssertActions(t *testing.T, got []kubetesting.Action, expected []ExpectedAction) {
	if len(expected) > len(got) {
		t.Fatalf("executed actions too short, expected %d, got %d", len(expected), len(got))
		return
	}

	for i, expectedAction := range expected {
		if !AssertExpectedAction(got, expectedAction) {
			t.Fatalf("action %d does not match any of the got actions", i)
		}
	}
}

func AssertExpectedAction(got []kubetesting.Action, expectedAction ExpectedAction) bool {
	for _, gotAction := range got {
		switch expectedAction.Verb {
		case "get":
			getAction, ok := gotAction.(kubetesting.GetAction)
			if !ok {
				continue
			}

			if getAction.GetName() != expectedAction.Name {
				continue
			}

			if !validateNamespaceAndResource(getAction, expectedAction) {
				continue
			}

			return true
		case "list":
			listAction, ok := gotAction.(kubetesting.ListAction)
			if !ok {
				continue
			}

			if !validateNamespaceAndResource(listAction, expectedAction) {
				continue
			}

			return true
		case "watch":
			watchAction, ok := gotAction.(kubetesting.WatchAction)
			if !ok {
				continue
			}

			if !validateNamespaceAndResource(watchAction, expectedAction) {
				continue
			}

			return true
		case "create":
			createAction, ok := gotAction.(kubetesting.CreateAction)
			if !ok {
				continue
			}

			if !validateNamespaceAndResource(createAction, expectedAction) {
				continue
			}

			return true
		case "update":
			updateAction, ok := gotAction.(kubetesting.UpdateAction)
			if !ok {
				continue
			}

			if !validateNamespaceAndResource(updateAction, expectedAction) {
				continue
			}

			return true
		case "delete":
			deleteAction, ok := gotAction.(kubetesting.DeleteAction)
			if !ok {
				continue
			}

			if deleteAction.GetName() != expectedAction.Name {
				continue
			}

			if !validateNamespaceAndResource(deleteAction, expectedAction) {
				continue
			}

			return true
		case "patch":
			patchAction, ok := gotAction.(kubetesting.PatchAction)
			if !ok {
				continue
			}

			if patchAction.GetName() != expectedAction.Name {
				continue
			}

			if !validateNamespaceAndResource(patchAction, expectedAction) {
				continue
			}

			if patchAction.GetPatchType() != expectedAction.PatchType {
				continue
			}

			patchBytes, err := json.Marshal(expectedAction.PatchPayload)
			if err != nil {
				continue
			}

			if !bytes.Equal(patchAction.GetPatch(), patchBytes) {
				continue
			}

			return true
		}
	}

	return false
}

func validateNamespaceAndResource(action kubetesting.Action, expectedAction ExpectedAction) bool {
	return action.GetNamespace() == expectedAction.Namespace && action.GetResource().Resource == expectedAction.Resource
}
```

This asserting function allows for more conditions to be added, like verifying list/watch restrictions and create/update bodies.

## Conclusion

The Kubernetes ecosystem is rich and every now and then we stumble upon this kind of treasure. I strongly recommend reading through the documentation not only of the [`k8s.io/client-go`](http://k8s.io/client-go) but also other like [`sigs.k8s.io/controller-runtime`](https://pkg.go.dev/sigs.k8s.io/controller-runtime) project and the [Kubernetes Reference API](https://kubernetes.io/docs/reference/kubernetes-api/) documentation.