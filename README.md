# AWS-Capstone-Project
This is documentation of building a highly available, scalable web application using AWS' cloud services for Example XYZ University.

# Preview
![Capstone Project_ Building a Highly Available, Scalable Web Application (1)](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/6dfefa12-416c-40c5-8036-d31cf8695af4)
**This is what the end result will look like after everything is put together.**

# Cost Estimation
In this scenario, I have a $80 monthly budget. I'm building a really simple infrastructure that contains:
+ VPC
+ EC2
+ Load Balancing
+ Auto Scaling

A lot AWS' services have a free tier, so I went under the budget. Here is my estimated budget for entire build using AWS Pricing Calculator: [My Estimate - AWS Pricing Calculator.pdf](https://github.com/Lynxee123/AWS-Capstone-Project/files/11668595/My.Estimate.-.AWS.Pricing.Calculator.pdf)


# Creating a VPC
First step, I needed to create a Virtual Private Cloud (VPC) named _ExampleUniversity-vpc_ to hold my infrastructure. To note, this infrastructure is placed in one region and uses two availability zones (**us-east-1a** & **us-east-1b**) for fault tolerance. The web application instance also lies on two public subnets (public subnet 1 & public subnet 2).

The VPC that I created has private subnets, but they are not utilized for simplicity. Using private subnets and a NAT gateway would have been more secure for the web application, but it’s complex, takes up too much time, and is costly.

![vpc](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/c78ab337-3078-4c2e-9d39-2c8e9d50ffdd)

After the VPC specifications were establishes, I had to create the VPC security group. I named it _Web Security Group_. Within _Inbound rules_, I allowed **HTTP from any IPv4** source which would permit web requests to the website. Using my own security group rather than the default security group allowed me to customize what protocols were allowed to communicate with the web application. 

![security group](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/bc0a61ba-feb8-48ac-9de5-6aea4c6e49ac)



# Launching an EC2 Instance
The next key part of the infrastructure was the computing power. I began by launching an EC2 instance labeled, _University Lab Instance 1_ into availability zone us-east-1a. I chose the **Ubuntu Amazon Machine Image** and chose the default **t2.micro** instance type and key pair. The university is small, so the free default options are suitable for the scenario. 

**Other EC2 Settings:**
+ Networking Settings
    + Enabled Auto-assign public IP
    + Selected _Web Security Group_ for the EC2 security group
+ Advanced Details
    + Entered Example University's website script in the User data box:

``` 
#!/bin/bash -xe
apt update -y
apt install nodejs unzip wget npm mysql-server -y
#wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP1-1-DEV/code.zip -P /home/ubuntu
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCAP1-1-79581/1-lab-capstone-project-1/code.zip -P /home/ubuntu
cd /home/ubuntu
unzip code.zip -x "resources/codebase_partner/node_modules/*"
cd resources/codebase_partner
npm install aws aws-sdk
mysql -u root -e "CREATE USER 'nodeapp' IDENTIFIED WITH mysql_native_password BY 'student12'";
mysql -u root -e "GRANT all privileges on *.* to 'nodeapp'@'%';"
mysql -u root -e "CREATE DATABASE STUDENTS;"
mysql -u root -e "USE STUDENTS; CREATE TABLE students(
            id INT NOT NULL AUTO_INCREMENT,
            name VARCHAR(255) NOT NULL,
            address VARCHAR(255) NOT NULL,
            city VARCHAR(255) NOT NULL,
            state VARCHAR(255) NOT NULL,
            email VARCHAR(255) NOT NULL,
            phone VARCHAR(100) NOT NULL,
            PRIMARY KEY ( id ));"
sed -i 's/.*bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl enable mysql
service mysql restart
export APP_DB_HOST=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
export APP_DB_USER=nodeapp
export APP_DB_PASSWORD=student12
export APP_DB_NAME=STUDENTS
export APP_PORT=80
npm start &
echo '#!/bin/bash -xe
cd /home/ubuntu/resources/codebase_partner
export APP_PORT=80
npm start' > /etc/rc.local
chmod +x /etc/rc.local
```
For the other options when making the EC2 Instance, I kept default. Once the instance is launched, to check that the website is publicly accessible, I copied the IP address into a web browser. I knew it was successful when the web page popped up without any errors.

