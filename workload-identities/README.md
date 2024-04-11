# Workload Identity Configuration in Upbound

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