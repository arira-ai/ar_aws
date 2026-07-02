Here is **Part 3**, the final phase of our enterprise-grade architecture. This section covers operationalizing the deployment, ensuring production readiness, estimating costs, and wrapping up the project documentation.

---

### 1. Deployment Steps

Follow these steps to deploy the infrastructure and code to your AWS account.

**Prerequisites:**

* AWS CLI installed and configured with Administrator privileges.
* Python 3.12 installed.
* Ensure you are deploying the CloudFormation stack in `us-east-1` if you want CloudFront to automatically attach the ACM certificate.

**Step 1: Package the Lambda Function**

```bash
mkdir package
pip install -r src/lambda/image_processor/requirements.txt -t package/
cp src/lambda/image_processor/app.py package/
cd package
zip -r ../lambda-code.zip .
cd ..
# Upload to a deployment artifacts S3 bucket (create one if you don't have it)
aws s3 cp lambda-code.zip s3://your-deployment-artifacts-bucket/lambda-code.zip

```

**Step 2: Deploy the CloudFormation Stack**

```bash
aws cloudformation deploy \
  --template-file infrastructure/main-template.yaml \
  --stack-name prod-image-platform \
  --parameter-overrides ProjectName=imageplatform Environment=prod DomainName=images.example.com HostedZoneId=Z1234567890 \
  --capabilities CAPABILITY_NAMED_IAM

```

**Step 3: Deploy Frontend Assets**

```bash
# Sync your compiled static website (HTML/JS/CSS) to the Website Bucket
aws s3 sync ./frontend/ s3://imageplatform-prod-website-us-east-1/ --delete

# Invalidate the CloudFront Cache to serve new content
aws cloudfront create-invalidation \
  --distribution-id <YOUR_DISTRIBUTION_ID> \
  --paths "/*"

```

### 2. Testing Procedures

* **Unit Testing:** Run `pytest` locally on the Lambda `app.py` using mock `boto3` objects (via `moto`).
* **Integration Testing:** 1. Upload a test image to the Upload Bucket using the AWS CLI.
2. Check the Website Bucket `uploads/` and `thumbnails/` prefixes to ensure the processed images exist.
* **End-to-End Testing:** 1. Navigate to `https://images.example.com`.
2. Perform an upload via the UI.
3. Verify the image and thumbnail render correctly on the website without broken links.
* **Failure Testing:** Upload an unsupported file type (e.g., `.txt` or `.pdf`) and verify CloudWatch Logs confirm graceful rejection without crashing the Lambda.

### 3. Rollback Plan

If a deployment causes a critical failure:

1. **Code Rollback:** Re-upload the previous stable version of `lambda-code.zip` to your artifacts bucket and update the Lambda function via AWS CLI.
2. **Frontend Rollback:** Sync the previous git commit of the frontend directory to the Website Bucket and invalidate the CloudFront cache.
3. **Infrastructure Rollback:** Run an AWS CloudFormation rollback or deploy the previous version of the `main-template.yaml`. Note: S3 buckets containing data will prevent automatic stack deletion.

### 4. Monitoring Setup

To achieve Operational Excellence, the following monitoring tools must be deployed (these can be added as CloudFormation resources):

* **CloudWatch Dashboards:** A centralized dashboard displaying:
* Lambda Invocations, Duration, and Errors.
* CloudFront 4XX/5XX Error Rates.
* S3 Bucket Size and Object Counts.


* **CloudWatch Alarms:**
* `LambdaErrorAlarm`: Triggers if Lambda Errors > 0 for 5 consecutive minutes.
* `CloudFront5xxAlarm`: Triggers if 5XX error rate > 1% over 5 minutes.


* **SNS Notifications:** An SNS Topic (`OperationsAlertsTopic`) subscribed to by the DevOps team's email or PagerDuty. The CloudWatch Alarms publish to this topic.

### 5. Security Review

* **Data Isolation:** Raw uploads and public-facing assets are physically separated into two different S3 buckets.
* **Least Privilege:** The Lambda function can *only* read from the upload bucket and write to the website bucket. It cannot delete resources.
* **Edge Protection:** CloudFront restricts access to S3 via Origin Access Control (OAC). S3 Public Access is completely blocked.
* **Encryption:** Data at rest is encrypted via SSE-KMS. Data in transit is enforced via TLS 1.2+ minimum policies on CloudFront.

### 6. AWS Well-Architected Review