![XYZ image](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/77e90f7e-1395-4fed-bd23-9a5c98771cb4)


# Creating a Load Balancing
Even though the university is small, I still want to make sure the website will be accessible in any chance of high volumes of incoming traffic. To do so, I created an application load balancer that will split traffic between instances in two different availability zones.

**The steps I took to complete this:**
+ Creating another EC2 instance
    + Choose the **Launch more like this** action on the first instance created. 
    + Renamed the copy to _University Lab Instance 2_
    + Changed the availability zone to us-east-1b
+ Create an application load balancer
    + Named it _ExampleUniversityLB_
    + Selected us-east-1a and us-east-1b for availability zones
    + Selected the _Web Server Security Group_
    + In Listeners and routing panel: Created a new target group named _albTG_. This will allow me to select which instances that are available for traffic distribution.

For every other option, I kept default.


# Creating an Auto Scaling
Final component in the infrastructure is auto scaling. This will allow new instances to be automatically created to accommodate any potential incoming traffic. 

**The steps I took to complete this:**
+ Navigated to _Create launch template_
    + Enabled Auto Scaling guidance
    + AMI: Ubuntu
    + Instance type: t2.micro
    + Key pair name: vockey
    + Subnet: Don't include in launch template
    + Advanced network configuration:
        + Enabled Auto-assign public IP
        + Selected the _Web Server Security Group_
        + Enabled Delete on termination
    + Create launch template
+ Create Auto Scaling Group named _University Auto Scaling Group_
    +  Launched the previously made template
    +  Version: Latest
    +  Selected _ExampleUniversity-vpc_ for VPC
    +  Selected Public Subnet 1
    +  Set the:
        + Desired capacity to 1
        + Min capacity to 0
        + Max capacity to 4
    +  Create Auto Scaling group
    +  Attach the instances into the auto scaling group (in the EC2 section under Actions)

To check that I successfully implemented auto scaling, I navigated back to my EC2 instances and notice that a new instance has been started. 


# Testing
To make sure that the infrastructure I created was correctly built, I needed to put the web application under heavy load to see if:
1. EC2 instances distributes traffic between availability zones (i.e. load balancing works)
2. New EC2 instances are created when a certain percentage in CPU utilization is crossed (i.e. auto scaling works)

To do this, I needed to create an environment that is attached to the VPC and the availability zones. I can then put that environment under stress and if everything is correct, it should distribute traffic to the previously made instances and create new ones if a certain threshold is crossed (I configured it to 60%-65%). 

**The steps I took to complete this:**
+ Navigated to AWS Management Console, chose Cloud9
    + Created an environment named UniversityIDE
    + Environment type: New EC2 instance
    + Instance type: t2.micro
    + Connection: Secure Shell (SSH)
    + VPC: ExampleUniversity-vpc
    + Subnet: Public Subnet 1
 
For the other options, I kept default. 
 
 ![cloud9environment](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/5d398a50-075c-4857-b8e3-1a567c9b29f7)

 
Once the AWS Cloud9 environment is created, I performed a load test on the application by running scripts in the AWS Cloud9 terminal:

The following command installs the loadtest package to perform load testing on the application
```
npm install -g loadtest
```
This command simulates user load on the application.
- Replace with the load balancer DNS name
```
loadtest --rps 2000 -c 1000 -k http://<LoadBalancerDNS>
```
![scriptrun](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/b37af176-ec3c-4076-81a1-1045fe6ca4e6)

Once I ran this command, I had to wait about 10-15 mins until I saw results (primarily auto scaling) because the CPU utilization had to cross a certain point. As it ran the script, it also popped up warnings, but it was fine to ignore them. 

To know that my infrastructure was built correctly, I navigated to the EC2 console and observed the amount of instances. I noticed that they increased in quantity. To observe the load balancing, I clicked on one of the instances and monitored the CPU utilization. All instances seemed to have the same amount of traffic, which means it did distribute the traffic between availability zones. 

![cpuutil](https://github.com/Lynxee123/AWS-Capstone-Project/assets/117693278/8b93665b-d03c-414c-a523-afc2e7bcdbf2)

