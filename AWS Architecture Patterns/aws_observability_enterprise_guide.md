# AWS Observability Enterprise Guide

**Document purpose:** Provide an enterprise-grade AWS observability reference covering logging, monitoring, auditing, tracing, alerting, dashboards, production implementation examples, AWS CLI steps, operational use cases, and basic-to-advanced concepts.

**Audience:** Cloud engineers, DevOps engineers, SREs, platform teams, security operations teams, cloud architects, and production support teams.

**Scope:** Amazon CloudWatch, AWS CloudTrail, AWS X-Ray, AWS Distro for OpenTelemetry, Amazon Managed Service for Prometheus, Amazon Managed Grafana, Amazon OpenSearch Service, AWS Config, EventBridge, SNS, Systems Manager, and enterprise observability operating models.

---

## 1. Executive Summary

Observability in AWS is the ability to understand the internal state of workloads, infrastructure, security activity, and user experience by collecting and correlating telemetry signals: **metrics, logs, traces, events, and audit records**.

In enterprise production environments, observability is not only about seeing CPU or memory. It supports:

- Faster incident detection and response.
- Compliance and audit readiness.
- Application performance monitoring.
- Security investigation and threat detection.
- Service-level objective tracking.
- Capacity planning and cost optimization.
- Root-cause analysis across accounts, regions, applications, and teams.

AWS provides native observability services such as **Amazon CloudWatch** for metrics, logs, alarms, and dashboards; **AWS CloudTrail** for API activity and audit history; **AWS X-Ray** and **OpenTelemetry** for distributed tracing; **Amazon Managed Service for Prometheus** for Prometheus-compatible metrics; and **Amazon Managed Grafana** for operational visualization.

---

## 2. Core Observability Concepts

### 2.1 Monitoring vs Observability

**Monitoring** tells you whether known systems are healthy.

Example:

- CPU is above 80%.
- ALB 5XX errors exceeded threshold.
- RDS free storage is low.

**Observability** helps you investigate unknown problems by correlating telemetry.

Example:

- A deployment increased checkout latency.
- A specific API route is failing for one tenant.
- An IAM role was assumed before suspicious S3 access.
- A Lambda cold start caused downstream timeout spikes.

### 2.2 The Five Enterprise Telemetry Signals

| Signal | Purpose | AWS Services |
|---|---|---|
| Metrics | Numerical time-series data | CloudWatch Metrics, Managed Prometheus |
| Logs | Detailed application and infrastructure records | CloudWatch Logs, OpenSearch, S3 |
| Traces | End-to-end request path and latency | X-Ray, OpenTelemetry, CloudWatch Application Signals |
| Events | State changes and operational events | EventBridge, CloudWatch Events |
| Audit records | Who did what, when, from where | CloudTrail, AWS Config |

### 2.3 Production Observability Goals

A production observability platform should answer:

1. Is the service available?
2. Are customers impacted?
3. What changed recently?
4. Which dependency is failing?
5. Was there unauthorized activity?
6. Is the issue isolated to an account, region, AZ, tenant, service, or deployment?
7. What action should operations take?

---

## 3. Enterprise Reference Architecture

### 3.1 Recommended Multi-Account Observability Model

For enterprise AWS environments, use a multi-account strategy:

| Account Type | Responsibility |
|---|---|
| Workload accounts | Application workloads, service logs, metrics, traces |
| Security account | GuardDuty, Security Hub, Detective, security analytics |
| Log archive account | Immutable centralized log storage |
| Observability account | Dashboards, cross-account CloudWatch, Grafana, alerting |
| Audit account | CloudTrail organization trails and compliance review |

### 3.2 High-Level Architecture

```text
Application / Infrastructure Accounts
        |
        | Metrics, Logs, Traces, Events, CloudTrail
        v
CloudWatch Logs / Metrics / X-Ray / EventBridge
        |
        | Cross-account observability / subscriptions / streams
        v
Central Observability Account
        |
        | Dashboards, Alarms, Incident Response, SRE Views
        v
CloudWatch Dashboards, Managed Grafana, SNS, PagerDuty/Opsgenie, Slack, ITSM

Security and Audit Flow
        |
        v
CloudTrail Organization Trail -> S3 Log Archive -> CloudTrail Lake / Athena / Security Lake
```

### 3.3 Enterprise Design Principles

- Enable observability by default for all production workloads.
- Centralize logs and audit records across accounts and regions.
- Separate security/audit logs from application operational logs.
- Use encryption with AWS KMS.
- Apply least-privilege IAM to telemetry readers and writers.
- Define retention policies by environment and compliance requirement.
- Use infrastructure as code for repeatable dashboards, alarms, and trails.
- Monitor the observability platform itself.
- Avoid alert fatigue by alerting on customer impact and SLO burn rate, not every noisy metric.

---

## 4. Amazon CloudWatch

