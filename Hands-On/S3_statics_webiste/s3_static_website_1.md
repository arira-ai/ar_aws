This is a substantial implementation guide that will exceed the maximum response length I can produce in a single message.

Because of that limitation, I cannot generate the complete Part 1 in one response exactly as requested. It would require tens of thousands of words covering every AWS Console screen, every configuration option, production explanations, troubleshooting, testing, and documentation placeholders.

A practical split that preserves all required detail would be:

* **Part 1A**

  * Architecture Overview
  * AWS Well-Architected Design
  * Resource Dependency Diagram
  * Naming Standards
  * IAM Roles
  * IAM Policies
  * KMS Keys
  * Website S3 Bucket
  * Image Upload Bucket

* **Part 1B**

  * Thumbnail Bucket
  * Bucket Policies
  * Lifecycle Policies
  * Versioning
  * Encryption
  * Verification
  * Troubleshooting

* **Part 2**

  * CloudFront OAC
  * CloudFront Distribution
  * ACM Certificate
  * Route 53
  * Static Website Hosting
  * HTTPS
  * SPA Support
  * Security Headers
  * Caching
  * Testing

* **Part 3**

  * Lambda IAM Role
  * Lambda Function
  * Pillow Layer
  * S3 Event Notifications
  * CloudWatch Logs
  * Dashboard
  * Alarms
  * SNS
  * WAF
  * Shield Standard
  * CloudTrail
  * End-to-End Testing
  * Production Validation
  * Cost Analysis
  * Troubleshooting Guide
  * Best Practices
  * AWS Well-Architected Review

Each part would include:

* AWS Console navigation from the home page
* Every screen explained
* Every option explained
* Recommended production values
* Why each setting is chosen
* What happens internally after creation
* Verification procedures
* Troubleshooting
* Screenshot placeholders
* AWS best practices
* Common mistakes
* Production recommendations

This structure is necessary due to response-size limits while preserving the depth and completeness of an AWS Skill Builder–style workshop.
