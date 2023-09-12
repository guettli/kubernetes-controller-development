# Kubernetes Controller Development

Since January 2023 one of my tasks is developing Kubernetes Controllers and CRDs with Go.

This is not an introduction to the topic.

## Basics

First, you need to learnthe basics Go: [go.dev/learn](https://go.dev/learn/)

You need to learn the basics of Kubernetes: [kubernetes.io](https://kubernetes.io)

You need to learn [Kubernetes Client Go](https://github.com/kubernetes/client-go#how-to-use-it). Unfortunately only same examples exist.


Learn the [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)


## Things I learned

Here are some things I learned.

## status.conditions are great.

Cluster-API has a more precise way of handling conditions. The proposal explains it: [Condition Proposal](https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20200506-conditions.md#data-model-changes)

cluster-api has some nice helpers: [conditions](https://pkg.go.dev/sigs.k8s.io/cluster-api/util/conditions)

I wrote a small tool to check all conditions of all resources in a cluster: [check-conditions](https://github.com/guettli/check-conditions)

## CRDs

Custom Resource Definitions are a way to extend the API schema of Kubernetes. It is like adding new tables to relational database.

You should know one big drawback of CRDs: CRDs are not namespaces. This means you can only install one version of your CRD in the cluster.

You can't install version v1 in namespace "foo" and version v2 in namespace "bar".

If you just need a way to configure your application, then a ConfigMap might be enough. This has the benefit that you can run two versions
of your applications in parallel.

Related: [Should I use a ConfigMap or a custom resource?](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#should-i-use-a-configmap-or-a-custom-resource)

If you want to create a CRD, then read the newbie friendly docs of [Kubebuilder](https://book.kubebuilder.io/)

## CRDs: `make run`

If you write a CRD and your own controller, then it is likely that you will use Tilt to run your controller in a development cluster (in most cases a kind cluster on your device).

But remember: a controller is just a Kubernetes API client. You can run it anywhere you like. If you created your project with Kubebuilder, you can use `make run` to 
run the controller without any container. This makes it easy for debugging with `delve`.

Unfortunately, if you use web-hooks, then this will not work, except you set up a way so that the API server can reach your webhook.


## CRDs: status will get lost upon backup+restore

Guideline: A controller does not read its own status.

After a backup+restore the status will be empty. The controller need to check the actual state and then create a matching status.

## CRDs: ownerRef will be lost upon backup+restore

https://github.com/vmware-tanzu/velero/issues/4707



