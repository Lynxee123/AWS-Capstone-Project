# AWS-Capstone-Project
This is documentation of building a highly available, scalable web application using AWS' cloud services for Example XYZ University.


# VPC
First step, I needed to create a Virtual Private Cloud (VPC) to hold my infrastructure. To note, this infrastructure is placed in one region and uses two availability zones for fault tolerance. The web application instance also lies on two public subnets.

The VPC that I created has private subnets, but they are not utilized for simplicity. Using private subnets and a NAT gateway would have been more secure for the web application, but it’s complex, takes up too much time, and is costly.

After the VPC specifications were establishes, I had to create the VPC security group. I named it _Web Security Group_. Within _Inbound rules_, I allowed **HTTP from any IPv4** source which would permit web requests to the website. Using my own security group rather than the default security group allowed me to customize what protocols were allowed to communicate with the web application. 

# EC2
The next key part of the infrastructure was the computing power. I began by launching an EC2 instance labeled, _University Lab Instance_. I chose the **Ubuntu Amazon Machine Image** and chose the default **t2.micro** instance type and key pair. The university is small, so the free default options are suitable for the scenario. 

**EC2 Settings**
+ Networking Settings
    + Enabled Auto-assign public IP
    + Selected Web Security Group for the EC2 security group
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

