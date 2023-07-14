# Rhino-Security-Labs
Practicing the Rhino Security Labs

# Installation
1. Install Terraform https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli
2. Install Cloudgoat - a sandbox environment to perform cloud based vulnerbility testing https://github.com/RhinoSecurityLabs/cloudgoat#quick-start

# Scenarios
## Easy
### Vulernable Lambda - Privilege escalation in lamdba to retrieve secret in secrets manager
https://github.com/janelhuang28/Rhino-Security-Labs/blob/main/vulnerable%20lambda/README.md 

### IAM Privilege Escalation - Privilege escalation in IAM 
https://github.com/janelhuang28/Rhino-Security-Labs/tree/main/iam_privesc_by_rollback

### Lamdba Privilege Escalation - Privilege escalation using IAM roles and Lambda
https://github.com/janelhuang28/Rhino-Security-Labs/tree/main/lambda_privesc

## Medium 
### IAM privilege escalation by attachment
https://github.com/janelhuang28/Rhino-Security-Labs/tree/main/iam_privesec_by_attachment

### EC2 SSRF
https://github.com/janelhuang28/Rhino-Security-Labs/tree/main/ec2_ssrf

### ECS Takeover
https://github.com/janelhuang28/Rhino-Security-Labs/tree/main/ecs_takeover

## Hard
### EFS ECS Attack - Privilege escalation through ecs containers to mount an efs 
https://github.com/janelhuang28/Rhino-Security-Labs/blob/main/efs_ecs/README.md

# Trouble Shooting
## Installation

#### Cloudgoat - ```No whitelist.txt file was found at /Users/<username>/cloudgoat/whitelist.txt ... Unknown error: Unable to retrieve IP address.```
With the --auto feature, the whitelist is created automatically. However, if it is not able to find the IP address due to VPN, etc., restrictions, then perform the following commands
```
ifconfig <- For MAC, retrieve IP Address
cd cloudgoat
nano whitelist.txt <- Ensure that the IP address has a CIDR of /32 at the end
python3 cloudgoat.py config whitelist
```

## Scenarios
#### ```The create command requires the use of the --profile flag, or a default profile defined in the config.yml file (try "config profile")```
Update the config file for cloudgoat to add your aws profile. An access key and secret access key must be created first.
```
cd cloudgoat
nano ./config.yml
<Edit cloudgoat_aws_access_key with the appropriate access key. Same with secret access key.>
```
#### ```ssh: connect to host <IP address> port 22: Operation timed out```
This error could be caused if your whitelist IP address was not correct. To find the correct IP address, in the ec2 console of the account, change the targeted instance, SSH rule to allow 0.0.0.0/0.
Then in the terminal SSH as normal e.g. ```ssh -i <key pair> ubuntu@<IP address>```
Find your ip address by: ```who am i```
Copy the IP address to the EC2 console SSH rule to change. 
