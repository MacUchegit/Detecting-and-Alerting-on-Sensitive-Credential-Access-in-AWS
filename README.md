# Detecting and Alerting on Sensitive Credential Access in AWS

## A Real-Time Security Monitoring Project for Sanifeo

### Project Overview

Organizations commonly store database passwords, API keys, tokens, and other application credentials in AWS Secrets Manager rather than hard-coding them inside applications.

However, securely storing a credential solves only part of the problem.

An organization must also be able to answer:

* Who accessed a sensitive credential?
* When was it accessed?
* Which AWS identity retrieved it?
* Where did the request originate?
* Can the security team be notified when sensitive access occurs?

In this project, I worked as a **Cloud Security Engineer for Sanifeo**, a fictional e-commerce organization.

During a security review, I identified a monitoring gap: Sanifeo had no immediate alerting mechanism when a sensitive production application credential was retrieved from AWS Secrets Manager.

I designed and implemented a lightweight security monitoring solution that:

1. Stores a dummy production credential in AWS Secrets Manager.
2. Records access activity using AWS CloudTrail.
3. Detects every `GetSecretValue` API call using Amazon EventBridge.
4. Sends a security alert through Amazon SNS every time the protected secret is accessed.
5. Allows the security team to investigate the identity, source, time, and nature of the access.
6. Validates the solution through a controlled credential-access simulation.
7. Removes the temporary resources after testing.

---

# Business Problem

Sanifeo's payment application uses a third-party payment provider and requires an API credential.

The credential is securely stored in AWS Secrets Manager.

During a security assessment, I discovered that while access to the secret could be audited later, the Cloud Security team was not automatically notified when the credential was retrieved.

This created a visibility gap.

A compromised administrator account, overly privileged developer, or unexpected workload could potentially retrieve the credential without the security team being immediately aware.

### Security Requirement

I translated the problem into the following requirement:

> **Generate a security notification every time the protected Sanifeo credential is retrieved and retain enough audit information to identify who accessed it, when the access occurred, and where the request originated.**

Because this credential represents sensitive information, there is **no minimum access threshold**. A single retrieval is enough to trigger a security notification.

The objective was **detection and visibility**, not automatic remediation.

---

# Project Objectives

By the end of the project, I wanted to prove that:

* the credential could be stored securely;
* credential retrieval generated an auditable AWS event;
* every occurrence of the specific API operation could be detected automatically;
* the Cloud Security team could receive an alert whenever the credential was accessed;
* the initiating AWS identity could be investigated;
* the architecture remained simple enough to operate and understand;
* temporary laboratory resources could be completely cleaned up.

---

# Architecture

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/b8423065-4046-4851-a1c8-057d1c27a21e" />

*Figure 1: Sanifeo sensitive credential monitoring architecture. AWS CloudTrail records retrieval of the protected credential, EventBridge detects every **`GetSecretValue`** API operation, and Amazon SNS delivers a security notification whenever the secret is accessed.*

---

# AWS Services Used

| AWS Service         | Purpose in the Project                                   |
| ------------------- | -------------------------------------------------------- |
| AWS Secrets Manager | Securely stores the dummy Sanifeo application credential |
| AWS CloudTrail      | Records API activity associated with the credential      |
| Amazon EventBridge  | Detects every security-relevant `GetSecretValue` event   |
| Amazon SNS          | Sends the resulting security notification by email       |
| Amazon S3           | Stores CloudTrail trail logs                             |

I intentionally avoided adding unnecessary services.

Because the credential represents sensitive production information, I did not want the system to wait for multiple retrievals or a time-based threshold before notifying the security team.

The security requirement was:

> **Notify security whenever this specific security-sensitive API operation occurs, including the first retrieval.**

The detection model is therefore:

```text
1 GetSecretValue event
        =
1 Security Alert
```

For that reason, an EventBridge event rule provided a simple event-driven architecture.

---

# Security Event Being Monitored

