# ElectricEye (ElectricEye-Response)
Continuously monitor your AWS services for configurations that can lead to degradation of confidentiality, integrity or availability. All results will be sent to Security Hub for further aggregation and analysis.

***Up here in space***<br/>
***I'm looking down on you***<br/>
***My lasers trace***<br/>
***Everything you do***<br/>
<sub>*Judas Priest, 1982*</sub>

## Table of Contents
- [Description](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#description)
- [Solution Architecture](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#solution-architecture)
- [Prerequisites](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#prerequisites)
- [Setting Up](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#setting-up)
  - [Deploying ElectricEye-Response Cross-Account Role via StackSets](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#deploying-electriceye-response-cross-account-role-via-stacksets)
  - [Semi-Auto ElectricEye-Response with CloudFormation](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#semi-auto-electriceye-response-with-cloudformation)
  - [Full-Auto ElectricEye-Response with Terraform](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#full-auto-electriceye-response-with-terraform)
- [Playbook Reference Repository](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#playbook-reference-repository)
- [Known Issues and Limitations](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#known-issues-and-limitations)
- [License](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/README.md#license)

## Description
ElectricEye-Response is a multi-account automation framework for response and remediation actions heavily influenced by [work I did when employed by AWS](https://aws.amazon.com/blogs/security/automated-response-and-remediation-with-aws-security-hub/). From your Security Hub Master, you can launch response and remediation actions by using CloudWatch Event rules, Lambda functions, Security Token Service (STS) and downstream services (such as Systems Manager Automation or Run Command). You can run these in a targetted manner (using Custom Actions) or fully automatically (using the CloudWatch detail type of `Security Hub Findings - Imported`).

These are written to support native Security Hub security standards (CIS, PCI-DSS, etc.) as well as ElectricEye Auditor checks. The role that ElectricEye-Response will assume to perform cross-account response actions will be deployed via CloudFormation (to take advantage of StackSets). The bulk of the codebase will be deployed via Terraform for fully-automatic remediation or via CloudFormation (for Custom Actions using custom providers).

***You should choose which version of ElectricEye-Response you use based on your organizational standard operating procedures (SOPs) for security engineering and/or incident response. The full-auto playbooks will not take into consideration any exceptions you have in place as currently designed.*** Due to the lack of exception acknowledgement, the full-auto version of ElectricEye-Response will not contain all actions.

## Solution Architecture
![ResponseThis](https://github.com/jonrau1/ElectricEye/blob/master/screenshots/electriceye-response-sad.jpg)
1.	A CloudFormation StackSet is used to deploy the cross-account role to multiple member accounts
2.	Findings from compliance standards and ElectricEye are collected in Security Hub, identifying compliant and non-compliant resources
3.	A Security Hub Master will aggregate findings from member accounts
4.	CloudWatch Events / EventBridge rules will monitor for non-compliant findings and automatically invoke Lambda functions based on the type of failed check
5.	Depending on the Account that owns the finding, Lambda will either attempt to assume a cross account role or use its own execution role to perform remediation actions. **Note**: Two Lambda function icons are shown for illustration purposes only, a single function will contain this “if/else” logic
6.	If the finding belongs to a Member account, Lambda will assume the role deployed in Step 1
7.	Temporary security credentials with permissions to perform the remediation is given to the Lambda function and the non-compliant resource is brought back into a compliant state
8.	Findings that are remediated will have an annotation added via the Security Hub UpdateFindings API noting if the remediation was successful, successfully remediated findings will be archived

## Prerequisites
- ElectricEye-Response must be deployed to the account your Security Hub Master is located
- For using StackSets you must have the required [execution roles](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html) configured
- AWS Config and the PCI-DSS / CIS security standards enabled in your member accounts (ElectricEye-Response will work without these turned on)

## Setting Up
These steps are split across their relevant sections. All CLI commands are executed from an Ubuntu 18.04LTS [Cloud9 IDE](https://aws.amazon.com/cloud9/details/), modify them to fit your OS.

**Important Notes:** The IAM policy for deploying Terraform is highly dangerous and potentially destructive, as with the core module. Additionally, the cross-account role and the Master account's Lambda execution role are potentially dangerous due to the amount of Update and Delete permissions they have. The cross-account role also trusts SSM and Backup in addition to your Master account and should be monitored closely for any abuse.

### Deploying ElectricEye-Response Cross-Account Role via StackSets
In this stage we will use CloudFormation Stack Sets to deploy the multi-account role to our Security Hub member accounts
1.	Download the CloudFormation template, it is titled `ElectricEye-Response_CrossAccount_CFN.yml`
2.	Navigate to the CloudFormation Console and select **StackSets** from the navigation pane on the left-hand side
3.	Select **Create StackSet**, choose **Upload a template file** and select **Choose file** to upload the CloudFormation template you download and select **Next**
4.	Enter a StackSet name and enter in the account number of your Security Hub Master account for the IAM role to trust and select **Next** as shown below
![StackSetParams](https://github.com/jonrau1/ElectricEye/blob/master/screenshots/electriceye-response-xaccount-stackset-param.JPG)
5.	In the next screen select the **IAM admin role** and enter the **IAM role name** for the execution role in your member accounts and select **Next** as shown below
![StackSetPerms](https://github.com/jonrau1/ElectricEye/blob/master/screenshots/electriceye-response-xaccount-stackset-perms.JPG)
6.	For **Deployment locations** either manually enter your Security Hub member *account* or *organizational unit (OU)* numbers. You can also provide a CSV file of all account numbers instead of manually entering them
7.	Specify the **region(s)** you will deploy this stack to and optionally modify the **Deployment options** and select **Next** as shown below
![StackSetRegions](https://github.com/jonrau1/ElectricEye/blob/master/screenshots/electriceye-response-xaccount-stackset-regions.JPG)
8.	On the next screen acknowledge the information box under **Capabilities** and select **Submit**
9.	Select the **Stack instances** tab and continually refresh as the StackSet deploys the IAM role. Once all accounts are showing the status of “current” you can proceed to the next stage

### Semi-Auto ElectricEye-Response with CloudFormation
In this section we will deploy Custom Actions, CloudWatch Events and Lambda functions to the Security Hub Master using a CloudFormation template.

***WORK IN PROGRESS***

### Full-Auto ElectricEye-Response with Terraform
In this section we will deploy CloudWatch Events and Lambda functions to the Security Hub Master using Terraform. **Important Note:** Only certain playbooks are contained within the full-auto version of ElectricEye-Response. Actions such as Shield Advanced protection, creating one-time Backups or deleting EC2 instances should likely **NOT** be executed without a human pulling the trigger.

All CLI commands are executed from an Ubuntu 18.04LTS [Cloud9 IDE](https://aws.amazon.com/cloud9/details/), modify them to fit your OS. Before starting [attach this IAM policy](https://github.com/jonrau1/ElectricEye/blob/master/add-ons/electriceye-response/policies/electriceye-response-terraform-policy.json) to your [Instance Profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) (if you are using Cloud9 or EC2).

***WORK IN PROGRESS***

## Playbook Reference Repository
There are currently **32** supported response and remediation Playbooks with coverage across **23** AWS services / components supported by ElectricEye-Response.

| Playbook Name                                | AWS Service In Scope                                                                                  | Action Taken                                                                                                                                                                                                                 |
|----------------------------------------------|-------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| CloudTrail_FileValidation_Playbook.py        | CloudTrail                                                                                            | Re-enable Log File Validation                                                                                                                                                                                                |
| EBS_Snapshot_Private.py                      | EBS Snapshot                                                                                          | Remove Public access to Snapshot                                                                                                                                                                                             |
| EC2_Isolation_Playbook.py                    | EC2 Instance                                                                                          | Create a new Security Group without any rules and <br>attach it to the instance, thus isolating it                                                                                                                           |
| ELBV2_DelProt_Playbook.py                    | ELBv2 (ALB/NLB)                                                                                       | Enable ELBv2 deletion protection                                                                                                                                                                                             |
| ELBV2_Drop_Invalid_HTTP_Header_Playbook.py   | ELBv2 (ALB/NLB)                                                                                       | Configure ELBv2 to drop Invalid<br>HTTP headers                                                                                                                                                                              |
| KDS_Apply_Encryption_Playbook.py             | Kinesis Data Stream                                                                                   | Applies AWS-managed encryption to stream                                                                                                                                                                                     |
| KMS_CMK_Rotation_Playbook.py                 | KMS CMK                                                                                               | Enable KMS CMK Rotation                                                                                                                                                                                                      |
| RDS_DelProt_Playbook.py                      | RDS Instance                                                                                          | Enable RDS instance deletion<br>protection                                                                                                                                                                                   |
| RDS_Multi_AZ_Playbook.py                     | RDS Instance                                                                                          | Configure RDS instance in Multi-AZ                                                                                                                                                                                           |
| RDS_Privatize_Instance_Playbook.py           | RDS Instance                                                                                          | Remove Public access to RDS Instance                                                                                                                                                                                         |
| RDS_Privatize_Snapshot_Playbook.py           | RDS Snapshot                                                                                          | Remove Public access to Snapshot                                                                                                                                                                                             |
| Redshift_Privatize_Playbook.py               | Redshift cluster                                                                                      | Remove Public access to Redshift<br>cluster                                                                                                                                                                                  |
| Release_EIP_Playbook.py                      | Elastic IP                                                                                            | Release unallocated EIP                                                                                                                                                                                                      |
| Release_SG_Playbook.py                       | Security Group                                                                                        | Release unattached SG                                                                                                                                                                                                        |
| Remove_All_SG_Rules_Playbook.py              | Security Group                                                                                        | Remove ALL ingress and egress rules<br>from SG                                                                                                                                                                               |
| Remove_Open_MySQL_Playbook.py                | Security Group                                                                                        | Remove ingress to 3306 from SG                                                                                                                                                                                               |
| Remove_Open_RDP_Playbook.py                  | Security Group                                                                                        | Remove ingress to 3389 from SG                                                                                                                                                                                               |
| Remove_Open_SSH_Playbook.py                  | Security Group                                                                                        | Remove ingress to 22 from SG                                                                                                                                                                                                 |
| S3_Encryption_Playbook.py                    | S3 Bucket                                                                                             | Enable SSE-S3 encryption on Bucket                                                                                                                                                                                           |
| S3_PrivateACL_Playbook.py                    | S3 Bucket                                                                                             | Puts 'PRIVATE' ACL on bucket                                                                                                                                                                                                 |
| S3_Put_Lifecycle_Playbook.py                 | S3 Bucket                                                                                             | move current and versioned objects to OZ_IA after 180<br>move current and versioned objects to Glacier after 365<br>delete current and versioned objects after 7 years (2555 days)<br>delete multi-part fails after 24 hours |
| S3_Versioning_Playbook.py                    | S3 Bucket                                                                                             | Enable bucket versioning                                                                                                                                                                                                     |
| ShieldAdv_AutoRenew_Playbook.py              | AWS Account (Shield)                                                                                  | Sets Shield Subscription to auto-renew                                                                                                                                                                                       |
| ShieldAdv_Protection_Playbook.py             | Route53 Hosted Zone<br>CloudFront distro<br>ELB<br>ELBv2<br>Global Accelerator<br>Elastic IP          | Creates Shield Advanced protection for<br>a resource                                                                                                                                                                         |
| SSM_ApplyPatch_Playbook.py                   | EC2 Instance                                                                                          | Invokes AWS-RunPatchBaseline document on instance                                                                                                                                                                            |
| SSM_DeleteEC2_Playbook.py                    | EC2 Instance                                                                                          | Invokes AWS-TerminateEC2Instance document on instance                                                                                                                                                                        |
| SSM_InspectorAgent_Playbook.py               | EC2 Instance                                                                                          | Invokes AmazonInspector-ManageAWSAgent document on instance                                                                                                                                                                  |
| SSM_RefreshAssoc_Playbook.py                 | EC2 Instance                                                                                          | Invokes AWS-RefreshAssociation document on instance                                                                                                                                                                          |
| SSM_UpdateAgent_Playbook.py                  | EC2 Instance                                                                                          | Invokes AWS-UpdateSSMAgent document on instance                                                                                                                                                                              |
| Start_Backup_Playbook.py                     | EC2 Instance<br>EBS Volume<br>DyanmoDB Table<br>Storage Gateway<br>RDS Instance<br>Elastic File Share | Creates a "one-time" backup for a resource in AWS Backup                                                                                                                                                                     |
| WAFv1_GuardDutyProbe_UpdateIPSet_Playbook.py | WAFv1 IP Set                                                                                          | From a Security Hub port-probe finding, retrieves all<br>malicious IP addresses from original GuardDuty finding<br>and adds them to a WAFv1 IP Set                                                                           |
| WAFv2_GuardDutyProbe_UpdateIPSet_Playbook.py | WAFv2 IP Set                                                                                          | From a Security Hub port-probe finding, retrieves all<br>malicious IP addresses from original GuardDuty finding<br>and adds them to a WAFv2 IP Set                                                                           |

## Known Issues and Limitations
- Security Hub Security Standards currently use a finding that is scoped to the `AwsAccount` resource in the ASFF to roll up all results and give you a pass/fail/not available score on the control. Due to this, you may encountere failures or exceptions in your CloudWatch logs from these findings.

- As designed these playbooks will not consider any exceptions you may have. Please open a PR for this functionality, it may make sense to develop that as a Step Function state machine versus a Lambda function.

## License
This library is licensed under the GNU General Public License v3.0 (GPL-3.0) License. See the LICENSE file.