### 4.1 What Is CloudWatch?

Amazon CloudWatch is the central AWS service for metrics, logs, alarms, dashboards, events, and application observability. AWS services publish many metrics automatically to CloudWatch, and applications can publish custom metrics using the CloudWatch API or OpenTelemetry Protocol.

CloudWatch supports:

- Metrics.
- Logs.
- Log Insights queries.
- Alarms.
- Dashboards.
- Synthetic canaries.
- Real User Monitoring.
- Application Signals.
- Container Insights.
- Lambda Insights.
- Contributor Insights.
- Cross-account and cross-region observability.

### 4.2 Basic Concepts

| Concept | Meaning |
|---|---|
| Namespace | Logical container for metrics, such as `AWS/EC2` or `AWS/ApplicationELB` |
| Metric | Time-series measurement, such as CPUUtilization |
| Dimension | Key-value metadata used to filter metrics |
| Datapoint | Metric value at a timestamp |
| Alarm | Threshold or anomaly-based alert |
| Log group | Logical collection of log streams |
| Log stream | Sequence of log events from a source |
| Dashboard | Visual operational view |

### 4.3 Production Example: E-Commerce Checkout Monitoring

**Scenario:** An enterprise e-commerce platform runs on ALB, ECS Fargate, RDS, ElastiCache, and Lambda.

Key CloudWatch metrics:

| Layer | Metrics |
|---|---|
| ALB | `HTTPCode_Target_5XX_Count`, `TargetResponseTime`, `RequestCount` |
| ECS | CPU, memory, running task count, restart count |
| RDS | CPU, connections, replica lag, free storage, deadlocks |
| ElastiCache | CPU, evictions, cache hit rate, memory usage |
| Lambda | Errors, duration, throttles, iterator age |
| Business | Checkout success count, payment failures, order latency |

Critical alarms:

- Checkout 5XX error rate > 2% for 5 minutes.
- Payment API latency p95 > 1.5 seconds for 10 minutes.
- RDS CPU > 85% for 15 minutes.
- ECS running tasks below desired count.
- Lambda throttles > 0 in production.

### 4.4 AWS CLI: CloudWatch Metrics

List metrics for EC2:

```bash
aws cloudwatch list-metrics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization
```

Get metric statistics:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2026-06-26T00:00:00Z \
  --end-time 2026-06-26T01:00:00Z \
  --period 300 \
  --statistics Average Maximum
```

Publish a custom business metric:

```bash
aws cloudwatch put-metric-data \
  --namespace Enterprise/ECommerce \
  --metric-data '[
    {
      "MetricName": "CheckoutSuccessCount",
      "Dimensions": [
        {"Name": "Environment", "Value": "prod"},
        {"Name": "Service", "Value": "checkout"}
      ],
      "Value": 1,
      "Unit": "Count"
    }
  ]'
```

### 4.5 AWS CLI: CloudWatch Alarms

Create a CPU alarm:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name prod-ec2-high-cpu \
  --alarm-description "EC2 CPU exceeds 80 percent for 10 minutes" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:111122223333:prod-alerts
```

Create an ALB 5XX alarm:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name prod-checkout-alb-5xx \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/prod-checkout-alb/1234567890abcdef \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 20 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:111122223333:prod-alerts
```

### 4.6 AWS CLI: CloudWatch Logs

Create a log group:

```bash
aws logs create-log-group \
  --log-group-name /enterprise/prod/checkout
```

Set retention to 90 days:

```bash
aws logs put-retention-policy \
  --log-group-name /enterprise/prod/checkout \
  --retention-in-days 90
```

Create a metric filter for application errors:

```bash
aws logs put-metric-filter \
  --log-group-name /enterprise/prod/checkout \
  --filter-name checkout-error-count \
  --filter-pattern "ERROR" \
  --metric-transformations \
      metricName=ErrorCount,metricNamespace=Enterprise/ECommerce,metricValue=1
```

Query logs with CloudWatch Logs Insights:

```bash
aws logs start-query \
  --log-group-name /enterprise/prod/checkout \
  --start-time 1782422400 \
  --end-time 1782426000 \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
```

Get query results:

```bash
aws logs get-query-results \
  --query-id QUERY_ID_FROM_PREVIOUS_COMMAND
