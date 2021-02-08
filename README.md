# Automation in Management of Networking Resources using AWS services
This repository contains code to create a Ci/CD pipeline for automating the deployment of networking resources such as Security Groups, Route Tables, Transit gateways etc 



# Introduction

In today’s modern era, organizations are moving from traditional monolithic data center networks into an agile api driven cloud networks. Customers are looking for an efficient and reliable way to make changes to their cloud network infrastructure. The desire is to be able to perform network changes in cloud using pipeline driven automation by following devops best practices (https://aws.amazon.com/devops/what-is-devops/) and with infrastructure as code. 

For example, you want to make a change to your security groups in a multi-account setup spanning across different vpc’s and or regions. You want to implement these changes using CI/CD pipeline with infrastructure as code in a seamless, fast and efficient manner. To perform this operation today, you need to setup a CI/CD pipeline and integrate your code repository to carry out a specific task. This is not straightforward task and requires a lot of engineering cycles.

In this post, we show how you can set up a CI/CD (https://en.wikipedia.org/wiki/CI/CD) pipeline across a multi account setup and carry out common networking tasks in AWS cloud. We leverage AWS CloudFormation (https://aws.amazon.com/cloudformation/) to setup the infrastructure that is required and use IAM roles (https://aws.amazon.com/iam/) to securely access resources across multiple AWS accounts and use cloudformation (https://aws.amazon.com/cloudformation/)to deploy the required network infrastructure changes. We show how you can update your network infrastructure on an ongoing basis using AWS native services that includes AWS Code Commit (https://aws.amazon.com/codecommit/) and AWS Code Pipeline (https://aws.amazon.com/codepipeline/). We exhibit how you can perform static and security analysis using tools like cfn-lint (https://github.com/aws-cloudformation/cfn-python-lint) and cfn-nag (https://github.com/stelligent/cfn_nag).  We finally show an example of how to deploy the required security group changes across multiple accounts that exists under organization. This process of deployment helps catch issues early, which subsequently saves time for deployment.


# Architecture 

Let’s get started by understanding the setup and functionality of the individual components deployed during the setup. At a bare minimum, we need two AWS accounts; 1) A Shared Services Account and, 2) any other AWS customer account.

The CI/CD platform is built using in Shared Services Account using the following components :


