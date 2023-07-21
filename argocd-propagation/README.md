# ArgoCD Tracking-ID Propagation

This repository contains a Composition demonstrating the propagation of the ArgoCD `argocd.argoproj.io/tracking-id` annotation from a CompositeResource down to the kubernetes manifest wrapped in an `Object`.

Currently this will result in the wrapped object being tracked as a top-level resource under the Application with no sync status. Future enhancements to ArgoCD may support showing this resource as a child of the Object. See [here](https://github.com/argoproj/argo-cd/issues/8683#issuecomment-1155723396)