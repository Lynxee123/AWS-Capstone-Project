# AWS-Capstone-Project
This is documentation of building a highly available, scalable web application using AWS' cloud services for Example XYZ University.


# VPC
First step, I needed to create a Virtual Private Cloud (VPC) to hold my infrastructure. To note, this infrastructure is placed in one region and uses two availability zones for fault tolerance. The web application instance also lies on two public subnets.

The VPC that I created has private subnets, but they are not utilized for simplicity. Using private subnets and a NAT gateway would have been more secure for the web application, but it’s complex, takes up too much time, and is costly.

After the VPC specifications were establishes, I had to create the VPC security group. I named it _Web Security Group_. Within _Inbound rules_, I allowed HTTP from any IPv4 source which would permit web requests to the website. Using my own security group rather than the default security group allowed me to customize what protocols were allowed to communicate with the web application. 
