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
  - [Helm Consul deployment](#helm-consul-deployment)
  - [Choices and tradeoffs](#choices-and-tradeoffs)
      - [Security](#security)
      - [EKS / k8s](#eks--k8s)
      - [Helm](#helm)
      - [Network](#network)
      - [s3](#s3)
  - [Possible ways to improve the code](#possible-ways-to-improve-the-code)
      - [linting](#linting)
      - [Adding additonal env](#adding-additonal-env)
    - [tagging resources](#tagging-resources)
    - [Using terraform.tfvars](#using-terraformtfvars)
  - [Conclusion](#conclusion)
    - [minikube](#minikube-1)
    - [AWS EKS](#aws-eks-1)
  - [References](#references)

## Summary

Demonstrating using terraform-cdktf (cloud development kit for terraform) and traditional terraform to setup a kubernetes (k8s) cluster and deploy Consul Helm chart.

## Key takeaways

- Authentication - Using AWS SSO to setup rotating keys enabling security
- Using two sets of k8s environments
  - AWS EKS - setup using CDKTF <https://github.com/ggrover/gauravgrover-k8s-cdktf-demo>
  - Minikube - local environment
- Setting and configuring S3 bucket to store Terraform state
- Using Terraform to deploy Helm chart

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

- Multi-account access (scalability) - IAM users are configured only for a single AWS account. If you need to access multiple AWS accounts, like production, testing, and development, you'll need to create multiple IAM users on each account. Thus, maintainability becomes an issue. However, using SSO user eliminates this need as the user can be granted access to all accounts within an AWS organization 

Skip this section if you are using IAM user instead of SSO

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
 . ./auth.sh && sso aws-personal #https://github.com/ggrover/gauravgrover-k8s-cdktf-demo/blob/main/auth.sh

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
<https://github.com/ggrover/gauravgrover-k8s-cdktf-demo/blob/main/auth.sh>
```bash
. ./auth.sh && mk
```

### AWS EKS

Instruction to setup and deploy EKS cluster
<https://github.com/ggrover/gauravgrover-k8s-cdktf-demo/blob/main/README.md>

## Helm Consul deployment

Follow the instructions here <https://github.com/ggrover/gauravgrover-helm-tf>

## Choices and tradeoffs

#### Security

- Not enabling MFA with AWS user. Not needed for demo purposes however for production systems definitely adds an extra protection layer

- Implicitly trusting deployment of helm consul image into the cluster which is risky. The image could be compromised some kind of admission controller or run time scanner would make sense to protect the cluster

#### EKS / k8s

- Due to cost of running personal EKS cluster I've only setup a single node cluster with a `t3.medium` instance type. In a real world scenario you'll
  probably have more than one node and larger instance types
- To show cluster reliability I selected two availability zones `us-west-2a` and `us-west-2b` each with its on public subnet cidr blocks. Ideally you should have three availability zones

#### Helm

- The basic set of configuration to install helm consul was only considered to validate the install. According to the docs there are multitude of configuration options
  
#### Network

- Enabling public subnet for demo allows the cluster to reach the outside world. Ideally want to use private subnet with some ingress controller and rules
  to allow traffic to pods

#### s3

- The recommended best practice to store terraform remote state is to include dynamoDB. The use case is that if you have more than one member of the team modifying the state at any given time it can cause problems. Using DynamoDB a user you can lock the state which will prevent other users from writing to the state while it is locked. However for this exercise we will just use s3 bucket for now.

 ## Possible ways to improve the code

 #### linting

- Adding typescript and terraform linting support can improve overall structure of the code. You can also have some type of code templates
 which allows all members of the team to conform to same formatting rules

 #### Adding additonal env

- EKS cluster can easily be setup to deploy to multiple different environments such as development, staging and production. To do this you'll need to expand the config section under `gauravgrover-k8s-cdktf-demo/config/` to include configuration files such as `production.json`, `development.json` or `staging.json`. You'll also need to setup  `NODE_ENV` variable to point to these envs during deployments. One also assume you'll have multiple different AWS account to perform these actions.

 ### tagging resources

 - Adding tags to resources that were created. The benefits of tags can help you manage, identify, organize, search for, and filter resources.

### Using terraform.tfvars

- When deploying to multiple different environments using `*.tfvars` can override default values making the code more flexible 

## Conclusion

### minikube

I was able to successfully deploy helm consul to minikube. To verify the install on minikube run the following command

- Get the minikube context `. ./auth.sh && mk ` https://github.com/ggrover/gauravgrover-k8s-cdktf-demo/blob/main/auth.sh
- `helm list`
```sh                                                                                 1 ✘  minikube ⎈
NAME            	NAMESPACE	REVISION	UPDATED                            	STATUS  	CHART       	APP VERSION
helm-consul-demo	default  	1       	2023-02-05 15:20:23.59376 -0800 PST	deployed	consul-1.0.3	1.14.4
```

- Verified helm-consul-demo-consul-ui by enabling kubectl port forwarding (EG `kubectl port-forward helm-consul-demo-consul-ui 28015:27017`)
For example setup port forwarding to http://localhost:60590

![ UI ](./images/Screen%20Shot%202023-02-05%20at%2010.36.35%20PM.png)
![ UI ](./images/Screen%20Shot%202023-02-05%20at%2010.36.42%20PM.png)
![ UI ](./images/Screen%20Shot%202023-02-05%20at%2010.36.48%20PM.png)
![ UI ](./images/Screen%20Shot%202023-02-05%20at%2010.36.55%20PM.png)
![ UI ](./images/Screen%20Shot%202023-02-05%20at%2010.37.02%20PM.png)


### AWS EKS

Even though the cluster was successfully created using cdktf, installing helm consul was problematic

the end result after running `terraform apply` is an error:

```
 Warning: Helm release "helm-consul-demo" was created but has a failed status. Use the `helm` command to investigate the error, correct it, then run Terraform again.
│ 
│   with helm_release.helm-consul-demo,
│   on main.tf line 7, in resource "helm_release" "helm-consul-demo":
│    7: resource "helm_release" "helm-consul-demo" {
│ 
╵
╷
│ Error: timed out waiting for the condition
│ 
│   with helm_release.helm-consul-demo,
│   on main.tf line 7, in resource "helm_release" "helm-consul-demo":
│    7: resource "helm_release" "helm-consul-demo" {
│ 
╵
8.16s user 1.81s system 3% cpu 5:14.71s total
```
Looking at the pods  I notice the following issue with `helm-consul-demo-consul-connect-injector-7c8d94dcf4-pmbxs`
```
kubectl get pods
NAME                                                            READY   STATUS    RESTARTS      AGE
helm-consul-demo-consul-connect-injector-7c8d94dcf4-pmbxs       0/1     Running   5 (27s ago)   5m31s
helm-consul-demo-consul-server-0                                0/1     Pending   0             5m31s
helm-consul-demo-consul-webhook-cert-manager-5b54969b74-52sqr   1/1     Running   0             5m31s
```

```
helm list                                                                                                                                                                                                                                        ✔  k8s-cluster ⎈ 
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS  CHART           APP VERSION
helm-consul-demo        default         1               2023-02-05 18:54:39.731385 -0800 PST    failed  consul-1.0.3    1.14.4     

``` 
If I run `kubectl describe pods helm-consul-demo-consul-connect-injector-7c8d94dcf4-q5nj8 ` I notice cert errors when mounting volume. I don't see this issue
on minikube

## References

- CDKTF - https://developer.hashicorp.com/terraform/cdktf
- https://github.com/hashicorp/terraform-cdk
- https://www.terraform.io/
- https://dev.to/aws-builders/create-aws-infrastructure-using-cdk-for-terraform-3eg5#cdk-for-terraform-some-key-concepts
- https://github.com/pahud/cdktf-aws-eks