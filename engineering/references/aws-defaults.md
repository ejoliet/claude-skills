# AWS Defaults Reference

Naming, tagging, and common resource conventions for IPAC/Caltech AWS work.

---

## Account & Region

- **Account ID**: `765894972596` (IPAC Caltech)
- **Default region**: `us-east-1`
- **EKS cluster**: target existing cluster via IRSA

---

## Naming Convention

```
{service}-{component}-{env}
```

Examples:
- `roman-file-monitor-prod`
- `ads-mcp-server-dev`
- `jenkins-cpu-alerts` (SNS topic)

---

## Required Tags (all resources)

```hcl
tags = {
  Project     = "roman" | "ads" | "jenkins" | ...
  Environment = "prod" | "dev" | "test"
  Owner       = "ejoliet"
  ManagedBy   = "terraform" | "helm" | "manual"
}
```

---

## SNS / CloudWatch Alarm Pattern

```python
import boto3

sns = boto3.client("sns", region_name="us-east-1")
cw  = boto3.client("cloudwatch", region_name="us-east-1")

# Existing SNS topic for Jenkins alerts:
JENKINS_SNS_ARN = "arn:aws:sns:us-east-1:765894972596:jenkins-cpu-alerts"

def create_cpu_alarm(instance_id: str, threshold: float = 40.0):
    cw.put_metric_alarm(
        AlarmName=f"CPUUtilization-{instance_id}",
        MetricName="CPUUtilization",
        Namespace="AWS/EC2",
        Statistic="Average",
        Period=3600,        # 1 hour
        EvaluationPeriods=168,  # 7 days
        Threshold=threshold,
        ComparisonOperator="GreaterThanThreshold",
        Dimensions=[{"Name": "InstanceId", "Value": instance_id}],
        AlarmActions=[JENKINS_SNS_ARN],
        TreatMissingData="notBreaching",
    )
```

---

## S3 Bucket Defaults

```python
s3 = boto3.client("s3")

# On creation, always apply:
s3.put_bucket_versioning(
    Bucket=bucket_name,
    VersioningConfiguration={"Status": "Enabled"}
)
s3.put_public_access_block(
    Bucket=bucket_name,
    PublicAccessBlockConfiguration={
        "BlockPublicAcls": True, "IgnorePublicAcls": True,
        "BlockPublicPolicy": True, "RestrictPublicBuckets": True,
    }
)
```

---

## IRSA (IAM Roles for Service Accounts)

```yaml
# k8s ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::765894972596:role/my-service-role
```

IAM trust policy must reference the EKS OIDC provider endpoint.

---

## CloudFormation Best Practices

```yaml
# Always use Change Sets for production updates:
# aws cloudformation create-change-set --stack-name my-stack \
#   --template-body file://template.yaml \
#   --change-set-name my-change-$(date +%Y%m%d) --capabilities CAPABILITY_IAM

# Enable termination protection on critical stacks:
# aws cloudformation update-termination-protection \
#   --enable-termination-protection --stack-name my-stack

# Minimal stack template pattern
AWSTemplateFormatVersion: "2010-09-09"
Description: "IPAC service stack"
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, test, prod]
  Owner:
    Type: String
    Default: ejoliet

Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Sub "ipac-${AWS::StackName}-${Environment}"
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        IgnorePublicAcls: true
        BlockPublicPolicy: true
        RestrictPublicBuckets: true
      Tags:
        - Key: Project
          Value: !Ref AWS::StackName
        - Key: Environment
          Value: !Ref Environment
        - Key: Owner
          Value: !Ref Owner
        - Key: ManagedBy
          Value: cloudformation

Outputs:
  BucketArn:
    Value: !GetAtt MyBucket.Arn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"
```

---

## Cloud Custodian (c7n) Patterns

Install: `pip install c7n`
Dry-run always first: `custodian run --dryrun -s out/ policy.yml`

```yaml
# Tag compliance — notify on untagged EC2
policies:
  - name: ec2-missing-required-tags
    resource: aws.ec2
    description: "Flag EC2 instances missing Owner or Project tags"
    filters:
      - or:
          - "tag:Owner": absent
          - "tag:Project": absent
    actions:
      - type: notify
        to:
          - ejoliet@ipac.caltech.edu
        subject: "EC2 missing required tags: {account} {region}"
        transport:
          type: sns
          topic: arn:aws:sns:us-east-1:765894972596:jenkins-cpu-alerts

  - name: s3-enforce-encryption
    resource: aws.s3
    description: "Auto-remediate S3 buckets without server-side encryption"
    filters:
      - type: bucket-encryption
        state: false
    actions:
      - type: set-bucket-encryption
        crypto: AES256

  - name: ec2-stop-untagged-dev
    resource: aws.ec2
    description: "Stop untagged instances in dev — run with --dryrun first"
    filters:
      - "tag:Environment": dev
      - "tag:Owner": absent
      - "instance-state-name": running
    actions:
      - stop
```

Run in CI (GitHub Actions / Jenkins):
```bash
pip install c7n
custodian validate policy.yml
custodian run --dryrun -s /tmp/out/ policy.yml
# Review output, then:
custodian run -s s3://ipac-custodian-reports/$(date +%Y%m%d)/ policy.yml
```

---

## EventBridge → SSM Run Command Pattern

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "bucket": { "name": ["my-bucket"] } }
}
```
Target: SSM `AWS-RunShellScript` with `commands` parameter.
