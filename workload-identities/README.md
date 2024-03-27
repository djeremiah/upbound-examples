# Workload Identity Configuration in Upbound self-hosted Spaces

Desired experience:
0. Install a configuration in the space corresponding to your use case
0. Deploy WorkloadIdentityAssociation for each controlplane 

Inputs:
- ControlPlane
- Provider
-


## Space running in AWS
### Access AWS
0. Set up IRSA role(s)
0. Configure DeploymentRuntimeConfig
0. Configure ProviderConfig

## Space running in Azure
### Access AWS

### Access Azure
0. Set up UserManagedIdentity
0. Set up FederatedIdentityCredential
0. Configure DeploymentRuntimeConfig
0. Configure ProviderConfig

## Space running in GCP
