# Kubernetes Controller Development

Since January 2023 one of my tasks is developing Kubernetes Controllers and CRDs with Go.

## Basics

First, you need to learn the basics of Go: [go.dev/learn](https://go.dev/learn/)

You need to learn the basics of Kubernetes: [kubernetes.io](https://kubernetes.io)

Then read the [Kubebuilder docs](https://book.kubebuilder.io/)

Learn the [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)

Read [So you wanna write Kubernetes controllers?](https://ahmet.im/blog/controller-pitfalls/)
(Except "Expectations pattern". I am unsure if this is really needed).

## Maybe you do not need a Controller?

You do not like Helm, and you want to develop a better alternative?

Great! Do it. But do not write a controller!

Read and understand [RMP: Rendered Manifest
Pattern](https://akuity.io/blog/the-rendered-manifests-pattern). This blog post is from Akuity (the
company behind ArgoCD!).

A better alternative to Helm creates manifests (yaml files). Keep both (the input to your tool, and
the created yaml files) in git.

## OpenTofu/TerraForm vs Kubernetes Manifests

Kubernetes Manifests have the big advantage, that they reconcile again and again.

If you run apply your OpenTofu/TerraForm config, and some machines are not reachable at the moment,
then these machines won't get configured.

Kubernetes Manifests get reconciled over and over again by the controller.

Kubernetes Manifests have Spec (desired state) and Status (current state).

## CrossPlane vs Infra Provider CRDs

I have not used CrossPlane or Infra Provider CRDs yet.

But currently I would prefer to use the CRDs of InfraProviders (AWS Service Operator CRDs , Azure
Service Operator CRDs, Google Cloud Config Connector CRDs)

## Random Hints

Controllers are allowed to read their own Status. At the beginning I was told that this is not
allowed, but it is fine.

Use [patch.Helper](https://pkg.go.dev/sigs.k8s.io/cluster-api/util/patch). Update the resource via
`defer` in Reconcile().

Use package [condition](https://pkg.go.dev/sigs.k8s.io/cluster-api@v1.11.1/util/conditions) to
update the Conditions.

When developing a controller, you use a controller-runtime Client, not a client-go Client.

## The controller-runtime cache

When I was new to controller development, the "cache" confused me a lot.

It took me some time to understand that all reads of a controller-runtime client read from the local
cache. Writes go directly to the api-server.

This means that if you try to read your changes directly after you updated a resource, you will read
outdated data!

If you return `reconcile.Result{Requeue: true}` from Reconcile(), then it is very likely that the
second Reconcile() will not read the changes to the resource of the first Reconcile(). That can be
very confusing.

## use wait.PollUntilContextTimeout() before returning reconcile.Result{}

You can use `wait.PollUntilContextTimeout()` before returning reconcile.Result{}.

This snippets waits until the changes of your current Reconcile have arrived in your local cache.

```go
func (r *FooReconciler) Reconcile(ctx context.Context, req reconcile.Request) (res reconcile.Result, reterr error) {
 // ...

 initialFoo := foo.DeepCopy()

 defer func() {
  // update resource. For example via cluster-api package `patch.Helper`.

  if !cmp.Equal(initialFoo, foo) {
   // The foo was changed. Wait until the local cache contains the revision
   // which was created by above update of foo.
   // We want to read our own writes.
   err := wait.PollUntilContextTimeout(ctx, 100*time.Millisecond, 5*time.Second, true, func(ctx context.Context) (done bool, err error) {
    // new resource, read from local cache
    latest := &foov1.Foo{}
    getErr := r.Get(ctx, client.ObjectKeyFromObject(foo), latest)
    if apierrors.IsNotFound(getErr) {
     return true, nil
    }
    if getErr != nil {
     return false, getErr
    }
    // When the ResourceVersion has changed, then it is very likely that the local
    // cache has the new version.
    return latest.ResourceVersion != foo.ResourceVersion, nil
   })
   if err != nil {
    // Not a serious error. Logging is enough.
    log.Error(err, "cache sync failed")
   }
  }
 }()

 // here comes your Reconcile code ...
}
```

## status.conditions are great

Cluster-API has a more precise way of handling conditions. The proposal explains it: [Condition
Proposal](https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20200506-conditions.md#data-model-changes)

cluster-api has some nice helpers:
[conditions](https://pkg.go.dev/sigs.k8s.io/cluster-api/util/conditions)

I wrote a small tool to check all conditions of all resources in a cluster:
[check-conditions](https://github.com/guettli/check-conditions)

## CRDs

Custom Resource Definitions are a way to extend the API schema of Kubernetes. It is like adding new
tables to relational database.

You should know one big drawback of CRDs: CRDs are not namespaces. This means you can only install
one version of your CRD in the cluster.

You can't install version v1 in namespace "foo" and version v2 in namespace "bar".

If you just need a way to configure your application, then a ConfigMap might be enough. This has the
benefit that you can run two versions of your applications in parallel.

Related: [Should I use a ConfigMap or a custom
resource?](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#should-i-use-a-configmap-or-a-custom-resource)

## Maybe you do not need Tilt

Tilt quickly syncs your code into running pods—faster than rebuilding an image and rolling the
Deployment.

At has a web UI to show the status of the Pods and Logs.

But, configuring (language Starlark) can take long, and if I want custom Log filters, then I prefer
the unix pipe with `grep -P` and `grep -vP`.

I came to the conclusion, that I do not need Tilt.

A script like
[update-operator-dev-deployment.sh](https://github.com/syself/cluster-api-provider-hetzner/blob/main/hack/update-operator-dev-deployment.sh)
works fine for me. The update is some seconds slower than Tilt, but it is easier to reason about
than Tilt/Starlark.

## Controllers can run on your local device

It is common to run the controller you develop in a deployment/container in a Kubernetes cluster.

But remember: a controller is just a Kubernetes API client. You can run it anywhere you like. If you
created your project with Kubebuilder, you can use `make run` to run the controller without any
container.

Unfortunately, if you use web-hooks, then this will not work, except you set up a way so that the
API server can reach your webhook.

## CRDs: no foreign keys

In a relational database you can create a foreign key, and you can be shure that the foreign key
will never be broken.

I admit, that I missed that feature at the beginning.

## CRDs: Webhooks: Order of restore

Imagine you have a CRD which represents a datacenter.

And you have a second CRD which represents a server.

We created a validating web hook which ensures that for a server spec.datacenter contains a string
which has a corresponding datacenter.

The web hook works fine.

Except during a restore.

Related:
<https://velero.io/docs/main/troubleshooting/#known-issue-with-restoring-resources-when-admission-webhooks-are-enabled>

Disabling the web hook during restore is a manual task which I would like to avoid. This is not
solved yet.

## Client.Get() finds nothing?

If `controllerRuntimeClient.Get()` does not find a resource, but you can see it with kubectl, then
two things could be wrong:

* The new resource has not arrived in your local cache yet.
* RBAC
* Client Cache

To check RBAC, you can use that (if your Go code can't find a secret):

```Go
kubectl describe secret \
   --as=system:serviceaccount:your-namespace-of-the-controller:your-serviceaccount \
    -n namespace-of-secret \
    your-secret
```

You can find the name of the service-account by looking at the pod.

Client Caching: check `ctrl.NewManager()` (usualy in main.go). Maybe there is some filter which
makes the cache hide a lot of objects which would be visible otherwise.

## Use controllerutil.CreateOrUpdate()

The
[controllerutil.CreateOrUpdate()](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller/controllerutil#CreateOrUpdate)
function is very handy, if you reconcile child resources.

Example: You have CRD MyParent which creates "child" resources of type "MyChild". During the
reconcile of MyParent you can use above function to ensure that the child resource is in the desired
state. This detects changes, so it is not only "create child, if id does not existing yet".

## Use controllerutil.SetControllerReference()

If you controller creates sub-resources, then set the controller reference, so that Garbage
Collection will delete this resource, when the parent resource gets deleted.

[controllerutil.SetControllerReference()](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller/controllerutil#SetControllerReference)

## Use GenerationChangedPredicate

Your FooReconciler will most likely update the Status of foo resources.

You do not want that Reconcile gets called by your own changes to the status.

To ignore these changes you can use a predicate like this:

```go
Controller.Watch(
 &source.Kind{Type: v1.FooKind},
 &handler.EnqueueRequestForObject{},
 predicate.Or(predicate.GenerationChangedPredicate{},
        predicate.AnnotationChangedPredicate{}),
        predicate.LabelChangedPredicate{}),
        )
```

Docs:
[controller-runtime/pkg/predicate](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/predicate)

## Do not fiddle with SyncPeriod

Imagine your controller has reconciled a resource, so that the desired state is equal to the desired
state.

Then you usualy return `reconcile.Result{}, nil`.

This means you Reconcile method will not be called again.

Is that what you want?

Maybe you want do check if there a drift every hour?

Then return `reconcile.Result{RequeueAfter: 1 * time.Hour}, nil`.

Do not change [cache.Options
SyncPeriod](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache#Options).

If you use `GenerationChangedPredicate`, then SyncPeriod will do nothing.

## The Kube API Linter

[kubernetes-sigs/kube-api-linter: KAL - The Kube API
Linter](https://github.com/kubernetes-sigs/kube-api-linter)

Installing the this linter is not straight forward, so I have not used it much yet.

## Apply is easy, prune is hard.

When a controller creates resources, it is usually via structured resources, and the controller uses `SetControllerReference`.

But sometimes you want to apply some "random" yaml, just like `kubectl` does. You should know:

> Apply is easy, prune is hard.

Imagine you apply 5 yaml manifests today. Updating these 5 manifests tomorrow is easy.

But imagine some weeks later, your code receives only 4 yaml manifests, because one manifest is no longer needed.

How to delete this obsolete manifest?

There are two solutions:

* Option 1: Label all objects you create, so that you can find all objects you applied in the past
* Option 2: Keep some kind of inventory. For example in a configMap.

Option2 is available via [kubernetes-sigs/cli-utils](https://github.com/kubernetes-sigs/cli-utils)

## Tools

Tools I use for developing with Golang and Kubernetes

* vscode
* [check-conditions](https://github.com/guettli/check-conditions) my tool the check all conditions
  of all resources in a cluster
* [dumpall](https://github.com/guettli/dumpall) Dump all Kubernetes resources into a directory
  structure
* [watchall](https://github.com/guettli/watchall) Watch all Kubernetes Resources
* [Bash Strict Mode](https://github.com/guettli/bash-strict-mode)

## Related

* [Thomas WOL: Working out Loud](https://github.com/guettli/wol)

## Feedback

I love feedback and I love to hear from you. Just create an issue and tell me what's on your mind.

## CRDs: ownerRef will be lost upon backup+restore

<https://github.com/vmware-tanzu/velero/issues/4707>
