# Serverless Applications with AWS and Upbound

This repository demonstrates the use of Upbound to deploy a control plane for the deployment and management of serverless applications on AWS Lambda and surrounding services.


## Available APIs
### For the Platform Team
#### XAccountSetup
This API is for the Platform Team to configure the control plane for use by Application Teams. 

Assuming the following environment:
* Multi-tenant control plane running in EKS, with each application team/environment/management unit receiving a namespace 
* Each namespace maps to one AWS account
* A role for managing AWS resources already exists, and can be assumed by the configured identity of the control plane provider
* A common set of tags should be applied to every AWS resource created in this namespace

You can quickly provision the necessary EnvironmentConfig and ProviderConfig for this tenant with the XAccountSetup API
```yaml
apiVersion: aws.example.com/v1alpha1
kind: XAccountSetup
metadata:
  name: project-a
spec:
  parameters:
    namespace: eu-xxxx
    region: eu-west-1
    roleARN: arn:aws:iam::${accountid}:role/eu-projectrole
    tags:
      owner: Murph
      projectCode: xxxx
```

The Developer APIs listed below rely on the existence of the appropriate EnvironmentConfig.


### For the Developers
#### ServerlessApp
This API is the main entrypoint for Development Teams seeking to deploy Lambda Functions with optional integration into other services.

It is composed of finer-grained elements

#### Function
Assuming the following environment:
* Function code is built and delivered into an S3 bucket accessible to the app team account

A Function can be deployed, along with a default execution role and policy:
```yaml
apiVersion: lambda.aws.example.com/v1alpha1
kind: Function
metadata:
  name: lambda
spec:
  parameters:
    name: lambda-test
    runtime: java11
    handler: example.Handler::handleRequest
    architecture: "arm64"
    package:
      type: Zip
      zip:
        s3bucket: lambda-bucket
        s3key: lambda-0.1.1.zip
```


#### Release
Versioning and concurrency management of Functions is managed by the Release API, mapping published versions to a defined alias, and optionally assigning provisioned concurrency:
```yaml
apiVersion: lambda.aws.example.com/v1alpha1
kind: Release
metadata:
  name: lambda-release
spec:
  parameters:
    name: release
    function: lambda-test
    version: "2"
    concurrency: 2
```


## Deploying the example
The apis can be deployed by running `kubectl apply -f apis/accountsetup`, `kubectl apply -f apis/function`, `kubectl apply -f apis/release`
Individual sample files can be found in the `.up/examples/` folder

## Tracking work in progress
This example is still under development. A high-level set of goals can be seen here. Subject to change.

:black_circle: Done
:white_circle: Todo
:red_circle: Blocked

Phase 1
Function
- :black_circle: Execution Role
- :black_circle: Default Execution Policy
- :black_circle: Refactor image build to zip
Alias
- :red_circle: Version Mapping <- there seems to be a bug in the id function for this resource
- :black_circle: Provisioned Concurrency
ServerlessApp
- Function with Version
- Publish new Version


Phase 2
Event Source Mappings
- :white_circle: DynamoDB
- :white_circle: MSK
- :white_circle: Kinesis
- :white_circle: SNS direct
- :white_circle: SQS
- :white_circle: SNS over SQS
:white_circle: Conditional Configuration of source in ServerlessApp
:black_circle: Namespaced Environment in Compositions

Phase 3 
:white_circle: Provisioning of Sources as part of Serverless App

