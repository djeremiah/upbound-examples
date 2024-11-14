# Workload Identity Configuration in Upbound

- [AWS](#aws)
- [Azure](#azure)

## AWS
This setup will use the same IAM role for all control planes.

### Pre-requisites
1. Create an IAM Role with the appropriate permissions for managing the desired resources
1. Assign a trust policy to the Role similar to the following
    ```yaml
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "<OIDC Provider ARN>"
                },
                "Action": "sts:AssumeRoleWithWebIdentity",
                "Condition": {
                    "StringEquals": {
                        "<OIDC Provider URL>:aud": "sts.amazonaws.com",
                        "<OIDC Provider URL>:sub": "system:serviceaccount:*:provider-aws"
                    }
                }
            }
        ]
    }
    ```
1. Copy the ARN of the new IAM Role, we will use it later

### Configure Control Planes to use this role
1. Create a `ControlPlane`
1. Connect to the ControlPlane using `up ctp connect` or equivalent
1. Create a Service Account with the correct IRSA annotations
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
        annotations:
            eks.amazonaws.com/role-arn: <ARN Copied Above>
        name: provider-aws
    ```
1. Configure a `DeploymentRuntimeConfig` to use the new service account
    ```yaml
    apiVersion: pkg.crossplane.io/v1beta1
    kind: DeploymentRuntimeConfig
    metadata:
        name: static-service-account-name
    spec:
        deploymentTemplate:
            spec:
                selector: {}
                template:
                    spec:
                        serviceAccountName: provider-aws
                        containers: 
                            - name: package-runtime
    ```
1. Deploy a `Provider`, using the runtime config
    ```yaml
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
        name: provider-aws-s3
    spec:
        package: xpkg.upbound.io/upbound/provider-aws-s3:<version>
        runtimeConfigRef: 
            name: static-service-account-name
    ```
1. Create a `ProviderConfig` to specify IRSA as the credential type
    ```yaml
    apiVersion: aws.upbound.io/v1beta1
    kind: ProviderConfig
    metadata:
        name: default
    spec:
        credentials:
            source: IRSA
    ```

## Azure
### Prerequisites - Set up the Host Cluster
1. Ensure the AKS cluster is configured to enable workload identity

1. Create a User Assigned Managed Identity that will have permissions in the Subscription
```sh
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --subscription "${SUBSCRIPTION}"
export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' --output tsv)"
export USER_ASSIGNED_TENANT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'tenantId' --output tsv)"
```
1. Retrieve the OIDC Issuer URL for the AKS host cluster
```sh
export AKS_OIDC_ISSUER="$(az aks show --name "${CLUSTER_NAME}" --resource-group "${RESOURCE_GROUP}" --query "oidcIssuerProfile.issuerUrl" --output tsv)"
```

### Set up the Upbound software
1. Configure your space to propagate pod labels to the hostcluster
values.yaml
```yaml
controlPlanes:
  syncer:
    extraSyncLabels: azure.workload.identity/use
```

1. Create a control plane that will host the providers
```sh
export GROUP_NAME=default
export CONTROLPLANE_NAME=azwi
up ctp create -g ${GROUP_NAME} ${CONTROLPLANE_NAME}
```

1. Create a service account in the control plane that we will federate to a User Assigned Managed Identity
```sh
up ctx ./${GROUP_NAME}/${CONTROLPLANE_NAME}
kubectl create serviceaccount provider-azure -n crossplane-system 
```

1. Create the Federated Identity Credential for the service account
```sh
up ctx ../../
export CONTROLPLANE_ID=$(up ctp get -g ${GROUP_NAME} ${CONTROLPLANE_NAME} --format json | jq -r '.metadata.annotations["internal.spaces.upbound.io/control-plane-id"]')
export SERVICE_ACCOUNT_NAMESPACE=mxp-${CONTROLPLANE_ID}-system
export SERVICE_ACCOUNT_NAME=provider-azure-x-crossplane-system-x-vcluster

az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --issuer "${AKS_OIDC_ISSUER}" --subject system:serviceaccount:"${SERVICE_ACCOUNT_NAMESPACE}":"${SERVICE_ACCOUNT_NAME}" --audience api://AzureADTokenExchange
```

1. Create a DeploymentRuntimeConfig to reference the service account an annotate the provider pod
```sh
up ctx ./${GROUP_NAME}/${CONTROLPLANE_NAME}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: azwi
spec:
  deploymentTemplate:
    spec:
      selector: {}
      template:
        metadata:
          labels:
            azure.workload.identity/use: "true"
        spec:
          serviceAccountName: provider-azure
          containers: 
            - name: package-runtime
EOF
```

1. Deploy a provider using the runtime config
```sh
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-family-azure
spec:
  package: xpkg.upbound.io/upbound/provider-family-azure:v1.8.0
  runtimeConfigRef:
    name: azwi
EOF
```

1. Deploy a providerconfig using Workload Identity
```sh
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: OIDCTokenFile
  clientID: ${USER_ASSIGNED_CLIENT_ID}
  subscriptionID: ${SUBSCRIPTION_ID}
  tenantID: ${USER_ASSIGNED_TENANT_ID}
EOF
```

1. Start using the provider to deploy resources
```sh
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
kind: ResourceGroup
metadata:
  name: test-rg
spec:
  forProvider:
    location: West Europe
EOF
```
