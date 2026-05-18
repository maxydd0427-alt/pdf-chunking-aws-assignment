

# Infrastructure as Code Design Document

## 1. Purpose

This document describes the planned AWS infrastructure for the PDF chunking comparison web application. It translates the architecture diagram into deployable AWS resources, including networking, compute, storage, database, event-driven processing, security groups, IAM permissions, and deployment order.

The purpose of this document is to provide a clear implementation plan before writing the final Infrastructure as Code template in `infra/main.yaml`.

## 2. Deployment Overview

The system is deployed inside a custom Amazon VPC across two Availability Zones. Users access the web application through an internet-facing Application Load Balancer. The web application runs on an EC2 instance managed by an Auto Scaling Group, with desired capacity set to 1 to reduce cost.

Uploaded PDF files are stored in Amazon S3. Application state, document metadata, processing status, generated chunks, and processing statistics are stored in Amazon RDS.

To decouple the upload request from PDF processing, the web application publishes a document-uploaded event to Amazon SNS after storing the PDF and metadata. Amazon SNS fans out the event to two Amazon SQS queues. A background worker EC2 instance runs two worker processes: one for fixed-size chunking and one for paragraph-aware chunking. Each worker consumes jobs from its corresponding SQS queue, reads the PDF from S3, processes the document, and writes results back to RDS.

## 3. Resource Inventory

| Layer | AWS Resource | Planned Name | Purpose |
|---|---|---|---|
| Network | VPC | `pdfchunk-vpc` | Custom network for the deployment |
| Network | Public Subnet A | `pdfchunk-public-subnet-a` | Hosts ALB subnet attachment and NAT Gateway |
| Network | Public Subnet B | `pdfchunk-public-subnet-b` | Hosts ALB subnet attachment |
| Network | Private App Subnet A | `pdfchunk-private-app-subnet-a` | Hosts Web App EC2 and Worker EC2 |
| Network | Private App Subnet B | `pdfchunk-private-app-subnet-b` | Supports ASG scale-out or instance replacement |
| Network | Private DB Subnet A | `pdfchunk-private-db-subnet-a` | RDS DB subnet group member |
| Network | Private DB Subnet B | `pdfchunk-private-db-subnet-b` | RDS DB subnet group member |
| Network | Internet Gateway | `pdfchunk-igw` | Allows public Internet access to the ALB |
| Network | NAT Gateway | `pdfchunk-nat-gateway` | Provides outbound Internet access for private EC2 instances |
| Compute | Application Load Balancer | `pdfchunk-alb` | Public entry point for the web application |
| Compute | Target Group | `pdfchunk-web-tg` | Routes ALB traffic to Web App EC2 instances |
| Compute | Launch Template | `pdfchunk-web-launch-template` | Defines Web App EC2 configuration |
| Compute | Auto Scaling Group | `pdfchunk-web-asg` | Manages Web App EC2 instance capacity |
| Compute | Worker EC2 | `pdfchunk-worker-ec2` | Runs background worker processes |
| Storage | S3 Bucket | `pdfchunk-upload-bucket` | Stores uploaded PDF files |
| Database | Amazon RDS | `pdfchunk-rds` | Stores metadata, status, chunks, and statistics |
| Event | Amazon SNS Topic | `pdfchunk-upload-topic` | Distributes document-uploaded events |
| Event | SQS Fixed Queue | `pdfchunk-fixed-queue` | Buffers jobs for fixed-size chunking |
| Event | SQS Paragraph Queue | `pdfchunk-paragraph-queue` | Buffers jobs for paragraph-aware chunking |
| Security | ALB Security Group | `pdfchunk-alb-sg` | Allows public HTTP access to the ALB |
| Security | Web App Security Group | `pdfchunk-web-sg` | Allows traffic from ALB to Web App EC2 |
| Security | Worker Security Group | `pdfchunk-worker-sg` | Controls Worker EC2 network access |
| Security | RDS Security Group | `pdfchunk-rds-sg` | Allows DB access only from Web App and Worker EC2 |
| IAM | Web App EC2 Role | `pdfchunk-web-role` | Allows Web App to access S3 and SNS |
| IAM | Worker EC2 Role | `pdfchunk-worker-role` | Allows Worker to access S3 and SQS |

## 4. Network Design

