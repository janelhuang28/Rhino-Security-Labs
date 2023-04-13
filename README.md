# Rhino-Security-Labs
Practicing the Rhino Security Labs

# Installation
1. Install Terraform https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli
2. Install Cloudgoat - a sandbox environment to perform cloud based vulnerbility testing https://github.com/RhinoSecurityLabs/cloudgoat#quick-start

# Scenarios
## EFS ECS attack
The goal of this mission is to mount a file system on EFS and retrieve the flag. To build the resources for the scenario:
```
./cloudgoat.py create ecs_efs_attack
cd ecs_efs_attack<random generated strings>
```
Privilege Enumeration
1. SSH into the instance ```ssh -i cloudgoat ubuntu@<IP address of ruse>```
2. Create a new profile (inherits the permissions from the EC2) ```aws configure --profile ruse```
3. Get the instance profile for the EC2 instance ```aws sts get-caller-identity```. The role name is found like so: arn:aws:sts::<account id>:assumed-role/<role name>/<instance id>
4. List the attached policies - 
  ```
  aws iam list-attached-role-policies --role-name <Role name>
   {
    "AttachedPolicies": [
        {
            "PolicyName": "cg-ec2-ruse-role-policy-ecs_efs_attack_cgid6sec93nj4n",
            "PolicyArn": "arn:aws:iam::<account id>:policy/<policy arn>"
        },
        {
            "PolicyName": "AmazonSSMManagedInstanceCore",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
        }
    ]
  }
  ```
5. Get the actual policy document - 
  ```
  aws iam get-policy-version --policy-arn arn:aws:iam::672021671131:policy/cg-ec2-ruse-role-policy-ecs_efs_attack_cgid6sec93nj4n --version-id v1
  {
      "PolicyVersion": {
          "Document": {
              "Statement": [
                  {
                      "Action": [
                          "ecs:Describe*",
                          "ecs:List*",
                          "ecs:RegisterTaskDefinition",
                          "ecs:UpdateService",
                          "iam:PassRole",
                          "iam:List*",
                          "iam:Get*",
                          "ec2:CreateTags",
                          "ec2:DescribeInstances",
                          "ec2:DescribeImages",
                          "ec2:DescribeTags",
                          "ec2:DescribeSnapshots"
                      ],
                      "Effect": "Allow",
                      "Resource": "*",
                      "Sid": "VisualEditor0"
                  }
              ],
              "Version": "2012-10-17"
          },
          "VersionId": "v1",
          "IsDefaultVersion": true,
          "CreateDate": "2023-04-13T04:01:32+00:00"
      }
  }
  ```
  We can see that the actions that are allowed e.g. EC2 (read), ECS (read and write) and iam (read)

EC2 Enumeration

# Trouble Shooting
## Installation

#### Cloudgoat - ```No whitelist.txt file was found at /Users/huajanel/cloudgoat/whitelist.txt ... Unknown error: Unable to retrieve IP address.```
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
