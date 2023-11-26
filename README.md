# Implementing VProfile Web App: Efficiently Deployed within a Kubernetes Cluster Environment.

## Overview
Utilizing the 'Containerized VProfile Web App Project' (accessible at https://github.com/Arup-gope/containerized-vpro-web), I successfully deployed this application 
within a Kubernetes cluster. The primary objective revolves around establishing the VProfile web app in a production environment, ensuring global accessibility for 
users across the world.

## Pre-requisites and Installation Instructions 
I used Kops, a multi-node Kubernetes setup on AWS, to deploy my project. Within this project, I employed images from my previous project's docker repositories. 
One critical aspect was configuring a MYSQL container requiring a volume for data storage, and I utilized Elastic Block Store (EBS) for this purpose. Additionally, 
I purchased the domain arupdevops.de specifically for Kubernetes DNS records, enabling me to access the website via a URL.

The command below creates a Kubernetes cluster with two nodes distributed across two zones. Both the master and worker nodes have 8 GB volumes. The master node uses 
a t3.medium instance, while the worker node uses a t3.small instance.
~~~
kops create cluster --name=kubevpro.arupdevops.de --state=s3://vprofile-kops-state-112 --zones=us-east-1a,us-east-1b --node-count=2 --node-size=t3.small 
--master-size=t3.medium --dns-zone=kubevpro.arupdevops.de --node-volume-size=8 --master-volume-size=8
~~~
![Create Cluster](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/b352764c-666c-4f36-8bc4-c2146c18dd5a)
Fig 01. Cluster Create

Afterward, we'll bring up and validate this cluster using the commands. The Kubernetes cluster's configuration for "kubevpro.arupdevops.de" will reference the state
 stored in the "vprofile-kops-state-112" S3 bucket. This particular S3 bucket was created in the zone where my Kops VM is currently operating.

~~~
kops update cluster --name kubevpro.arupdevops.de --state=s3://vprofile-kops-state-112  --yes --admin

kops validate cluster --name=kubevpro.arupdevops.de --state=s3://vprofile-kops-state-112 
~~~
![healthy nodes](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/fbe7f80e-d41d-4354-8235-68cb64cc2717)
Fig 02. Healthy Nodes



With the below command, we can make a new EBS volume in the 'us-east-2a' availability zone on AWS. The volume is 3 GB in size and is set to use the 'gp2' volume type.
We need to ensure that our Database Definition file runs in the exact zone where we created our EBS volume. After creating the volume, we'll assign a tag to it and 
then utilize this tag within the DB definition file.
~~~
aws ec2 create-volume --availability-zone=us-east-2a --size=3 --volume-type=gp2
~~~


## Defination Files


I've set up four Deployments and Services each for Database, Memcached, RabbitMQ, and Tomcat. Alongside these, I've also established a secret to 
securely store the encrypted passwords for MYSQL and RabbitMQ. 

This YAML defines a Kubernetes Secret named "app-secret" of type "Opaque." It contains encoded data for two keys: "db-pass" with the value "dnByb2RicGFzcw==" 
(representing an encoded form of the database password) and "rmq-pass" with the value "Z3Vlc3Q=" (representing an encoded form of the RabbitMQ password).

~~~
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  db-pass: dnByb2RicGFzcw==
  rmq-pass: Z3Vlc3Q=

~~~


This YAML defines a Kubernetes Service named "vprodb" that routes traffic on port 3306 to the "vprodb-port" within the selected pods labeled with "app: vprodb". 
It's configured as a ClusterIP type service, accessible only within the Kubernetes cluster.
~~~
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
  type: ClusterIP
~~~

This YAML defines a Kubernetes Deployment named "vprodb" with a single replica, managing a MySQL container ("vprodb") using the "vprofile/vprofiledb" image. It 
mounts an AWS EBS volume ("vol-092294d4dd43c49ab") to store MySQL data and runs an initContainer to remove the "lost+found" directory within the MySQL data directory. 
Additionally, it sets the MYSQL_ROOT_PASSWORD using a secret named "app-secret" to retrieve the encoded password for the database.

~~~
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
          image: vprofile/vprofiledb
          #args:
          #- "--ignore-db-dir"
          #- "lost+found"
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
          # This AWS EBS volume must already exist.
          awsElasticBlockStore:
             volumeID: vol-092294d4dd43c49ab
             fsType: ext4

      initContainers:
        - name: busybox
          image: busybox:latest
          args: ["rm", "-rf", "/var/lib/mysql/lost+found"]
          volumeMounts:
            - name: vpro-db-data
              mountPath: /var/lib/mysql
~~~

This YAML defines a Kubernetes Service named "vprocache01," routing traffic on port 11211 to the "vpromc-port" within pods labeled with "app: vpromc," 
configured as a ClusterIP type service.
~~~
apiVersion: v1
kind: Service
metadata:
  name: vprocache01
spec:
  selector:
    app: vpromc
  ports:
    - port: 11211
      targetPort: vpromc-port
      protocol: TCP
  type: ClusterIP
~~~

This YAML sets up a Kubernetes Deployment named "vpromc," managing a single instance of the Memcached container ("vpromc") using the default "memcached" 
image on port 11211 within pods labeled as "app: vpromc."

~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpromc
  labels:
    apps: vpromc
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
~~~

This YAML defines a Kubernetes Service named "vpromq01," directing traffic on port 15672 to the "vpromq01-port" within pods labeled as "app: vpromq01," 
configured as a ClusterIP type service.
~~~
apiVersion: v1
kind: Service
metadata:
  name: vpromq01
spec:
  selector:
    app: vpromq01
  ports:
  - port: 15672
    targetPort: vpromq01-port
    protocol: TCP

  type: ClusterIP
~~~
This YAML creates a Kubernetes Deployment called "vpromq01," managing a single instance of the RabbitMQ container ("vpromq01") using the default "rabbitmq" image on port 
15672 within pods labeled as "app: vpromq01." It also sets the RabbitMQ default password by retrieving the encoded password from a secret named "app-secret."

~~~
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
            - name: RABBITMQ_DEFAULT_PASS
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: rmq-pass
            - name: RABBITMQ_DEFAULT_USER
              value: "guest"

~~~
This YAML defines a Kubernetes Service named "vproapp-service," routing incoming traffic on port 80 to the "vproapp-port" within pods 
labeled as "app: vproapp." The service is set as a LoadBalancer type to manage external access to the application.
~~~
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
~~~


This YAML sets up a Kubernetes Deployment named "vproapp," managing a single instance of the "arupgope/vprofileapp:latest" image within
 a pod labeled as "app: vproapp." The container runs on port 8080, allowing communication via the specified port configuration.
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vproapp
  labels: 
    app: vproapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vproapp
  template:
    metadata:
      labels:
        app: vproapp
    spec:
      containers:
      - name: vproapp
        image: arupgope/vprofileapp:latest
        ports:
        - name: vproapp-port
          containerPort: 8080

~~~

## Result
Upon executing the 'kubectl create -f .' command, all four services and deployments are now operational without issues. Accessing via the provided URL from the LoadBalancer works, and thorough testing confirms MySQL, Memcached, and RabbitMQ functionalities are working perfectly.

![ALl wroking](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/355763a2-fb9d-441b-a739-d29aa0d887e8)
Fig 03. All Pods and Services are working.


![aws_Url](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/fdc924c4-46b9-4977-b41f-10e497881a2b)
Fig 04. The website is accessible through the Load Balancer URL.


![MysqlWorks](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/81ae101a-15a8-4156-a910-d33171217d9b)
Fig 05. MYSQL Works and all data are visible.

![RabbitMqWorks](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/0b8924a4-92e7-4ba0-9953-04de8033c2de)
Fig 06. RabbitMQ is working.

![MemcachedWorks](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/7735c9fd-758d-4a51-bdfc-3f07705aa9a9)
![MemcachedWorks2](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/4af16032-5c04-492d-9b5b-e435f734f336)
Fig 07. The Memcached is working if I reaccess the same profile.

Now, I'm mapping the LoadBalancer URL to Route53 for DNS resolution, allowing universal access to the website via my domain.

![route53updateblog](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/83fe4a10-56ee-4f4e-a972-0c3bc7cfe75f)
Fig 08. DNS Mapping in Route53.

The website is now accessible through my domain's URL: http://blog.kubevpro.arupdevops.de

![ProfileInNewUrl](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/70861175-8a7a-4982-881d-a79e9ba86033)
Fig 09. Website accessible through my domain URL

Starting fresh with a new profile sounds like a good way to test things out again. I am creating a new profile to test it.

![newprofiledevops](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/5cc8ab26-6165-4aa6-8712-22267a807e25)
![ProfileInNewUrl](https://github.com/Arup-gope/kube-app-kubernetes/assets/64405321/27147922-d2a2-4428-8a3d-07c21d0c9d07)
Fig 10. A new profile was created on the website.

## Acknowledgement
Acknowledging the invaluable guidance and knowledge imparted by Imran Teli in his remarkable DevOps course has significantly enriched my understanding and skills in this field. 


























