Retrieving a secret from AWS Secrets Manager uses the API operation:

```text
GetSecretValue
```

The required IAM permission is:

```text
secretsmanager:GetSecretValue
```

AWS Secrets Manager generates a CloudTrail event when this API operation occurs.

It is important to distinguish the operation correctly:

`GetSecretValue` is a **read-only management operation** because it retrieves information without modifying the AWS resource.

The secret value itself is not written into the CloudTrail log.

This allowed me to monitor access without exposing the sensitive credential inside the audit record.

Because the credential is considered sensitive, **every occurrence of ****`GetSecretValue`**** should generate an alert**.

---

# Implementation

## Step 1 — Verify My AWS Identity and Region

Before creating any resources, I first verified which AWS identity and account my local AWS CLI was using.

I ran:

```bash
aws sts get-caller-identity --profile sanifeo-fin-user
```

This returned information about the authenticated AWS principal and account.

The IAM user used for this Sanifeo environment was:

```text
sanifeo-fin-user
```

I also confirmed that I would perform the project consistently in:

```text
eu-west-2 — Europe (London)
```

Keeping resources in one Region simplified troubleshooting and cleanup.

### Why I Did This

Cloud security investigations frequently fail because engineers search the wrong:

* account;
* Region;
* identity; or
* environment.

Verifying these values before deployment gave me a known baseline.

<img width="976" height="362" alt="image" src="https://github.com/user-attachments/assets/217c37c8-a135-4123-a4a1-f3c7ed2a1f04" />

*Figure 2: Verification of the **`sanifeo-fin-user`** AWS identity used for the Sanifeo security monitoring project. I confirmed the active account and IAM principal before deploying any resources.*

---

# Step 2 — Create the Sensitive Sanifeo Test Credential

I next created the asset that the monitoring solution would protect.

From the AWS Management Console, I opened:

**AWS Secrets Manager → Store a new secret**

I selected:

```text
Secret type:
Other type of secret
```

For the test value, I used dummy information only.

Example:

```text
Key:   api_key
Value: SANIFEO-DEMO-KEY-NOT-REAL
```

I did **not** use a real password, API key, AWS credential, or customer information.

For encryption, I used the AWS-managed Secrets Manager encryption key rather than creating a separate customer-managed KMS key for this temporary lab.

I named the secret:

```text
sanifeo/prod/payment-api
```

I added a description similar to:

```text
Dummy payment API credential used for Sanifeo
security monitoring training.
```

Automatic rotation was unnecessary for this temporary dummy credential.

I completed creation of the secret.

### Why I Did This

Secrets Manager provides a central location for protecting application secrets rather than placing credentials inside:

* application source code;
* configuration files;
* repositories;
* scripts; or
* emails.

The important security question for this project, however, was what happened **after someone retrieved that secret**.

### Screenshot Evidence

Do **not** publish a screenshot showing the secret value.

Capture the secret's overview page showing:

* secret name;
* description;
* encryption configuration;
* Region.

<img width="1633" height="419" alt="image" src="https://github.com/user-attachments/assets/3c140920-13f7-41da-8fa3-16852cd19803" />

*Figure 3: Dummy Sanifeo payment credential stored in AWS Secrets Manager. Only metadata is shown; the secret value was intentionally excluded from portfolio evidence.*

---

# Step 3 — Configure CloudTrail Audit Logging

I needed an audit source capable of recording access to the secret.

AWS CloudTrail records API operations performed through:

* the AWS Management Console;
* AWS CLI;
* AWS SDKs;
* AWS APIs;
* AWS services.

I opened:

**AWS CloudTrail → Trails → Create trail**

I named the trail:

```text
sanifeo-security-monitoring-trail
```

For storage, I created a dedicated S3 bucket using a unique name such as:

```text
sanifeo-cloudtrail-logs-02665******
```

I did not create a customer-managed KMS key solely for this temporary project.

