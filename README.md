# Mitigate an economic denial-of-service (EDoS) bot attack targeting Azure JA4 TLS fingerprints

This repository documents a real-world case study from JURASSIC PARK TECHNOLOGY LTD on how to detect and mitigate an economic denial-of-service (EDoS) bot attack targeting Azure AppService using JA4 TLS fingerprints, while cascading protection through Azure Front Door (AFD) and Azure Application Gateway WAF.
Unlike traditional DDoS attacks that try to take your site down, this attack was designed to inflate CDN and data transfer costs without overloading the origin, exploiting cached static assets and globally distributed IPs.



## Diagram
<img width="2214" height="1159" alt="image" src="https://github.com/user-attachments/assets/522ad414-9629-4841-b8c6-be5a45a25610" />



## Scenario: Tiger Bank Cloud Exit Use Case
Tiger Bank Ltd., a financial institution, currently operates its IT systems on AWS. Concerned about rising costs and eager to leverage Azure's integration with OpenAI features for enhanced customer experience, Tiger Bank Ltd. decides to migrate its workloads and applications from AWS to Azure.

## Terraform Structure

According to best practices, the terraform structure is as follows:
```
├── environments
│   ├── dev
│   │   ├── main.tf
│   │   ├── terraform-dev.tfvars
│   │   └── variables.tf
│   ├── prod
│   │   ├── main.tf
│   │   ├── terraform-prod.tfvars
│   │   └── variables.tf
│   └── readme.md
├── modules
│   └── {LIST_OF_ALL_MODULES}
└── Readme.md
```

# AWS to Azure Migration Services (high level inventory)

Below is a comparison of equivalent services between AWS and Azure to assist with the above migration process:

| AWS | AZURE |
| --- | --- |
| Virtual Private Cloud (VPC) | Virtual Network (VNET) |
| Accounts | Subscriptions |
| CloudWatch | Azure Monitor |
| CloudFront | Content Delivery Network |
| Route 53 | DNS |
| Codedeploy | Azure DevOps |
| EC2 | VM |
| Application Load Balancer | Application Gateway |
| Auto Scaling | Virtual Machine Scale Sets |
| S3 | Blob storage |
| RDS | Database for MySQL | 
| DynamoDB | Cosmos DB |
| Lambda | Functions |
| Certificate Manager | Key Vault  |
| Web Application Firewall | Application Gateway |
| SNS | Event Grid |

# Databackup
Data backup ensures data integrity, prevents loss, and enables seamless recovery during cloud exit, reducing migration risks significantly.
Below is how Tiger bank perform on Database, S3 and VM for databackup.

In additional to native AWS backup, Tiger bank will use 3rd party tools such as Rubrik for better Multi-Cloud support, zero-trust achitecture and advacned encryption to backup data.

#Backup Frequency:
1. Database(RDS/DynamoDB):
   - Perform full backups at beginning and daily incremental backup to ensure data integrity.
   - Enable **Point-in-Time Recovery (PITR)** for continuous backups, allowing you to restore to any specific time within the retention period.
   - Retain backups for at least 30 days or as per compliance requirements.

2. S3 (Object Storage):
   - Use **continuous replication** for critical data to Azure Blob Storage using Azure Data Box or AWS S3 Replication.
   - Schedule weekly backups for less frequently accessed data.
   - Enable versioning in S3 to maintain historical versions of objects.

3. VM (EC2 Instances):
   - Schedule daily snapshots of EC2 instances and attached volumes to capture the VM state.
   - Retain snapshots for at least 7-14 days, depending on recovery objectives.
  
# Handy tools during migration:
1. AWS Backup:
   - Automates backup scheduling, retention policies, and recovery for databases, S3, EC2, and other AWS resources.
   - Centralized management for all AWS backups.

2. Azure Migrate:
   - Facilitates the migration of VMs, databases, and applications from AWS to Azure.
   - Includes tools for assessment, replication, and cutover.

3. Azure Data Box:
   - A physical device for transferring large datasets securely from AWS to Azure.

4. AWS DataSync:
   - Automates and accelerates data transfer between AWS and Azure.

5. Azure Site Recovery (ASR):
   - Ensures business continuity by replicating workloads from AWS to Azure for disaster recovery.

6. Terraform:
   - Use Terraform scripts to automate infrastructure provisioning and configuration in Azure post-migration.
  
7. Gitlab CI/CD:
   - Use GitLab CI/CD to automate backups, data transfer, infrastructure provisioning, and testing, streamlining the cloud exit process for efficient, reliable, and error-free AWS-to-Azure migration.
  
8. Azure Data Factory
   - enables seamless data migration from S3 to Azure Blob with parallel processing, secure transfer, checkpointing, and large-scale data pipeline orchestration for efficient migration.
