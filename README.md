
The CloudFomation Stack used throughout this was created by Amazon Web Services. 

AWS Teams (2021) CloudFormation YAML Files & source code (Version 2.0) [Source code]. [Highly Available Web Application Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/3de93ad5-ebbe-4258-b977-b45cdfe661f1/en-US)

# Highly Available Web Application Project Lab


### Overview

 I will create a highly available (HA), scalable deployment of the  [WordPress](https://wordpress.org/) application stack. I will then deploy the WordPress application in such a way that the application server, database, and file server can scale independently of one another. I will also deploy the application's components into two availability zones to distribute it and guard against failure of any one availability zone. The WordPress application will be deployed in a stateless fashion so that I can add or remove web application servers in response to the requests flowing into the system.

To create this scalable, HA web application I will use various AWS services to create an architecture similar to the **full scale** reference architecture below. 

![full scale highly available Web Application](https://i.ibb.co/yRpCg9N/HA-Web-App.png)

The sequence of steps followed in building the architecture are as follows.

1.  Create a virtual network spread across multiple availability zones in your region of choice using  [Amazon Virtual Private Cloud](https://aws.amazon.com/vpc) .
2.  Deploy a highly available & fully managed relational database across those availability zones using  [Amazon RDS](https://aws.amazon.com/rds/) .
3.  After deploying the database the next step will be to create a database caching layer using  [Amazon ElastiCache](https://aws.amazon.com/elasticache/) . This will provide a cache around the database for frequently run queries, improving HTTP response time performance and reducing strain on the database instantiated in the previous step.
4.  For the application tier, I'll provision a shared storage layer powered by NFS. Using  [Amazon EFS](https://aws.amazon.com/efs/) create an NFS cluster across multiple availability zones.
5.  I'll create a load-balanced grouped of web servers that will automatically scale in response to load variations addressed to the application tier.
6.  I will then connect the application to a Content Delivery Network using amazon CloudFront to minimise the load on the application origin server and reduce latency.

# Part 1: Configure the Network

> Amazon Virtual Private Cloud (Amazon VPC) is a service that lets you launch AWS resources in a logically isolated virtual network that you define. You have complete control over your virtual networking environment, including selection of your own IP address range, creation of subnets, and configuration of route tables and network gateways.

### Here is what I will create in this section
![section1](https://i.ibb.co/ZKV30SJ/SECTION1-VPC.png)


### Launch CloudFormation Stack
I will use the CloudFormation stack to create a virtual network spread across multiple availability zone, in my region of choice with a private Subnet for Database and Public subnet for public access.
 
[Launch VPC Stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module1-vpc&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module1.yaml) *The CloudFormation Stack will create a VPC, Public/Private Subnets in two Availability Zones, and an Internet Gateway and two NAT Gateways. It will then handle association and routing of gateways and subnets.*

![Specify stack details](https://i.ibb.co/6sGLtzP/image.png)

the stack has already prepopulated all the field for me and I just made a few modifications and then I created the stack. 
![creation progress](https://i.ibb.co/tmrgNss/image.png)

After a few minute the CloudFormation Stack was ready and showed a status of *CREATE_COMPLETE*.
![enter image description here](https://i.ibb.co/DL79QDP/image.png)

## Verifying my configuration


**VPC created**
![enter image description here](https://i.ibb.co/7K0pHvx/vpc-verification.png)

**VPC Subnets created**
![enter image description here](https://i.ibb.co/jbLnxPv/image.png)

**VPC Internet Gateway created**
![enter image description here](https://i.ibb.co/crRPhY6/image.png)

**VPC NAT Gateway created**
![NAT Gateway](https://i.ibb.co/FJYLZDD/image.png)

### What I have done so far 
I have created a virtual private cloud network across two availability zones within an AWS region. I have created six subnets, three in each availability zone, and have configured a route so that the internet can communicate with resources in the public subnets and vice versa. The application subnets have been configured, via routing table, to communicate with the internet via NAT gateways in the public subnets, and the data subnets can only communicate with resources in the six subnets, but not the internet.

### Next 
Now I am ready to move to the next section where I will deploy a highly available database for use by WordPress.

# Part 2: Build the Data Tier

> Now that I have created a virtual network across multiple data centers I will create a resilient, cached, highly-available data tier designed to support my WordPress installation. To do this I will, over the next 3 sections, create an active / passive database deployment using  [Amazon Relational Database Service](https://aws.amazon.com/rds/) (RDS) and then add caching to the database using the managed caching service  [Amazon ElastiCache](https://aws.amazon.com/elasticache/)

## Set up RDS database
Amazon Relational Database Service (Amazon RDS) is a web service that makes it easier to set up, operate, and scale a relational database in the AWS Cloud. It provides cost-efficient, resizable capacity for an industry-standard relational database and manages common database administration tasks. Amazon Aurora is a MySQL and PostgreSQL-compatible relational database built for the cloud, that combines the performance and availability of traditional enterprise databases with the simplicity and cost-effectiveness of open source databases.


### Here is what I will create in this Section
![database tier](https://i.ibb.co/L6xC0gP/section2.png)
I will deploy a highly available relational database across availability zones and in DB Subnets using Amazon RDS. I will use CloudFormation to automate the resource deployment. 

### Launch CloudFormation Stack
I will use the CloudFormation stack to create  MySQL database using Amazon Aurora in RDS in multiple availability zones, as well as security groups and subnet groups for communication to a client server.

[Launch Stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module2-rds&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module2.yaml)

![enter image description here](https://i.ibb.co/yqvDTVN/image.png)

-  Under  **Parameters**  section:  I Inputted the  **Database Username**  and  **Database Password**
-  I Selected the two database subnets  **James-O-Wordpress-Project Data Subnet A (AZ1)**  and  **James-O-Wordpress-Project Data Subnet B (AZ2)** and then i provisioned the stack. 

**Creation in progress** 
![enter image description here](https://i.ibb.co/9YSVVLt/image.png)

**Creation in complete** after a few minutes the CloudFormation Stack will show status of _CREATE_COMPLETE_.
![enter image description here](https://i.ibb.co/qjr4FY2/image.png)

I copied the value for the the key named **DatabaseClusterEndpoint** in the output tab into a notepad as the value will be required in future section.

> A cluster endpoint (or writer endpoint) for an Aurora DB cluster connects to the current primary DB instance for that DB cluster.

![enter image description here](https://i.ibb.co/7Y424FQ/image.png)

## Verifying my configuration
My active / passive database should now be available and running in two different availability zones, waiting for connections from any EC2 resource with the client security group associated to it.

**RDS**
I clicked on the **DatabaseCluster** _Physical ID_ within the **Resources** tab to open the RDS console in a new tab 
![enter image description here](https://i.ibb.co/0GkhXqJ/image.png)

You can see the newly created Aurora RDS MySQL Highly available, Multi-AZ Cluster.
- The _Writer_ and a _Reader_ Instance are in different Availability Zones
- The Cluster Endpoint  points to the current Writer instance.
- The Reader Endpoint points to the Reader instance.		

![enter image description here](https://i.ibb.co/3yp4Wq4/image.png)	 

**Security Group**
I clicked on the **WordPressDatabaseClientSG** _Physical ID_ within the **Resources** tab to open the RDS console in a new tab 
![enter image description here](https://i.ibb.co/xmx6t9F/image.png)
I Select the Security Group with name **WP Database SG**. This is the security group for the Database cluster I just created.

-   The  **Inbound rules**  for the security group only allow TCP connection on port 3306 from  **WP Database Client SG**  Security Group.
-   Only Instance/service which are part of  **WP Database Client SG**  Security Group will have access to the Database.
-   The application servers I will create in Section 7 will be assigned  **WP Database Client SG**  to allow access to Database.


![enter image description here](https://i.ibb.co/NLqSTF0/image.png)

## Next step 
WordPress uses its database to store articles, users, and configuration information. This can result in a lot of unnecessary queries against the database which consistently return the same data set. To offset the strain on the database and improve performance of the WordPress web application I'll add a caching layer for common SQL requests.

## Set up Elasticache for Memcached

> Amazon ElastiCache allows you to seamlessly set up, run, and scale popular open-source compatible in-memory data stores in the cloud. Amazon ElastiCache is a popular choice for real-time use cases like Caching, Session Stores, Gaming, Geospatial Services, Real-Time Analytics, and Queuing. Amazon ElastiCache offers fully managed Redis and Memcached for your most demanding applications that require sub-millisecond response times.

### Here is what you will create in this Module
![enter image description here](https://i.ibb.co/xF1PP4W/section3.png)

### Launch CloudFormation Stack
I will use the  CloudFormation stack to create a Memcached cluster, then create a client and a server security group to protect the Memcached cluster.

[Launch stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module3-elasticache&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module3.yaml)

In **Specify stack details**

-   Under  the **Parameters**, I selected the Data Tier subnets:  **Wordpress- Data Subnet A (AZ1)**  and  **Wordpress- Data Subnet B (AZ2)**  from the drop down.

![enter image description here](https://i.ibb.co/3k7H0XN/image.png)

**Creation in progress**
![enter image description here](https://i.ibb.co/BtBqtXV/image.png)

**Creation in complete**
![enter image description here](https://i.ibb.co/3dB6Lhw/image.png)
## Verifying my configuration
**ElasticCache Security Groups**
Under the **Resources** tab I clicked on the **ElastiCacheClientSecurityGroup** _Physical ID_ to open EC2 console in the new tab.
![enter image description here](https://i.ibb.co/cw7RyMP/image.png)
1.   Two security groups,  _WP Cache Client SG_  and  _WP Cache SG_, was created in this section.
2.  within the  **WP Cache SG**
    -   into the  **Inbound Rules**  of the  _WP Cache SG_  security group, it has a rule with type  `Custom TCP Rule`  allowing traffic on port  `11211`  from the  _WP Cache Client SG_.
    -   Only Instance/service which are part of  **WP Cache Client SG**  Security Group will have access to the Cache cluster.
    -   The application servers I will create in module7 will be assigned  **WP Cache Client SG**  to allow access to Cache cluster.

![enter image description here](https://i.ibb.co/Rgpk8NZ/image.png)

**Memcached setup**
Under the **Resources** tab I clicked on the **WordPressDatabaseClientSG,** _Physical ID_ to open ElastiCache console in the new tab.

![enter image description here](https://i.ibb.co/Cthq6C3/image.png)

**Memcached** cluster that was created in this Section. is now showing a status of _Available_.

![enter image description here](https://i.ibb.co/pbYj7JM/image.png)

## Next Steps
I have now created a resilient active / standby database deployment with a caching layer that is distributed across multiple data centers.

In the next set of labs I will build the application tier which will deploy Wordpress to begin using the data tier and serving customers.

# Part 3: Create the shared filesystem

> Amazon Elastic Filesystem (EFS) provides a simple, serverless, set-and-forget, elastic file system that lets you share file data without provisioning or managing storage. It can be used with AWS Cloud services and on-premises resources, and is built to scale on demand to petabytes without disrupting applications. With Amazon EFS, you can grow and shrink your file systems automatically as you add and remove files, eliminating the need to provision and manage capacity to accommodate growth.

### Here is what I will create in this section 
![enter image description here](https://i.ibb.co/r42Zcg6/section4.png)
### Launch CloudFormation Stack

I will use the CloudFormation stack to create an Elastic File System along with two Mount Targets for your Application servers. It will also create a client and a server security group to protect the provisioned Elastic File System.

[Launch Stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module4-efs&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module4.yaml)

** Creation in progress **
![enter image description here](https://i.ibb.co/xDJnw92/image.png)

** Creation in complete **
![enter image description here](https://i.ibb.co/XysWQ1h/image.png)
## Verifying my configuration

**Filesystem security groups**
Under the **Resources** tab I clicked on the ****EFSSecurityClientGroup,** _Physical ID_ to open ElastiCache console in the new tab.
![enter image description here](https://i.ibb.co/T2jfhJ3/image.png)
 
1.   The file system has two security groups,  _WP FS Client SG_  and  _WP FS SG_, created in this module.
2.  The  **WP FS SG** 
    -   within the **Inbound Rules**  of the  _WP FS SG_  security group, it has one rule with type  `Custom TCP Rule`  allowing traffic on port  `2049`  from the  _WP FS Client SG_.
    -   Only Instance/service which are part of  **WP FS Client SG**  Security Group will have access to the Cache cluster.
    -   The application servers I will create in section 7 will be assigned  **WP FS Client SG**  to allow access to Cache cluster.

![enter image description here](https://i.ibb.co/VVCmdpM/image.png)

**Filesystem cluster**
![enter image description here](https://i.ibb.co/cbWrVxh/image.png)
![enter image description here](https://i.ibb.co/wSxyn0M/image.png)

1.  under the  _Network_  tab
    -   The two  _Mount targets_, each one resides in a subnet that was created in section1 for the application tier (application subnet A and B).
    -   The  _Security groups_  for mount targets, they are associated with the  _WP FS SG_  security group.

![enter image description here](https://i.ibb.co/2jchjk0/image.png)

## Next step
I now have an EFS cluster ready and available for use.
in the next section I will create the application servers that will connect to and use these resources to serve clients.

# Part 4: Building the Application Tier

I  have created a shared filesystem, a central highly available database, and a caching layer to improve application response times.

It is now time to deploy the WordPress application itself in a highly available fashion. To do this I will create an EC2 AutoScaling group which will add and remove servers to a fleet of application servers in response to network traffic. I will also deploy a load balancer to distribute traffic across these instances as they are added and removed. The load balancer will provide a single endpoint for the users even as the backend application servers are scaled according to user traffic demands.

## Create the app server

> Amazon EC2 Auto Scaling group helps you maintain application availability and allows you to automatically add or remove EC2 instances according to conditions you define.

### Here is what I will create in this section

![enter image description here](https://i.ibb.co/vq034wB/section5.png)

### Launch CloudFormation Stack
I will use the  CloudFormation stack to create an Elastic File System along with two Mount Targets for the Application servers. the stack will also create a client and a server security group to protect the provisioned Elastic File System.

[Launch stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module5-elb&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module5.yaml)

In the **Specify stack details**
-   Under  **Parameters**, I will be launching the Application Load Balancer in the Public subnets: selecting  **James-O-Wordpress-Project Public Subnet A (AZ1) & Public Subnet B (AZ2)**  from the drop down.

![enter image description here](https://i.ibb.co/DbfPvy2/image.png)

**Creation in progress** 
![enter image description here](https://i.ibb.co/XW24jxK/image.png)

**Creation complete**
![enter image description here](https://i.ibb.co/VWJNKy7/image.png)

## Verifying Configuration

**Application Load Balancer Security group**

 On the  **Resources**  tab,  I will  click on the  _Physical ID_ of the **WPLoadBalancerSecurityGroup**,  to open EC2 console in the new tab.

![enter image description here](https://i.ibb.co/CQNmTrD/image.png)

1.  As seen below the security group,  _WPLoadBalancerSecurityGroup_  has been created in this section.
       -   under the  **Inbound Rules** tab  of the security group, it has one rule allowing traffic on port 80.  allowing global public access from port 0.0.0.0/0.
    -   I would normally restrict the inbound connections to the load balancer from my workstation only, by updating update the  **Inbound Rules**  to set the Source IP to my workstation for tighter security control.
     
![enter image description here](https://i.ibb.co/wW9m0Nk/image.png)

**Application Load Balancer Security group**
- I will copy  **DNS name**  of the load balancer,  as it will be required in next section.
-   The load balancer has load balancer nodes in two availability zones, in the public subnets I selected during the creation step.

![enter image description here](https://i.ibb.co/qMjHhVm/image.png)

Under the listener tab you can see the listener that is created for the ALB.
 -   The listener is configured to listen for http requests on port 80 and forward the request to a  **Target Group**
> A listener checks for connection requests from clients, using the protocol and port that you configure. The rules that you define for a listener determine how the load balancer routes requests to its registered target.

![enter image description here](https://i.ibb.co/pfZ8GFv/image.png)

**Target Group**

 > Each target group routes requests to one or more registered targets, such as EC2 instances.

![enter image description here](https://i.ibb.co/Dr2DCjf/image.png)

## Next step 
I have now a created a public facing load balancer.
In the next section I will create the Launch configuration which will be used to launch WordPress servers.

# Part 5: Create a launch configuration

> A launch configuration is an instance configuration template that an Auto Scaling group, which I will create in next section, uses to launch EC2 instances. When you create a launch configuration, you specify information for the instances, Including the ID of the Amazon Machine Image (AMI), the instance type, one or more security groups, and a block device mapping. It is similar to the information you provide when you launch and EC2 instance.

### Here is what I will create in this section 
 So far I have created a virtual private network in AWS, spanning across multiple fault-isolated data centers. Deployed an Aurora MySQL database with a primary and secondary Database in two different Availability Zones as well as a managed Memcached instance, and a distributed NFS cluster for shared storage. In this Section, I will create the Launch Configuration for launching EC2 instances.
 
[launch Stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module6-launchconfig&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module6.yaml)

The CloudFormation Stack will create a Launch Configuration that I will use to launch EC2 instances. It will also create a security group to protect the provisioned EC2 instances.
![enter image description here](https://i.ibb.co/zVTsW49/image.png)
In **Specify stack details**

-   Under  **Parameters**, the Wordpress servers launched will be connecting to the MySQL database cluster I created in section2. I Provided the DB parameters to the template.
    
    -   **DB name** = `wordpress`
    -   **Database Endpoint** = `section2-rds-databasecluster-3wwrqvg41agk.cluster-copnrtvs7ubp.us-east-1.rds.amazonaws.com`
        *This is the cluster endpoint of my database, which points to the Writer instance.* 
        
    -   **Database User Name** = `admin`
        This is the database username I specified in section2. 
        
    -   **Database Password** = `********`
        This is the password for the database I created in section2.
      
-   I will also be installing and configuring WordPress, so I provided a wordpress parameters to the template.
    
    -   **admin username** = `jok01`
        admin user for wordpress management.
        
    -   **admin password**  = `********`
    - password for the admin user.
and finally I provisioned the stack. 

**Creation in progress**
![enter image description here](https://i.ibb.co/BLvQVmY/image.png)

**Creation complete**
![enter image description here](https://i.ibb.co/zhzXFfR/image.png)

## Verify  Configuration
I will click on the _Physical ID_ of the **WordPressLaunchConfig**, under the **Resources** tab, to open a new EC2 console in the new tab.
![enter image description here](https://i.ibb.co/ph8G9xy/image.png)

-   Launch configuration page shows the  _AMI ID_,  _Instance Type_  and  _Security Groups_  for the instance that will be launched using this Launch Configuration.
-   You will notice that I have provided a User Data to the launch config.

> When you launch an instance in Amazon EC2, you have the option of passing user data to the instance that can be used to perform common automated configuration tasks and even run scripts after the instance starts. You can pass shell scripts as User data.


![enter image description here](https://i.ibb.co/f8MpVtJ/image.png)

 I want to **view user data**  to verify the script that will run when the instance is launch to do instance configuration for WordPress Instance launched using this launch config.

 -   I am installing PHP and WordPress
 -   Mounting the NFS mount point
 -   Configuring WordPress
 -   Configuring and enabling OPcache (*OPcache is a caching engine built into PHP. When enabled, it **dramatically increases the performance of websites that utilize PHP**.*)

![enter image description here](https://i.ibb.co/QHR2C5F/image.png)

 ````bash 
 #!/bin/bash -xe            
DB_NAME=wordpress
DB_HOSTNAME=section2-rds-databasecluster-3wwrqvg41agk.cluster-copnrtvs7ubp.us-east-1.rds.amazonaws.com
DB_USERNAME=admin
DB_PASSWORD=Victory01
WP_ADMIN=jok01
WP_PASSWORD=Victory01
LB_HOSTNAME=Secti-WordP-9VF5JIEJRI0A-19248653.us-east-1.elb.amazonaws.com
yum update -y
yum install -y awslogs httpd24 mysql56 php70 php70-devel php7-pear php70-mysqlnd gcc-c++ php70-opcache
mkdir -p /var/www/wordpress
mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 fs-06aa933d3ea2a02ef.efs.us-east-1.amazonaws.com:/ /var/www/wordpress

## create site config
cat <<EOF >/etc/httpd/conf.d/wordpress.conf
ServerName 127.0.0.1:80
DocumentRoot /var/www/wordpress/wordpress
<Directory /var/www/wordpress/wordpress>
  Options Indexes FollowSymLinks
  AllowOverride All
  Require all granted
</Directory>
EOF

## install cache client
ln -s /usr/bin/pecl7 /usr/bin/pecl #just so pecl is available easily
pecl7 install igbinary
wget -P /tmp/ https://s3.amazonaws.com/aws-refarch/wordpress/latest/bits/AmazonElastiCacheClusterClient-2.0.1-PHP70-64bit.tar.gz
cd /tmp
tar -xf '/tmp/AmazonElastiCacheClusterClient-2.0.1-PHP70-64bit.tar.gz'
cp '/tmp/artifact/amazon-elasticache-cluster-client.so' /usr/lib64/php/7.0/modules/
if [ ! -f /etc/php-7.0.d/50-memcached.ini ]; then
    touch /etc/php-7.0.d/50-memcached.ini
fi
echo 'extension=igbinary.so;' >> /etc/php-7.0.d/50-memcached.ini
echo 'extension=/usr/lib64/php/7.0/modules/amazon-elasticache-cluster-client.so;' >> /etc/php-7.0.d/50-memcached.ini

## install wordpress
if [ ! -f /bin/wp/wp-cli.phar ]; then
  curl -o /bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
  chmod +x /bin/wp
fi
# make site directory
if [ ! -d /var/www/wordpress/wordpress ]; then
  mkdir -p /var/www/wordpress/wordpress
  cd /var/www/wordpress/wordpress
  # install wordpress if not installed
  # use public alb host name if wp domain name was empty
  if ! $(wp core is-installed --allow-root); then
      wp core download --version='4.9' --locale='en_GB' --allow-root
      wp core config --dbname="$DB_NAME" --dbuser="$DB_USERNAME" --dbpass="$DB_PASSWORD" --dbhost="$DB_HOSTNAME" --dbprefix=wp_ --allow-root
      wp core install --url="http://$LB_HOSTNAME" --title='Wordpress on AWS' --admin_user="$WP_ADMIN" --admin_password="$WP_PASSWORD" --admin_email='admin@example.com' --allow-root
      wp plugin install w3-total-cache --allow-root
      # sed -i \"/$table_prefix = 'wp_';/ a \\define('WP_HOME', 'http://' . \\$_SERVER['HTTP_HOST']); \" /var/www/wordpress/wordpress/wp-config.php
      # sed -i \"/$table_prefix = 'wp_';/ a \\define('WP_SITEURL', 'http://' . \\$_SERVER['HTTP_HOST']); \" /var/www/wordpress/wordpress/wp-config.php
      # enable HTTPS in wp-config.php if ACM Public SSL Certificate parameter was not empty
      # sed -i \"/$table_prefix = 'wp_';/ a \\# No ACM Public SSL Certificate \" /var/www/wordpress/wordpress/wp-config.php
      # set permissions of wordpress site directories
      chown -R apache:apache /var/www/wordpress/wordpress
      chmod u+wrx /var/www/wordpress/wordpress/wp-content/*
      if [ ! -f /var/www/wordpress/wordpress/opcache-instanceid.php ]; then
        wget -P /var/www/wordpress/wordpress/ https://s3.amazonaws.com/aws-refarch/wordpress/latest/bits/opcache-instanceid.php
      fi
  fi
  RESULT=$?
  if [ $RESULT -eq 0 ]; then
      touch /var/www/wordpress/wordpress/wordpress.initialized
  else
      touch /var/www/wordpress/wordpress/wordpress.failed
  fi
fi 


## install opcache
# create hidden opcache directory locally & change owner to apache
if [ ! -d /var/www/.opcache ]; then                    
    mkdir -p /var/www/.opcache
fi
# enable opcache in /etc/php-7.0.d/10-opcache.ini
sed -i 's/;opcache.file_cache=.*/opcache.file_cache=\/var\/www\/.opcache/' /etc/php-7.0.d/10-opcache.ini
sed -i 's/opcache.memory_consumption=.*/opcache.memory_consumption=512/' /etc/php-7.0.d/10-opcache.ini
# download opcache-instance.php to verify opcache status
if [ ! -f /var/www/wordpress/wordpress/opcache-instanceid.php ]; then
    wget -P /var/www/wordpress/wordpress/ https://s3.amazonaws.com/aws-refarch/wordpress/latest/bits/opcache-instanceid.php
fi

chkconfig httpd on
service httpd start
  
````

**Security Group**

I will click on Physical ID for _WebTierSecurityGroup_ under **Resources** tab of the provisioned stack for this section to open a new EC2 page.
![enter image description here](https://i.ibb.co/zQ3rCHw/image.png)

the _Wordpress Servers Security Group_ only allows incoming requests from Application Load Balancer Security group. Only Application load balance can communicate with the Wordpress servers

![enter image description here](https://i.ibb.co/zNYp4HH/image.png)

## Next Step 
I have now created a launch configuration, which will be used by auto-scaling group while launching instances.
In the next section I will create the Auto Scaling group.

# Part 6: Create the app server

> Amazon EC2 Auto Scaling group helps you maintain application availability and allows you to automatically add or remove EC2 instances according to conditions you define.

### Here is what I will create in this Section 
![enter image description here](https://i.ibb.co/5FsGbvC/section7.png)

I will use CloudFormation to automate the resource deployment. The CloudFormation Stack will create an Auto Scaling group for your EC2 instances. This will also include Amazon EC2 Auto Scaling features such as health check replacements and scaling policies.

[Launch Stack](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=module7-autoscalinggroup&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module7.yaml)

**Creation in progress**

![enter image description here](https://i.ibb.co/HHfC6H9/image.png)

**Creation in complete**
![enter image description here](https://i.ibb.co/B3R9NcT/image.png)

## Verify Configuration 

**Auto Scaling groups**
  I will click on the _Physical ID_ for **AutoScalingGroup**, on the **Resources** tab, to open EC2 console in the new tab.

![enter image description here](https://i.ibb.co/23NwRdz/image.png)

The created Auto Scaling Group has desired(4), minimum(4) and maximum(8) capacity.

![enter image description here](https://i.ibb.co/TqgF03X/image.png)

![enter image description here](https://i.ibb.co/74q0BCk/image.png)

````
**Automatic Scaling** of *section7-AutoScalingGroup*. has a scaling policy named something like *section7-ScalingPolicy* with Execute policy "As required to maintain Average CPU utilization at 80".
````
![enter image description here](https://i.ibb.co/f4xQ5Js/image.png)

**Target Group**
````
- The **Targets** of the *secti-WPAlb*.shows the registered targets status of the instances linked to ALB as **healthy**
````
![enter image description here](https://i.ibb.co/J5dCCvv/image.png)

**Load Balancer**
I am going to get the load balancer DNS name

![enter image description here](https://i.ibb.co/qySCLss/image.png)

Copied and pasted the DNS name `Secti-WordP-9VF5JIEJRI0A-19248653.us-east-1.elb.amazonaws.com`into a browser  and the result is the website shown below. 

![enter image description here](https://i.ibb.co/0Dhyh9G/Screenshot-2022-05-17-20-49-09.png)

![enter image description here](https://i.ibb.co/GTJKh4L/image.png)

## Next Step 
I have now created a highly-available auto-scaling deployment of WordPress that will scale in & out in response to client traffic hitting the website.

Next I am going to configure the WordPress caching.


# Part 7: Speed up the Website
I have create a highly available, multi tier WordPress application hosting platform.

In this section, I will apply techniques to improve the performance of the application by Caching Database responses and a Content Delivery Network.
 
 ## Configure Caching
 
 I created an [Elasticache for Memcached](https://aws.amazon.com/elasticache/memcached/) instance in Section 3. In this section I will configure WordPress to use that service. To improve the site's response time and to reduce the strain on your backend database, WordPress is configured to use Memcached as a caching layer for common requests. 

I will paste in the DNS name from the load balancer into brower so that I can login wordpress`Secti-WordP-9VF5JIEJRI0A-19248653.us-east-1.elb.amazonaws.com`

![enter image description here](https://i.ibb.co/3prMw2Z/image.png)

![enter image description here](https://i.ibb.co/5WHF2hJ/image.png)

After login into wordpress, on the left menu I will locate the **plugins**, and click on **Activate for W3 Total Cache**

![enter image description here](https://i.ibb.co/VxNfYVP/image.png)

In the left menu I'll select **Performance**, and then click on  _Accept_  for cache setup.
![enter image description here](https://i.ibb.co/QKnBzdJ/image.png)

Then I will click on skip at the bottom of the page and move o to database cache setup step.
![enter image description here](https://i.ibb.co/xf3HRg2/image.png)

In the left menu under **Performance**, I will click on **General Settings**. followed by **Database Cache**. Under this section, I will check the Database Cache _Enable_ box, and select _Memcached_ as Database Cache Method.
![enter image description here](https://i.ibb.co/K7yKL2W/image.png)

![enter image description here](https://i.ibb.co/2gfVPW2/image.png)

1. I will then open the  **[AWS ElastiCache Console](https://console.aws.amazon.com/elasticache)** link in a new tab and log in to my AWS account. This will open ElastiCache console in the new tab. In the left menu I will click on  **Memcached**  and select  _wp-elasticache_  cluster then copy the configuration endpoint for  _wp-elasticache_  to a clipboard.

![enter image description here](https://i.ibb.co/HxHrNVz/image.png)

![enter image description here](https://i.ibb.co/hYGj5ss/image.png)
1.  Go back to the WordPress management tab. In the left menu I select  **Performance**, then click on  _Database Cache_  and replace the *Memcached hostname:port/Ip:port* with the value copied in the previous step. I'll then click on the "Test" button to confirm the connection is successfully made and "Test Passed" message is displayed.
![enter image description here](https://i.ibb.co/BKzW1rQ/image.png)

Then I 'll scroll down to the bottom of the page and click on  _Save all settings_  on the left. The following message was displayed to indicate Database caching has been enabled.

![enter image description here](https://i.ibb.co/3SFXFy9/image.png)

### Add Content to the Website 
I will add dummy content to the website, by installing [WordPress Importer Plugin](https://wordpress.org/plugins/wordpress-importer/)  and then cloning and importing [Theme Unit Test codex](https://codex.wordpress.org/Theme_Unit_Test) into the wordpress. 

![enter image description here](https://i.ibb.co/pygzmyz/image.png)

After importing theme test the site respond time is lighting fast and  content loads up effortlessly on page refresh/reload testing. 
![enter image description here](https://i.ibb.co/4j7CSN6/image.png)
 
 Ran a quick test on the load balancer DNS `http://secti-wordp-9vf5jiejri0a-19248653.us-east-1.elb.amazonaws.com/` on [GTMetric](https://gtmetrix.com/) and the overall performance of the site was ok.  
![enter image description here](https://i.ibb.co/Sr0jnvn/image.png)

Then I ran another speed test on the load balancer DNS from  [pingdom](tools.pingdom.com) site and the result is good. 

![enter image description here](https://i.ibb.co/cbPWg6n/image.png)

# Part 8: Add Content Delivery Network

> Amazon CloudFront can speed up the delivery of your websites, whether its static objects (e.g., images, style sheets, JavaScript, etc.) or dynamic content (e.g., videos, audio, motion graphics, etc.), to viewers across the globe. The CDN offers a multi-tier cache by default that improves latency and lowers the load on origin servers when the object is not already cached at the Edge.

### Here is what I will create in the section
![enter image description here](https://i.ibb.co/1qnfCcL/section9.png)
In the previous section, I added caching to the WordPress configuration to cache frequently accessed database queries. In this section I will add a content delivery network using [Amazon CloudFront](https://aws.amazon.com/cloudfront/) to speed up global delivery of the application.

[launch Stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=module9-cloudfront&templateURL=https://ee-assets-prod-us-east-1.s3.amazonaws.com/modules/402582136e8b4a63866212aa6dc0e6e3/v1/module9.yaml)
The CloudFormation Stack will create a CloudFront distribution with the Elastic Load Balancer as the origin. This load balancer was created in Section 5 earlier.

**Creation in progress**
![enter image description here](https://i.ibb.co/rykqT2d/image.png)

**Creation complete**
![enter image description here](https://i.ibb.co/Hpz53Jg/image.png)

From the CloudFormation template execution Outputs, I note the value of the key "DnsHostname" which represents the  **CloudFront distribution endpoint** and will be used to update the WordPress configuration as the  **Site Address (URL)**.
![enter image description here](https://i.ibb.co/D52W2F2/image.png)

I will Login back into the WordPress Admin console using the Elastic Load Balancer DNS hostname (not the CloudFront one) from section 5 and then navigate to Settings > General tab. I will Only replace the the  **Site Address (URL)**  with the DNS hostname copied in the previous step and then save the changes.

![enter image description here](https://i.ibb.co/jrfXkk5/image.png)

I will now browse to the CloudFront DNS name to view the public facing WordPress site I created in this section. The CloudFront distribution caches the content , improves latency for users across the globe and lowers the load on origin WordPress servers.

![enter image description here](https://i.ibb.co/9s0CDYs/Screenshot-2022-05-18-02-05-45.png)


## Wrapping Up

In this project lab I used numerous AWS services to deploy a simple web application in a highly available fashion. 

# Summary 

During this project I used AWS VPC to create a software-defined network across multiple AWS availability zones in a single AWS region.

Using Amazon RDS I created a fully managed active / standby multi-node MySQL database deployment. This database is automatically backed up and recoverable to within the last 5 minutes and is also patched on my behalf. In the event of a failure the database will automatically failover to the standby node to which data is synchronously replicated with the active node, ensuring I never lose my data.

I also created a Memcached-powered cache using Amazon ElastiCache. This allows me to cache frequent database queries and reduce the number of queries I have to make against my HA MySQL database.

Then I created a distributed NFS share using Amazon Elastic Filesystem (EFS) to share a single WordPress installation across multiple application servers.

Using AWS auto scaling and the AWS Application Load Balancer I created a collection of virtual machines that will automatically scale in and out based on resource utilization and in response to the number of users hitting my application server. As new servers come online the load balancer will automatically get updated with their information and route traffic them. Similarly, as servers are removed the auto scaling group will update the load balancer configuration and drain connections from those servers.

Finally, I added AWS CloudFront a global content delivery network. CloudFront speeds up the delivery of my websites, whether its static objects (e.g., images, style sheets, JavaScript, etc.) or dynamic content (e.g., videos, audio, motion graphics, etc.), to viewers across the globe.

The outcome of this modular architecture design is a highly-available, distributed, and fault tolerate web application. 
 
 
AWS Teams (2021) CloudFormation YAML Files & source code (Version 2.0) [Source code]. [Highly Available Web Application Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/3de93ad5-ebbe-4258-b977-b45cdfe661f1/en-US)