For management events, I retained both:

```text
Read
Write
Exclude AWS KMS events
Exclude Amazon RDS Data API events
```

This was important because `GetSecretValue` is a read operation.

I then completed creation of the trail and confirmed that logging was enabled.

### Why a Trail Was Needed

CloudTrail Event History automatically provides recent management-event visibility, but EventBridge rules using the:

```text
AWS API Call via CloudTrail
```

event type require an actively logging CloudTrail trail for the appropriate event category.

The trail therefore served two purposes:

1. provide an ongoing audit configuration;
2. make the API activity available for the EventBridge detection workflow.

<img width="1381" height="586" alt="image" src="https://github.com/user-attachments/assets/998a8b3a-7e81-47b9-8623-458694308901" />

*Figure 4: Active CloudTrail trail configured to record Sanifeo AWS management activity. Read management events were retained so that **`GetSecretValue`** retrieval activity could be captured.*

---

# Step 4 — Establish a Baseline CloudTrail Event

Before creating the automated alert, I wanted to prove that AWS could actually record the security event I planned to monitor.

I retrieved the dummy secret once using the AWS CLI.

To avoid displaying the actual value in my terminal, I queried only the secret name:

```bash
aws secretsmanager get-secret-value ^
  --secret-id sanifeo/prod/payment-api ^
  --region eu-west-2 ^
  --profile sanifeo-fin-user ^
  --query Name ^
  --output text
```

Because I was running the command on Windows, I used the appropriate command-line formatting for my environment.

The command returned:

```text
sanifeo/prod/payment-api
```

Although I prevented the secret value from appearing in my terminal, AWS still processed a real:

```text
GetSecretValue
```

API request.

### Security Benefit

This gave me a controlled security event without exposing the dummy credential in my screenshots.

Because the monitoring requirement applies to **every secret retrieval**, this single API request represented exactly the type of activity that should ultimately generate a security alert once the full detection pipeline was configured.

### Screenshot Evidence

Capture:

* the AWS CLI command;
* the returned secret name;
* no secret value.

<img width="1381" height="135" alt="image" src="https://github.com/user-attachments/assets/f563997f-7093-438f-a6c4-04a760708510" />

*Figure 5: Controlled **`GetSecretValue`** request generated through the AWS CLI using **`sanifeo-fin-user`**. The CLI output was restricted to the secret name so that no credential value was exposed in portfolio evidence.*

---

# Step 5 — Verify the Event in CloudTrail

I opened:

**CloudTrail → Event history**

I filtered for:

```text
Event name:
GetSecretValue
```

<img width="1557" height="296" alt="image" src="https://github.com/user-attachments/assets/4f393fa6-c6c4-4a33-8005-f1129a13edf1" />

*Figure 6.0: CloudTrail evidence generated by the controlled secret retrieval.*

Alternatively, Secrets Manager activity could be located using:

```text
Event source:
secretsmanager.amazonaws.com
```

I located the event generated by my CLI test.

I opened the event details and reviewed fields including:

```text
eventTime
eventName
eventSource
userIdentity
sourceIPAddress
awsRegion
requestParameters
readOnly
```

The event allowed me to answer several investigation questions.

### Who?

The `userIdentity` field showed which AWS identity initiated the request.

In my controlled test, this was associated with:

```text
sanifeo-fin-user
```

### What?

```text
GetSecretValue
```

showed exactly which API action had occurred.

### When?

`eventTime` provided the event timestamp.

### Where?

`sourceIPAddress` helped identify the source from which the request originated.

### Which Region?

`awsRegion` confirmed where the API action occurred.

### What resource?

The request information identified the targeted secret.

This was an important milestone because it proved that Sanifeo already had the forensic evidence necessary to investigate the activity.

The remaining problem was:

> Nobody was being automatically notified.

### Screenshot Evidence

Capture the expanded CloudTrail event showing:

