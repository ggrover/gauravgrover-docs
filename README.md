# Terraform and Kubernetes 

## Summary

Demonstrating using terraform-cdktf (cloud development kit for terraform) and traditional terraform to setup a kubernetes (k8s) cluster and deploy Consul Helm chart.

## Key takesways

- Authentication - Using AWS SSO to setup rotating keys to enabling security
- Using two sets of k8s environments
  - AWS EKS - setup using CDKTF (link to repo)
  - Minikube - local environment
- Setting and configuring S3 bucket to store Terraform state
- Using Terraform to deploy Helm charts

## Prerequisite

- AWS Account - Needed to satisfy storing the terraform remote state. Also needed if you are looking to use EKS resource rather than minikube

### AWS Setup and configuration 

#### AWS Client Setup

Follow the documentation set out here https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html

#### Authentication 
(Configuring the AWS CLI to use AWS IAM Identity Center (successor to AWS Single Sign-On))

There are two approaches that can be taken for authenticating with AWS. 

- Adding IAM user
- Using Single Sign On (SSO)

Keeping security in mind the approach that I have taken is to use SSO. The reason behind this are as follows:

- Short-lived credentials - IAM users requires persisting the same access key and secret on your machine, which is normally stored in the following file `~/.aws/credentials file`. Compared with SSO, which issues temporary credentials at auth-time with an expiry configurable between 1 and 12 hours. If the keys are ever compromised the risk relatively low.

- Multi-account access (scalability) - IAM users are configured only for a single AWS account. If you need to access multiple AWS accounts, like production, testing, and development, you'll need to create multiple IAM users on each account. Thus maintainability becomes an issue. However, using SSO user eliminates this need as the user can be granted access to all accounts within an AWS organization 

Skip this section if you are user IAM user instead of SSO

IAM User setup: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html

##### SSO

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

#### AWS S3 Bucket setup