# Kubernetes Controller Development

Since January 2023 one of my tasks is developing Kubernetes Controllers and CRDs with Go.

This is not an introduction to the topic.

## Basics

First, you need to learnthe basics Go: [go.dev/learn](https://go.dev/learn/)

You need to learn the basics of Kubernetes: [kubernetes.io](https://kubernetes.io)

You need to learn [Kubernetes Client Go](https://github.com/kubernetes/client-go#how-to-use-it). Unfortunately only same examples exist.

Then learn [Kubebuilder](https://book.kubebuilder.io/). These docs are good.

## Things I learned

Here are some things I learned.

## status.conditions are great.

cluster-api has some nice helpers: [conditions](https://pkg.go.dev/sigs.k8s.io/cluster-api/util/conditions)