* `GetSecretValue`;
* timestamp;
* AWS identity type/name;
* source IP;
* Region;
* read-only status.

Before publishing:

* redact the AWS account ID;
* partially redact your public IP;
* remove any unique credential information.

<img width="1513" height="380" alt="image" src="https://github.com/user-attachments/assets/fee7fa01-e592-422b-8710-e5c7f531fbc1" />

*Figure 6.1: CloudTrail evidence generated by the controlled secret retrieval. The event identified the API operation, initiating AWS identity, timestamp, source IP address, and Region.*

---

# Step 6 — Create the Security Notification Channel

Next, I created the mechanism that would notify the security team.

I opened:

**Amazon SNS → Topics → Create topic**

I selected:

```text
Type:
Standard
```

I named the topic:

```text
sanifeo-security-alerts
```

After creating the topic, I created an email subscription.

I selected:

```text
Protocol:
Email
```

and entered the email address I was using for the lab.

AWS sent a confirmation message to that address.

I opened the message and selected:

```text
Confirm subscription
```

I then returned to SNS and confirmed that the subscription no longer displayed:

```text
Pending confirmation
```

### Why Confirmation Matters

SNS does not begin sending notifications to an email endpoint until the recipient confirms the subscription.

This prevents AWS users from arbitrarily subscribing unrelated email addresses to notification topics.

<img width="1359" height="619" alt="image" src="https://github.com/user-attachments/assets/3bcbdcea-ff31-47c5-a897-c26534b57dc2" />

*Figure 7: Confirmed Amazon SNS email subscription for the **`sanifeo-security-alerts`** topic. This topic acts as the notification channel for detected credential access.*

---

# Step 7 — Create the EventBridge Detection Rule

With audit logging and notification working independently, I connected them using Amazon EventBridge.

The detection logic was deliberately simple.

I wanted EventBridge to identify:

```text
AWS Secrets Manager
        +
GetSecretValue API call
```

Importantly, I did **not** configure a threshold such as a certain number of secret retrievals within five minutes.

The credential is considered sensitive.

Therefore:

```text
ANY GetSecretValue event
        ↓
Security alert
```

I opened:

**Amazon EventBridge → Rules → Create rule**

I entered:

```text
Name:
sanifeo-detect-secret-access
```

Description:

```text
Detects retrieval of the protected Sanifeo
payment credential and notifies security.
```

I selected:

```text
Event bus:
default
```

and:

```text
Rule type:
Rule with an event pattern
```

For the event source, I selected:

```text
AWS events / AWS services
```

I used an event pattern based on Secrets Manager API events delivered through CloudTrail.

The core pattern was:

```json
{
  "source": ["aws.secretsmanager"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["secretsmanager.amazonaws.com"],
    "eventName": ["GetSecretValue"]
  }
}
```

This means:

```text
IF

source = AWS Secrets Manager

AND

event type = API activity captured by CloudTrail

AND

API operation = GetSecretValue

THEN

the rule matches immediately.
```

There was no count threshold.

There was no five-minute waiting period.

Each matching `GetSecretValue` event represented a security-relevant access event.

For this controlled training account, the rule monitored `GetSecretValue` activity.

Because my lab contained the designated Sanifeo test credential, this was sufficient for demonstrating the control.

In a production environment containing many secrets, I would further scope the detection strategy to high-value secrets or approved naming/tagging conventions to reduce unnecessary alerts.

### Selecting the Target

For the EventBridge target, I selected:

```text
AWS service
```

and then:

```text
SNS topic
```

I selected:

```text
sanifeo-security-alerts
```

as the target.

I reviewed the configuration and created the rule.

### What I Had Built

At this point the monitoring workflow was:

```text
Secret retrieval
       ↓
CloudTrail
       ↓
EventBridge rule
       ↓
SNS
       ↓
Security email
```

The alerting principle was:

```text
1 sensitive secret retrieval
            ↓
      1 security alert
```


