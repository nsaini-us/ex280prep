# 1. Install DB Cockroach Operator
Create a namespace (project). Install operator into this namespace. Click on Operators -> OperatorHub
Select Database, to display the list of database operators. In the filters select "Certified" operators and then click on Cockroach Operator.
Install the operator by selecting the following options: "Update Channel = stable" and "Installation Mode = A specific namespace on the cluster" 
and enter the namespace above and then hit install.

# 2. Create DB cluster
Once operator is successfully installed, click on Operators -> Installed Operators and click on the installed operator.
Next select create instance of the db cluster. Select all defaults except turn off TLS and name as crdb-example, hit create.
Wait for the db cluster (3 nodes) to start.

`oc get po -w`

# 3. Check the service name 
check the name of the svc that contains the ClusterIP for db-cluster

`oc get svc`

Copy the cluster name `crdb-example-public`

# 4. Create the deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  labels:
    app: exoplanets
  name: exoplanets
spec:
  replicas: 1
  selector:
    matchLabels:
      app: exoplanets
  template:
    metadata:
      labels:
        app: exoplanets
    spec:
      containers:
      - env:
        - name: DB_HOST
          value: crdb-example-public
        - name: DB_NAME
          value: postgres
        - name: DB_PORT
          value: "26257"
        - name: DB_USER
          value: root
        image: quay.io/redhattraining/exoplanets:v1.0
        name: exoplanets
        ports:
        - containerPort: 8080
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
```
`oc create -f exoplanets.yaml`

The above can also be accomplished by running the following commands
```
oc new-app --name exoplanets --docker-image quay.io/redhattraining/exoplanets:v1.0 \
    -e DB_HOST=crdb-example-public -e DB_PORT=26257 -e DB_USER=root -e DB_NAME=postgres

oc set probe deployment/exoplanets --readiness --get-url http://:8080/healthz
```

# 5. create svc and route

`oc expose deployment/exoplanets --port 8080`

`oc expose svc/exoplanets --port 8080`

# 6. Test out the route using a browser
