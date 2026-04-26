# Migrating an EC2 Database to RDS (WordPress on AWS)

## Overview

This project demonstrates migrating a WordPress database from a self-managed MariaDB instance running on EC2 to a managed Amazon RDS MySQL instance. The goal is to offload database management to AWS by moving from an EC2-hosted database to a dedicated RDS instance — improving reliability and separation of concerns.

---

## Architecture

- **VPC:** Custom VPC (`a4lVPC1`) deployed via CloudFormation
- **Compute:** EC2 instance running WordPress (`A4L-Wordpress`)
- **Original DB:** MariaDB running on a separate EC2 instance (`A4L-DB-Wordpress`)
- **Target DB:** Amazon RDS MySQL instance (`a4lwordpress`)

---

## Architecture Diagram

### Before Migration
```
[ Browser ] → [ A4L-Wordpress EC2 ] → [ A4L-DB-Wordpress EC2 (MariaDB) ]
                    (both running inside a4lVPC1)
```

### After Migration
```
[ Browser ] → [ A4L-Wordpress EC2 ] → [ a4lwordpress RDS MySQL ]
                    (WordPress EC2 in a4lVPC1, RDS in dedicated subnet group across us-east-1a/b/c)
```

> **Note:** Replace this section with a screenshot of your architecture once you have one. A diagram image carries more weight on a portfolio than a text drawing.

---

## Steps

### 1. Environment Setup
- Deployed the base infrastructure using a CloudFormation 1-click template, which provisioned the VPC, subnets, and EC2 instances
- Accessed the WordPress site via the EC2 public IP, created an admin account, and published a test post with images to confirm the app was functional before migration

### 2. Deploy RDS Instance
- Created a DB subnet group (`a4lsngroup`) spanning three Availability Zones (`us-east-1a`, `us-east-1b`, `us-east-1c`) using the VPC's web-tier subnets
- Launched a MySQL RDS instance using the Free Tier template with no public access
- Created a dedicated security group (`a4lvpc-rds-sg`) to control inbound access to the database
- Updated the RDS security group's inbound rules to allow MySQL traffic from the EC2 instance security group

### 3. Export Database from EC2
Connected to the WordPress EC2 instance via Instance Connect and ran a `mysqldump` against the MariaDB instance using its private IP to export the WordPress database to a `.sql` file

### 4. Import Database into RDS
Used the `mysql` CLI to import the `.sql` dump file into the RDS instance, targeting it by its RDS endpoint (CNAME)

### 5. Update WordPress Configuration
- Edited `wp-config.php` on the EC2 instance to point the `DB_HOST` value to the RDS endpoint
- Verified the site still loaded correctly via the EC2 public IP

### 6. Validate Migration
- Stopped the original EC2-hosted database instance (`A4L-DB-Wordpress`)
- Confirmed WordPress continued to function using only the RDS instance as the backend

### 7. Cleanup
- Deleted the RDS instance (no final snapshot retained)
- Deleted the DB subnet group
- Deleted the `a4lvpc-rds-sg` security group
- Deleted the CloudFormation stack to remove all remaining infrastructure

---

## Key Concepts Demonstrated

- Separating application and database tiers using managed services
- Creating and configuring RDS subnet groups across multiple Availability Zones
- Database security using VPC security groups (no public RDS access)
- Using `mysqldump` and `mysql` CLI for live database migration
- Updating application configuration to redirect database connections

