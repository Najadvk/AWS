# CloudGoat – SNS Secrets

A walkthrough of the **SNS Secrets** scenario from CloudGoat.

This lab demonstrates how misconfigured permissions in **Amazon SNS** and **API Gateway** can expose sensitive information and allow attackers to access protected resources.

The scenario focuses on discovering an SNS topic, subscribing to it, receiving a leaked API key through notifications, and using that key to access an API Gateway endpoint.


---

## Scenario Overview

The lab begins with credentials for a **low-privilege IAM user**.

During enumeration, the user is found to have permissions to interact with **Amazon SNS**. By identifying an accessible SNS topic and subscribing to it, a notification is received containing a **leaked API Gateway key**.

Using this key, the API Gateway service can be enumerated to discover available stages and resources. With the constructed endpoint and the leaked API key, the protected API resource can be accessed, revealing sensitive information and the final flag.

---

## Attack Flow

```
Low Privilege IAM User
        ↓
SNS Topic Enumeration
        ↓
Subscribe to SNS Topic
        ↓
API Key Leaked via Notification
        ↓
API Gateway Enumeration
        ↓
Construct API Endpoint
        ↓
Access Protected API Resource
```

---

## Walkthrough

The full step-by-step exploitation process is documented here:

**Walkthrough:**  
`sns secrets/Walkthrough.md`

---

## Key Concepts Demonstrated

- SNS topic enumeration and subscription abuse  
- Exposure of sensitive information through automated notifications  
- API Gateway enumeration using AWS CLI  
- Abuse of leaked API keys to access protected endpoints  

---

## Disclaimer

This repository is intended for **educational and security training purposes only**.  
The techniques demonstrated are part of a controlled lab environment provided by CloudGoat.