```

### 4.7 Advanced CloudWatch Topics

#### 4.7.1 Metric Math

Metric math allows calculated metrics such as error rate:

```text
ErrorRate = 100 * 5XXErrorCount / RequestCount
```

Use it for:

- Error percentage.
- Availability percentage.
- Cost per request.
- Queue depth per worker.
- Saturation ratios.

#### 4.7.2 Anomaly Detection

CloudWatch anomaly detection learns historical metric patterns and creates dynamic expected bands. Use it for metrics with daily or weekly seasonality.

Good candidates:

- Request count.
- Login failures.
- Payment failures.
- CPU on predictable workloads.
- Data processing volume.

Avoid anomaly detection for extremely sparse or random metrics.

#### 4.7.3 Composite Alarms

Composite alarms combine multiple alarms into a higher-level signal.

Example:

```text
ALARM(ALBHigh5XX) AND ALARM(CheckoutLatencyHigh)
```

This reduces noise by alerting only when multiple symptoms indicate real customer impact.

#### 4.7.4 Cross-Account Observability

Enterprise teams should use a dedicated monitoring account to observe metrics, logs, and traces from multiple source accounts. This supports centralized dashboards and alarms without requiring engineers to switch accounts frequently.

#### 4.7.5 CloudWatch Agent

Use the CloudWatch Agent to collect:

- EC2 memory metrics.
- Disk metrics.
- Swap usage.
- Application logs.
- System logs.
- On-premises server metrics.

Install on Amazon Linux:

```bash
sudo yum install -y amazon-cloudwatch-agent
```

Start the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c ssm:AmazonCloudWatch-linux \
  -s
```

### 4.8 Industry Use Cases

| Industry | CloudWatch Use Case |
|---|---|
| Banking | Monitor transaction latency, fraud API errors, core banking integration failures |
| Retail | Track checkout conversion, inventory sync failures, payment gateway health |
| Healthcare | Monitor patient portal availability and audit integration workloads |
| Media | Monitor video encoding pipeline, CDN origin errors, live-stream latency |
| SaaS | Track tenant-specific error rates, API throttling, SLO burn rate |
| Manufacturing | Monitor IoT ingestion, machine telemetry pipelines, edge gateway health |

---

## 5. AWS CloudTrail

### 5.1 What Is CloudTrail?

AWS CloudTrail records AWS account activity. It captures actions made through the AWS Management Console, AWS CLI, SDKs, and AWS services. CloudTrail is essential for governance, compliance, security analysis, and operational troubleshooting.

CloudTrail provides:

- Event history.
- Trails.
- Organization trails.
- Management events.
- Data events.
- Insights events.
- CloudTrail Lake.

### 5.2 Basic Concepts

| Concept | Meaning |
|---|---|
| Event | Record of activity in an AWS account |
| Management event | Control-plane API activity such as creating an EC2 instance |
| Data event | Data-plane activity such as S3 object access or Lambda invoke |
| Trail | Configuration that delivers events to S3, CloudWatch Logs, or EventBridge |
| Organization trail | Trail applied across AWS Organizations accounts |
| CloudTrail Lake | Managed event data store for querying audit events |
| Insight event | Detected unusual API activity |

### 5.3 Production Example: Enterprise Audit Trail

**Scenario:** A financial services company must retain audit logs for seven years and detect unauthorized administrative activity.

Recommended setup:

- Enable an organization trail in all regions.
- Deliver logs to a centralized S3 bucket in a log archive account.
- Enable S3 Object Lock for compliance retention where required.
- Encrypt logs with KMS.
- Enable log file validation.
- Send critical CloudTrail events to CloudWatch Logs and EventBridge.
- Create alerts for root user activity, IAM policy changes, CloudTrail changes, KMS key deletion, and S3 bucket policy changes.
- Query long-term activity using CloudTrail Lake or Athena.

### 5.4 AWS CLI: Create a CloudTrail Trail

Create an S3 bucket for CloudTrail logs:

```bash
aws s3api create-bucket \
  --bucket enterprise-cloudtrail-logs-111122223333 \
  --region us-east-1
```

Create the trail:

```bash
aws cloudtrail create-trail \
  --name enterprise-org-trail \
  --s3-bucket-name enterprise-cloudtrail-logs-111122223333 \
  --is-multi-region-trail \
  --enable-log-file-validation
```

Start logging:

```bash
aws cloudtrail start-logging \
  --name enterprise-org-trail
```

Check trail status:

```bash
aws cloudtrail get-trail-status \
  --name enterprise-org-trail
```

### 5.5 AWS CLI: CloudTrail Event Lookup

Look up recent events:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 10
```

Look up events by username:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=admin-user \
  --max-results 10
```

### 5.6 AWS CLI: Enable CloudTrail Insights

```bash
aws cloudtrail put-insight-selectors \
  --trail-name enterprise-org-trail \
  --insight-selectors InsightType=ApiCallRateInsight
```

### 5.7 AWS CLI: Advanced Event Selectors for S3 Data Events

```bash
aws cloudtrail put-event-selectors \
  --trail-name enterprise-org-trail \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3::Object",
          "Values": ["arn:aws:s3:::prod-sensitive-data/"]
        }
      ]
    }
  ]'
```

### 5.8 Important CloudTrail Alerts

Create EventBridge rules for:

