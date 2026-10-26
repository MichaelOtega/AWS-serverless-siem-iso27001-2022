# Serverless AWS SIEM Pipeline (ISO/IEC 27001:2022 Compliant)

## Overview
This repository documents the architecture and deployment of a zero-spend, production-grade Security Information and Event Management (SIEM) pipeline in AWS. Designed with strict adherence to **ISO/IEC 27001:2022** compliance standards, this project captures AWS infrastructure events, routes them through a secure, restricted storage vault, and leverages a serverless architecture to deliver real-time threat detection and alerting via Slack.

## Tool Integration & Tech Stack
* **Telemetry & Auditing:** AWS CloudTrail
* **Immutable Storage:** Amazon S3
* **Serverless Compute:** AWS Lambda (Python)
* **SIEM & Analytics:** Datadog
* **Incident Response:** Slack API / Webhooks

## Architecture Flow
The pipeline operates on a serverless, event-driven model in `eu-north-1` to eliminate persistent infrastructure costs while ensuring high availability and auditability:

```text
[AWS Control Plane APIs]
          │
          ▼ (1. Captures Management & Auth Events)
   [AWS CloudTrail]
          │
          ▼ (2. Delivers Batched JSON Audit Logs)
 [Amazon S3 Audit Vault] ──(Enforces PoLP & Confused Deputy Protection)
          │
          ▼ (3. S3 ObjectCreated Event Trigger)
[AWS Lambda: Datadog Forwarder]
          │
          ▼ (4. TLS-Encrypted Log Forwarding)
  [Datadog Log Explorer] ──(5. Evaluates Query: source:cloudtrail @evt.name:ConsoleLogin)
          │
          ▼ (6. Webhook Trigger on Threshold > 0 in 5m)
 [Slack SOC Channel: #aws-alerts]




1. Telemetry Generation (AWS CloudTrail)
CloudTrail continuously monitors the AWS tenant, capturing all control-plane API calls, management events, and IAM authentication attempts.

2. Immutable Audit Vault & 365-Day Retention (Amazon S3)
Logs are batched into compressed JSON files and delivered to a dedicated S3 bucket. The bucket is hardened using a strict resource-based policy utilizing the aws:SourceArn condition key to prevent Confused Deputy attacks. To satisfy strict audit and forensic requirements, an S3 Lifecycle Rule automatically enforces 365-day log retention.

3. Event-Driven Log Forwarding (AWS Lambda)
To achieve zero-spend FinOps objectives, the pipeline relies on AWS Lambda rather than persistent virtual machines. The exact moment a new log file lands in the S3 vault, an ObjectCreated event triggers the Lambda function. It wakes up for milliseconds to decompress, parse, and securely forward the logs, then spins down immediately.

4. Continuous Monitoring (Datadog)
Parsed logs are ingested into Datadog where continuous monitoring rules evaluate the telemetry in real-time. Custom Detection-as-Code logic (logs("source:cloudtrail @evt.name:ConsoleLogin").index("*").rollup("count").last("5m") > 0) is utilized to identify unauthorized or anomalous access within rolling 5-minute windows.

5. Automated Incident Response (Slack)
Upon detecting a threshold breach, Datadog's alerting engine dispatches a high-priority webhook payload directly to a dedicated #aws-alerts Slack channel, equipping the Security Operations Center (SOC) with immediate context for triage and reducing Mean Time to Detect (MTTD).

ISO/IEC 27001:2022 Compliance Mapping
This architecture was purpose-built to satisfy specific Annex A controls for cloud security and continuous monitoring:

Control A.5.23 (Information security for use of cloud services): Enforces strict governance over cloud logging and telemetry routing.

Control A.8.15 (Logging): Guarantees the continuous generation of system audit logs, tamper protection via S3 PoLP, and forensic availability through a strictly enforced 365-day lifecycle retention policy.

Control A.8.16 (Monitoring activities): Implements Datadog for the continuous monitoring of anomalous activities and unauthorized AWS environment access.

Controls A.5.25 & A.5.26 (Assessment and decision on information security events / Response to information security incidents): Automates the classification of specific API events and orchestrates immediate incident notification via Slack webhooks.

Repository Contents
/policies/: Contains the sanitized JSON resource policies utilized to harden the Amazon S3 audit vault.

/detections/: Contains the exported Datadog JSON rule definitions for Detection-as-Code deployment.

/assets/: Contains visual proof-of-concept evidence detailing successful log ingestion, detection triggers, Slack integration, and automated 365-day retention lifecycle rules.
