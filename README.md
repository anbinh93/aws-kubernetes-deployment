# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

This repo available on Window, on Linux may different

## Getting Started

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup
#### 1. Create EKS Cluster
Install eksctl(opens in a new tab) and use it to create an EKS cluster. It's as simple as running a single command to create a cluster:
```
eksctl create cluster --name my-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2
```
After creating an EKS cluster, execute the command below to update the context in the local Kubeconfig file. This file allows configuring access to clusters, and should have at least one active context. It contains the information about clusters, users, namespaces, and authentication mechanisms.
```
aws eks --region us-east-1 update-kubeconfig --name my-cluster
```
#### 2. Configure a Database
Set up a Postgres database

2.1 Create a file pvc.yaml on your local machine, with the following content.

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

2.2. Create a file pv.yaml on your local machine, with the following content.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  hostPath:
    path: "/mnt/data"
```
2.3. Create a file postgresql-deployment.yaml on your local machine, with the following content. It will use the official Postgres Docker image for deployment.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:latest
        env:
        - name: POSTGRES_DB
          value: mydatabase
        - name: POSTGRES_USER
          value: myuser
        - name: POSTGRES_PASSWORD
          value: mypassword
        ports:
        - containerPort: 5432
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgresql-storage
      volumes:
      - name: postgresql-storage
        persistentVolumeClaim:
          claimName: postgresql-pvc
```
Note the database, user, and passwords in the YAML above. You can change them per your choice. Here are the sample values.
* Database name: <strong>mydatabase</strong>
* User name: <strong>myuser</strong>
* Password: <strong>mypassword</strong>

Apply YAML configurations in the following order.
```
kubectl apply -f pvc.yaml
kubectl apply -f pv.yaml
kubectl apply -f postgresql-deployment.yaml
```

1.4. Test database connection
View the pods
```
kubectl get pods
```
![Alt text](.\images\get_pod.png?raw=true "Title")

Assuming the postgres pod name is ```postgresql-77d75d45d5-b46w8```, run the following command to open bash into the pod.
```
kubectl exec -it postgresql-77d75d45d5-b46w8 -- bash
```
Once you are inside the pod, you can run
```
psql -U myuser -d mydatabase
```
Ensure to change the username and password, as applicable to you.
Once you are inside the postgres database, you can list all databases
```
\l
```
1.5. Connecting via Port Forwarding
Before you expose your database, you will need to create a service, and then expose it using port-forwarding approach.

Create a YAML file, <strong>postgresql-service.yaml</strong>, in your current working directory.

```
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgresql
```
This YAML defines a Service named ```postgresql-service```x` that targets pods with the label ```app.kubernetes.io/name=postgresql``` on port 5432, which is the default port for PostgreSQL. Apply this YAML to create the Service:
```
kubectl apply -f postgresql-service.yaml
```
When you have exited the pod and arrived back to your local environment, run the following
```
# List the services
kubectl get svc

# Set up port-forwarding to `postgresql-service`
kubectl port-forward service/postgresql-service 5433:5432
```

![Alt text](.\images\get_svc.png?raw=true "Title")
![Alt text](.\images\describe_service.png?raw=true "Title")
![Alt text](.\images\describe_deployment.png?raw=true "Title")


Now you can use PGAdmin to connect to your database
You can alse use command in <strong>/db</strong> folder to create tables

#### 2. Build the analytics application
Using ```Dockerfile``` in <strong>/analytics</strong> folder to build image 
```
docker build -t test-coworking-analytics .
```

After that you can the command below to test your application
```
docker run -e DB_USERNAME='myuser' -e DB_PASSWORD='mypassword' -e DB_PORT=5433 -e DB_NAME='mydatabase' -e DB_HOST='host.docker.internal' -p 5153:5153 test-coworking-analytics
```
Here ```host.docker.internal``` to make the application connect to the database in the localhost 

You can use Postman to test the API using 
```
GET http://127.0.0.1:5153/api/reports/user_visits
```
Here is the image result
![Alt text](.\images\send_success_api.png?raw=true "Title")

#### 3. Check log from Cloudwatch

You can access cloudwatch to watch the logs

![Alt text](.\images\cloudwatch.png?raw=true "Title")
![Alt text](.\images\log.png?raw=true "Title")