- Root account usage.
- Console login without MFA.
- IAM user creation.
- IAM policy attachment.
- Security group opened to `0.0.0.0/0`.
- CloudTrail stopped or deleted.
- KMS key disabled or scheduled for deletion.
- S3 bucket policy changed.
- RDS snapshot shared publicly.

Example EventBridge pattern for root usage:

```json
{
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "userIdentity": {
      "type": ["Root"]
    }
  }
}
```

### 5.9 Advanced CloudTrail Topics

#### 5.9.1 Organization Trails

Use organization trails to apply consistent audit logging across all AWS accounts in AWS Organizations. This avoids gaps when new accounts are created.

#### 5.9.2 CloudTrail Lake

CloudTrail Lake supports SQL-like querying of audit events. Use it when security, audit, or compliance teams need flexible queries across large event volumes.

Example investigations:

- Who changed a production IAM role?
- Which principal accessed a sensitive S3 object?
- Which APIs were called before a production outage?
- Which regions had administrative activity?

#### 5.9.3 CloudTrail Data Events

Data events are high-volume and can increase cost. Enable selectively for sensitive resources:

- Critical S3 buckets.
- Lambda functions processing regulated data.
- DynamoDB tables with sensitive records.

#### 5.9.4 CloudTrail and Incident Response

During an incident, CloudTrail helps answer:

- Which principal performed the action?
- Was MFA used?
- What source IP was used?
- Which role was assumed?
- Which resources were modified?
- Was the same identity active in multiple regions?

### 5.10 Industry Use Cases

| Industry | CloudTrail Use Case |
|---|---|
| Banking | Audit privileged access and IAM changes |
| Healthcare | Track access to regulated workloads and sensitive S3 buckets |
| Retail | Investigate production configuration changes before outages |
| SaaS | Prove tenant data access controls during customer audits |
| Government | Retain immutable evidence of administrative activity |
| Education | Monitor account misuse and unauthorized resource creation |

---

## 6. AWS X-Ray and Distributed Tracing

### 6.1 What Is X-Ray?

AWS X-Ray helps analyze and debug distributed applications by tracing requests across services. It shows service maps, latency, errors, throttles, and downstream dependency behavior.

X-Ray is useful for microservices, serverless applications, APIs, and event-driven systems.

### 6.2 Basic Concepts

| Concept | Meaning |
|---|---|
| Trace | End-to-end request journey |
| Segment | Work done by one service |
| Subsegment | Downstream call within a segment |
| Annotation | Indexed metadata used for filtering traces |
| Metadata | Non-indexed debugging details |
| Service map | Visual dependency map |

### 6.3 Production Example: API Latency Investigation

**Scenario:** Customers report slow checkout API responses.

Trace analysis shows:

```text
API Gateway -> Lambda checkout-service -> DynamoDB -> Payment Provider API
```

Findings:

- API Gateway latency is normal.
- Lambda duration p95 increased from 300 ms to 2.5 seconds.
- DynamoDB latency is normal.
- Payment provider API subsegment is slow.

Outcome:

- Incident is isolated to external payment provider.
- Circuit breaker threshold is reduced.
- Alert is added for external API latency.
- Fallback payment queue is activated.

### 6.4 AWS CLI: X-Ray Commands

Get trace summaries:

```bash
aws xray get-trace-summaries \
  --start-time 2026-06-26T00:00:00Z \
  --end-time 2026-06-26T01:00:00Z
```

Get full trace details:

```bash
aws xray batch-get-traces \
  --trace-ids 1-abcdef12-345678901234567890abcdef
```

Get service graph:

```bash
aws xray get-service-graph \
  --start-time 2026-06-26T00:00:00Z \
  --end-time 2026-06-26T01:00:00Z
```

### 6.5 Advanced X-Ray and OpenTelemetry

AWS is moving toward OpenTelemetry as the primary instrumentation standard for tracing and observability. For new enterprise workloads, prefer OpenTelemetry instrumentation through AWS Distro for OpenTelemetry where possible.

Benefits:

- Vendor-neutral instrumentation.
- Portable telemetry pipeline.
- Single instrumentation path for metrics and traces.
- Integration with CloudWatch, X-Ray, OpenSearch, Managed Prometheus, and third-party platforms.

### 6.6 AWS Distro for OpenTelemetry Collector Example

Example collector configuration:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:

