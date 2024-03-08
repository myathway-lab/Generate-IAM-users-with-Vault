## Generate-IAM-users-with-Vault

### Summary <br>

Our goal on this lab is to provide EC2FullAccess access to our vendors.  <br>
Access Period is just 5 days.  <br>
But in this lab, we will simulate it just 5min of lease time.  <br>
All the access must be expired after 5 min. 
If we don't users to access EC2, we may need to revoke the access.  <br>
In some case, we may need to extend the access period. 
<br>

As per the requirement, we will use Vault to create IAM users with EC2 Full permission. 
Vault can revoke, renew the credentials.

<br>

### Requirements <br>

- IAM user which has privileges to create IAM users & assign IAM permissions. 
- Install AWS CLI for verification purpose. 
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Note - Make sure to delete all the access key & secrets after this lab. 

<br>

### Steps to follow <br>

1) Install vault using helm & Setup a Vault cluster on K8s. Refer below link.
2) Authenticate with AWS
3) Create a vault role
4) Generate the IAM users using role.
5) Verify the IAM access.
6) Revoke the IAM access.
7) Rotate the Root Cretential


<br>

**1) Install vault using helm & Setup a Vault cluster on K8s. Refer below link.**

<li class="masthead__menu-item">
<a href="https://github.com/myathway-lab/Vault-on-K8s">Setup Vault Cluster on K8s</a>
</li>


<br>

**2) Authenticate with AWS**

- We need to authenticate with Vault before we access it.
  
```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault-0 -n vault -- sh
/ $ vault login 
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.eE0sBdsbP82tixEf1UUmN1a3
token_accessor       FiOMnIlKXDtqAiBraXs8vzV9
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

- Enable AWS engine using aws-iam-generate/ path.

```yaml
/ $ vault secrets enable -path=aws-iam-generate aws
Success! Enabled the aws secrets engine at: aws-iam-generate/
```
- We have one IAM account which has admin privileges.
- We will drive Vault to use this account to authenticate with AWS.
- Make sure to have access key and secret key of this IAM account. 
- Write this access and secret keys to aws-iam-generate root config. 
  
```yaml
/ $ export AWS_ACCESS_KEY_ID=AKIAQ3EGRHNZ4JS3ZEW2
/ $ export AWS_SECRET_ACCESS_KEY=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
/ $ vault write aws-iam-generate/config/root access_key=$AWS_ACCESS_KEY_ID secret_key=$AWS_SECRET_ACCESS_KEY region=ap-southeast-1
Success! Data written to: aws-iam-generate/config/root
```

- Now Vault is ready to talk with AWS. 

<br>


**3) Create a vault role**

- As per our goal, we need to create IAM users with "EC2FullAccess". 
- So we will create a Vault role with iam_user credential_type. Then we will assign policy with EC2FullAccess. <br>
 (We reference below policy json from AWS permissions page. )

```yaml
/ $ vault write aws-iam-generate/roles/aws-EC2FullAccess-role credential_type=iam_user \
> policy_document=-<<EOF
> {
>     "Version": "2012-10-17",
>     "Statement": [
>         {
>             "Action": "ec2:*",
>             "Effect": "Allow",
>             "Resource": "*"
>         },
>         {
>             "Effect": "Allow",
>             "Action": "elasticloadbalancing:*",
>             "Resource": "*"
>         },
>         {
>             "Effect": "Allow",
>             "Action": "cloudwatch:*",
>             "Resource": "*"
>         },
>         {
>             "Effect": "Allow",
>             "Action": "autoscaling:*",
>             "Resource": "*"
>         },
>         {
>             "Effect": "Allow",
>             "Action": "iam:CreateServiceLinkedRole",
>             "Resource": "*",
>             "Condition": {
>                 "StringEquals": {
>                     "iam:AWSServiceName": [
>                         "autoscaling.amazonaws.com",
>                         "ec2scheduled.amazonaws.com",
>                         "elasticloadbalancing.amazonaws.com",
>                         "spot.amazonaws.com",
>                         "spotfleet.amazonaws.com",
>                         "transitgateway.amazonaws.com"
>                     ]
>                 }
>             }
>         }
>     ]
> }
> EOF
Success! Data written to: aws-iam-generate/roles/aws-EC2FullAccess-role