* **Operational Excellence:** Infrastructure as Code (CloudFormation), structured logging (Powertools), automated CI/CD capabilities.
* **Security:** IAM least privilege, KMS encryption, OAC, AWS WAF.
* **Reliability:** S3 and CloudFront are highly available by default. Lambda scales automatically to handle burst uploads.
* **Performance Efficiency:** Edge caching via CloudFront, Brotli compression, Lambda memory tuned to 1024MB for fast Pillow image processing.
* **Cost Optimization:** Serverless pay-per-use model, S3 Lifecycle rules delete raw uploads, CloudFront reduces S3 GET requests.
* **Sustainability:** Serverless compute minimizes idle resource energy consumption. Intelligent Tiering reduces physical storage overhead over time.

### 7. Cost Estimation

*Assumptions: 100 GB storage, 10,000 image uploads/month, 500,000 website visits/month (avg 2MB per visit = 1 TB Outbound Data Transfer). US-East-1 region.*

| Service | Metric | Estimated Monthly Cost |
| --- | --- | --- |
| **Amazon S3** | 100 GB Standard Storage | ~$2.30 |
| **AWS Lambda** | 10,000 requests, 1024MB, 500ms avg | ~$0.08 |
| **CloudFront** | 1 TB Data Transfer Out | ~$85.00 |
| **Route 53** | 1 Hosted Zone + DNS Queries | ~$0.50 |
| **AWS KMS & CW** | Standard Key + Custom Metrics | ~$2.00 |
| **Total Estimated Cost** |  | **~$89.88 / month** |

### 8. Cleanup Guide

To tear down the environment and stop incurring costs:

1. Empty both S3 buckets manually or via CLI (CloudFormation will not delete non-empty buckets):
* `aws s3 rm s3://imageplatform-prod-upload-us-east-1 --recursive`
* `aws s3 rm s3://imageplatform-prod-website-us-east-1 --recursive`


2. Delete the CloudFormation stack:
* `aws cloudformation delete-stack --stack-name prod-image-platform`



### 9. Future Enhancements

* **Amazon DynamoDB:** Introduce a table to store rich metadata (uploader ID, tags, AI-generated descriptions via Amazon Rekognition).
* **AWS Step Functions:** If the processing pipeline grows (e.g., watermarking, multi-resolution generation, AI moderation), replace the direct Lambda trigger with a Step Functions state machine.
* **Presigned URLs:** Implement an API Gateway + Lambda pattern to securely generate S3 Presigned Upload URLs, removing the need for AWS credentials in the frontend.

### 10. README.md

```markdown
# Enterprise AWS Static Website & Image Processing Pipeline

## Overview
A Well-Architected, fully serverless platform for hosting static websites (SPAs) with an automated, idempotent image processing pipeline using AWS Lambda, S3, and CloudFront.

## Architecture Highlights
* **Edge Routing & Caching:** Amazon Route 53 + Amazon CloudFront.
* **Storage:** Private S3 Buckets encrypted with KMS.
* **Compute:** AWS Lambda triggered by S3 Event Notifications.
* **Security:** Least privilege IAM, CloudFront OAC, Enforced HTTPS/TLS 1.2+.

## Deployment
1. Build the Lambda package and upload to an artifacts S3 bucket.
2. Deploy `infrastructure/main-template.yaml` using AWS CloudFormation.
3. Sync frontend application code to the created Website Bucket.

## Operations & Monitoring
* View centralized logs in Amazon CloudWatch.
* Metrics for Lambda invocations, errors, and durations are automatically captured via AWS Lambda Powertools.

```

---

### Final Verification Checklist

| Deliverable | Status | Location |
| --- | --- | --- |
| High-Level / Detailed Architecture | ✅ | Part 1 |
| Mermaid Architecture Diagram | ✅ | Part 1 |
| AWS Service Selection Justification | ✅ | Part 1 |
| Resource Naming Convention | ✅ | Part 1 |
| Complete CloudFormation Templates | ✅ | Part 2 |
| IAM Policies (Least Privilege) | ✅ | Part 2 (CFN) |
| Bucket Policies & CloudFront OAC | ✅ | Part 2 (CFN) |
| Lambda Python Code (Pillow, Powertools) | ✅ | Part 2 |
| ACM & Route53 Configuration | ✅ | Part 2 (CFN) |
| Deployment, Testing, Rollback Guides | ✅ | Part 3 |
| Monitoring, Security, Well-Architected Review | ✅ | Part 3 |
| Cost Estimation & Cleanup Guide | ✅ | Part 3 |
| Future Enhancements & README.md | ✅ | Part 3 |

The complete, production-ready enterprise solution has been fully designed and delivered across all three parts.