The network uses a custom VPC with public subnets, private application subnets, and private database subnets across two Availability Zones.

| Subnet | CIDR | Availability Zone | Purpose |
|---|---|---|---|
| Public Subnet A | `10.0.1.0/24` | AZ A | ALB and NAT Gateway |
| Public Subnet B | `10.0.2.0/24` | AZ B | ALB |
| Private App Subnet A | `10.0.11.0/24` | AZ A | Web App EC2 and Worker EC2 |
| Private App Subnet B | `10.0.12.0/24` | AZ B | ASG scale-out or replacement |
| Private DB Subnet A | `10.0.21.0/24` | AZ A | RDS subnet group |
| Private DB Subnet B | `10.0.22.0/24` | AZ B | RDS subnet group |

### Route Tables

| Route Table | Associated Subnets | Main Route |
|---|---|---|
| Public Route Table | Public Subnet A, Public Subnet B | `0.0.0.0/0 -> Internet Gateway` |
| Private App Route Table | Private App Subnet A, Private App Subnet B | `0.0.0.0/0 -> NAT Gateway` |
| Private DB Route Table | Private DB Subnet A, Private DB Subnet B | No direct Internet route |

The NAT Gateway is used to provide outbound access for EC2 instances in private subnets. This allows the Web App and Worker instances to access AWS public service endpoints such as S3, SNS, and SQS, as well as software package repositories. The NAT Gateway is not part of the public inbound request path.

A single NAT Gateway is used for cost control. In a production environment, one NAT Gateway per Availability Zone would provide higher availability.

## 5. Compute Design

### 5.1 Web Application Tier

The web application tier is deployed using an Auto Scaling Group. The ASG is configured across two private application subnets, but the desired capacity is set to 1 to reduce cost.

| Item | Planned Configuration |
|---|---|
| Compute service | Amazon EC2 |
| Management | Auto Scaling Group |
| Desired capacity | 1 |
| Minimum capacity | 1 |
| Maximum capacity | 2 |
| Subnets | Private App Subnet A and Private App Subnet B |
| Public access | No direct public access |
| Inbound traffic | From ALB Security Group only |

The Application Load Balancer is the only public entry point. User requests reach the ALB through the Internet Gateway, and the ALB forwards HTTP requests to the Web Application Server in the private subnet.

### 5.2 Worker Tier

The worker tier is deployed on a separate EC2 instance in a private application subnet. The Worker EC2 instance runs two worker processes:

- Fixed-size chunking worker
- Paragraph-aware chunking worker

The worker processes do not receive public HTTP requests. They poll their corresponding SQS queues, read uploaded PDFs from S3, process the documents, and write generated chunks and statistics to RDS.

## 6. Storage and Database Design

### 6.1 Amazon S3

Amazon S3 stores the uploaded PDF files. The Web Application Server uploads the PDF file to S3 and stores the S3 object key in RDS.

S3 stores the original uploaded files only. Application metadata and generated processing results are stored in RDS.

### 6.2 Amazon RDS

Amazon RDS stores persistent application data, including:

- Document metadata
- S3 object key for each uploaded PDF
- Processing status for each chunking strategy
- Generated chunks
- Processing statistics
- Retrieval query results if required

Planned database tables:

| Table | Purpose |
|---|---|
| `documents` | Stores uploaded document metadata and S3 object keys |
| `processing_jobs` | Stores processing status for each chunking strategy |
| `chunks` | Stores generated chunks for fixed-size and paragraph-aware strategies |
| `retrieval_queries` | Optionally stores user retrieval queries and retrieved chunk IDs |

## 7. Event-driven Processing Design

The upload request is decoupled from PDF processing using Amazon SNS and Amazon SQS.

After receiving a PDF upload, the Web Application Server performs the following steps:

1. Stores the PDF file in Amazon S3.
2. Writes document metadata and initial processing status to Amazon RDS.
3. Publishes a document-uploaded event to Amazon SNS.
4. Returns a response to the user without waiting for PDF processing to complete.

Amazon SNS fans out the same event to two SQS queues:

- `pdfchunk-fixed-queue`
- `pdfchunk-paragraph-queue`

Each worker process polls its corresponding queue. The fixed-size worker performs fixed-size chunking, while the paragraph-aware worker performs paragraph-aware chunking.

Example event message:

