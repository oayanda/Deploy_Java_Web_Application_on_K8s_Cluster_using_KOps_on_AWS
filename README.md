# Deploy Java Web Application on K8s Cluster using KOps on AWS.

In this article example how you could deploy a java web application with Kubernetes. This is helpful when your system architecture needs to be high-availability, fault tolerance, easily scalable, portable and platform independent.

In this setup, I have used [KOps](https://kubernetes.io/docs/setup/production-environment/tools/kops/) to deploy the k8s cluster on AWS. However, you can provision your k8s cluster other options like [AWS EKS](https://aws.amazon.com/eks/) a managed service on AWS. I wrote a simple [script](https://dev.to/oayanda/bash-script-how-to-create-a-k8s-cluster-on-aws-eks-5cfc) to provision k8s cluster on AWS EkS using the eksctl tool.

The web application deployed in this tutorial uses docker images. I have containerized the java application as well as the database and are publicly available on my docker registry. I have also used official docker images for RabbitMQ (as the message broker) and Memcached( to speed up the database by reducing the amount of reads on the database.) and finally used AWS route53 as DNS.

Click below list to view the docker images
- [Java Application](https://hub.docker.com/layers/oayanda/vprofileapp/v1/images/sha256-b82849c5833d7ba61b136e1bf4f5a6e77dc3102e0c0ba8a9a2ed04fac0d75230?context=repo)
- [Database](https://hub.docker.com/layers/oayanda/vprofiledb/v1/images/sha256-050a9a08a7bc72cb2a026927f4cc9a9fe9077265efbd528a8140c5fff509c5ef?context=repo)
- [RabbitMQ](https://hub.docker.com/_/rabbitmq)
- [Memcached](https://hub.docker.com/_/memcached)


**Prerequisites**
- [AWSCLIv2 installed and configured](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Have k8s Cluster Setup](https://dev.to/oayanda/bash-script-how-to-create-a-k8s-cluster-on-aws-eks-5cfc)
- [AWS account](https://aws.amazon.com/free/) 
- [Understanding docker or containers](https://dev.to/oayanda/getting-started-docker-container-docker-image-dockerfile-2oj9)
- [Basic understanding of K8s objects](https://dev.to/oayanda/explained-pod-replicaset-and-deployment-in-kubernetes-2kf)


Let's Begin!!!

You should skip this step if you are using another cluster setup.

## Spin up KOps Cluster in the terminal

Create cluster

```bash
kops create cluster --name=kube.oayanda.com --state=s3://oayanda-kops-state --zones=us-east-1a,us-east-1b --node-count=2 --node-size=t3.small --master-size=t3.medium --dns-zone=kube.oayanda.com
```

![create cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/56e7o59u7tr3e455c2r9.png)

Update cluster

```bash
 kops update cluster --name kube.oayanda.com --state=s3://oayanda-kops-state --yes --admin
```

![update cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/crla1zb7dqv5i8lm1mfy.png)

Validate cluster

```bash
kops validate cluster  --state=s3://oayanda-kops-state
```

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tyztdje9gqku0d0fh5hj.png)


Create Persistent EBS volume for DB pod. Copy the volume ID for later use. *vol-023c6c76a8a8b98ce* and the AZ *us-east-1a*.

Copy the following code snippet into your terminal. 

```bash
aws ec2 create-volume --availability-zone=us-east-1a --size=5 --volume-type=gp2 --tag-specifications 'ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=kube.oayanda.com}]'
```

> _**Note:** For volume mapping, make sure the value of the tag is the same as your kubernetes cluster._

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uj7bf8hpvy28bcf13roq.png)

Verify from AWS console

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3tbt4larx1shp79hvnkb.png)

Verify which node is located in us-east-1a, which is where the volume was created.

```bash

# Get available Nodes
k get nodes

# Get more details about a node using the name
k describe node <name>
```
> _Note: You can create an alias for kubectl in your terminal. `alias k=kubectl`_

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1khkbrpfewgc3ydm77y0.png)

Create custom labels for nodes

```bash
# Create label for node
k label nodes i-033bf8399b48c258e zone=us-east-1a

# Verify label creation
k get node i-033bf8399b48c258e --show-labels
```

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sacuipl49rnv2ay0tl2s.png)

## Writing definition Files

***Secret definition File***
The secret object help to keep sensitive data like password. However, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. [Read more from official documentation](https://kubernetes.io/docs/concepts/configuration/secret/)

Encode for the application and RabbitMQ passwords with base64.

```bash
echo -n "<password>" | base64
```

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e4acknfk92zu3j1fqjjn.png)

Create a file app-secret.yaml
```bash
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  db-pass: cGFzcw==
  rmq-pass: Z3Vlc3Q=
```

```bash
# create secret object
k create -f app-secret.yaml

# Show secret
k get secret
```

> _Note: for production, the secret definition file should not be public because it might be decoded._

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0s8ex1v4eo5wtz4uz1z1.png)

***Database definition File***
For this file, you need the volume ID you created earlier and the zone the volume was created.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vprodb
  labels:
    app: vprodb
spec:
  selector:
    matchLabels:
      app: vprodb
  replicas: 1
  template:
    metadata:
      labels:
        app: vprodb
    spec:
      containers:
        - name: vprodb
          image: oayanda/vprofiledb:v1
          args:
            - "--ignore-db-dir=lost+found"
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: vpro-db-data
          ports:
            - name: vprodb-port
              containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: db-pass
      nodeSelector:
        zone: us-east-1a
      volumes:
        - name: vpro-db-data
          awsElasticBlockStore:
            volumeID: vol-023c6c76a8a8b98ce
            fsType: ext4
```

Create DB deployment

```bash
k create -f vprodbdep.yaml
k get pod 
```

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vkaoeqtxj60odb587ixa.png)

Verify volume is attached to pod

```bash
k describe pod pod vprodb-58b465f7f-zfth7
```

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f12qark9kjfhpzkttxpy.png)

***DB Service Definition***

This will only be exposed internally to application and not to the public.

Create definition file *db-cip.yaml*

```bash
apiVersion: v1
kind: Service
metadata:
  name: vprodb 
spec:
  ports:
    - port: 3306
      targetPort: vprodb-port
      protocol: TCP
  selector:
    app: vprodb
  type: ClusterI
```

***Memcached deployment Definition***

This will use the official docker image from docker hub.

Create definition file *mcdep.yaml*

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpromc
  labels:
    app: vpromc
spec:
  selector:
    matchLabels:
      app: vpromc
  replicas: 1
  template:
    metadata:
      labels:
        app: vpromc
    spec:
      containers:
        - name: vpromc
          image: memcached
          ports:
            - name: vpromc-port
              containerPort: 11211

```

***Memcached Service Definition***

This will only be exposed internally to application and not to the public as well.

Create definition file *mc-cip.yaml*

```bash
apiVersion: v1
kind: Service
metadata:
  name: vprocache01
spec:
  ports:
    - port: 11211
      targetPort: vpromc-port
      protocol: TCP
  selector:
    app: vpromc
  type: ClusterIP
```

***RabbitMQ Deployment Definition***

This will also use the official docker image from docker hub.

Create definition file *mcdep.yaml*

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpromq01
  labels:
    app: vpromq01
spec:
  selector:
    matchLabels:
      app: vpromq01
  replicas: 1
  template:
    metadata:
      labels:
        app: vpromq01
    spec:
      containers:
        - name: vpromq01
          image: rabbitmq
          ports:
            - name: vpromq01-port
              containerPort: 15672
          env:
            - name: RABBIT_DEFAULT_PASS
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: rmq-pass
            - name: RABBIT_DEFAULT_USER
              value: "guest"
```

***Rabbitmq Service Definition***

This will only be exposed internally to application using the Cluster IP type.

Create definition file *mc-cip.yaml*

```bash
apiVersion: v1
kind: Service
metadata:
  name: vprormq01
spec:
  ports:
    - port: 15672
      targetPort: vpromq01-port
      protocol: TCP
  selector:
    app: vpromq01
  type: ClusterIP
```

***Java Application Deployment***
I have used two inicontainers which are temporary containers which are dependencies for the Java application. Their job is to make sure the database and memcache container service are ready before the Java application container starts.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vproapp
  labels:
    app: vproapp
spec:
  selector:
    matchLabels:
      app: vproapp
  replicas: 1
  template:
    metadata:
      labels:
        app: vproapp
    spec:
      containers:
        - name: vproapp
          image: oayanda/vprofileapp:v1
          ports:
            - name: vproapp-port
              containerPort: 8080
      initContainers:
        - name: init-mydb
          image: busybox:1.28
          command: ['sh', '-c','until nslookup vprodb; do echo waiting for mydb; sleep 2; done;']
        - name: init-memcache
          image: busybox:1.28
          command: ['sh', '-c','until nslookup vprocache01; do echo waiting for memcache ; sleep 2; done;']
```

***Create Service Load balancer for Java application***

```bash
apiVersion: v1
kind: Service
metadata:
  name: vproapp-service
spec:
  ports:
    - port: 80
      targetPort: vproapp-port
      protocol: TCP
  selector:
    app: vproapp
  type: LoadBalancer
```

Now, let's deploy all the other definition files

```bash
k apply -f .
```

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2ct40t4c1varehhlkqws.png)

Verify deployment and service are created and working

```bash
k get deploy,pod,svc
```

> _Note: It might sometime for all objects to created including the Load balancer._

![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/no57fas6vko5fl4ba1qo.png)

Copy the Load balancer URL and view in browser

![load balancer](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k0br99xcjjk8hejkee5g.png)
![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z3hs0w5tux0rol92a0kd.png)

Login with the default *name:* ***admin_vp*** and *password:* ***admin_vp***
![validate cluster](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dj3a03nua04777g1p0hc.png)

**Host on Rout53**
_Step 1_
If you are using an external registrar (for example, GoDaddy).
- Create a Hosted Zone in Route53
- Copy the NS records and update it on your external registrar.
_Step 2_

In the Hosted zone on Rout53
Click create record

![create record](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8kpar4x1437zgj0dug1c.png)
- Enter a name for your application
- Click on the _Alias_ radio button
- Under the _Route traffic to_, select Alias to Application and Classic load balancer
- Select the region your application is deployed
- Select the Load balancer
- Click Create records
> This will take some seconds for propagation. 

![route 53](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/faf1xym7h81ilpcy6cqv.png)

View in the browser

![domain](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tyi1wj7zhcyjer6do0u9.png)

Clean Up
```bash
# Delete all objects
k delete -f .
```
![clean up](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pa0j50ppmbhybd2hyrc6.png)

Congratulations! you have successfully deploy an application on Kubernetes Cluster.

As always, I look forward to getting your thoughts on this article. Please feel free to leave a comment!
