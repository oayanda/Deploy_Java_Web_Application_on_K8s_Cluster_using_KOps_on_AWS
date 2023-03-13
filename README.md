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

Create Persistent Volume for DB Node and copy the volume ID for later use. *vol-023c6c76a8a8b98ce* and the AZ *us-east-1a*.

```bash
aws ec2 create-volume --availability-zone=us-east-1a --size=5 --volume-type=gp2 --tag-specifications 'ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=kube.oayanda.com}]'
```

> ***Note:** For volume mapping, make sure the value of the tag is the same as your k8scluster.*

![validate cluster](./images/4.png)

Verfiy from AWS console

![validate cluster](./images/9.png)

Verify the node in us-east-1a

```bash

# Get available Nodes
k get nodes

# Get more details about a node using the name
k describe node <name>
```

![validate cluster](./images/5.png)

Create custome labels for nodes

```bash
# Create label for node
k label nodes i-033bf8399b48c258e zone=us-east-1a

# Verify label creation
k get node i-033bf8399b48c258e --show-labels
```

![validate cluster](./images/6.png)

## Writing definition Files

View Docker images for application here oayanda/vprofileapp:v1
oayanda/vprofiledb:v1

***Secret definition File***

Encode for the application and rabbitmg passwords with base64.

```bash
echo -n "<password>" | base64
```

![validate cluster](./images/7.png)

Create Secret and deploy object

```bash
# create secret object
k create -f app-secret.yaml

# Show secret
k get secret
```

> ***Note** for production, the secret definition file should not be public because it might be decoded*.

![validate cluster](./images/8.png)

***Database definition File***

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

![validate cluster](./images/11.png)

Verify volume is attached to pod

```bash
k describe pod pod vprodb-58b465f7f-zfth7
```

![validate cluster](./images/10.png)

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

This will use the offical docker image from docker hub.

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

This will only be exposed internally to application and not to the public.

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

This will also use the offical docker image from docker hub.

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

This will only be exposed internally to application and not to the public.

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

***Tomcat Application Deployment***

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
          image: busybox
          command: ['sh', '-c','until nslookup vprodb; do echo waiting for mydb; sleep 2; done;']
        - name: init-memcache
          image: busybox
          command: ['sh', '-c','until nslookup vprocache01; do echo waiting for memcache ; sleep 2; done;']
```

***Create Service Load balancer for Tomcat application***

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

![validate cluster](./images/12.png)

Verify deployment and service are created and working

```bash
k get deploy,pod,svc
```

> Note: It might sometime for all to create including the Load balancer