exporters:
  awsxray:
  awsemf:
    namespace: Enterprise/ECommerce

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsemf]
```

### 6.7 Industry Use Cases

| Industry | Tracing Use Case |
|---|---|
| Retail | Identify checkout latency across API, payment, inventory, and shipping services |
| Banking | Trace loan application workflow across internal and external APIs |
| Healthcare | Debug patient portal delays across authentication and records systems |
| SaaS | Trace tenant-specific performance issues |
| Media | Analyze streaming entitlement and recommendation service latency |

---

## 7. Amazon Managed Service for Prometheus

### 7.1 What Is Amazon Managed Service for Prometheus?

Amazon Managed Service for Prometheus is a managed Prometheus-compatible monitoring service designed for containers and dynamic environments such as Amazon EKS. It supports PromQL and remote write from Prometheus agents.

### 7.2 When to Use CloudWatch vs Prometheus

| Requirement | Recommended Service |
|---|---|
| AWS service metrics | CloudWatch |
| EC2/Linux host metrics | CloudWatch Agent or Prometheus node exporter |
| Kubernetes pod/container metrics | Managed Prometheus |
| PromQL dashboards | Managed Prometheus + Grafana |
| Native AWS alarms | CloudWatch |
| SRE Kubernetes platform monitoring | Managed Prometheus |

### 7.3 AWS CLI: Create a Prometheus Workspace

```bash
aws amp create-workspace \
  --alias prod-eks-observability
```

List workspaces:

```bash
aws amp list-workspaces
```

Describe workspace:

```bash
aws amp describe-workspace \
  --workspace-id ws-12345678-1234-1234-1234-123456789012
```

### 7.4 Production Example: EKS Platform Monitoring

Collect metrics from:

- Kubernetes API server.
- Nodes.
- Pods.
- Deployments.
- Ingress controllers.
- Service mesh.
- Application containers.

Example PromQL alerts:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m])) > 0.02
```

```promql
kube_deployment_status_replicas_available
<
kube_deployment_spec_replicas
```

```promql
container_memory_working_set_bytes
/
container_spec_memory_limit_bytes > 0.9
```

### 7.5 Advanced Prometheus Topics

- Use recording rules for expensive queries.
- Keep high-cardinality labels under control.
- Avoid labels such as request ID, user ID, or session ID.
- Use service, environment, cluster, namespace, and workload labels consistently.
- Separate platform metrics from application metrics.
- Apply retention and cost controls.

### 7.6 Industry Use Cases

| Industry | Managed Prometheus Use Case |
|---|---|
| SaaS | Kubernetes platform SLO monitoring |
| FinTech | Microservice error budget tracking |
| Gaming | Real-time backend fleet monitoring |
| Retail | Containerized inventory and checkout metrics |
| Telecom | Large-scale network control-plane monitoring |

---

## 8. Amazon Managed Grafana

### 8.1 What Is Amazon Managed Grafana?

Amazon Managed Grafana is a managed visualization service that integrates with AWS and third-party data sources, including CloudWatch, X-Ray, OpenSearch, Timestream, and Amazon Managed Service for Prometheus.

### 8.2 Dashboard Strategy

Create dashboards for different personas:

| Dashboard | Audience | Purpose |
|---|---|---|
| Executive availability | Leadership | SLA/SLO and business KPIs |
| NOC overview | Operations | Current production health |
| Service dashboard | Engineering | App metrics, logs, traces |
| Security dashboard | SecOps | CloudTrail, GuardDuty, IAM events |
| Cost observability | FinOps | Cost, usage, anomaly signals |

### 8.3 Production Dashboard Example

For a production checkout service:

- Request rate.
- Error rate.
- Latency p50, p90, p95, p99.
- Deployment version.
- ECS task count.
- RDS CPU and connections.
- Payment provider latency.
- Checkout success count.
- Revenue per minute.
- Active incidents.

### 8.4 Advanced Dashboard Practices

- Use variables for account, region, environment, service, and cluster.
- Keep dashboards simple and actionable.
- Put customer-impacting metrics at the top.
- Link dashboards to runbooks.
- Show deployment markers.
- Separate symptoms from diagnostic panels.
- Avoid dashboards with hundreds of panels.

---

## 9. AWS Config for Compliance Observability

### 9.1 What Is AWS Config?

AWS Config records configuration changes and evaluates resources against rules. It complements CloudTrail by showing the state and configuration history of AWS resources.

CloudTrail answers:

```text
Who changed it?
```

AWS Config answers:

```text
What changed, and is it compliant?
```

### 9.2 Production Use Cases

- Detect public S3 buckets.
- Detect unencrypted EBS volumes.
- Detect security groups open to the internet.
- Track IAM policy drift.
- Validate required tags.
- Prove compliance posture to auditors.

### 9.3 AWS CLI: Config Recorder

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder name=enterprise-config-recorder,roleARN=arn:aws:iam::111122223333:role/AWSConfigRole
```

```bash
aws configservice start-configuration-recorder \
  --configuration-recorder-name enterprise-config-recorder
```

---

## 10. EventBridge for Operational Event Routing

### 10.1 What Is EventBridge?

Amazon EventBridge routes events from AWS services, SaaS providers, and custom applications. In observability, it is used to trigger automation when operational or security events occur.

### 10.2 Production Examples

- Send CloudTrail root login events to Slack and PagerDuty.
- Trigger Lambda remediation when a security group is opened to the internet.
- Notify operations when ECS deployment fails.
- Route GuardDuty high-severity findings to incident management.
- Start Systems Manager Automation documents for known remediation steps.

### 10.3 AWS CLI: EventBridge Rule for EC2 State Change

```bash
aws events put-rule \
  --name prod-ec2-state-change \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"]
  }'
