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

4. List the lambda according to the assumed role
```
aws sts assume-role --role-arn <lambda role> --role-session-name biblo --profile biblo
```
## Lambda Source Code

# References 
* Challenge: https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/vulnerable_lambda/README.md
