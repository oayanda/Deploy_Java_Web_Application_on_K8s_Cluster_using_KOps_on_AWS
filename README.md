# Deploy Java Web Application on K8s Cluster using KOps on AWS.

Prerequites

AWSCLI
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

Create Persistent Volume for DB Node and copy the volume ID for later use. *vol-09570d353208335bb* and the AZ *us-east-1a*.

```bash
aws ec2 create-volume --availability-zone=us-east-1a --size=3 --volume-type=gp2
```

![validate cluster](./images/4.png)

Verify the node in us-east-1a

```bash

# Get available Nodes
k get nodes

# Get more details about a node using the name
k describe node <name>
```

![validate cluster](./images/5.png)