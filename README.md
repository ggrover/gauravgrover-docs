# Terraform and Kubernetes

- [Terraform and Kubernetes](#terraform-and-kubernetes)
  - [Summary](#summary)
  - [Key takeaways](#key-takeaways)
  - [Prerequisite](#prerequisite)
    - [Technologies demonstrated](#technologies-demonstrated)
      - [CDK for Terraform (CDKTF)](#cdk-for-terraform-cdktf)
      - [Terraform](#terraform)
  - [Github setup for public repo](#github-setup-for-public-repo)
  - [AWS Setup and configuration](#aws-setup-and-configuration)
    - [AWS Client Setup](#aws-client-setup)
    - [Authentication](#authentication)
      - [SSO](#sso)
    - [AWS S3 Bucket setup](#aws-s3-bucket-setup)
  - [Kubernetes](#kubernetes)
    - [Minikube](#minikube)
      - [Setup](#setup)
      - [Set context](#set-context)
    - [AWS EKS](#aws-eks)
  - [Choices and tradeoffs](#choices-and-tradeoffs)
      - [Security](#security)
      - [EKS / k8s](#eks--k8s)
      - [Helm](#helm)
      - [Network](#network)
      - [Reliablity](#reliablity)
      - [s3](#s3)
  - [Possible ways to improve the code](#possible-ways-to-improve-the-code)
      - [linting](#linting)
      - [Adding additonal env](#adding-additonal-env)
    - [tagging resources](#tagging-resources)
  - [References](#references)

## Summary

Demonstrating using terraform-cdktf (cloud development kit for terraform) and traditional terraform to setup a kubernetes (k8s) cluster and deploy Consul Helm chart.

## Key takeaways

- Authentication - Using AWS SSO to setup rotating keys to enabling security
- Using two sets of k8s environments
  - AWS EKS - setup using CDKTF (link to repo)
  - Minikube - local environment
- Setting and configuring S3 bucket to store Terraform state
- Using Terraform to deploy Helm charts

## Prerequisite

- AWS Account - Needed to satisfy storing the terraform remote state. Also needed if you are looking to use EKS resource rather than minikube

### Technologies demonstrated

#### CDK for Terraform (CDKTF)
  
   The Cloud Development Kit for Terraform (CDKTF) generates JSON Terraform configuration from code in C#, Python, TypeScript, Java, or Go, and creates infrastructure using Terraform. With CDKTF, you can use hundreds of providers and thousands of module definitions provided by HashiCorp and the Terraform community. For more information take a look at <https://developer.hashicorp.com/terraform/cdktf>
  
#### Terraform

   A declarative approach, describing resources using HashiCorp Configuration Language (HCL), allowing to be machine and human readable. For more  information take a look at <https://www.terraform.io/>

## Github setup for public repo

The following repos were created for the task:

- The documentation outlining installation and configuration as well as improvements - <https://github.com/ggrover/gauravgrover-docs>
- Source code to install helm consul using Terraform - <https://github.com/ggrover/gauravgrover-helm-tf>
- Source code to install AWS EKS using CDKTF - <https://github.com/ggrover/gauravgrover-k8s-cdktf-demo>

For all the git repos used for the exercise the following security features have been enabled. This is to ensure that security is at the forefront before diving into the task at hand.

![ S3 Bucket ](./images/Screen%20Shot%202023-02-05%20at%209.12.31%20AM.png)

## AWS Setup and configuration

### AWS Client Setup

Follow the documentation set out here https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html

### Authentication

(Configuring the AWS CLI to use AWS IAM Identity Center (successor to AWS Single Sign-On))

There are two approaches that can be taken for authenticating with AWS. 

- Adding IAM user
- Using Single Sign On (SSO)

Keeping security in mind the approach that I have taken is to use SSO. The reason behind this are as follows:

- Short-lived credentials - IAM users requires persisting the same access key and secret on your machine, which is normally stored in the following file `~/.aws/credentials file`. Compared with SSO, which issues temporary credentials at auth-time with an expiry configurable between 1 and 12 hours. If the keys are ever compromised the risk relatively low.

- Multi-account access (scalability) - IAM users are configured only for a single AWS account. If you need to access multiple AWS accounts, like production, testing, and development, you'll need to create multiple IAM users on each account. Thus maintainability becomes an issue. However, using SSO user eliminates this need as the user can be granted access to all accounts within an AWS organization 

Skip this section if you are user IAM user instead of SSO

IAM User setup: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html

#### SSO

To setup SSO take a look at the following documentation: https://docs.aws.amazon.com/singlesignon/latest/userguide/getting-started.html

Once SSO has been setup copy and paste the following named aws profile into `.aws/config`. I've added comments to explain section of the config file

```
[profile aws-personal] // Your profile name that you setup
sso_start_url = https://d-92675590a2.awsapps.com/start  // This is your company unique url for AWS organization (found on the AWS SSO dashboard)
sso_region = us-west-1 // The region where SSO organization has been setup
sso_account_id = 386250199463 // organization acc number
sso_role_name = AWSPowerUserAccess // Role name could be Admin as well
region = us-west-2
```

To test AWS account setup run the following command
- `aws sso login --profile aws-personal` (`aws-personal` is the name profile defined in your `./aws/config`) 

This will take you to the aws website to authenticate

Once authentication has been setup you can use the bash script to automate the process. To run the bash script 
run the following cmd

```bash
 # This execute the bash script and calls sso function in the script with the aws profile as an argument
 . ./auth.sh && sso aws-personal

```

### AWS S3 Bucket setup

- Go to AWS console and find the S3 bucket
![ S3 Bucket ](./images/Screen%20Shot%202023-02-04%20at%208.32.34%20PM.png)
- Next, click create bucket and go through the wizard
![ S3 Bucket ](./images/Screen%20Shot%202023-02-04%20at%208.35.36%20PM.png)
![ S3 Bucket ](./images/Screen%20Shot%202023-02-04%20at%208.36.59%20PM.png)
- Enable encryption for the bucket
- After creation of the bucket, the aws s3 policy for the bucket needs to be set
- Select the new bucket and navigate to the permission tab
![ S3 Bucket ](./images/Screen%20Shot%202023-02-04%20at%208.41.20%20PM.png)
- Add the following to the bucket policy making sure that you replace `<user_arn>` and `<bucket_arn>` with your values.
To get the `user_arn` you can run the following cmd `aws sts get-caller-identity --profile <this is set in you ./aws/config file>`
The bucket arn can be found in the aws console
![ S3 Bucket ](./images/Screen%20Shot%202023-02-04%20at%208.49.19%20PM.png)

```script
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "<user_arn>"
            },
            "Action": "s3:ListBucket",
            "Resource": "<bucket_arn>"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "<user_arn>"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "<bucket_arn>"
        }
    ]
}
```

- Save the s3 configuration
- As an enhancement you can also use DynamoDB. The use case is that if you have more than one member of the team modifying the state 
at any given time it can cause problems. Using DynamoDB a user you can lock the state which will prevent other user from writing to the state while it is locked. However for this exercise we will just use s3 bucket for now.

## Kubernetes

For the purpose of this exercise I will show two different kubernetes options taken, one locally and the other in the cloud. This includes

- Minikube
- AWS EKS

### Minikube

#### Setup

To setup minikube env go through the following documentation https://minikube.sigs.k8s.io/docs/start/

#### Set context

Run the following command

```bash
. ./auth.sh && mk
```

### AWS EKS


## Choices and tradeoffs 

#### Security
#### EKS / k8s
#### Helm
#### Network
#### Reliablity 
- EKS cluster uses public subnet thus allowing access from the outside world to the cluster. Easier to setup and get up and running Improvements would be to use private subnet to prevent access to the cluster from the outside world
- Allow users to use MFA with AWS to be authenticated adding extra layer of protection  . For production system definitly wnt to secure it don
#### s3

 ## Possible ways to improve the code

 #### linting
 #### Adding additonal env
 ### tagging resources

## References

- CDKTF - https://developer.hashicorp.com/terraform/cdktf
- https://github.com/hashicorp/terraform-cdk
- https://www.terraform.io/
