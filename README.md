# AWS Security Challenges

![AWS](https://img.shields.io/badge/Platform-AWS-orange)
![Focus](https://img.shields.io/badge/Focus-Cloud%20Security-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-red)

# AWS Security Challenges

This repository contains practical AWS security exercises focused on identifying misconfigurations, discovering exposed credentials, and performing cloud privilege escalation in controlled environments.  

The exercises demonstrate techniques commonly used during cloud security assessments, including resource enumeration, IAM analysis, and secrets discovery.

## Challenges

### Beanstalk Secrets
Explore an AWS Elastic Beanstalk environment to uncover **secondary credentials** stored in environment variables.  
By enumerating IAM permissions and exploiting the **CreateAccessKey** method, it is possible to escalate privileges to an administrator account and access secrets stored in AWS Secrets Manager.

**Skills practiced:**
- Elastic Beanstalk enumeration  
- Credential discovery in environment variables  
- IAM permission analysis  
- Privilege escalation via `CreateAccessKey`  
- Retrieving secrets from AWS Secrets Manager  

### SNS Secrets
Investigate **Amazon SNS topics** to uncover sensitive information.  
After subscribing to an SNS topic, messages may contain leaked **API Gateway keys**. By enumerating API Gateway resources and using the key, the full API endpoint can be accessed to retrieve sensitive data.

**Skills practiced:**
- IAM permission enumeration  
- SNS topic enumeration and subscription  
- Data leakage analysis via SNS notifications  
- API Gateway enumeration  
- Using API keys to access protected resources  
---
