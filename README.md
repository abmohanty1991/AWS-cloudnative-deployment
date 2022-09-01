
# Hosting a web application using AWS cloud Native resources

This project demonstrates how to host a sample Java application using AWS Cloud native resources.
The motive is to use as much AWS managed services as we can, so as to minimize the overhead of configuring and managing the resources.


# Setting the Environment right
Before going ahead with the project, we need folowing pre-requisites to be installed in our machine. I am using a Windows OS and am using Chocolatey(https://chocolatey.org/docs/installation) package installer. Depedning on your OS, you can carve a way out for following along.

    (a) Virtualbox : choco install virtualbox
    (b) Vagrant (for automation) : choco install vagrant
    (c) Git : choco install git
    (d) JDK : choco install jdk8
    (e) Maven : choco install maven
    (f) AWSCLI (for accessing AWS resources) : choco install awscli
    (g) Choice of your IDE


## AWS Services to be used

- **Route 53** - for DNS
- **Elastic Beanstalk** - For Application Load balancer, EC2 and ASG
- **Amazon MQ** - As a messging broker
- **Elasticache** - Caching
- **Amazon RDS(MySQL)** : Database
- **Cloudwatch** - for monitoring

## Architecture
![architecture](https://user-images.githubusercontent.com/108822178/187946019-3c1ff1e7-3137-4fa3-bed2-2f3160e00178.jpg)

## Execution flow

- Login to AWS account
- Create a keypair 
- Create a security group for backend resources i.e. Amazon MQ, Elasticache and RDS
- Provision Backend resources
- Create security group for front end resources
- Edit backend sg to allow internal traffic
- Edit backend sg to accept connection from frontend sg
- Create Elastic Beanstalk Environment
- Launch EC2 instance for initialising RDS
- Change healthcheck in Load Balancer of Elastic Beanstalk to the app url (suppose /login)
- Click stickiness in LB settings
- Add 443 listener to ELB to connect from Route 53
- Pull artifact from git and store it to S3.
- Deploy artifact to Beanstalk
- Update entry in the domain DNS
- Test and cleanup

# Step-1 : Keypair and security groups
- Login to your AWS account, open EC2 services. Go to Key Pairs. Generate a new key pair.
- Go to security groups and create security group for further utilisation by Amazon MQ, Elasticache and RDS
- In the inbound rules, just add any one rule for creation. It is because we will have to self refer this security group later.That means for enabling the resources inside this backend sg to communicate to each other, the sg needs to have inbound traffic allowed from itself.
![backend security grp](https://user-images.githubusercontent.com/108822178/187949928-bd4affed-6e12-4f7c-b5ee-3abd2f9781bb.jpg)

# Step-2 : Provision RDS

- Now before we create RDS, we need to create its own subnet group and parameter group. Subnets are segments of a VPC’s IP address range that you designate to group your resources based on security and operational needs.We can include all the three backend resources in single subnet group. Here we are creating different ones for each of them.
![rds_default_vpc_subnet_grp](https://user-images.githubusercontent.com/108822178/187949023-8b08664f-1481-40cf-9db9-56a7a1e66d05.jpg)
- We can select multiple subnets and AZs here

- Parameter groups (version selection is there, so select required version of MySQL here) present default parameters for setting the environment. If we are not very familiar, it is better to leave it as default.

- Go to Databases > Create Databases > MySQL
 
- Select Templates from Prod , Dev/Test (lets you configure things) and Free tier(autoselects all free tier settings)
![rds_dev_single_az setting](https://user-images.githubusercontent.com/108822178/187946910-c7794a8c-f974-4690-85c5-473e62a0f73b.jpg)
- **Availability** - Under this head, we can select whether to go for Single Instance or Multi Instance deployment of RDS, for high availability.
- Under the **Credentials** header, enter the username and password. **Caution!!!!** Please copy the username and password during this step and save it elsewhere. These credentials wont be visible again and would be required when we configure our application prior deployment.
![rds_copy_credentialsjpg](https://user-images.githubusercontent.com/108822178/187946699-31eaeb08-c205-46f4-b53b-2bce8ce346fb.jpg)
- Under **Instance Configuration**, we can choose use Burstable classes for keeping it to t2 class. Obviously, production environment needs will be much higher than dev and accordingly the instance capacity can be selected.
![rds_instance_capacity](https://user-images.githubusercontent.com/108822178/187947966-a60b59f2-812d-4ec8-8e95-29950d403d94.jpg)

- Attach the already created Backend security group to it.
- After the RDS is provisioned, copy the ARN and the port (**3306**) and save for future use.

# Step-4 : Provision Elasticache

- Elasticache isa  fully managed in-memory caching service.
- Like previous one, we need to create subnet group and parameter group first. 
![elasti_subnet_grp](https://user-images.githubusercontent.com/108822178/187948345-0ed6cabe-2b94-4795-b21a-fe2a0d4a91d3.jpg)

- While creating parameter group, we will have to select between any engine of either **Memcached** or **Reddis** 
![elasti_paragrp_created](https://user-images.githubusercontent.com/108822178/187948455-b57559e6-68d1-4309-ab08-e038d4c55fde.jpg)
- Creation of subnet is like before. 
- Click on create Elasticache cluster, now this presents two options, we can create either in Cloud or even on-prem.
- Port of Elasticache is **11211**. Save it
- Be careful while selecting the node capacity.
- Attach the created subnet group and select the backend security group.
![elasti_paragrp_created](https://user-images.githubusercontent.com/108822178/187948455-b57559e6-68d1-4309-ab08-e038d4c55fde.jpg)
- After the Elasticache is provisioned, copy the ARN and the port (3306) and save for future use.


# Step-5 : Provision Amazon MQ

- Amazon MQ is a managed message broker service for RabbitMQ and ActiveMQ. 
![mq_apache_vs_rabbit](https://user-images.githubusercontent.com/108822178/187948879-85340e0a-7f21-4ab5-a82a-84094fc94e1b.jpg)
- We can select multiple subnets and AZs here
- We get to select between a Single Broker or a Cluster broker, whereb a cluster of brokers are interconnected in a mesh spread over multi AZ for HA. Select single mq.
- **Caution!!!** Save the Username and password, as mentioned earlier also and save it for use in application config later.
- Attach the created subnet group and backend security group to it. 

# Step-6 : Initialise RDS using EC2 instance

- Check the MySQL database connectivity

![rds_connected](https://user-images.githubusercontent.com/108822178/187949701-dce1c55c-d125-40c0-ae71-eb24fe130849.jpg)
- The MySQL database we have provisioned required an initialisation and we can do that by launching an EC2 instance for it.
- We can choose Ubuntu AMI/ or any AMI 
- Update the backend sg to accept connection from the Security group of this sqlclient EC2.
![allow_ec2sqlclient_sg_to-_connect to ds](https://user-images.githubusercontent.com/108822178/187948153-47a57512-4449-43b0-8d39-0bc67ac49f8b.jpg)
- In user data, put following
```
#!/bin/bash
sudo apt update
sudo apt install mysql-client -y
```
- Keep the schema statements ready in a file, basically the create table, insert records commands etc in a file .
- SSH into the sqlclient EC2 and issue the above commands. After the sqlclient is installed, issue following command to initialise the mysql with the username and password
```
mysql -h rds-arn -u username -p password db_name < db_config.sql

```
- This will initialise the MySQL db and will create tables as mentioned in the db_config.sql file. SHow tables will display the tables created and validates that the DB is initialised.
![nitialise_db_show_tables](https://user-images.githubusercontent.com/108822178/187949795-2bcc20f1-0c74-4a84-a704-f95f28c151a0.jpg)

# Step-7 : Set up Elastic Beanstalk

- Elastic Beanstalk is a fully managed suite that provisions resources required for hosting any application such as Load balancer, EC2 instances, security groups, autoscaling methodologies and cloudwatch monitoring, without having to manage it.
- EB has application and inside that there are environments. One app can have multiple environments.
- Go to EB > create application ....
- Select platform as Tomcat as we are going to host the project in tomcat. 
![eb_setup](https://user-images.githubusercontent.com/108822178/187951299-46c94731-c2d2-4b09-95f9-adb06a635087.jpg)
- We can configure various transients in the EB such as min-max instances, instance capacity, Load balancer settings etc.
- We will attach the backend sg here, becasue no user is going to hit the EC2 instances directly.
- In Load balancer settings, we will have to change the health check path to the path of our website(suppose /home) and click stickiness. This will enable the users to go to the last connected instances.
- Under **Rolling Updates & Deployments** setting, we can select Rolling and the %, based on which the instances will be loaded with our app. There are other options like Immutable : A new stack of instances are launched and new app is hosted thr and also traffic slpitting : for red-blue deployment, between old and new version of the app.

![rolling_updates](https://user-images.githubusercontent.com/108822178/187953000-b832b59e-b4f0-4dad-8a16-ac98349e6b9a.jpg)

- When we create app, an application is created inside of which an environment is also created.

# Step-8 : Updating Security Group and LB

- Our Env is healthy now.
![bs_env](https://user-images.githubusercontent.com/108822178/187953189-c26522aa-c57b-4552-87d0-d9a422e6df02.jpg)
- Now we need to edit the backend sg inbound rules to accept connections from the frontend sg ie sg of the load balancer, ie the lb that we created while launching the EB.
- We can either mention three diff security rules with individual port numbers of Amazon MQ, Elasticache and RDS and say that traffic from the EB security gp is allowed to backend sec grp. This is the preferred way.
- Otehrwise we can mention all traffic fromthe eb security gp is allowed.
![add eb ec2 sec grp rule to backend sec grp](https://user-images.githubusercontent.com/108822178/187953992-0130f566-78f9-4456-98de-7441c404d17f.jpg)
- Edit the LB configuration again to see whether the health check is pointing to our corrected url or not and also the stickiness is enabled or not. If not, do the necessary correction.
- Add HTTPS listener at 443 and select the certificate(created during provisioning ur domain---godaddy etc)

# Step - 9 : Git clone, build and Deploy
- I am using the public repo of vproject for deployment. Clone it to the local directory using git clone
``` git clone https://github.com/devopshydclub/vprofile-project.git```
- There should be a application.properties in your aven project. If doing in Django, the necessary such contents will be part of settings.py file. Go to the respective section and paste the ARN, ports, username and password that we have copied earlier.
- Finally use ```mvn install``` to build the project in local machine.
- If build is successful, you will get a project.war file generated.
- In EB > Application versions > upload the war file.
- Deploy.

# Step - 10 : Release the resources
- You can go ahead and release the Elasticache, Amazon MQ, RDS, EB.

Hope this could help you enough to carry out a web application hosting using AWS native resources.
