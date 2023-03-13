# Deploy Java Web Application on K8s Cluster using KOps on AWS.

## Spin up KOps Cluster

Create cluster

```bash
kops create cluster --name=kube.oayanda.com --state=s3://oayanda-kops-state --zones=us-east-1a,us-east-1b --node-count=2 --node-size=t3.small --master-size=t3.medium --dns-zone=kube.oayanda.com
```

![create cluster](./images/1.png)

Update cluster

```bash
 kops update cluster --name kube.oayanda.com --state=s3://oayanda-kops-state --yes --admin
```

![update cluster](./images/2.png)

Validate cluster

```bash
kops validate cluster  --state=s3://oayanda-kops-state
```
![validate cluster](./images/3.png)