```

Add SNS target:

```bash
aws events put-targets \
  --rule prod-ec2-state-change \
  --targets "Id"="1","Arn"="arn:aws:sns:us-east-1:111122223333:prod-alerts"
```

---

## 11. Alerting and Incident Response

### 11.1 Alert Design Principles

Alert only when action is required.

A good production alert has:

- Clear service name.
- Environment.
- Customer impact.
- Severity.
- Runbook link.
- Dashboard link.
- Recent change link.
- Escalation owner.

### 11.2 Severity Model

| Severity | Meaning | Example |
|---|---|---|
| SEV1 | Major customer impact | Checkout unavailable globally |
| SEV2 | Significant degradation | Payment latency high in one region |
| SEV3 | Limited impact | One background job failing |
| SEV4 | Low urgency | Non-critical warning or capacity trend |

### 11.3 Recommended Alarm Types

| Alarm Type | Example |
|---|---|
| Availability | Health check failure, 5XX spike |
| Latency | p95 latency above SLO |
| Error | Application error rate > threshold |
| Saturation | CPU, memory, disk, queue depth |
| Security | Root login, IAM policy change |
| Cost | Unusual spend or usage spike |
| Deployment | Failed deployment or rollback |

### 11.4 SLO-Based Alerting

Instead of alerting on every low-level signal, use service-level objectives.

Example SLO:

```text
99.9% of checkout requests complete successfully within 1 second over 30 days.
```

Useful indicators:

- Availability SLI.
- Latency SLI.
- Error budget burn rate.
- Customer journey success rate.

---

## 12. Log Strategy

### 12.1 Logging Standards

Use structured JSON logs.

Example:

```json
{
  "timestamp": "2026-06-26T10:15:30Z",
  "level": "ERROR",
  "service": "checkout",
  "environment": "prod",
  "region": "us-east-1",
  "correlation_id": "abc-123",
  "tenant_id": "tenant-001",
  "order_id": "ord-789",
  "message": "Payment authorization failed",
  "error_code": "PAYMENT_TIMEOUT",
  "latency_ms": 1840
}
```

### 12.2 Required Log Fields

| Field | Purpose |
|---|---|
| timestamp | Event time |
| level | DEBUG, INFO, WARN, ERROR |
| service | Service name |
| environment | dev, test, stage, prod |
| correlation_id | Request tracing |
| tenant_id | SaaS tenant investigation |
| user_id_hash | User troubleshooting without exposing PII |
| error_code | Grouping errors |
| latency_ms | Performance analysis |

### 12.3 Sensitive Data Rules

Never log:

- Passwords.
- API keys.
- Access tokens.
- Credit card numbers.
- Full personal health information.
- Full bank account numbers.
- Unmasked personally identifiable information unless explicitly approved.

### 12.4 Retention Example

| Log Type | Retention |
|---|---|
| Debug logs | 7-14 days |
| Application prod logs | 30-90 days |
| Security logs | 1-7 years depending on compliance |
| CloudTrail audit logs | 1-7 years depending on compliance |
| Metrics | Based on operational and reporting needs |

---

## 13. Enterprise Production Implementation Blueprint

### 13.1 Baseline Services to Enable

For every production AWS account:

- CloudTrail organization trail.
- CloudWatch metrics and logs.
- AWS Config recorder.
- GuardDuty.
- Security Hub.
- CloudWatch alarms for critical infrastructure.
- Centralized log forwarding.
- KMS encryption.
- S3 log archive.
- EventBridge rules for security-sensitive activity.

### 13.2 Production Naming Standards

Use consistent naming:

```text
<environment>-<application>-<component>-<signal>
```

Examples:

```text
prod-checkout-api-latency-p95
prod-checkout-alb-5xx-rate
prod-payments-lambda-errors
prod-corebank-rds-cpu-high
prod-security-root-login
```

### 13.3 Tagging Standards

Required tags:

| Tag | Example |
|---|---|
| Environment | prod |
| Application | checkout |
| ServiceOwner | sre-team |
| CostCenter | cc-1001 |
| DataClassification | confidential |
| Compliance | pci |
| Criticality | tier-1 |

### 13.4 Production Dashboard Layout

Recommended dashboard sections:

1. Customer impact.
2. Availability and latency.
3. Error rate.
4. Traffic and saturation.
5. Dependency health.
6. Infrastructure health.
7. Recent deployments.
8. Security and audit events.
9. Runbook links.

---

## 14. End-to-End Production Example

### 14.1 Workload

A payment-processing platform:

```text
Route 53 -> CloudFront -> WAF -> ALB -> ECS Payment API -> RDS PostgreSQL
                                      -> Lambda Fraud Check
                                      -> External Bank API
