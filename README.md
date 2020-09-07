# k8s-tenancy-example

## Create Users

Generate private key
```
$ openssl genrsa -out tenant-a-developer-1.pem 2048
```

Generate CSR. The value of `/CN=` will be user name in K8s.
```
$ openssl req -new -key tenant-a-developer-1.pem -out tenant-a-developer-1.csr -subj "/CN=tenant-a-developer-1"
```

Encode CSR in base64
```
$ cat tenant-a-developer-1.csr | base64 | tr -d '\n'
```

Create CSR object for K8s API
```
$ vim csr.yaml

apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: user-request-tenant-a-developer-1
spec:
  groups:
  - system:authenticated
  request: [Copy and paste base64-encoded CSR here]
  usages:
  - digital signature
  - key encipherment
  - client auth
```

Send CSR to K8s API
```
$ kubectl create -f csr.yaml
```

Administrator can see that there is a CSR with `Pending` status.
```
$ kubectl get csr
NAME                   AGE       REQUESTOR   CONDITION
user-request-tenant-a-developer-1   2s        admin       Pending
```

Approve CSR
```
$ kubectl certificate approve user-request-tenant-a-developer-1
```

Get certificate
```
$ kubectl get csr user-request-tenant-a-developer-1 -o jsonpath='{.status.certificate}' | base64 -d > tenant-a-developer-1.crt
```

Now the user can access K8s API with private key `tenant-a-developer-1.pem` and client certificate `tenant-a-developer-1.csr` . The user can configure kubectl as following.

```
$ kubectl config set-cluster cluster_name --insecure-skip-tls-verify=true --server=[kubernetes API endpoint]
$ kubectl config set-credential tenant-a-developer-1 --client-certificate=tenant-a-developer-1.crt --client-key=tenant-a-developer-1.pem --embed-certs=true
$ kubectl config set-context cluster_name --cluster=cluster_name --user=tenant-a-developer-1
$ kubectl config use-context cluster_name
```

At this point, this user can't do anything because there's no RoleBinding and ClusterRoleBinding assined to the user.
```
$ kubectl get pods
Error from server (Forbidden): User "tenant-a-developer-1" cannot list pods in the namespace "default". (get pods)
```


## Create Namespace,Role,RoleBinding

See example in `tenant.yaml`

