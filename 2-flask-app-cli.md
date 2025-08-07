
## Overview

This project builds on Part 1, which covered deploying a basic EC2 instance and NGINX server. If you haven’t completed that, please refer to **[1. EC2 Web Server Deployment]** first. In this project, you will automate the deployment of a Flask application in a scalable and highly available architecture using a **Launch Template**, **Application Load Balancer (ALB)**, **Auto Scaling Group (ASG)**, and a **custom VPC**. This project focuses on using the CLI to set up resources. Part 3 will expand on this project by using IaC to create resources. Using the CLI is not the best solution,, however by manually creating each resource it helps to learn the process.


At completion, you'll have:
- A Flask web app running on EC2 instances
- Auto Scaling and health check recovery
- A public ALB managing HTTP traffic
- Infrastructure created entirely via AWS CLI

---
## Prerequisites

- AWS CLI installed and configured
- EC2 Key Pair created (e.g., `MyEC2KeyPair`)
- IAM Role for EC2 with SSM permissions (e.g., `EC2SSMRole`)
- Basic familiarity with EC2, VPCs, and the CLI

---
## Project Workflow

1.   [Create the Flask App Bootstrap Script](#create-the-flask-app-bootstrap-script)
2.   [Create a Custom VPC with Public and Private Subnets](#2-create-a-custom-vpc-with-public-and-private-subnets)
3.   [Create a Security Group for Flask](#3-create-a-security-group-for-flask)
4.   [Create a Launch Template](#4-create-a-launch-template)
5.   [Create a Target Group](#5-create-a-target-group)
6.   [Create the Application Load Balancer](#6-create-the-application-load-balancer)
7.   [Configure the Listener](#7-configure-the-listener)
8.   [Create the Auto Scaling Group](#8-create-the-auto-scaling-group)
9.  [Test ALB & ASG Behavior](#9-test-alb-and-asg-behavior)
10.  [Simulate Failure & Observe Auto Healing](#10-simulate-failure-and-observe-auto-healing)
11.  [Clean Up Resources](#11-clean-up-resources)
12.  [Troubleshooting & Lessons Learned](#12-troubleshooting-and-lessons-learned)
13.  [Final Thoughts](#final-thoughts)

---

![[ec2-project2-architecture.png]]

---
<a name="create-the-flask-app-bootstrap-script"></a>
### 1. Create the Flask App Bootstrap Script

Create a shell script to install dependencies, create the app.py, and run Flask on boot. Save this as `user-data.sh`:

From the `/myapp` directory use `nano user-data.sh` to open a terminal editor and paste in the following user-data.sh code.

**user-data.sh
```bash
#!/bin/bash
exec > /var/log/user-data.log 2>&1
set -x

# Update and install Python/Flask
dnf update -y
dnf install -y python3-pip -y
pip3 install --upgrade pip
pip3 install flask

# Create app directory
mkdir -p /home/ec2-user/myapp
cd /home/ec2-user/myapp

# Create Flask app
cat << 'EOF' > app.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return 'Hello from Flask on EC2 with ALB and ASG!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Run the app as ec2-user (important!)
runuser -l ec2-user -c 'nohup python3 /home/ec2-user/myapp/app.py > /home/ec2-user/myapp/flask.log 2>&1 &'
```

Once we have this file we have to change it to a base 64 for it to be processed correctly by the ec2 instance. to do that run `base64 -w 0 user-data.sh > user-data.b64`

We will now reference the "user-data.b64" script when creating our launch template

---
### 2. Create a Custom VPC with Public and Private Subnets

First we need to create a VPC. If you need a review of a VPC, here are some good resources: 

https://aws.amazon.com/vpc/ 

https://hamzahabdulla1.medium.com/aws-vpc-101-c46dc258d27d

We will create the following in this section:

- A VPC with DNS support
- 2 Public Subnets for ALB
- 2 Private Subnets for EC2
- Internet Gateway for public access
- NAT Gateway for private instance updates
- Route tables and proper associations

Before we create anything we need a CIDR range for our VPC.

For this project we will use the standard of `10.0.0.0/16` . This is part of the Private IP Range (RFC 1918), that means it isn't routable on the public internet. It is safe to use internally within cloud or corporate networks. The `/16` ensures 65,536 IP addresses.

Some other ranges we could have chosen are `10.0.0.0/8`, `172.16.0.0/12`, or `192.168.0.0/16` 

 First we need to create a VPC
 
```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'
```
 
  Save the VPC ID in a variable
  
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=MyVPC" \
  --query "Vpcs[0].VpcId" \
  --output text)
```

We then need to enable DNS support. Enabling **DNS support** in an Amazon VPC allows instances within your VPC to **resolve domain names** (like `google.com`, `amazonaws.com`, or internal AWS service endpoints like `ec2.amazonaws.com`) to IP addresses.

`enableDnsSupport` Allows instances to **resolve DNS names** (e.g., via Route 53 or AWS service endpoints). Must be `true` to use AWS DNS.

`enableDnsHostnames` Assigns a **public DNS hostname** to instances with public IPs. 

To enable DNS support run the following commands:
```bash
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
```

Now that we have a VPC with DNS support we need to create 2 Public Subnets in different Availability Zones to provide high availability, fault tolerance, and resiliency.

Load balancers and ASGs rely on public subnets in at least 2 AZs

```bash
# Subnet 1 (Saving the output to the variable PUBLIC_SUBNET_ID for later)
PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet1}]' \
  --query 'Subnet.SubnetId' \
  --output text)


# Subnet 2
aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet2}]'
```

Now we need to create 2 more subnets however these will be Private Subnets for our EC2 instances

```bash
# Private Subnet 1
PRIVATE_SUBNET1_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.101.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet 2
PRIVATE_SUBNET2_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.102.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet2}]' \
  --query 'Subnet.SubnetId' \
  --output text)
```

Next we need to modify our Public Subnets to Automatically Assign Public IPs. We do this so that EC2 instances launched in those subnets can be accessed directly from the internet without manually assigning a IP each time.

```bash
SUBNET1_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=PublicSubnet1" \
  --query "Subnets[0].SubnetId" \
  --output text)

SUBNET2_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=PublicSubnet2" \
  --query "Subnets[0].SubnetId" \
  --output text)

aws ec2 modify-subnet-attribute --subnet-id $SUBNET1_ID --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUBNET2_ID --map-public-ip-on-launch
```

In order for our resources in our VPC to access the internet and vice versa we need to setup an Internet Gateway (IGW). 

Run the following to create an Internet Gateway (for Public Subnets only):

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

IGW_ID=$(aws ec2 describe-internet-gateways \
  --filters "Name=tag:Name,Values=MyIGW" \
  --query "InternetGateways[0].InternetGatewayId" \
  --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

To set up the route table use the following:

```bash
aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]'

RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=PublicRouteTable" \
  --query "RouteTables[0].RouteTableId" \
  --output text)

aws ec2 create-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```

Now that we have the route table we have to associate it with the Public Subnets we created

```bash
aws ec2 associate-route-table --subnet-id $SUBNET1_ID --route-table-id $RT_ID
aws ec2 associate-route-table --subnet-id $SUBNET2_ID --route-table-id $RT_ID
```

We have created the following up to this point:

- A VPC
- 2 Public subnets
- 2 Private subnets
- An Internet Gateway 
- A Route Table routing 0.0.0.0/0 traffic to the IGW
- Subnets with auto-assign public IPs
  
Were still missing one last piece, and that is we need our EC2 instances in private subnets to reach the internet (for updates and installs).

In order to do this we need to create a Network Address Translation (NAT) Gateway in a public subnet. A NAT Gateway allows our EC2 instances to reach the internet while blocking inbound traffic.

Disclaimer: While most of this can be accomplished in the free tier a NAT Gateway charges $.045/hour. Make sure to delete the NAT Gateway when done with testing your project with:

`aws ec2 delete-nat-gateway --nat-gateway-id nat-xxxxxxxxxxxxxx

In order to set up the NAT Gateway we need to create an Public Elastic IP to associate with.

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
echo "Elastic IP Allocation ID: $EIP_ALLOC_ID"
```

Now that we have the Elastic IP we can create the NAT Gateway in a public subnet (using PUBLIC_SUBNET_ID  from earlier).

```bash
NAT_GW_ID=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET_ID --allocation-id $EIP_ALLOC_ID --query 'NatGateway.NatGatewayId' --output text)
echo "NAT Gateway ID: $NAT_GW_ID"
```

Note: it might take a couple minutes for the NAT Gateway status to become available. We have to wait for it to be available to move on. Run the following to check the status:

```bash
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
echo "NAT Gateway is now available"
```

Once available we can now create a route table for our Private Subnets.

```bash
PRIVATE_RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTable}]' --query 'RouteTable.RouteTableId' --output text)
echo "Private Route Table ID: $PRIVATE_RT_ID"
```

Now that we have the route table we need to create a route in the private route table that points to the NAT gateway. Do that via the following.

```bash
aws ec2 create-route --route-table-id $PRIVATE_RT_ID --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_ID
```

Lastly, we need to associate the private subnets with the private route table.

```bash
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET1_ID --route-table-id $PRIVATE_RT_ID
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET2_ID --route-table-id $PRIVATE_RT_ID
```

We have successfully created:
-  A VPC with DNS support
- 2 Public Subnets for ALB
- 2 Private Subnets for EC2
- Internet Gateway for public access
- NAT Gateway for private instance updates
- Route tables and proper associations

---
### 3. Create a Security Group for Flask

We can't use the same Security Group from Part 1 [[Project 1. EC2 Web Server Deployment]] Because we created a new VPC. Security Groups are scoped to a single VPC, in part 1 we scoped it to the default VPC with the free tier AWS account. In this case, you’ll define one for HTTP and optionally SSH (if testing public EC2).

To assign it to our newly created VPC we first need to create it

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name flask-sg \
  --description "Allow HTTP and SSH access" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"
```

Next we need to allow inbound HTTP (port 80) from anywhere.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

Next we need to allow inbound SSH from port 22 (your IP only). This is only if you want to create a EC2 instance in a public subnet. You can't SSH into the Auto Scaling Group EC2 instances because they are in private subnets

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 93.112.39.161/32

```

---
### 4. Create a Launch Template

A launch Template in AWS is a reusable configuration for launching EC2 instances. It defines everything needed to start an instance. ASG and ELB all reference the Launch Template to know what kind of instances to spin up.

Run the following in a terminal configured for AWS CLI not 

```bash
aws ec2 create-launch-template \
  --launch-template-name MyFlaskTemplate \
  --version-description "v1" \
  --launch-template-data '{
	"IamInstanceProfile": { "Name": "EC2SSMRole" },
    "ImageId": "ami-07caf09b362be10b8", 
    "InstanceType": "t2.micro",
    "KeyName": "MyEC2KeyPair",
    "SecurityGroupIds": ["$SG_ID"],
    "UserData": "'"$(< user-data.b64)"'"
  }'
```

If you need to update the Launch Template you do so with versioning. to do that run the following command

```bash
aws ec2 create-launch-template-version \
  --launch-template-name MyFlaskTemplate \
  --version-description "v8-ec2ssmrole" \
  --launch-template-data '{
    "IamInstanceProfile": { "Name": "EC2SSMRole" },
    "ImageId": "ami-07caf09b362be10b8",
    "InstanceType": "t2.micro",
    "KeyName": "MyEC2KeyPair",
    "SecurityGroupIds": ["'$SG_ID'"] ,
    "UserData": "'$(< user-data.b64)'"
  }'
```

---
### 5. Create a Target Group for ALB

A Target Group is a logical group of compute resources that a Load Balancer sends traffic to.

The ALB handles incoming HTTP/HTTPS requests. The Target Group tells the ALB where to send those requests ex: EC2 Instances.

It also performs health checks, ex: hitting `/` to determine if an instance is healthy and should receive traffic.

If you followed the steps above you can use the same variables, If you're using an existing VPC you first need to find our VPC ID. Use this command:

`aws ec2 describe-vpcs --query 'Vpcs[*].{ID:VpcId}' --output table
`
If you have never created a VPC there should only be one option and it should look like this:

vpc-12fd98c473j9487098

Replace the $VPC_ID in the code below with your actual vpc-id.

Target Group
```bash
aws elbv2 create-target-group \
  --name flask-targets-v2 \
  --protocol HTTP \
  --port 5000 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-path /
```

`--health-check-path /` means the ALB will periodically send an HTTP request to:

`http://<instance-ip>/`

 If the response is successful (HTTP 200 OK), the instance is marked as **healthy**.

 If it fails (e.g., returns 500 or times out), the instance is marked as **unhealthy**, and **the ALB stops routing traffic to it** until it recovers.

---
### 6. Create the Application Load Balancer

An Application Load Balancer (ALB) is a service in AWS that automatically distributes incoming HTTP and HTTPS traffic across multiple targets like EC2 instances, containers (ECS), or lambda functions.

It operates at the Application Layer (Layer 7) of the OSI model. It understands HTTP/HTTPS traffic and route based on things like:
- Paths (ex: /api/*)
- Host-based (ex: admin.example.com)
- Query parameters or headers

In our example we put the ALB in front of our EC2 instances. It handles all the HTTP traffic from the internet. It checks which instance is healthy via health checks and sends traffic to only those. If one EC2 dies the Auto Scaling Group (ASG) can replace it and the ALB routes traffic to the new one seamlessly.

One bonus of ALB is they can terminate SSL certificates so you can offload HTTPS decryption to the ALB and forward plain HTTP to the backend - keeping your app code simpler. The following will create the ALB and capture the ARN to a variable for creating the listner

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name flask-alb \
  --subnets subnet-0ddde542e96119398 subnet-093eb674a5c6324d7 \
  --security-groups $SG_ID \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)
```


---
### 7. Configure the Listener

An ALB listener in AWS is a process that checks for connection request on a specific port and protocol. It then forwards those requests to target groups based on defined rules. In the code below you can see we are forwarding request on port 80 to the target group by passing the target group ARN. 

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TGT_GROUP_ARN
```

---
### 8. Create the Auto Scaling Group

An Auto Scaling Group (ASG) in AWS is a service that automatically manages a fleet of EC2 instances to meet demand and ensure availability. ASG automatically launches or terminates EC2 instances based on CPU usage, request count, schedules, or other CloudWatch metrics.

ASGs can span multiple AZs within a region to protect against failure of a single AZ. If a healt check fails the ASG automatically replaces it. It uses the Launch Template from step 4 know how to create instances. It also sets the `MinSize`, `MaxSize`, and `DesiredCapacity` for the instances.

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name flask-asg-2 \
  --launch-template LaunchTemplateName=MyFlaskTemplate \
  --min-size 2 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$PRIVATE_SUBNET1_ID, $PRIVATE_SUBNET2_ID" \
  --target-group-arns $TGT_GROUP_ARN
```

This will create 2 instances automatically and will refresh with new instances anytime their is a failure

![[Pasted image 20250807083051.png]]

---
### 9. Test ALB and ASG Behavior

Verify your deployment by retrieving the DNS name of your ALB and accessing the Flask app via browser.

Test Plan:
- Load the site via ALB BNS
- Confirm correct app response "Hellow from Flask on EC2 with ALB and ASG!"
- Manually terminate an instance and observe ASG healing

```bash
aws elbv2 describe-load-balancers \
  --names flask-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text
```

Visit the DNS in a browser.

`http://flask-alb-1234567890.us-east-1.elb.amazonaws.com/
`
You should see your Flask response:
"Hello from Flask on EC2 with ALB and ASG!"

![[Pasted image 20250807082446.png]]

### ### **10. Simulate Failure and Observe Auto Healing

Use CLI commands to view and terminate EC2 instances. Watch as the ASG replaces unhealthy or terminated instances automatically. 

To see the status of your instances use

```bash
`aws ec2 describe-instances \
	--filters "Name=tag:aws:autoscaling:groupName,Values=flask-asg" \
	--query "Reservations[*].Instances[*].[InstanceId,State.Name]" \
	--output table`
```

To see the target health of the instances you can run

```bash
aws elbv2 describe-target-health \
	--target-group-arn $TGT_GROUP_ARN \
	--query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
	--output table
```

![[Target Health.png]]

Choose one instance and terminate using the command below. In this example we terminated i-0eb8ba3584681226b:

```bash
aws ec2 terminate-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

Watch it heal with the following command. You will see the instance terminate and be replaced with a new one:

```bash
watch -n 5 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names flask-asg-2 --query 'AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]' --output table"
```



![[Pasted image 20250804070314.png]]

![[Pasted image 20250804070403.png]]

![[Pasted image 20250804070445.png]]

---

## 11. Clean Up Resources

To avoid unexpected charges, clean up all provisioned infrastructure.

> **Reminder:** NAT Gateway incurs hourly charges even if unused.

```bash
# Delete ASG 
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name flask-asg --force-delete  

# Delete launch template 
aws ec2 delete-launch-template --launch-template-name MyFlaskTemplate  

# Delete target group 
aws elbv2 delete-target-group --target-group-arn $TGT_GROUP_ARN  

# Delete ALB 
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN  

# Delete Security Groups
for sg_id in "${SECURITY_GROUP_IDS[@]}"; do
  aws ec2 delete-security-group --group-id $sg_id
  
# Delete Subnets
for subnet_id in "${SUBNET_IDS[@]}"; do
  aws ec2 delete-subnet --subnet-id $subnet_id
done

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id <nat-gateway-id>
```


---
## 12. Trouble Shooting and Lessons Learned

> **Common Issues Resolved:**
>
> - Accessing private EC2 via SSM
> - Bootstrap script failures (e.g., using `yum` vs `dnf`)
> - Port binding limitations for unprivileged users
> - ALB-to-EC2 communication via port mismatch or SG misconfigurations


During testing, my EC2 instances were showing a status of unhealthy. Trying to debug the situation I learned that because the instances are in a private subnet we can't SSH into them directly. We have to use Session Manager to access the private instance.

To set up AWS Systems Manager (SSM) to connect without SSH there are a couple things that have to be done:

- **Create an IAM Role** with SSM permissions
- **Attach that IAM Role** to your EC2 instance or Launch Template
- **Ensure your EC2 instance** meets SSM requirements (SSM agent installed, internet access via NAT/IGW)
- **Verify security group rules** (443 Outbound)
- **Access the instance via Session Manager**

To create an IAM role with SSM permissions use the following command:

```bash
aws iam create-instance-profile --instance-profile-name EC2SSMRole

aws iam create-role \
  --role-name EC2SSMRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'

aws iam attach-role-policy \
  --role-name EC2SSMRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2SSMRole \
  --role-name EC2SSMRole
```

Next, if the EC2 instance already exists we can attach the IAM Role to the instance

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-0ebc7828b467a622c \
  --iam-instance-profile Name=EC2SSMRole
```

Now you can use session manager from the console or the command line to directly access the EC2 instances in a private subnet.

Wen connecting via SSM you have to use the command `sudo -i` in order to change directories because SSM defaults to a non-root user. `sudo -i` elevates the entire shell. You can't just do `sudo cd/root` Because `cd` changes the **current shell’s directory**, and `sudo` runs a **subshell** with elevated privileges — it doesn't change your current shell's working directory.

Another issue I encountered was the user-data.sh script. Originally I had used linux commands, `yum install` however because we launched a Amazon Linux machine we have to use the command `dnf install`. It's an easy fix but updating the start up script also has some ripple effects.

---

If you need to update a launch template, you need to update the version specifying the new version description.

```bash
aws ec2 create-launch-template-version \
  --launch-template-name MyFlaskTemplate \
  --version-description "v6" \
  --launch-template-data '{
    "ImageId": "ami-07caf09b362be10b8",
    "InstanceType": "t2.micro",
    "KeyName": "MyEC2KeyPair",
    "SecurityGroupIds": ["'"$SG_ID"'"],
    "UserData": "'"$(< user-data.b64)"'"
  }'
```

Then you need to make this version the default

```bash
aws ec2 modify-launch-template \
  --launch-template-id "lt-0f7b8084ed33bdb28" \
  --default-version "9" \
  --region "us-east-1"
```

Next you need to update the auto scaling group to reference the latest launch template version

```bash
aws autoscaling update-auto-scaling-group   \
	--auto-scaling-group-name flask-asg-2  \
	--launch-template LaunchTemplateName=MyFlaskTemplate,Version='$Latest'
```


We also originally set up this app to use port 80 **BUT the user-data script runs as ec2-user**, which can’t bind to port 80. So we had to change it to port 5000.

---

I originally used the same security group for instances and the ALB. I learned that they should be separate. This is how I created a new ALB security group with port 80 access and the ec2 security group with port 5000 access from the EC2 security group.

```bash
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name flask-alb-sg \
  --description "Security group for ALB to access Flask app" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
```


```bash
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 5000 \
  --source-group $ALB_SG_ID
```

```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $ALB_ARN \
  --attributes Key=access_logs.s3.enabled,Value=false

aws elbv2 set-security-groups \
  --load-balancer-arn $ALB_ARN \
  --security-groups $ALB_SG_ID
```