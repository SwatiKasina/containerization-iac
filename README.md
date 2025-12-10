# EKS Containerization IaC Pipeline

This repository contains Infrastructure as Code (IaC) templates and a dynamic CI/CD pipeline to deploy AWS VPC and EKS Clusters across Dev and QA environments.

## Project Overview

The solution uses **AWS CloudFormation**, **CodePipeline**, and **CodeBuild** to automate infrastructure deployment. It supports a multi-cluster strategy where each cluster's configuration is isolated in its own directory, but deployed using a single, reusable `pipeline.yaml` template.

### Key Features
- **Dynamic Pipeline**: The pipeline adapts to the `ClusterName` parameter.
- **Environment Isolation**: Separate deployments for Dev and QA with independent parameter files.
- **Manual Promotion**: Automated deployment to Dev, followed by a manual approval step before deploying to QA.
- **Artifact Versioning**: Builds are tagged with the specific Commit ID.

## Repository Structure

The project follows a directory-per-cluster pattern:

```text
.
├── pipeline.yaml                  # The reusable master pipeline template
├── README.md                      # This documentation
└── <ClusterName>/                 # e.g., my-eks-project/
    ├── cfn/                       # CloudFormation templates for this cluster
    │   ├── vpc.yaml               # VPC definition
    │   └── eks-cluster.yaml       # EKS Cluster definition
    └── params/                    # Environment-specific parameters
        ├── dev.json               # Parameters for Dev environment
        └── qa.json                # Parameters for QA environment
```

## Prerequisites

Before deploying the pipeline, ensure you have the following AWS resources ready:

1.  **CodeStar Connection**: A connection to your GitHub repository.
2.  **S3 Bucket**: A bucket to store pipeline artifacts.
3.  **IAM Roles**:
    *   `PipelineServiceRole`: For CodePipeline.
    *   `CodeBuildServiceRole`: For CodeBuild projects.
    *   `CloudFormationServiceRole`: For CloudFormation to create resources (VPC, EKS, etc.).

## How to Onboard a New Cluster

To deploy a new cluster (e.g., named `analytics-cluster`):

1.  **Create Directory**: Create a folder named `analytics-cluster` in the root of the repo.
2.  **Add Templates**: Copy `vpc.yaml` and `eks-cluster.yaml` into `analytics-cluster/cfn/`.
3.  **Add Parameters**: Create `analytics-cluster/params/dev.json` and `analytics-cluster/params/qa.json` with appropriate values (e.g., CIDR blocks).
4.  **Commit**: Push the changes to the `main` branch.
5.  **Deploy Pipeline**: Launch the `pipeline.yaml` stack (see below).

## Deployment Instructions

Run the following command to deploy the pipeline for your cluster. Replace the placeholder values with your specific configuration.

```bash
aws cloudformation deploy \
  --template-file pipeline.yaml \
  --stack-name my-eks-project-pipeline \
  --parameter-overrides \
    ClusterName="my-eks-project" \
    GitHubOwner="your-github-username" \
    GitHubRepo="containerization-iac" \
    GitHubBranch="main" \
    CodeStarConnectionArn="arn:aws:codestar-connections:us-east-1:123456789012:connection/example-uuid" \
    ArtifactBucketName="your-artifact-bucket-name" \
    PipelineServiceRoleArn="arn:aws:iam::123456789012:role/service-role/AWSCodePipelineServiceRole" \
    CodeBuildServiceRoleArn="arn:aws:iam::123456789012:role/service-role/codebuild-service-role" \
    CloudFormationServiceRoleArn="arn:aws:iam::123456789012:role/CloudFormationDeployRole" \
  --capabilities CAPABILITY_NAMED_IAM
```

## Pipeline Workflow

1.  **Source**: Detects changes in the GitHub repository.
2.  **Build**:
    *   Packages the templates and configuration.
    *   Tags the artifact with the Git Commit ID.
3.  **Deploy Dev**:
    *   Deploys/Updates the VPC stack using `params/dev.json`.
    *   Deploys/Updates the EKS Cluster stack using `params/dev.json`.
4.  **Approval**:
    *   Pipeline pauses for manual review.
    *   Approve in the AWS Console to proceed.
5.  **Deploy QA**:
    *   Uses the **same build artifact** from step 2.
    *   Deploys/Updates the VPC stack using `params/qa.json`.
    *   Deploys/Updates the EKS Cluster stack using `params/qa.json`.

## SSM Parameters & Outputs

The VPC stack automatically exports critical values to SSM Parameter Store for the EKS stack to consume:

*   `/<ClusterName>/<EnvironmentName>/vpc/id`
*   `/<ClusterName>/<EnvironmentName>/vpc/public-subnet`
*   `/<ClusterName>/<EnvironmentName>/vpc/private-subnet`
