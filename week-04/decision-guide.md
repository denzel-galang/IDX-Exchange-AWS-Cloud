# Week 4 Decision Guide

| Service | Type | Scope | Use this when... | Avoid when... |
|---|---|---|---|---|
| **S3** | Object storage | Regional (data spread across AZs) | You're storing files or blobs (backups, logs, static assets, exports) and reading them by key or URL | You need a mountable disk, or low-latency queries across fields |
| **EBS** | Block storage | One Availability Zone, normally attached to one EC2 instance | An EC2 instance needs a disk: its root volume, or a database you run yourself | Several instances need to share the same files |
| **EFS** | Network file system (NFS) | Regional, mountable from many instances at once | Multiple Linux instances need a shared, growing file system | One instance needs fast private disk (EBS is cheaper and simpler) |
| **RDS** | Managed relational database (SQL) | One region, optional Multi-AZ for failover | Your data has relationships, joins, and transactions, like the `rets_property` table | Your access pattern is simple key lookups at huge scale |
| **DynamoDB** | Managed NoSQL key-value/document | Regional, fully managed | You look items up by a known key (like `L_ListingID`) and want serverless scaling | You need ad-hoc queries, joins, or complex reporting |

## Where I used each one

- **S3:** Lab 4.1, a bucket with a lifecycle rule moving objects to Glacier after 30 days.
- **EBS:** Week 3, the root volume of `property-api-01` and its snapshot.
- **EFS:** concepts only, not used in a lab.
- **RDS:** Lab 4.2, the `training-db` MySQL instance holding the cleaned `rets_property` data.
- **DynamoDB:** Lab 4.3, the `Properties` table keyed on `L_ListingID`.