<img width="1165" height="679" alt="image" src="https://github.com/user-attachments/assets/4e0dacb2-9728-482d-a447-f42ad6944ff7" />

*Figure 8A: EventBridge event pattern configured to detect every AWS Secrets Manager **`GetSecretValue`** API activity recorded through CloudTrail.*

<img width="1513" height="588" alt="image" src="https://github.com/user-attachments/assets/aa989f4c-367f-462f-ad7c-1ff3afa17dea" />

*Figure 8B: Amazon SNS configured as the target of the Sanifeo EventBridge security rule, completing the event-driven notification path.*

---

# Step 8 — Test the Complete Monitoring System

Creating security controls is not enough.

I needed to prove that the control actually worked.

I performed another controlled credential retrieval using the AWS CLI:

```bash
aws secretsmanager get-secret-value ^
  --secret-id sanifeo/prod/payment-api ^
  --region eu-west-2 ^
  --profile sanifeo-fin-user ^
  --query Name ^
  --output text
```

This represented the security event Sanifeo wanted to detect.

Only **one retrieval** was necessary.

<img width="1252" height="89" alt="image" src="https://github.com/user-attachments/assets/07910d64-886f-48cd-bf86-76f8f3d4a32e" />

I did not need to repeatedly retrieve the credential because the control was explicitly designed to alert on every individual access.

Conceptually:

<img width="1902" height="658" alt="image2" src="https://github.com/user-attachments/assets/e1ad2298-2edb-4a1e-bf75-664d13e23679" />


During the first end-to-end test, however, the SNS email did **not** arrive.

Rather than rebuilding the entire workflow, I verified each component individually. CloudTrail was successfully recording `GetSecretValue`, the SNS subscription was confirmed, and the EventBridge rule was enabled. EventBridge monitoring, however, showed no matched events or target invocations.

This led me to identify an EventBridge configuration requirement for the read-only `GetSecretValue` management event. I corrected the rule configuration through the AWS CLI, retested the same controlled retrieval, and then successfully received the SNS notification.

This troubleshooting process is explained briefly in the **Troubleshooting and Lessons During Validation** section below.

After applying the fix, I checked the email account subscribed to the SNS topic.

The notification confirmed that the event had successfully travelled through the monitoring pipeline.

This demonstrated that the detection was operational rather than simply configured.

The expected behavior was:

```text
First retrieval
      ↓
Alert

Second retrieval
      ↓
Alert

Third retrieval
      ↓
Alert
```

Every access to the protected sensitive credential should result in a corresponding security notification.


<img width="1555" height="587" alt="image" src="https://github.com/user-attachments/assets/3f3a92fb-1b91-4567-a0bc-bf39d04783e3" />

*Figure 9: Security notification generated after a single controlled retrieval of the Sanifeo payment credential. This validated that every **`GetSecretValue`** event can trigger the CloudTrail → EventBridge → SNS detection pipeline without waiting for an access threshold.*

---

# Step 9 — Investigate the Simulated Security Event

Receiving an alert is only the start of security monitoring.

A Cloud Security Engineer must be able to investigate it.

I returned to:

**CloudTrail → Event history**

and located the new:

```text
GetSecretValue
```

event.

I treated the test as a small incident investigation.

## Investigation Questions

### 1. What happened?

A Secrets Manager credential was retrieved using:

```text
GetSecretValue
```

### 2. Which resource was involved?

```text
sanifeo/prod/payment-api
```

### 3. Who initiated the request?

I reviewed:

```text
userIdentity
```

to determine the AWS identity involved.

For my controlled test, the request was associated with the Sanifeo IAM identity:

```text
sanifeo-fin-user
```

### 4. When did it happen?

I reviewed:

```text
eventTime
```

### 5. Where did the request originate?

I reviewed:

```text
sourceIPAddress
```

### 6. Was this an administrative modification?

No.

The CloudTrail record identified the activity as read-only.

The secret was accessed, but it was not modified.

