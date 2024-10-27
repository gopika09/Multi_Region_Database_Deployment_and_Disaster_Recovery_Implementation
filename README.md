# Multi-Region Database Deployment and Disaster Recovery Implementation

This project focuses on creating a highly available and resilient database architecture across multiple AWS regions. By deploying an Amazon RDS instance with cross-region read replicas, this project ensures data availability and minimizes downtime in the event of regional failures. The disaster recovery plan incorporates AWS Route 53 failover policies, which automatically reroute traffic to the backup region, guaranteeing uninterrupted database access during outages. This setup demonstrates the effective use of AWS services to achieve scalability, data redundancy, and fault tolerance, ensuring business continuity.

## Overview :
![diagram](https://github.com/gopika09/Python_Flask_App_Terraform_Code/blob/main/diagram.png)

## Features

- **Cross-Region Database Replication**: Deployed Amazon RDS with cross-region read replicas to ensure data availability across multiple AWS regions.
- **Disaster Recovery**: Implemented a disaster recovery plan using Route 53 failover routing policies to automatically switch traffic to a healthy replica during regional failures.
- **Data Redundancy**: Ensures continuous data replication between regions, protecting against data loss during outages.
- **High Availability**: Configured a highly available architecture by distributing database resources across multiple regions, minimizing downtime and ensuring seamless user access.
- **Scalability**: Leveraged AWS’s scalable infrastructure to easily manage and expand database resources as needed.


## Tools and Technologies

- **Amazon RDS**: Managed relational database service that handles automated backups, scaling, and cross-region replication.
- **AWS VPC**: Provides isolated and secure networking environments for database instances in multiple regions.
- **Route 53*8: AWS’s DNS service used for failover routing, ensuring smooth traffic redirection during regional failures.
- **AWS EC2**: Virtual servers used to manage and monitor the database environment and any related application workloads.
- **Amazon S3** : Utilized for backup storage, ensuring long-term data availability and recovery.
- **Elastic Load Balancer (ELB)**: Distributes incoming traffic across EC2 instances, ensuring high availability and redundancy.

## Solution Overview:

This solution is deployed across two AWS Regions using an active-passive strategy. The primary (active) region hosts the workload and handles traffic, while the secondary (passive) region serves as the disaster recovery site. To route internet traffic between the two regions, an Amazon Route 53 public hosted zone with a failover policy is configured. Private hosted zone CNAME records are used to store the Amazon RDS for SQL Server endpoints, allowing the application to connect to the database using these records.

Amazon Route 53 public hosted zone failover policies follow a "standby takes over primary" approach. In this setup, Route 53 performs health checks on the primary region by verifying the accessibility of an HTTP endpoint, which corresponds to an Amazon S3 file stored in the secondary region.

If the HTTP response from this endpoint returns a status code of 4xx or 5xx (indicating that Route 53 health check agents cannot resolve the S3 file), the health check is considered healthy. However, if a status code of 2xx or 3xx is returned (indicating the endpoint was resolved), the health check fails. This approach, known as Route 53 Invert Health Check, enables manual control over failover between regions and prevents accidental failover if resources in the standby region experience issues.

In the event of a disaster, uploading a designated file to the S3 bucket in the secondary region triggers the failover by causing the Route 53 health check for the primary region to fail, allowing the secondary region to take over.
 

## Steps to Deploy

I started by creating two VPCs: one in the ap-south-1 region (the primary region) and another in ap-northeast-1 (the secondary region). Each VPC had one public and one private subnet. Once I had that basic network architecture set up, I moved on to configuring the RDS.

![VPC Architecture](path_to_image/vpc.png)



I created a DB subnet group and associated it with subnets, then launched an RDS instance using MySQL as the database engine. For the instance class, I went with "db.t3.micro." After setting the database username, password, and the initial database name, I made sure to attach the correct security group so that access to the RDS instance was secure. Once everything looked good, I created a read replica in the ap-northeast-1 region for redundancy and failover purposes. I captured the endpoints for the primary and secondary RDS instances to use later in the configuration.

![Security Groups](path_to_image/security_groups.png)



Next, I deployed EC2 instances and set up load balancers in both the primary and secondary regions and associate security groups to it. I updated application configuration in primary and secondary region and use CNAME record as the database host name in database connection string



![Security Groups](path_to_image/security_groups.png)


Then came the Route 53 setup. I created a private hosted zone in Route 53 and associated the VPCs for both regions with it. This private hosted zone would handle the internal routing of traffic within the VPCs. I added two CNAME records to this zone—one for the RDS instance in the primary region and another for the secondary region. This step ensured that the application would always use these CNAME records to connect to the database, simplifying things when a failover occurred.

![Security Groups](path_to_image/security_groups.png)



To prepare for disaster recovery, I created S3 buckets in both the primary and secondary regions. These buckets would store the disaster recovery file, and I restricted access to them to ensure security. At this stage, the buckets were empty because both regions were still functioning normally. 

![Security Groups](path_to_image/security_groups.png)



The next step was setting up a public hosted zone in Route 53 to manage internet traffic for the domain. I also created two Route 53 health checks—one for the primary region and one for the secondary. For the primary region's health check, I set it up to monitor the S3 endpoint in the secondary region. If that S3 file became accessible (meaning the secondary region was in use), the health check would fail, triggering the failover. I enabled the 'Invert health check status' option to make sure the failover would only happen if the primary region became inaccessible. I repeated these steps to create a health check for the secondary region, this time monitoring the S3 endpoint in the primary region. 

![Security Groups](path_to_image/security_groups.png)


Finally, I returned to the Route 53 public hosted zone and created the failover records. For each region, I defined the failover routing policy and linked it to the Application Load Balancers in both the primary and secondary regions. I also associated each record with the appropriate health check so that Route 53 would know when to switch between regions.


![Security Groups](path_to_image/security_groups.png)

After completing these steps, the solution was in place. The application was now set up for seamless failover between the primary and secondary regions. Should a disaster occur, I could simply upload a file to the S3 bucket in the secondary region, causing Route 53 to automatically switch traffic to the secondary RDS and EC2 instances without needing to make manual changes to the application configuration. This ensured that the setup was resilient, reliable, and capable of handling any regional outages.

##Key Considerations for Cross-Region Failover

When preparing for a cross-region failover, several critical factors must be taken into account:

1. **Service Availability**: 
   Ensure that one or more AWS services your application relies on in the primary region are not responding. This step is crucial, as failing services can impede a successful failover.

2. **Application Readiness**: 
   Confirm that the application tier deployed in the secondary region is fully updated and configured to assume the primary role. This ensures a seamless transition without disruptions.

3. **Live Traffic Switching**: 
   The only option to restore customer services is to switch live traffic to the secondary region. This action must be taken with careful consideration to maintain service continuity.

4. **Replica Lag Acceptance**: 
   Monitor the replica lag for the RDS for SQL Server read replica in the secondary region. It must remain within an acceptable range for the business (Recovery Point Objective - RPO). Initiating a cross-region failover with non-zero replica lag can potentially lead to data loss, which is a significant risk to manage.


##Implementing the Failover:

With these considerations in place, it was time to implement the failover. I declared a disaster recovery event, and the first step was to disable writes on the RDS for SQL Server instance in the primary region. To avoid any accidental writes during the failover process, I stopped the application, ensuring that no data would be inadvertently modified.

I then turned my attention to the secondary region. Here, I verified the lag on the RDS for SQL Server read replica, ensuring it was under control. Confident in the status of the replica, I promoted it to become the new primary database. This was a pivotal moment in my recovery plan.

The next task was to upload the recovery file to the Amazon S3 bucket in the secondary region. This action was crucial because it would trigger a Route 53 health check failure for the primary region. Given that I had enabled the "invert health check status" option in Route 53, I knew that this failure would signal the health check agents that something was wrong.


![Security Groups](path_to_image/security_groups.png)


As expected, once the primary region was marked as unhealthy, Route 53 initiated the failover, redirecting internet traffic to the secondary region. I watched the process unfold, knowing that the delays in failover would depend on the health check settings, request intervals, and failure thresholds I had previously configured.

## Results:

Primary region service unavailable:

![Security Groups](path_to_image/security_groups.png)

Secondary region service up and running:

![Security Groups](path_to_image/security_groups.png)

## Conclusion


​The Multi-Region Database Deployment and Disaster Recovery Implementation project demonstrates the importance of resilience and availability in cloud-based infrastructures. By deploying an Amazon RDS instance with cross-region read replicas and utilizing Route 53 failover policies, the project ensures that critical data remains accessible even during regional outages or disasters. This architecture not only improves data redundancy and minimizes downtime but also provides a scalable and fault-tolerant solution that guarantees business continuity. Ultimately, this setup highlights best practices for disaster recovery and showcases the ability to build robust, highly available systems in AWS.
