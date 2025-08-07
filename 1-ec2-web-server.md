# Step-by-Step CLI Guide for Basic EC2 Web Server Deployment [[EC2]] 

### Prerequisites:

- AWS CLI installed and configured with your credentials (`aws configure`)
- SSH client installed 
---
### Step 1: **Create a key pair (for SSH) via CLI**


`aws ec2 create-key-pair --key-name MyEC2KeyPair --query 'KeyMaterial' --output text > MyEC2KeyPair.pem chmod 400 MyEC2KeyPair.pem`

- Saves your private key locally as `MyEC2KeyPair.pem` with proper permissions.
- The command `chmod 400 MyEC2KeyPair.pem` sets **file permissions** on your EC2 private key file so it can be used securely with SSH.

---
### Step 2: **Create a security group that allows HTTP (port 80) and SSH (port 22)**

- Create security group for EC2 instance. This is a virtual firewall that provides access rules for the EC2 instance. It defines who can connect from where and on which ports/protocols.
- Security groups are stateful, if you allow inbound traffic the return is also allowed

`aws ec2 create-security-group --group-name MyWebSG --description "Allow SSH and HTTP access"  

- You have now created a Security Group inside of the region associated with your AWS account. You will see a GroupID as well as the ARN for that Security Group

![[Pasted image 20250803223653.png]]

`aws ec2 create-security-group --group-name MyEC2SG --description "Allow SSH and http access"

```bash
{
    "GroupId": "sg-xxxxxxxxxxxxx",
    "SecurityGroupArn": "arn:aws:ec2:us-east-1:XXXXXXXXXXXX:security-group/sg-XXXXXXXXXXXXXXXXXX"
}
```


-  Capture the Security Group ID to a shell variable SG_ID by looking up a security group with the name that was just created.
`SG_ID=$(aws ec2 describe-security-groups --group-names MyEC2SG --query "Security Groups[0].GroupId" --output text) 
 
-  Allow SSH from your IP 
	1. Get the public IP and store it in a variable MY_IP
		`MY_IP=$(curl -s https://checkip.amazonaws.com) 

	2. Authorize inbound SSH from your public IP
		`aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr ${MY_IP}/32  

		You will see the newly created SG Rules for you SSH

```bash
{
	"Return": true,
	"SecurityGroupRules": [
		{
			"SecurityGroupRuleId": "sgr-xxxxxxxxxxxxxxxx",
			"GroupId": "sg-xxxxxxxxxxxxxxxx",
			"GroupOwnerId": "xxxxxxxxxxxx",
			"IsEgress": false,
			"IpProtocol": "tcp",
			"FromPort": 22,
			"ToPort": 22,
			"CidrIpv4": "xxx.xxx.xxxx.xxx/32",
			"SecurityGroupRuleArn": "arn:aws:ec2:us-east 1:xxxxxxxxxxxx:security-group-rule/sgr-xxxxxxxxxxxxxxxx"
		}
	]
}
```

 - Allow HTTP from anywhere. A new security group rule will be created for port 80 allowing HTTP access from any IP address
 
	 `aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0`

---
### Step 3: **Launch an EC2 instance**

To create an instance use the following. This will create a Amazon Linux instance running on the free tier t2 micro type

```bash
aws ec2 run-instances \
  --image-id ami-07caf09b362be10b8 \ # Amazon Linux 2023
  --count 1 \
  --instance-type t2.micro \
  --key-name MyEC2KeyPair \
  --security-group-ids $SG_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyWebServer}]' \
  --query 'Instances[0].InstanceId' \
  --output text
```

- The instance ID will be output as text "i-XXXXXXXXXXXXXXXXXX"
- Save the instance ID:

	`INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=MyWebServer" --query "Reservations[0].Instances[0].InstanceId" --output text)

---
### Step 4: **Get the public IP of the instance**

Wait a few minutes, then:

`PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query "Reservations[0].Instances[0].PublicIpAddress" --output text) 

Print the Public IP to the terminal

`echo "Instance public IP is: $PUBLIC_IP"`

---
### Step 5: **SSH into the instance**

`ssh -i MyEC2KeyPair.pem ec2-user@$PUBLIC_IP

If it is the first time connecting via SSH you should get this message:

The authenticity of host 'XXX.XX.XXX.XXX  can't be established.
XXXXXX key fingerprint is SHA256:gXXXXXXXXXXXXXXXXXXXXXX.
This key is not known by any other names.

This is normal, it is just an alert because it's the first time connecting. You can type yes.


---
### Step 6: **Install Nginx and create a static "Hello World" page**

Once inside the EC2 instance run the following:

`sudo dnf update -y 

- This updates all packages to the latest version. It is similar to the yum command in Linux but Amazon Linux 2023 uses dnf

`sudo dnf install nginx -y 

- This install the NGINX web server from the default package repo. The -y confirms the installation automatically.

`sudo systemctl enable nginx 

- This enables the NGINX service to start at boot time

`sudo systemctl start nginx  

- This starts the NGINX service immediately.

`echo "Hello World from EC2 Web Server!" | sudo tee /usr/share/nginx/html/index.html

- This outputs "Hello World from EC2 Web Server" to the server.`| sudo tee ...` pipes it to tee with sudo to write it to the default nginx web root file: `/usr/share/nginx/html/index.html`
- `tee` takes input and writes it to a file similar to `>` it also prints it to the screen
- This overwrites the default index page with the custom message

---
### Step 7: **Verify Nginx is running**

`sudo systemctl status nginx`

- if everything was done correctly it should look like this:

```bash
sudo systemctl status nginx
	  nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     **Active: active (running)** since Sun 2025-08-03 21:17:22 UTC; 51s ago
     Process: 26332 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
     Process: 26333 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
     Process: 26334 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
```

---
### Step 8: **Exit SSH and test your webpage**

From your local machine, open:

`http://$PUBLIC_IP`

You should see **"Hello World from EC2 Web Server!"**

![[Pasted image 20250804002213.png]]

---
### Step 9: Clean up

When done, terminate the EC2 instance:

`aws ec2 terminate-instances --instance-ids $INSTANCE_ID`