### 7. Was the access authorized?

For my test, yes.

The event was intentionally generated as part of the security validation.

However, in a genuine incident, I would compare the identity against:

* expected application roles;
* approved administrators;
* change records;
* working hours;
* known IP ranges;
* other surrounding CloudTrail activity.

---

# Mini Incident Record

## Incident

Unexpected retrieval of a sensitive Sanifeo application credential.

## Detection

Amazon EventBridge matched a CloudTrail `GetSecretValue` event and delivered an alert through Amazon SNS.

The alert did not depend on repeated activity.

A single credential retrieval was sufficient to activate the detection.

## Asset

```text
sanifeo/prod/payment-api
```

## Evidence Source

AWS CloudTrail Event History.

## Investigation Fields

```text
eventName
eventTime
eventSource
userIdentity
sourceIPAddress
awsRegion
requestParameters
```

## Test Conclusion

The event was generated intentionally as part of a controlled validation exercise by the `sanifeo-fin-user` AWS identity.

The monitoring system successfully detected the individual access event and generated the expected security notification.

<img width="1061" height="577" alt="image" src="https://github.com/user-attachments/assets/e51115f2-af23-4f56-b9da-18cc623bfb79" />

*Figure 10: Investigation of the simulated credential-access event using CloudTrail. The audit record provided the identity, timestamp, API operation, source address, Region, and affected secret required for initial triage.*

---

# Step 10 — Evaluate the Security Control

After proving that the system worked, I evaluated what the solution could and could not do.

## What the Control Provides

The solution provides:

* audit visibility;
* event-driven detection;
* notification on every sensitive credential retrieval;
* email notification;
* identity attribution;
* source information;
* investigation evidence.

## What the Control Does Not Provide

The monitoring system does **not** prevent an authorized identity from retrieving the secret.

This is a detective control.

```text
Preventive Control
        ↓
Stops an unwanted action

Detective Control
        ↓
Identifies that an action occurred
```

My solution is primarily:

```text
DETECTIVE
```

This distinction is important.

A mature production solution should combine the monitoring control with preventive mechanisms such as:

* least-privilege IAM permissions;
* tightly scoped workload roles;
* restrictions on human access;
* credential rotation;
* monitoring for unusual identities or sources;
* separation between development and production;
* periodic access reviews.

---

# Troubleshooting and Lessons During Validation

The first end-to-end validation did not produce an SNS email, even though the EventBridge rule appeared correctly configured.

I troubleshot the workflow from source to destination rather than making random configuration changes:

1. **Verified CloudTrail** — `GetSecretValue` was successfully recorded and the trail was configured for both read and write management events.
2. **Verified SNS** — the email subscription was confirmed, eliminating an unconfirmed subscription as the cause.
3. **Checked EventBridge monitoring** — `MatchedEvents`, `TriggeredRules`, and `Invocations` showed no activity. This demonstrated that EventBridge was not matching the read-only event, so SNS had never been invoked.
4. **Identified the EventBridge rule-state issue** — `GetSecretValue` is a read-only CloudTrail management event. The standard `ENABLED` EventBridge state was not sufficient for processing this class of `Get*` management events.
5. **Updated the rule through the AWS CLI** — I changed the rule state to:

```text
ENABLED_WITH_ALL_CLOUDTRAIL_MANAGEMENT_EVENTS
```

While applying the change, I also encountered a PowerShell JSON quoting issue. Instead of passing the event pattern as inline JSON, I stored the pattern in a local `event-pattern.json` file and referenced it with:

```text
file://event-pattern.json
```

This avoided PowerShell stripping the JSON quotation marks.

After updating the EventBridge rule, I generated one new `GetSecretValue` request and successfully received the SNS security notification.

### Troubleshooting Outcome

The incident reinforced an important engineering lesson:

> **A correctly written event pattern does not guarantee that the underlying event is eligible to reach the rule.**