```

### 14.2 Observability Controls

| Component | Observability |
|---|---|
| Route 53 | Health checks |
| CloudFront | Access logs, error rate, cache hit ratio |
| WAF | Blocked requests, sampled requests |
| ALB | Request count, 4XX, 5XX, target latency |
| ECS | CPU, memory, task health, container logs |
| Lambda | Errors, throttles, duration, cold starts |
| RDS | CPU, connections, locks, slow queries |
| External API | Custom latency and error metrics |
| Security | CloudTrail, GuardDuty, Config |

### 14.3 Example Incident Flow

1. CloudWatch alarm fires: `prod-payment-api-5xx-rate-high`.
2. SNS sends alert to incident management.
3. On-call opens Grafana payment dashboard.
4. Error rate panel shows 5XX increase after deployment.
5. X-Ray trace shows timeout in external bank API.
6. CloudTrail confirms no infra change except ECS deployment.
7. Runbook instructs rollback to previous task definition.
8. ECS deployment rollback completes.
9. Error rate returns to normal.
10. Post-incident review adds synthetic canary for bank API.

### 14.4 Example CloudWatch Logs Insights Queries

Find top errors:

```sql
fields @timestamp, level, service, error_code, message
| filter level = "ERROR"
| stats count(*) as errors by error_code, service
| sort errors desc
| limit 20
```

Find slow API requests:

```sql
fields @timestamp, service, path, latency_ms, correlation_id
| filter latency_ms > 1000
| sort latency_ms desc
| limit 50
```

Find tenant-specific issue:

```sql
fields @timestamp, tenant_id, service, error_code, message
| filter tenant_id = "tenant-001"
| sort @timestamp desc
| limit 100
```

---

## 15. Security Observability

### 15.1 Key Signals

Monitor:

- Root account usage.
- IAM changes.
- Failed console logins.
- Console login without MFA.
- Security group changes.
- S3 public access changes.
- KMS key changes.
- CloudTrail stopped.
- GuardDuty findings.
- AWS Config noncompliance.

### 15.2 Example Security Alarms

| Event | Action |
|---|---|
| Root login | Immediate SEV2 security alert |
| CloudTrail stopped | Immediate SEV1 security alert |
| Public S3 bucket policy | Auto-remediate or quarantine |
| KMS key deletion scheduled | Security approval required |
| IAM admin policy attached | Notify SecOps |

### 15.3 Incident Investigation Checklist

- Identify principal ARN.
- Check source IP and geolocation.
- Confirm MFA status.
- Check role assumption chain.
- Review CloudTrail before and after event.
- Check related AWS Config changes.
- Check GuardDuty and Security Hub findings.
- Preserve logs in immutable storage.

---

## 16. Cost Observability

### 16.1 Observability Cost Drivers

Costs can increase due to:

- High CloudWatch custom metric cardinality.
- Excessive log ingestion.
- Long log retention.
- High-volume CloudTrail data events.
- Prometheus high-cardinality labels.
- Frequent dashboard queries.
- Large OpenSearch clusters.

### 16.2 Cost Optimization Practices

- Use log sampling for high-volume debug logs.
- Set retention policies on every log group.
- Avoid custom metrics with unbounded dimensions.
- Use metric filters carefully.
- Enable CloudTrail data events only for critical resources.
- Archive long-term logs to S3.
- Use different retention by environment.
- Review unused dashboards and alarms.

---

## 17. Maturity Model

| Level | Capabilities |
|---|---|
| Level 1 - Basic | CloudWatch metrics, basic logs, manual dashboards |
| Level 2 - Standard | Alarms, structured logs, CloudTrail, Config |
| Level 3 - Enterprise | Centralized multi-account observability, runbooks, SLOs |
| Level 4 - Advanced | Distributed tracing, anomaly detection, automation |
| Level 5 - Proactive | Predictive alerts, auto-remediation, error budget governance |

---

## 18. Implementation Roadmap

### Phase 1: Foundation

- Enable CloudTrail organization trail.
- Enable AWS Config.
- Define logging and tagging standards.
- Create central log archive account.
- Create production SNS topics.
- Set baseline CloudWatch alarms.

### Phase 2: Application Observability

- Add structured JSON logs.
- Add custom business metrics.
- Add service dashboards.
- Add CloudWatch Logs Insights saved queries.
- Add synthetic checks for critical endpoints.

### Phase 3: Distributed Tracing

- Instrument services with OpenTelemetry.
- Send traces to X-Ray.
- Add correlation IDs across services.
- Add trace IDs to logs.

### Phase 4: Platform Observability

- Add Managed Prometheus for EKS.
- Add Managed Grafana dashboards.
- Add cross-account observability.
- Build executive and SRE dashboards.

### Phase 5: Automation and Governance

- Add EventBridge remediation workflows.
- Add SLO-based alerts.
- Add cost observability.
- Add regular observability reviews.
- Add audit-ready evidence reports.

---

## 19. Common Mistakes to Avoid

- Alerting on noisy metrics without customer impact.
- Not setting log retention policies.
- Logging sensitive data.
- Creating too many high-cardinality custom metrics.
- Not enabling CloudTrail in all regions.
- Not centralizing logs across accounts.
- Building dashboards without runbooks.
- Ignoring business metrics.
- Using traces without correlation IDs.
- Not monitoring third-party dependencies.

---

## 20. Enterprise Checklist

### CloudWatch

- [ ] Critical AWS service metrics monitored.
- [ ] Custom application metrics published.
- [ ] Production alarms configured.
- [ ] Alarm actions routed to incident management.
- [ ] Log groups have retention policies.
- [ ] Dashboards exist for tier-1 services.
- [ ] Logs use JSON structure.
- [ ] CloudWatch Agent installed where needed.

### CloudTrail

- [ ] Organization trail enabled.
- [ ] Multi-region logging enabled.
- [ ] Log file validation enabled.
- [ ] Logs encrypted with KMS.
- [ ] Logs delivered to central S3 bucket.
- [ ] CloudTrail stop/delete alerts configured.
- [ ] Sensitive S3 data events enabled where required.
- [ ] CloudTrail Lake or Athena query process defined.

### Tracing

- [ ] OpenTelemetry standard selected.
- [ ] Critical services instrumented.
- [ ] Trace IDs added to logs.
- [ ] External dependencies traced.
- [ ] Sampling strategy defined.

### Dashboards and Operations

- [ ] Service dashboards created.
- [ ] Executive dashboards created.
- [ ] Runbooks linked.
- [ ] On-call escalation configured.
- [ ] Post-incident dashboard improvements tracked.

### Security

- [ ] Root usage monitored.
- [ ] IAM changes monitored.
- [ ] Public access changes monitored.
- [ ] GuardDuty and Security Hub integrated.
- [ ] Security event workflows tested.

---

## 21. Appendix: Useful AWS CLI Commands

Check current AWS identity:

```bash
aws sts get-caller-identity
```

List CloudWatch alarms:

```bash
aws cloudwatch describe-alarms
```

List log groups:

```bash
aws logs describe-log-groups
```

Describe a log group:

```bash
aws logs describe-log-groups \
  --log-group-name-prefix /enterprise/prod