* AWS CodeCommit : A fully-managed source control (https://aws.amazon.com/devops/source-control/) service that hosts secure Git-based repositories. CodeCommit makes it easy for teams to collaborate on code in a secure and highly scalable ecosystem. Our solution uses CodeCommit to create a repository to store the deployment codes.
* AWS CodeBuild : A fully managed continuous integration service that compiles source code, runs tests, and produces software packages that are ready to deploy, on a dynamically created build server. Our solution uses CodeBuild to build and test the code, which we deploy later.
* AWS CodePipeline : A fully managed continuous delivery (https://aws.amazon.com/devops/continuous-delivery/) service that helps you automate your release pipelines for fast and reliable application and infrastructure updates. Our solution uses CodePipeline to create an end-to-end pipeline that fetches the application code from CodeCommit, builds and tests using CodeBuild, and finally deploys using AWS CloudFormation (https://aws.amazon.com/cloudformation/).
* Identity Access Management : An IAM identity that you can create in your account that has specific permissions. Our solution leverages  Security Token Service (STS) with Assume Roles (https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) to access resources across cross account boundaries. 
* Parameter Store : Parameter Store (https://aws.amazon.com/ec2/systems-manager/parameter-store/) is part of Amazon EC2 Systems Manager (https://aws.amazon.com/ec2/systems-manager/parameter-store/). It provides a centralized, encrypted store to manage your configuration data, whether it is plain text data (to store resource Id strings) or secure strings and secrets (such as passwords, and API keys). Our solution uses parameter store to store cross account resource IDs that later require to be managed via pipeline.
* AWS Organizations :  Our solution leverages AWS Organizations (https://docs.aws.amazon.com/controltower/latest/userguide/organizations.html) which is an account management service that lets you consolidate multiple AWS accounts into an organization that you create and centrally manage. You can organize those accounts into groups and attach policy-based controls. Our solution uses AWS Organization to manage multiple AWS accounts. 
* Simple Notification Service : Our solution leverages Amazon (https://aws.amazon.com/sns/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc)Simple Notification Service (https://aws.amazon.com/sns/) (Amazon SNS) which is a managed service that provides message delivery from publishers to subscribers (also known as producers and consumers). Our solution uses SNS to publish updates for all infrastructure networking changes that are deployed via the code pipeline.


The changes via code pipeline are deployed in customer accounts for the following networking resources :

* Security Groups : A Security Group (https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html#VPCSecurityGroups) is a stateful virtual firewall for your EC2 instance to control inbound and outbound traffic. Security groups act at the instance level, not the subnet level. Therefore, each instance in a subnet in your VPC can be assigned to a different set of security group. The solution will demonstrate how to create security groups and manage the security group rules in a multi-account setup using CI/CD framework.
* Network Access Control Lists : A Network Access Control List (https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html#nacl-basics) is an additional layer of security for your VPC that acts as a stateless firewall for controlling traffic in and out of one or more subnets. The solution will demonstrate how to create network access control lists and manage the ACL rules in a multi-account setup using CI/CD framework.
* VPC Route tables : A VPC has an implicit router, and you use route tables (https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#RouteTables) to control where network traffic is directed. It contains a set of rules, called routes, that are used to determine where network traffic from your subnet or gateway is directed. The solution will demonstrate how to create route tables and manage the routes in a multi-account setup using CI/CD framework.
* Transit gateway : A Transit gateway (https://aws.amazon.com/transit-gateway/) is a network transit hub that you can use to interconnect your virtual private clouds (VPC) and on-premises networks. The solution will demonstrate how to create transit gateway and manage its routing domains using CI/CD framework.
* Private hosted zone : A private hosted zone (https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html) is a container that holds private domain records. The solution will show how to create private hosted zone and manage the Route 53 records using CI/CD framework.


This example uses a four account setup, but the pattern is suitable for any number of AWS accounts under an AWS Organization.

[Image: blogpost-Network-orchestrator (2).png]

Prerequisites

Before getting started, you must complete the following prerequisites:

* Create a repository in CodeCommit and provide access to your user
* Copy the sample source code from GitHub (https://github.com/ki2a/MultiRegionCodePipeline.git) under your repository
* Create an Amazon S3 (http://aws.amazon.com/s3) bucket in the current Region and each target Region for your artifact store

Solution Overview 

Our solution provisions the following resources :

1. A service role in shared service account using AWS Organization, refer to the document Enable Trusted Access to Cloudformation (https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-enable-trusted-access.html) for further details. 
2. A CodeCommit Repository that serves as your source code location.
3. A CodePipeline which is triggered using a Cloudwatch event to build a CodeBuild project.
4. Appropriate cross account IAM roles that has permissions to deploy changes to network resources using Cloudformation. 

Deployment Steps 

We deploy our solution using the provided AWS CloudFormation template in the Organization shared services account. Here is an example of how you can make changes to your security group across three accounts. The solution can be extended to any number accounts within your organization and for any networking resources.

* Launch the stack using  _Link to template_ (https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=CloudWatchAlarmSetup&templateURL=https://s3.amazonaws.com/public-customer-cloudwatch/create-cloudwatch-alarms.yaml) by logging into the AWS console and selecting CloudFormation service. The template provisions an AWS CodeCommit repository, CodeBuild project, and fully managed AWS Codepipeline in the organization shared services account.
* Next, acknowledge the permissions required to provision IAM roles, and cross account permissions granted via Organization policies and create the stack. 

An example walkthrough of steps to deploy security group changes into your accounts :

* Create codecommit repository using the following command : 
```
aws codecommit create-repository --repository-name NetworkingResourcesRepository --repository-description "Networking Resouces Repository"
```
    
* Create a clone of the CodeCommit repository to your local machine. For HTTPS using *git-remote-codecommit* (https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html), assuming the default profile and AWS Region configured in the AWS CLI, use the following command to clone the repository :
    
```git clone codecommit://NetworkingResourcesRepository NetworkingResourcesRepository```



* Create a folder with a suitable name and add the following 2 files to the folder :
        * Cloudformation template to be deployed.  (This template will consist of the resources to be provisioned) 
        * Config.json consisting of the account list in which the cloudformation template needs to be deployed.
    * Example : Create directory named “SecurityGroups” inside the “NetworkingResourcesRepository” and add the following files into the folder : 
        Cloudformation template : ecs-security-groups.yaml (https://github.com/aws-samples/ecs-refarch-cloudformation/blob/master/infrastructure/security-groups.yaml)
        Config file (config.json) 
        
* **Syntax for config.json** :
    {
    "account_list": "<list_of_accounts_to_deploy_security_groups>"
    "EnvironmentName": <dev/qa/stage/prod>
    "VPC": <enter_your_vpc_id_here>
    } 
    * 
        *Example config.json:*
* **Example config.json* :* ** 
    
    {
    "account_list": "111111111111,2222222222222,333333333333"
    "EnvironmentName": "Dev"
    "VPC": "vpc-1111111"
    } 

Note : The accounts parameter is list of account ids where the networking changes will be deployed. This is a mandatory parameter that needs to be passed to ensure successful deployment. 
eg. { “account_list” : “111111111,222222222,333333333”}

* Commit and push the changes to the repository using the following commands: 
    
* git add .
    git commit -m "adding cloudformation template to deploy security groups" 
    git push -u origin master



* Commit the change to CodeCommit. A CodePipeline will be triggered using a Cloudwatch event which then builds a CodeBuild project. The CodeBuild project will perform the necessary security checks and static code analysis using cfn-lint (https://github.com/aws-cloudformation/cfn-python-lint) utility to ensure cloudformation templates are valid. 

Validate security group changes are deployed in the accounts listed in the configuration file. 

<ADD_STEPS_FOR_UPDATES>

Note : 

1. Using this pipeline, you can Deploy your networking resources into multiple Regions using the CloudFormation template; for example, us-east-1, us-east-2, and ap-south-1 and multiple accounts such as "dev environment" , "test environments" etc. 
2. Config file in step 6 can also consists of parameters required for Cloudformation template in the following format : 

“parameterKey” : “parametervalue”, “parameterKey2”:“parameterValue2”
  eg. {“Environment”:“Development” , “VPCId“ : ”vpc-123456“}

The above setup can be extended to deploy transit gateway changes, private hosted zone dns changes , VPC route table changes. You can find the reference templates to carry out these operations <github_link here>




Conclusion 

In this post, We have shown a CI/CD solution using AWS Code Commit , AWS Code Pipeline and Cloudformation to implement most common network infrastructure changes that you are likely to implement on AWS. We removed the undifferentiated heavy lifting for setting up CI/CD pipeline. We also provided examples of how you can carry out common networking tasks by providing the necessary CloudFormation templates.

We hope that you’ve found this post informative and we look forward to hearing how you use this feature!



APPENDIX:

Scenario : SG changes : Create or update SG with inbound and outbound rules on port X (parameter from the user)
Cloudformation template : ecs-security-groups.yaml (https://github.com/aws-samples/ecs-refarch-cloudformation/blob/master/infrastructure/security-groups.yaml)
Config file : config.json

Scenario : ACL changes  : Create or update SG with inbound and outbound rules on port X (parameter from the user)
Cloudformation template : 
Config file : config.json

Scenario : VPC Route table changes : Create or update Routes with a particular destination and a target (both parameters from the user) - 192.168.0.0/16 → TGW/PCX/DXGW/Appliance 

Scenario : Private hosted zones association/authorization : Create or update private hosted zone. In terms of Update operation, it is either adding new vpc’s/accounts to the private hosted zone or removing them from a particular private hosted zone.

Scenario : TGW changes : 
Create or update Transit gateway, Transit route tables, and manage them (adding/removing attachments from a TGW route table)
 - Create/Updates  TGW 
 - Route tables within the route table  - add/remove a propagation or an association from a particular routing domain and add it to other