By checking the monitoring pipeline one component at a time, I isolated the failure to EventBridge rather than incorrectly assuming that SNS was responsible.

---

# Security Findings

The project demonstrated several important cloud-security principles.

### Finding 1 — Secure Storage Is Not the Same as Monitoring

Secrets Manager protects credential storage, but organizations also need visibility into credential usage.

Because the protected credential is sensitive, even one retrieval can warrant security awareness.

### Finding 2 — AWS API Activity Provides Strong Investigation Evidence

CloudTrail allowed me to attribute activity to an AWS identity and examine contextual information surrounding the request.

### Finding 3 — Event-Driven Detection Can Remain Simple

EventBridge allowed me to detect every relevant API operation without creating a larger monitoring stack than the requirement justified.

### Finding 4 — Sensitive Access Should Be Visible Immediately

The solution was deliberately designed so that:

```text
First secret retrieval
        ↓
Security notification
```

The security team does not need to wait for repeated retrievals before gaining visibility.

### Finding 5 — Alerts Must Be Tested

A configured rule is not proof that a monitoring system works.

Only an end-to-end test demonstrated that:

```text
event
 → detection
 → notification
 → investigation
```

was operational.

### Finding 6 — Detection Does Not Replace Least Privilege

The strongest solution would prevent unnecessary users from retrieving the production secret in the first place and then monitor the identities that legitimately retain access.

---

# Cleanup

After completing the validation, I removed the temporary resources used for the project:

* Deleted the `sanifeo-detect-secret-access` EventBridge rule.
* Deleted the `sanifeo-security-alerts` SNS topic and subscription.
* Deleted the dummy `sanifeo/prod/payment-api` secret.
* Deleted the `sanifeo-security-monitoring-trail` CloudTrail trail.
* Emptied and deleted the associated CloudTrail S3 bucket.
* Removed the local `event-pattern.json` troubleshooting file.
* Reviewed the AWS Billing dashboard to confirm that no unintended project resources remained.

---

# Skills Demonstrated

Through this project, I gained practical experience with:

* AWS Secrets Manager and secure credential storage;
* AWS CloudTrail auditing and event investigation;
* Amazon EventBridge event-pattern design;
* Amazon SNS security notifications;
* AWS CLI and PowerShell troubleshooting;
* identifying and resolving event-delivery failures;
* interpreting CloudTrail JSON records;
* testing detective security controls end to end;
* incident triage and evidence collection;
* secure cloud-resource cleanup.

---

# What I Would Improve for Production

This project intentionally used a small architecture suitable for demonstrating the security problem.

For a real Sanifeo production environment, I would extend the control by:

1. limiting `secretsmanager:GetSecretValue` to approved application roles;
2. reducing or eliminating routine human access to production secrets;
3. separating development and production AWS permissions;
4. defining higher-severity alerts for unexpected identities;
5. reviewing source IP information against known corporate networks;
6. implementing credential rotation where appropriate;
7. centralizing security events from multiple AWS accounts;
8. developing an incident-response procedure for unauthorized credential access.

These would be incremental improvements rather than requirements for proving the core monitoring concept.

The fundamental monitoring requirement would remain unchanged:

> **Every access to the sensitive credential should remain visible to the security team.**

---

# Key Takeaway

The biggest lesson from this project was that:

> **Protecting sensitive information requires both access control and visibility.**

Storing a credential securely is important, but security teams must also be able to identify when sensitive information is accessed and quickly determine:

```text
WHO
WHAT
WHEN
WHERE
HOW
```

For a sensitive credential, even the first access may be important.

Therefore, the detection model should not depend on repeated retrievals:

```text
Secret accessed once
        ↓
Alert generated
```

AWS CloudTrail supplied the audit evidence.

Amazon EventBridge turned each relevant event into a detection.

Amazon SNS turned the detection into an actionable notification.

Together, these services provided Sanifeo with a simple but effective security monitoring capability.
