# pizza-app
Notes 

This document is a natural continution of the simpleAppInK8s.md doc
Idea of this is to evolve a deployment that has postgres deployed into a cluter
based off the digital ocean article and helped by chatGPT ¯\_(ツ)_/¯  

The initial parts of this are definetly not product ready, more useful for throwing something
together to investigate how postgres can sit in a cluster, later version my evolve the secret handling and
load balancing to be more product ready

Starting with a deployment for now, should move to a stateful set in reality
secret in config map... really you should not be that silly in production, but again this is just quick sandpit
use secrets for that....

postgres comes with docker images already available.  Since I will be using a KIND cluster I need to download that image
```
docker pull postgres:latest
docker images
```
once present it will need loading into teh kind cluster so that its available for a deployment
start the cluster and load the image into it

```bash
kind create cluster --config kind-config.yaml
kind load docker-image postgres:latest
kind load docker-image bashonly:latest
docker exec -it kind-control-plane crictl images    # to see what is loaded
kubectl create namespace pizza-app

kubectl -n pizza-app apply -f pv.yaml
kubectl -n pizza-app apply -f pvc.yaml
kubectl -n pizza-app apply -f pizza-deployment.yaml
kubectl -n pizza-app apply -f bashonly-deployment.yaml

```

chatGPT wrote a simple deployment yaml for the postgres file
```yaml
# deployment.yaml for postgres that uses an ephemeral disk for storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:latest # Adjust version as needed
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER           # do not do this in production, only for sandboxes where you can 
          value: "testuser"             # store password into git and not care
        - name: POSTGRES_PASSWORD
          value: "testpassword"
        - name: POSTGRES_DB
          value: "testdb"
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-data
      volumes:
      - name: postgres-data
        emptyDir: {}            # this create a temp, ephemeral disk that postgres will use
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  type: ClusterIP                   # clusterIP since teh (eventual) connection will be from internal pods only
```

```
kubectl -n pizza-app apply -f pizza-deployment.yaml
```

to test
connect to postgres


Create a new database:

```
CREATE DATABASE mydatabase;
```
 

Connect to mydatabase:
```
\c mydatabase
```

Create a table, insert data:
```
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(128),
    email VARCHAR(128)
);

INSERT INTO users (name, email) VALUES ('Attila', 'attila@example.com'), ('Ghengis', 'ghengis@example.com');
``` 

Run a sample query:

```
SELECT * FROM users;
```

Output:
 id | name    |  	email  	 
----+---------+------------------
  1 | Attila  | attila@example.com
  2 | Ghengis | ghengis@example.com


# mext step 
Install a bash container in the same cluster so that we can access postgre over the cluster network and prove its there

Simple bash container can b ebuildt form beloe dockerfile  (call it dockerfile.bashonly)

```bash
  FROM debian:bullseye-slim

# Install PostgreSQL client
RUN apt-get update && apt-get install -y postgresql-client && rm -rf /var/lib/apt/lists/*

# Set default command to bash
CMD ["bash"]
```
then 
`docker build -t bashonly .`

then copy over to the kind cluster

`kind load docker-image bashonly:latest`

then deploy using a deploymemnt chart 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bash-client
  labels:
    app: bash-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bash-client
  template:
    metadata:
      labels:
        app: bash-client
    spec:
      containers:
      - name: bash-client
        image: bashonly:latest  # Replace with your built image
        imagePullPolicy: IfNotPresent  # NEEDED SO KUBENETES WILL USE THE LOCAL IMAGE 
        stdin: true
        tty: true
        env:
        - name: DB_HOST
          value: "postgres-service"  # Name of the PostgreSQL service
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "testdb"
        - name: DB_USER
          value: "testuser"
        - name: DB_PASSWORD
          value: "testpassword"
```

then deploy
`kubectl -n pizza-app apply -f bashonly-deployment.yaml`

exec in 
`kubectl exec -it <bash-client-pod-name> -- /bin/bash`

open postgres tools (passeord is testpassword)
`psql -h postgres-service -p 5432 -U testuser -d testdb`


