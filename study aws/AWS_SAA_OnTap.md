# 📚 AWS Solutions Architect Associate — Tài Liệu Ôn Tập

> Được tổng hợp từ ghi chú Notion + khóa học Udemy  
> Cập nhật: 22/5/2026

## 📋 Mục Lục

1. [IAM Users & Groups Hands On](#iam-users-groups-hands-on)
2. [EC2 Fundamentals](#ec2-fundamentals)
3. [Solution Architect Associate Level](#solution-architect-associate-level)
4. [EC2 Instance storage ](#ec2-instance-storage-)
5. [High Availability and Scalability: ELB & ASG](#high-availability-and-scalability-elb-asg)

---

# IAM Users & Groups Hands On

IAM Introduction: Users, Groups, Policies


IAM: Users & Group

IAM = Identity and Access Management, Global service

Root account created by default, shouldn’t be used or shared

Users are people within your organization, and can be grouped

Groups only contain users, not other groups

Users don’t have to belong to a group, and user can belong to multiple groups

![image](attachment:d505df45-7016-4518-942d-f5101abdc228:image.png)


IAM Permissions

Users or Groups can be assigned JSON documents called policies

These policies define the permissions of the users

In AWS, you apply the “Least privilege principle”: don’t give more permissions than a user needs.

![image](attachment:7fed0efa-7b23-40f9-8a2b-18d4650a4f70:image.png)

IAM Users & Groups Hands On

Why create IAM Users

Currently, you are using the root user, which is identified by the account ID. It is not best practice to use the root account for everyday tasks. Instead, we create IAM users, such as admin users, to use our accounts more safely

Some reasons you need to create IAM users:

The root account has full access to everything

Creating IAM users lets you avoid sharing the root account, which reduces risks

Users can be organized into groups to simplify permission management

AWS allows customizations of the sign-in URL through account aliases for easier access

Step:
Create user:


![image](attachment:e17cc103-d0e6-4db7-a5c9-e596cb4a1d24:image.png)

For simplicity and exam purposes, choose to create an IAM user

![image](attachment:054d8b6e-5423-4635-b8de-a002efd64eed:image.png)

create admin group ( include assign permissions)


![image](attachment:e8bbea13-81bf-4f7d-907d-0cbdfb173a4b:image.png)

Signing in with the IAM User


![image](attachment:a35a38bf-998b-4f35-a059-52192ce93472:image.png)

Click “Create”


![image](attachment:dd6549b0-1eb5-4e0f-bba4-16913028a3ee:image.png)

after that


![image](attachment:1ab26b84-7a3c-4f44-8932-44b7baec622e:image.png)

Open a private browser and access via the link and log in with the IAM user

![image](attachment:dd3bdd71-dd1e-40b7-832c-691c0ed8ec52:image.png)

IAM Policy Structure

Structure of IAM policies

![image](attachment:ff171f7b-f506-47e6-a43d-4ed9de45582a:image.png)

You can see the above image, which is the structure of IAM policies

IAM policy document consists of:

Version: The policy language version, usually “2012-10-17”

ID: An optional identifier for the policy

Statement(s): One or multiple statements defining permissions

Each statement includes several important parts:

Sid: Statement ID, an optional identifier

Effect: Specifies whether the statement allows or denies access to certain APIS, for example, “Allow” or “Deny”

Principal: Defines the accounts, users, or roles to which this policy applies. For instance, it could apply to the root accounts of your AWS accounts

Action: A list of API calls that are either allowed or denied based on the Effect

Resource: Specifies the resources to which the actions apply, such as an S3 bucket

Condition: Optional; defines when the statement should be applied

Key Takeaways

IAM policies can be attached at the group, user, or team levels, affecting access permissions accordingly

Users can inherit multiple policies from different groups or teams they belong to.

The IAM structure includes key elements above.

Understanding Effect, Principal, Action, and Resource is essential for managing AWS IAM policies effectively.

https://www.notion.so/Bucket-level-lock-trong-ConcurrentHashMap-21647f0876c180369850c484d4cbe30a?source=copy_link

IAM policies Hands On

You can manage user policies by either attaching the IAMReadOnlyAccess policy directly to the user or by adding the user to a group that has this policy.



![image](attachment:0a808106-830a-4fca-97a4-fc5d60154d7e:image.png)

You can view the details of IAM policies by selecting one of the policies. Then, you can see the JSON representation of the policy or a summary view

![image](attachment:461818ce-31ed-43cf-af3a-4f7e176c2a8f:image.png)

![image](attachment:bb4b8f0b-3f00-43fd-bb69-c09d69d436b7:image.png)

Key Takeaways 

IAM user permissions are managed through group memberships and attached policies

Removing a user from an admin group revokes their administrator access immediately

Read-only access policies allow viewing but restrict modification actions

Custom IAM policies can be created using the visual or JSON editor to specify precise permissions

IAM MFA Overview

MFA Device Options in AWS

Virtual MFA Device: This is what we will use in the hands-on. You can use apps like Google Authenticator, which works on one phone at a time, or Authy, which supports multiple tokens on a single device.

Univeral 2nd Factor (U2F) Security Key: This is a physical device, such as a Yubikey by Yubico ( a third party to AWS). 

Hardware Key Fob MFA Device: For example, a device provided by Gemalto, which is also a third party to AWS

AWS GovCloud Special Key Fob: IF you are using the AWS GovCloud in the US, there is a special key fob provided by SurePassID, another third party

Key Takeaways

Password policies enhance account security by enforcing strong password requirements

Multi-Factor Authentication (MFA) combines something you know (password) with something you own (security device) for stronger protection

AWS supports various MFA devices, including virtual MFA apps, U2F security key, and hardware key fobs

Protecting root and IAM user accounts with MFA significantly reduces the risk of unauthorized access.

IAM MFA Hands On

Key Takeaways

Defined and customized password policies in IAM to enhance security.

Enabled multi-factor authentication (MFA) for the root AWS account to add an extra security layer

Demonstrated the setup process of an authenticator app as an MFA device

Highlighted the importance of safeguarding MFA devices to prevent account lockout

AWS Access Keys, CLI and SDK

How can users access AWS?

To access AWS, you have three options:

AWS Management Console (protected by password + MFA)

AWS Command Line Interface (CLI): protected by access keys

AWS Software Developer Kit (SDK) - for code: protected by access key

Access key are generated through the AWS console

Users manage their own access keys

Access Keys are secret, just like a password. Don’t shre them

Access Key ID ~= username

Secret Access Key ~= password

![image](attachment:060768c4-364e-4b87-9f4d-ef2de86bd9ae:image.png)

What’s the AWS CLI

A tool that enables you to interact with AWS services using commands in your command-line shell

Direct access to the public APIs of AWS services

You can develop scripts to manage your resources

It’s open-source



---

# EC2 Fundamentals

AWS Budget Setup

Key Takeaways

Access to billing data requires enabling IAM user and role access in the root account 

The AWS Billing console provides detailed cost breakdowns by service and month 

The Free Tier dashboard helps monitor usage and forecast potential charges

Setting up budgets and alarms can prevent unexpected AWS costs by sending alerts when thresholds are reached 

IAM Users cannot access billing information unless access is explicitly enabled in Account Settings 

EC2 Basics 

Amazon EC2

EC2 is one of the most popular of AWS offerings

EC2 = Elastic Compute Cloud = Infrastructure as a service

It mainly consists in the capability of:

Renting virtual machines (EC2)

Storing data on virtual drives (EBS)

Distributing load across machines (ELB)

Scaling the services using an auto-scaling group (ASG)

Knowing EC2 is fundamental to understanding how the Cloud works

EC2 sizing and configuration options 

Operating System (OS): Linus, Window or Mac OS

How much compute power & cores (CPU)

How much random-access memory (RAM)

How much storage space

Network-attached (EBS & EFS)

Hardware (EC2 Instance Store)

Network card: speed of the cared, Public IP address

Firewall rules: security group

Bootstrap script (configure at first launch): EC2 User Data

EC2 User Data and Bootstrapping

What is Bootstrapping

Bootstrapping means running scripts or commands automatically when a server (EC2 instance) starts for the first time

It allows you to configure the server without manual intervention

Common uses include installing software, configuring services, and downloading required files

What is EC2 User Data

User Data is a special field where you can provide a script to run when the EC2 instance boots up for the first time only

AWS automatically executes this scripts during the initial launch of the instance

That script is only run once at the instance's first start

EC2 user data is used to automate boot tasks such as

Installing updates

Installing software

Downloading common files from the internet

Anything you can think of

For example: above image content some instance of EC2

![image](attachment:9e54d950-0413-4ed1-939a-da738218443b:image.png)

Key Takeaways

Amazon EC2 stands for Elastic Compute Cloud and provides Infrastructure as a Service (IaaS) on AWS

EC2 allows renting virtual machines called instances, with customizable operating systems, CPU, memory, storage, and networking

EC2 User Data scripts enable bootstrapping by automating tasks during the first launch of an instance

Various instance types exist to fit different application needs, with the t2.micro instance included in the AWS free tier

Create an EC2 Instance with EC2 User Data to have a Website Hands-On

**The step below is how to create an EC2 instance of Stephan in the course AWS Solution Architect Associate**
Key Takeaways

Launched an EC2 instance using Amazon Linux 2 AMI with a t2.micro instance type

Configured key pairs for SSH access and set up security groups to allow SSH and HTTP traffic

Used EC2 user data to automatically install and start a web server on the instance

Learned how to start, stop, and terminate EC2 instances and understood the behavior of public and private addresses upon instance restart

EC2 Instance Types Basics

Overview

Amazon EC2 provides virtual servers with various instance types optimized for specific workloads

Choosing the right instance saves cost and improves performance

7 major families:

General Purpose (T, M, A)

Compute Optimized (C)

Memory Optimized ( R, X, Z, u-High Memory)

Storage Optimized (I, D, H)

Accelerated Computing (P, G, F, Inf, Trn)

HPC Optimized 

Instance Features

Naming Convention 

Example: m5.2xlarge

m: Instance class → General Purpose

5: Generation → higher = newer hardware

2xlarge: denotes the size within the instance class → defines vCPU, RAM ( scales: small → large → xlarge→ 2xlarge →….)

Instance Families

General Purpose (T, M, A) 

Balanced: CPU ↔ Memory ↔ Network

Use cases: web/app servers, microservices, code repositories.

Examples:

t2.micro → Free Tier ( 1 vCPU, 1 GB RAM, burstable CPU)

m5.large → common production workloads

Compute Optimized (C)

High CPU performance, high CPU-to-memory ratio 

Use cases: batch processing, gaming servers, ML inference, HPC, media transcoding

These instances belong to the C series, such as C5 and C6

Example: c5.4xlarge ( 16 vCPU, 32 GB RAM)

Memory Optimized (R, X, Z, u-High Memory)

Provide high performance for workloads that process large datasets in memory

Large RAM, optimized for memory-heavy workloads

Use cases: databases (RDS, NoSQL), in-memory cache (Redis), big data, real-time analytics

The R series represents memory-optimized instances, with other types including X1 high memory and Z1 instances

Example:

r5.16xlarge ( 64 vCPU, 512 GB RAM)

x1e.32xlarge (3.9 TB RAM for SAP HANA) 

Storage Optimized (I, D, H)

Storage-optimized instances are ideal when accessing large datasets on local storage

Use cases: OTLP, data warehousing, NoSQL DB, search engines, log processing

These instances typically start with the letters I, G, or H

Example: i3.4xlarge ( 16 vCPU, 122 GB RAM, NVMe SSD)

Storage Optimized Instances

Storage Optimized instances are ideal when accessing large datasets on local storage

GPU or specialized hardware for acceleration

Use cases: ML training, AI, deep learning, video rendering, HPC

Example: p3.8xlarge (NVIDIA Tesla V100 GPU)

key Takeaways

EC2 Instances are categorized into types optimized for different workloads, such as general purpose, compute optimized, memory optimized, and storage optimized

AWS uses a naming convention for instances: Instance Class (e.g., M), generation number (e.g., 5), and size (e.g., 2xlarge).

General-purpose instances balance compute, memory, and networking, suitable for diverse workloads like web servers

Compute optimized instances ( C series) are ideal for CPU-intensive tasks like batch processing and machine learning

Memory optimized instances (R series and others) are designed for workloads requiring large RAM, such as databases and real-time big data processing

Storage-optimized instances (I, G, H series) excel at local storage access for transactional processing and data warehousing

The t2.micro instance is part of the AWS free tier, offering 750 hours per month

The website instituteinstance.info is a useful resource to compare EC2 instance types, costs, and specifications 

Security Groups & Classic Ports Overview

Introduction to Security Groups

Security Groups are the fundamental of network security in AWS

They control how traffic is allowed into or out of our EC2 instances

Security groups only contain “allow” rules. Only “ALLOW rules”. It means you can’t write the rule “DENY” in the Security Group. If SG doesn’t have a rule, the traffic will be prevented as a default 

Security groups rules can reference by IP or by security group

![image](attachment:c93ad044-052f-4630-8521-92a8a2344c36:image.png)

Security Groups Deeper Dive

Security groups are acting as a “firewall” on EC2 instances

The regulation:



---

# Solution Architect Associate Level

Private vs Public vs Elastic IP 

![image](attachment:501441c9-b189-4eaf-884a-9d2988fab5d1:IMG_3024.jpeg)

Key Takeaways

IPv4 is the most common IP format, consisting of four numbers separated by dots, allowing approximately 3.7 bilion unique public addresses

Public IPs are unique across the internet and allow machines to be accessible globally, while private IPs are unique only within a private network

Private IPs enable communication within a private network and connect to the internet via NAT devices and internet gateways

Elastic IPs are static public IPv4 addresses in AWS that can be reassigned to different instances, but are limited in number and generally discouraged in favor of DNS or load balancers

EC2 Placement Groups

Introduction to Placement Groups

![image](attachment:7294b06f-d9ae-4677-b3ae-b3c3abaa28b6:image.png)

When creating a placement group, you have three strategies available:

Cluster: Instances are grouped together in a low-latency hardware setup within a single availability zone

Spread: Instances are spread across different hardware

Partition:  Instances are spread across multiple partitions, which are sets of racks within an availability zone

Cluster Placement Group

![image](attachment:7897b72d-0824-4c1c-a89e-3dcf9ba4f3c2:image.png)

All your EC2 instances are located within the same availability zone. This setup provides great networking capabilities.

Spread Placement Group

![image](attachment:1ed35972-78e2-4574-9943-b1c27227dfb2:image.png)

Spread placement groups aim to minimize failure risk by placing EC2 instances on different hardware.

Partition Placement Group

![image](attachment:60689214-ddc7-4f8e-9708-55076809330e:image.png)

Partition placement groups allow instances to be spread across multiple partitions within availability zones. Each partition corresponds to a rack of hardware. For example, you can have up to seven partitions per availability zone, each containing multiple EC2 instances.

Summary

Placement groups provide strategic control over EC2 instance placement to optimize performance and availability:

Cluster: High performance within a single AZ, but higher risk

Spread: High availability by spreading instances across hardware, limited to seven per AZ,

Partition: Scalable, partition-aware distribution across racks and AZs for large applications

Choosing the appropriate placement group depends on your application’s performance and fault tolerance requirements

Key takeaways

Placement groups allow control over EC2 instance placement within AWS infrastructure

Cluster placement groups provide low-latency, high-throughput networking within a single availability zone, but carry a higher risk

Spread placement groups distribute instances across different hardware to minimize simultaneous failures, limited to seven instances per AZ

Partition placement groups spread instances across multiple partitions (racks) within AZs, supporting large-scale, partition-aware applications like Hadoop and Kafka

Elastic Network Interfaces (ENI)

![image](attachment:5fe236d5-c74d-4a50-aef8-00eab2c12df3:image.png)

ENI = a virtual network card in a VPC

Provides network connectivity to EC2 instances

Main Components

Primary Private IPv4 → The main private IP address ( always exists)

Secondary Private IPv4 → Optional extra IPs for multi-IP setups

Elastic/Public IP → Optional public or Elastic IP mapped to a private IP

Security Groups → Act as firewalls controlling inbound/outbound traffic

MAC address → Unique hardware address of the ENI


Attachment & Mobility

Each EC2 has a primary ENI (eth0) by default

You can create extra ENIs and attach them as secondary (eth1,eth2…)

ENIs can be moved between EC2 instances ( for failover or maintenance)

ENIs are bound to a specific Availability Zone ( AZ) → cannot move across AZs

Use Cases

High Availability/ Failover: move ENI ( and its IP) to a standby instance

Multiple Network Interfaces: connect to multiple subnets or networks

Security Isolation: Separate traffic types via different ENIs & security groups

Key Takeaways

Elastic Network Interfaces (ENIs) are virtual network cards within a VPC that provide network connectivity to EC2 instances

Each ENI can have a primary private IPv4 address, multiple secondary IPv4 addresses, and an associated elastic or public IPv4 address

ENIs can be created independently of EC2 instances and attached or moved between instances within the same Availability Zone for failover

ENIs are bound to a specific Availability Zone and support multiple security groups and MAX addresses

EC2 Hibernate 

Introduction to EC2 Hibernate

![image](attachment:3f233091-b536-4e8c-a2f6-d40b89fcfdd3:image.png)

When we stop an instance, the data on disk, specifically on your EBS volume, is kept intact until the next start. This behavior makes sense

If you terminate an instance, and if you have set up the root volume to be destroyed with your instance, it will be destroyed. However, any volume not set to be destroyed upon termination will be retained

When you start an instance, the operating system boots up, then the EC2 User Data script runs. After that, your application starts, and caches get warmed. This process can take time because you are booting your machine from scratch

Introducing EC2 Hibernate

The in-memory (RAM) state is preserved 

The instance boot is much faster! ( the OS is not stopped/restarted)

Under the hood: the RAM state is written to a file in the root EBS volume

The root EBS volume must be encrypted 

Use Cases for EC2 Hibernate

Running long-lived processes without actually stopping them

Saving the RAM state to avoid reinitialization

Fast rebooting when services take time to initialize, allowing them to stay up and running even after hibernation 

Important Details about EC2 Hibernate

It supports many different instance families

The instance RAM size must be less than 150 gigabytes 

It does not work for bare metal instances

It supports multiple operating systems, including Linux and Windows

The root volume must be an encrypted EBS volume with enough space to store the RAM dump

Available for on-demand, reserved, and spot instances

Hibernate is intended for durations no longer than 60 days

Key Takeaways

EC2 Hibernate preserves the in-memory RAM state by saving it to the root EBS volume, enabling faster instance boot times

The root EBS volume must be encrypted and have sufficient space to store the RAM dump

Hibernate supports many instance families and operating systems, but excludes bare metal instances

Hibernation is intended for use up to 60 days and is available for on-demand, reserved, and spot instances

EC2 Hibernate - Hands On 

Key Takeaways

The uptime value proves that the instance resumed from its previous state

Enabled EC2 hibernation by configuring an encrypted root EBS volume with sufficient storage

Demonstrated stopping and restarting an instance with hibernation, preserving RAM state




---

# EC2 Instance storage 

EBS Overview

What’s an EBS Volume

An EBS ( Elastic Block Store ) Volume is a network drive you can attach to your instances while they run

It allows your instance to persist data, even after its termination

They can only be mounted to one instance at a time ( at the CCP level) 

They are bound to a specific availability zone

AnAddAmazon

EBS VolumeAddAmazon

It’s a network drive ( i.e not a physical drive )

It uses the network to communicate with the instance, which means there might be a bit of latency

It can be detached from an EC2 instance and attached to another one quickly

It’s locked to an Availability Zone 

An EBS Volume in us-east-1a cannot be attached to us-east-1b

To move a volume across, you first need to snapshot it

Have a provisioned capacity ( size in GBs, and IOPS)

You get billed for all the provisioned capacity

You can increase the capacity of the drive over time

![image](attachment:8c67788d-5cdf-42e9-831d-380b4c398d07:image.png)

EBS - Delete on Termination attribute

Controls the EBS behavior when an EC2 instance terminates

By default, the root EBS volume is deleted ( attribute enabled) 

By default, any other attached EBS volume is not deleted ( attribute disabled)

This can be controlled by the AWS console? AWS CLI

Use case: preserve root volume when the instance is terminated 

Key Takeaways

EBS volumes are network-attached storage devices that persist data independently of EC2 instance lifecycles

Each EBS volume is bound to a specific availability zone and can only be attached to one instance at a time

EBS volumes must be provisioned with capacity and IOPS in advance, and their performance can be adjusted over time

The “delete on termination” attribute controls whether EBS volumes are deleted when their associated EC2 instance is terminated, with root volumes deleted by default

EBS Hands On

Key Takeaways

EBS volumes are attached to EC2 instances within specific availability zones

You can create, attached, and delete EBS volumes dynamically through the AWS console

EBS volumes have a “delete on termination” attribute that determines if they are deleted when the instance is terminated

EBS volumes must be in the same availability zone as the EC2 instance to be attached

EBS Snapshots

Make a backup (snapshot) of your EBS volume at a point in time

Not necessary to detach volume to do snapshot, but recommended

Can copy snapshots across AZ to region

![image](attachment:18f4a543-b69d-4dcb-99cc-5dd9b4d253c6:image.png)

EBS Snapshots Features

EBS Snapshot Archive 

Move a Snapshot to an “archive tier” that is 75% cheaper

Takes within 24 to 72 hours for restoring the archive

Recycle Bin for EBS Snapshots

Setup rules to retain deleted snapshots so you can recover them after an accidental deletion 

Specify retention form 1 day to 1 year

Fast Snapshot Restore (FSR)

Force full initialization of snapshot to have no latency on the first use ($$)

Conclusion

This concludes the lecture on EBS Snapshots. These features provide flexibility, cost savings, and data protection options for managing your EBS volumes effectively

Key Takeaways

EBS Snapshots serve as backups of EBS volumes at any point in time without necessarily detaching the volume from the EC2 instance

Snapshots can be copied across different Availability Zones Regions, facilitating volume transfer

The EBS Snapshot Archive tier offers up to 75% cost savings but requires 24 to 72 hours for restoration

Re Recycle Bin feature allows recovery of accidentally deleted snapshots with retention periods from one day to one year

Fast Snapshot Restore enables immediate volume initialization with zero latency on first use but incurs higher costs.

EBS Snapshots - Hands On

Key Takeaways

Created and managed EBS snapshots to back up volumes

Demonstrated copying snapshots across AWS regions for disaster recovery

Restored volumes from snapshots in different Availability Zones

The Recycle Bin feature to protect snapshots from accidental deletion

AMI Overview

What is an AMI?

AMI ( Amazon Machine Image ) is a template used to launch EC2 instances

It includes everything your instance needs

Operating System (OS)

Software configuration

Monitoring or security tools

Type of AMIs

Public AMI: Provided by AWS for everyone to use. Ex: Amazon Linux 2 AMI

Custom AMI: Created by you with your own software/configs. Ex: Your custom app server

Marketplace AMI: Provided by sold third parties on AWS Marketplace. Ex: WordPress. MongoDB AMIs

Benefits of Creating Your Own AMI

Faster boot and configuration times ( everything preinstalled)

Consistency across environments (same setup for all instances)

Time-savings for repetitive deployments

Portable across regions - can copy AMIs to other AWS regions

AMI Creation Process

First, we start an EC2 instance and customize it, Then, we stop the instance to ensure data integrity is correct

Nex, we build an AMI from the instance. This process also creates EBS snapshots behind the scenes. Finally, we can launch instances from other AMIs

Key Takeaways 

An AMI customizes EC2 instances with preconfigured software and operating systems

Creating your own AMI results in faster boot and configuration time for EC2 instance

AMIs can be region-specific but can be copied across AWS regions to leverage global infrastructure

AWS Marketplace offers AMIs created and sold by third parties, enabling users to save time or even sell their own AMIs

AMI Hands On

Key Takeaways

Launched an EC2 instance using Amazone Linux 2 and configured it with Apache HTTPD server

Created an AMI from the configured instance to save its state for faster future deployments

Demonstrated launching new instances from the created AMI, significantly reducing setup time

Highlighted the benefits of AMIs for packaging pre-installed software and speeding up instance boot-up

EC2 Instance Store

EBS volumes are network drives with good but “limited” performance

If you need a high performance hardware disk, use EC2 Instance Store

Better I/O performance

EC2 Instance Store lose their storage if they’re stopped ( ephemeral ) 

Good for buffer / cache / scratch data / temporary content

Risk of data loss if hardware fails

Backups and Replication are your responsibility

Benefits and Characteristic of EC2 Instance Store

It is an excellent choice when extremely high disk performance is required. However, there is an important caveat → If you stop or terminate your EC2 instance that has an Instance Store, the storage will be lost.

Because of this, EC2 Instance Store is called ephemeral storage. This mean it cannot be used as a durable, long-term place to store your data

Performance Illustration 

![image](attachment:6125248e-9e02-47b5-a247-466e457ea49a:image.png)

To illustrate the performance benefits, consider the instance size 13.x instances, which come with an Instance Store attached. The Read IOPS and Write IOPS. which correspond to the number of input/output operations per second, can reach up to 3.3 millions and 1.4 million respectively for the most performant instance

In comparison, an EBS volume of type gp2 can reach up to 32,000 IOPS. This demonstrates the significantly higher performance of Instance Store volumes

From an exam perspective, whenever you see a very high-performance hardware-attached volume for your EC2 instances, think of local EC2 Instance Store

Key Takeaways

EC2 Instance Store provides hardware-attached disk storage for EC2 instances, offering extremely high I/O performance

Instance Store storage is ephemeral and data is lost if the instance is stopped or terminated

Ideal use cases for Instance Store include buffers, caches, and temporary data, but not long-term storage

Users are responsible for backing up and replicating data stored pn Instance Store to prevent data loss

EBS Volume Types

General Purpose SSD



---

# High Availability and Scalability: ELB & ASG


High Availability and Scalability 

What is Scalability?

It mean that your application system can handle a greater load by adapting accordingly. There are two types of scalability

Vertical Scalability

Horizontal scalability

It is important to note that scalability is different from high availability, although they are related concepts

Vertical Scalability


![image](attachment:326d949e-f09f-4664-a2ad-3f14955c0258:image.png)

Vertical Scalability means increasing the size or capacity of your instance. For example, consider a phone operator

For example, your application runs on a t2.micro

Scaling that application vertically means running it on a t2.large

Vertical scalability is very common for non distributed system, such as a database

RDS, ElastiCache are services that can scale vertically

There’s usually a limit to how much you can vertically scale ( hardware limit)

Horizontal Scalability

![image](attachment:64833c9b-b2db-4629-b08d-0d064e3e0937:image.png)

Horizontal Scalability means increasing the number of instances/system for your application

Horizontal scaling implies distributed systems

This is very common for web applications / modern applications

It’s easy to horizontally scale thanks the cloud offerings such as Amazone EC2

High Availability

High availability usually goes hand in hand with horizontal scaling but is not the same

It means running your application or system in at least two data centers or availability zones in AWS

The goal is to survive a data center loss, if one center goes down, the system continue running

For example, having three phone operators in a New York building and three in a San Francisco building ensures that if the New York building loses connection, the San Francisco operators can still take calls.

This setup is highly available.

High availability can be:

Passive: RDS Multi-AZ deployments where a standby instance exists but is not actively serving

Active: When multiple instances actively serve traffic simultaneously such as horizontally scaled phone operators in two buildings both taking calls at the same time

Scalability and High Availability in AWS

Vertical scaling: Increasing instance size, such as from t2.nano (0.5 GB RAM, 1 vCPU) to a1.12tb1.metal (12.3 TB RAM, 450 vCPUs).

Horizontal scaling: Increasing the number of instances, known as scaling out (adding instances) or scaling in (removing instances).

High availability: Running the same application instance across multiple availability zones, enabled in auto-scaling groups or load balancers.

Key Takeaways

Scalability allows an application system to handle increased load by adapting, either vertically or horizontally.

Vertical scalability involves increasing the size or capacity of a single instance, such as upgrading a server or operator's capability.

Horizontal scalability involves increasing the number of instances or systems, commonly used in distributed systems and modern web applications.

High availability ensures system operation across multiple data centers or availability zones to survive failures, often implemented alongside horizontal scaling.

Elastic Load Balancing (ELB) Overview

Benefits of Using a load balancer

Provides a single point of access to your applications

Seamlessly handles failures of downstream instances through health check mechanisms

Supports health checks to determine instance availability

Provides SSL termination for HTTPS encrypted traffic

Enforces stickiness with cookies

Ensures high availability across zones

Separates public traffic from private traffic in your cloud environment

Why use an Elastic Load Balancer

An ELB is a managed load balancer

AWS guarantees that it will be working

AWS takes care of upgraders, maintenance, high availability

AWS provides only a few configuration knobs

It costs less to setup your own load balancer but it will be a lot more effort on your end 

It is integrated with many AWS offerings / services

EC2, EC2 auto Scaling Groups, Amazon EC2

AWS Certificate Manager (ACM) , CloudWatch

Router 53, AWS WAF, AWS Global Accelerator

Health Check

![image](attachment:76ae9981-e005-47fb-8da2-736d28ea1ce2:image.png)

Types of Managed Load Balancers on AWS

AWS provides four kinds of managed load balancers:

Classic Load Balancer (CLB): Older generation from 2009, supports HTTP, HTTPS, TCP, SSL, and Secure TCP. ( Không sài được nữa)

Application Load Balancer (ALB): Introduced around 2016, supports HTTP, HTTPS, and WebSocket protocols.

Network Load Balancer (NLB): Introduced in 2017, supports TCP, TLS, Secure TCP, and UDP protocols.

Gateway Load Balancer (GWLB): Introduced in 2020, operates at the network layer and supports the IP protocol.

It is recommended to use the newer generation load balancers as they provide more features.

Load balancers can be configured as internal (private) for private network access or external (public) for websites and public applications.

Security Groups


![image](attachment:f997d267-9a24-4c52-9327-a98e905bc77e:image.png)

✔ Load Balancer SG:

ALLOW từ 0.0.0.0/0 (public)

Port 80 (HTTP) hoặc 443 (HTTPS)

✔ EC2 SG:

Chỉ allow từ Security Group của Load Balancer

KHÔNG BAO GIỜ mở EC2 ra 0.0.0.0/0

Key Takeaways

Load balancers distribute incoming traffic across multiple backend EC2 instances to ensure efficient resource use and reliability.

Elastic Load Balancers provide a single endpoint for users, handle health checks, SSL termination, and support high availability.

AWS offers four types of managed load balancers: Classic Load Balancer (deprecated), Application Load Balancer, Network Load Balancer, and Gateway Load Balancer.

Security groups should be configured to allow user traffic to the load balancer and restrict EC2 instance traffic to only originate from the load balancer's security group.

Application Load Balancer (ALB)

Application load balancers is 7 Layer (HTTP)

Load balancing to multiple HTTP applications across machines (target groups)

Load balancing to multiple applications on the same machine (ex: containers)

Support for HTTP/2 and WebSocket

Support redirects (from HTTP to HTTPs for example)

Routing Capabilities of ALB

Routing tables to different target groups:

Routing based on path in URL (example.com/users & example.com/posts)

Routing based on hostname in URL (one.example.com & other.example.com)

Routing based on Query String, Headers (example.com/users?id=123&order=false)

ALB are a great fit for micro services & container-based application (ex: Docker and Amazon ECS)

Has a port mapping feature to redirect to a dynamic port in ECS

In comparison, we’d need multiple Classic Load Balancer per application

![image](attachment:28ebb1a7-d34f-48b1-a1cd-17ee148da278:image.png)

Target Groups

EC2 instances (can be managed by an Auto Scaling Group) - HTTP

ECS tasks (managed by ECS itself) - HTTP

Lambda functions - HTTP request is translated into a JSON event

IP Addresses - must be private IPs

ALB can route to multiple target groups

Health checks are at the target group level

![image](attachment:4d7e10e8-043c-4fcd-ad4c-b7ac4a9c779c:image.png)

Additional ALB Features

![image](attachment:fbba2884-6c0d-4747-8d6c-8d88e361d9b4:image.png)

Before we proceed to the hands-on section, here are some important points:

You get a fixed hostname with your Application Load Balancer, similar to the Classic Load Balancer.

The application servers do not see the client's IP address directly. Instead, the true client IP is inserted into the HTTP header called X-Forwarded-For.

