## CloudGoat – Beanstalk Secrets
A walkthrough of the Beanstalk Secrets scenario from CloudGoat.

This lab demonstrates how a small configuration mistake in a cloud environment can lead to privilege escalation and full account compromise.

The scenario focuses on discovering exposed credentials in AWS Elastic Beanstalk and abusing IAM permissions to eventually retrieve a secret from AWS Secrets Manager.

All credentials shown in this repository are sanitized placeholders, and the lab environment used for the exercise has already been destroyed.


 ## Scenario

The lab begins with credentials for a low-privilege IAM user.
During enumeration of the Elastic Beanstalk environment, additional AWS credentials are discovered inside application environment variables.

Those credentials belong to another IAM user inside the same account.

After switching to this user and inspecting the available permissions, a privilege escalation path becomes visible. By abusing the iam:CreateAccessKey permission it is possible to generate credentials for another user and obtain administrator access.

With administrative privileges, secrets stored in the AWS account can be enumerated and retrieved.

## Attack Path
<img width="1113" height="554" alt="image" src="https://github.com/user-attachments/assets/1a4e5e30-9713-46bb-ad72-d340d86ff3dd" />


## Tools

AWS CLI
Used for interacting with AWS resources and verifying identities.

Pacu
Used for enumeration, IAM analysis, and privilege escalation testing.

CloudGoat
Provides the intentionally vulnerable AWS environment used in this lab.