```

List CloudTrail trails:

```bash
aws cloudtrail describe-trails
```

Validate CloudTrail logging status:

```bash
aws cloudtrail get-trail-status \
  --name enterprise-org-trail
```

List X-Ray groups:

```bash
aws xray get-groups
```

Create SNS topic for alerts:

```bash
aws sns create-topic \
  --name prod-alerts
```

Subscribe email to SNS topic:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:111122223333:prod-alerts \
  --protocol email \
  --notification-endpoint sre-team@example.com
```

---

## 22. References

The following AWS documentation sources were used to align this guide with current AWS service behavior:

1. Amazon CloudWatch overview and capabilities: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
2. CloudWatch metrics: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html
3. CloudWatch dashboards and cross-account observability: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Dashboards.html
4. CloudWatch Logs documentation: https://docs.aws.amazon.com/cloudwatch/
5. AWS CloudTrail overview: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
6. CloudTrail concepts: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-concepts.html
7. CloudTrail event history: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html
8. CloudTrail Lake: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake.html
9. CloudTrail organization event data stores: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-organizations.html
10. AWS X-Ray overview: https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html
11. AWS Distro for OpenTelemetry and X-Ray: https://docs.aws.amazon.com/xray/latest/devguide/xray-services-adot.html
12. OpenTelemetry OTLP endpoint for X-Ray and CloudWatch Application Signals: https://docs.aws.amazon.com/xray/latest/devguide/xray-opentelemetry.html
13. Amazon Managed Service for Prometheus documentation: https://docs.aws.amazon.com/prometheus/
14. AWS Observability Accelerator for EKS: https://docs.aws.amazon.com/prometheus/latest/userguide/obs_accelerator.html
15. Amazon Managed Grafana overview: https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html

---

## 23. Final Recommendation

For enterprise production, implement AWS observability as a platform capability rather than a per-application afterthought. Start with CloudTrail, CloudWatch, AWS Config, centralized logs, and critical alerts. Then mature toward OpenTelemetry, distributed tracing, SLOs, Managed Prometheus, Managed Grafana, automated remediation, and cross-account operational visibility.

A strong enterprise observability platform should help teams detect incidents quickly, understand customer impact, prove compliance, reduce mean time to recovery, and continuously improve production reliability.