```json
{
  "document_id": "123",
  "s3_bucket": "pdfchunk-upload-bucket",
  "s3_key": "uploads/example.pdf",
  "filename": "example.pdf"
}
```

This design allows PDF processing to happen asynchronously in the background.

## 8. Security Group Design

| Security Group | Inbound Rules | Outbound Rules | Purpose |
|---|---|---|---|
| `pdfchunk-alb-sg` | HTTP 80 from `0.0.0.0/0` | To Web App SG | Allows users to access the ALB |
| `pdfchunk-web-sg` | App port from ALB SG only | To RDS, S3, SNS, Internet via NAT | Protects Web App EC2 |
| `pdfchunk-worker-sg` | No public inbound traffic | To RDS, S3, SQS, Internet via NAT | Protects Worker EC2 |
| `pdfchunk-rds-sg` | DB port from Web App SG and Worker SG only | Default outbound | Protects RDS |

The Web Application Server is not directly exposed to the Internet. It only accepts traffic forwarded by the ALB.

Amazon RDS is deployed in private database subnets and does not allow public access. It only accepts database connections from the Web App and Worker security groups.

## 9. IAM Permission Design

### 9.1 Web App EC2 Role

The Web App EC2 role requires permissions to:

- Upload PDF files to the S3 bucket.
- Read PDF files from the S3 bucket if needed.
- Publish document-uploaded events to the SNS topic.
- Write application logs to CloudWatch if logging is enabled.

Example permissions:

- `s3:PutObject`
- `s3:GetObject`
- `sns:Publish`
- `logs:CreateLogStream`
- `logs:PutLogEvents`

### 9.2 Worker EC2 Role

The Worker EC2 role requires permissions to:

- Read uploaded PDF files from S3.
- Receive and delete messages from SQS.
- Read SQS queue attributes.
- Write logs to CloudWatch if logging is enabled.

Example permissions:

- `s3:GetObject`
- `sqs:ReceiveMessage`
- `sqs:DeleteMessage`
- `sqs:GetQueueAttributes`
- `logs:CreateLogStream`
- `logs:PutLogEvents`

Permissions may be slightly broader than strict least privilege for simplicity, but they should be limited to the assignment-specific S3 bucket, SNS topic, and SQS queues where possible. In a production environment, permissions should follow the least privilege principle more strictly.

## 10. Deployment Order

The planned deployment order is:

1. Create the custom VPC.
2. Create public, private app, and private DB subnets across two Availability Zones.
3. Create and attach the Internet Gateway.
4. Create route tables and subnet associations.
5. Create the NAT Gateway for private subnet outbound access.
6. Create security groups.
7. Create the S3 bucket.
8. Create the RDS DB subnet group and RDS database.
9. Create the SNS topic and two SQS queues.
10. Subscribe both SQS queues to the SNS topic.
11. Create IAM roles for the Web App EC2 and Worker EC2.
12. Deploy the Worker EC2 instance and start both worker processes.
13. Create the Web App launch template.
14. Create the Web App Auto Scaling Group.
15. Create the ALB, target group, and listener.
16. Deploy the web application code.
17. Test PDF upload, event delivery, queue consumption, database updates, and result retrieval.

## 11. Cost-control Decisions

The deployment uses several cost-control decisions:

- The Auto Scaling Group desired capacity is set to 1.
- A single Worker EC2 instance runs both worker processes.
- A single NAT Gateway is used instead of one NAT Gateway per Availability Zone.
- Small EC2 and RDS instance types are preferred for the assignment deployment.

These choices reduce cost while still satisfying the required architectural properties. In a production environment, the architecture could be improved by using one NAT Gateway per Availability Zone, multiple Web App instances, multiple Worker instances, and Multi-AZ RDS.

## 12. Notes for CloudFormation Implementation

The final CloudFormation template in `infra/main.yaml` should create or define the following major resource groups:

- Network resources: VPC, subnets, route tables, Internet Gateway, NAT Gateway.
- Security resources: security groups and IAM roles.
- Storage resources: S3 bucket and RDS database.
- Event resources: SNS topic, SQS queues, and queue subscriptions.
- Compute resources: Worker EC2 instance, Web App launch template, Auto Scaling Group, ALB, target group, and listener.

This document should be used as the planning reference when implementing `infra/main.yaml`.