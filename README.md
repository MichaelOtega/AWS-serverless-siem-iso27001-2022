# Serverless AWS SIEM Pipeline (ISO/IEC 27001:2022 Compliant)

## Overview
This repository documents the architecture and deployment of a zero-spend, production-grade Security Information and Event Management (SIEM) pipeline in AWS. Designed with strict adherence to **ISO/IEC 27001:2022** compliance standards, this project captures AWS infrastructure events, routes them through a secure, restricted storage vault, and leverages a serverless architecture to deliver real-time threat detection and alerting via Slack.

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