```

<br>

**4) Generate the IAM users using role.**

- It's time to generate IAM users with EC2Full access from Vault. 
- Before we generate, below is our current user list. 

![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/5ed75715-12c7-473a-9441-37b81a77f416)


<br>

- Now let's generate user from vault ui.

  <br>
  
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/43ffa582-4c69-4e96-9c90-4cc2588ac14a)

- Copy the Credentials for later verification.
  
    <br>
    
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/186f3dbd-ddf8-4f51-acfd-0a713879817b)

- Go to AWS IAM and verify the new user was created.

  <br>
  
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/ca315eb4-3995-453a-8a48-78556d62ec06)

  <br>

- We can also generate using Vault CLI.

```yaml
/ $ vault read aws-iam-generate/creds/aws-EC2FullAccess-role
Key                Value
---                -----
lease_id           aws-iam-generate/creds/aws-EC2FullAccess-role/wMsFogtixgVD8OilMI1PLXks
lease_duration     5m
lease_renewable    true
access_key         AKIAQ3EGRHNZ5P3PRV4I
secret_key         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
security_token     <nil>
/ $
```

<br>

**5) Verify the IAM access.**

- Let's verify whether these new IAM users are getting EC2 FULL permissions.

- Connect to AWS using the IAM access key & secret.

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ aws configure
AWS Access Key ID [****************E7NR]: AKIAQ3EGRHNZ5P3PRV4I
AWS Secret Access Key [****************mpMH]: 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
Default region name [us-east-1]: 
Default output format [None]:
```

- We have one EC2 running under us-east-1 region.
  
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/bd866136-c1e2-421b-9b31-7cfc83d8d4e8)


- Verify if user can list EC2 server infomation.
  
```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ aws ec2 describe-instances
{
    "Reservations": [
        {
            "Groups": [],
            "Instances": [
                {
                    "AmiLaunchIndex": 0,
                    "ImageId": "ami-0f403e3180720dd7e",
                    "InstanceId": "i-0b0ffacae28421d68",
                    "InstanceType": "t2.micro",
                    "KeyName": "test",
                    "LaunchTime": "2024-03-08T12:50:04+00:00",
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "Placement": {
                        "AvailabilityZone": "us-east-1b",
                        "GroupName": "",
                        "Tenancy": "default"
                    },
                    "PrivateDnsName": "ip-172-31-94-122.ec2.internal",
                    "PrivateIpAddress": "172.31.94.122",
                    "ProductCodes": [],
                    "PublicDnsName": "ec2-54-166-76-210.compute-1.amazonaws.com",
                    "PublicIpAddress": "54.166.76.210",
```

- Verify if user can terminate the EC2. 

```yaml
agrant@kindcluster-box:~$ aws ec2 terminate-instances --instance-ids i-0b0ffacae28421d68
    "TerminatingInstances": [
        {
            "CurrentState": {
                "Code": 32,
                "Name": "shutting-down"
            },
            "InstanceId": "i-0b0ffacae28421d68",
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
```

- Now we can confirm IAM user was successfully created using Vault Role and users are able to get EC2 Full access.

  <br>

**6) Revoke / Renew the Lease.**

- I have generated one IAM access. 
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/6cc585ca-9ca3-478b-9acf-319ff87db1da)

- If we don't want that IAM user to access EC2 anymore, we can revoke it.
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/e5459396-9acb-4d5e-b4fe-ac87573bab5b)

- We can also renew the lease time as below.
![image](https://github.com/myathway-lab/Generate-IAM-users-with-Vault/assets/157335804/223d1c05-f2dc-439e-a264-88c4fb6aa72e)

  <br>

  
**7) Tune Lease duration**

- We can extend TTL lease time for whole secret engine path.
  
```yaml
/ $ vault secrets tune -default-lease-ttl=10m /aws-iam-generate
Success! Tuned the secrets engine at: aws-iam-generate/
/ $ 
/ $ vault read sys/mounts/aws-iam-generate/tune
Key                  Value
---                  -----
default_lease_ttl    10m
description          n/a
force_no_cache       false
max_lease_ttl        768h
```

<br> 

**To note** <br>

- Tuned lease time will not update on current gerenrated Leases.
- It will only effect on upcoming leases.
- But if we renew the leases that was generated before tuned, it will use new tuned TTL.





