# Application Workspace Provisioning

This example demonstrates using Upbound to provision workspaces into a workload cluster and deploy applications into it. Example deployments can be found in the [`.up/examples`](.up/examples/) folder.

## Available APIs

This repository implements Compositions for EKS Clusters, Application Deployments, and Team Setups.

- `Cluster.aws.caas.upbound.io`
    - Provision/Manage an EKS Cluster

- `App.example.upbound.io`
    - Provision a Namespace into the target Cluster and register an ArgoCD Application targeting that Namespace

- `Team.example.upbound.io`
    - WIP; Provision a new Group within GitLab and assign LDAP groups to the key access roles


The [`apis`](apis/)
folder has the CompositeResourceDefinitons (XRDs) that define the schemas for
the APIs.

Once you clone the repository, you can modify the included compositions to fit your organizations needs.
