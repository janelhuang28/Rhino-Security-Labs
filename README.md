# Rhino-Security-Labs
Practicing the Rhino Security Labs

# Installation
1. Install Terraform https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli
2. Install Cloudgoat - a sandbox environment to perform cloud based vulnerbility testing https://github.com/RhinoSecurityLabs/cloudgoat#quick-start

# Scenarios
## EFS EC2 attack
The goal of this mission is to mount a file system on EFS and retrieve the flag. To build the resources for the scenario:
```
./cloudgoat.py create ecs_efs_attack
```


# Trouble Shooting
## Installation

#### Cloudgoat - ```No whitelist.txt file was found at /Users/huajanel/cloudgoat/whitelist.txt ... Unknown error: Unable to retrieve IP address.```
With the --auto feature, the whitelist is created automatically. However, if it is not able to find the IP address due to VPN, etc., restrictions, then perform the following commands
```
ifconfig <- For MAC, retrieve IP Address
cd cloudgoat
nano whitelist.txt <- Ensure that the IP address has a CIDR of /32 at the end
python3 cloudgoat.py config whitelist
```

## Scenarios
#### Cloudgoat - ```The create command requires the use of the --profile flag, or a default profile defined in the config.yml file (try "config profile")```
Update the config file for cloudgoat to add your aws profile. An access key and secret access key must be created first.
```
cd cloudgoat
nano ./config.yml
<Edit cloudgoat_aws_access_key with the appropriate access key. Same with secret access key.>
```
