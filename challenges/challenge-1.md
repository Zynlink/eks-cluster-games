## Secret Seeker
Goal: Listing all the secrets in the cluster and spot the flag among them.

Permissions:
```
{
    "secrets": [
        "get",
        "list"
    ]
}
```

Walkthrough:

```bash
kubectl get secrets -A
```
Command above will trigger an error (Forbidden) as the service account __service-account-challenge1__ has permission to get and list secrets in the __challenge1__ namespace. 

```bash
kubectl get secrets 
```
```bash
Output

NAME         TYPE     DATA   AGE
log-rotate   Opaque   1      8d
```

Get the secret's YAML
```bash
kubectl get secrets log-rotate -o yaml
```
```bash
Output

apiVersion: v1
data:
  flag: <...>
kind: Secret
metadata:
  creationTimestamp: ...
  name: log-rotate
  namespace: challenge1
  resourceVersion: ...
  uid: ...
type: Opaque
```
Retrieve the __flag__ key value from the data field. 

```bash
kubectl get secrets log-rotate -o jsonpath={.data.flag}
```
Secret data in Kubernetes is stored in base64 format, we will need to decode it to get the actual flag. 

```bash
kubectl get secrets log-rotate -o jsonpath={.data.flag} | base64 -d
```
