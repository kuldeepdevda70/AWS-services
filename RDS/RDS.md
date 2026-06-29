# RDS
## **What is Amazon RDS? What problem does it solve, and why do we use it instead of installing a database directly on an EC2 instance**

Amazon RDS (Relational Database Service) is a fully managed database service provided by AWS for relational databases such as MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and Aurora.

Instead of installing and managing a database on an EC2 instance, AWS handles tasks like database installation, patching, backups, monitoring, automated failover, software updates, and scaling.

We use RDS because it reduces operational effort, improves availability and security, and lets developers focus on building applications rather than managing the database infrastructure.

## Aurora offers:
Up to 5x the throughput of MySQL Community Edition
& 3x of PostGres

Up to 128 TB of autoscaling SSD storage

Six-way replication across three Availability Zones

Up to 15 read replicas with replica lag under 10-ms

Automatic monitoring with failover

## Benefits of Using RDS:
High availability and fault tolerance.

Vertical and Horizontal Scaling

Automated backups and recovery.

Read replicas for improved read performance

Multi AZ setup for DR (Disaster Recovery)

Cost-effectiveness.

---
<br></br>

## What is the difference between an RDS Snapshot and an Automated Backup?
Amazon RDS provides two backup options: Automated Backups and Manual Snapshots.

### Automated Backup:

1. Enabled by AWS when you configure the backup retention period.
2. AWS automatically takes daily backups and stores transaction logs.
3. You can restore the database to any point in time within the configured retention period (0–35 days).
4. When the retention period expires, older backups are automatically deleted.


### Manual Snapshot:

1. Created manually whenever needed.
2. Used before major changes such as database upgrades, migrations, or deletion.
3. A manual snapshot remains available until you delete it.
4. You can restore a new RDS instance from the snapshot at any time.

In short: Automated Backups are for disaster recovery and point-in-time recovery, while Manual Snapshots are for long-term backups or before important changes.


## Features
<img width="911" height="641" alt="image" src="https://github.com/user-attachments/assets/6029a2fc-3a04-4891-b155-24a06a00a448" />


<br></br>



## RDS Read Replica - Multi Region
<img width="890" height="507" alt="image" src="https://github.com/user-attachments/assets/2826c40a-ef52-409a-bd0e-38c0f92dd277" />

<br></br>



## RDS Multi - AZ
<img width="727" height="671" alt="image" src="https://github.com/user-attachments/assets/a0edc89e-194c-4912-9022-398fafd2113c" />


<br></br>


## RDS  Multi - AZ

<img width="759" height="684" alt="image" src="https://github.com/user-attachments/assets/758c45b3-dc89-4605-811d-12706c3152b3" />

<br></br>



## **Common Use Cases for RDS:**
Web Applications: Relational databases are ideal for web
apps requiring structured data.

E-commerce Platforms: For handling inventory, customer
data, and order transactions.

Business Applications: ERP, CRM, and financial applications
with strong data integrity needs.







