# Coding MongoDB deployment

The first thing we are going to create a yaml file for this mongodb deployment and we are going to name it as mongo.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password

```

Realize that we will need to create a secret yaml file to store those variables MONGO_INITDB_ROOT_USERNAME and MONGO_INITDB_ROOT_PASSWORD 


# Creating our Secret

Now we will create our Secret yaml file

```
apiVersion: v1
kind: Secret
metadata:
    name: mongodb-secret
type: Opaque
data:
    mongo-root-username: dXNlcm5hbWU=
    mongo-root-password: cGFzc3dvcmQ=
```

To generate the above username and password encrypted we should use the base64 tool 

```
degutos@pop-os:~/git/kube-deployment-lab$ echo -n 'username' | base64
dXNlcm5hbWU=
degutos@pop-os:~/git/kube-deployment-lab$ echo -n 'password' | base64
cGFzc3dvcmQ=
degutos@pop-os:~/git/kube-deployment-lab$ 
```

Now we can create our secret

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl apply -f secret.yaml 
secret/mongodb-secret created
```

Lets list our secret recently created

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl get secret
NAME             TYPE     DATA   AGE
mongodb-secret   Opaque   2      117m
```



# Creating our MongoDB deployment object 

Now that we have created our Secrects and referenced them on our mongo.yaml we can apply the mongo.yaml file and create the resource 

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl apply -f mongo.yaml 
deployment.apps/mongodb-deployment created
```

Lets list all resources now

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/mongodb-deployment-67dcfb9c9f-cjnf4   1/1     Running   0          119s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5h10m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongodb-deployment   1/1     1            1           119s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/mongodb-deployment-67dcfb9c9f   1         1         1       119s
```

Realize that we have created now mongodb pod, service, deployment and replicaset


# Creating our MongoDB internal service 

We can use the same mongo.yaml file to create our service in the same yaml file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017


```

Let create now the service

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl apply -f mongo.yaml 
deployment.apps/mongodb-deployment unchanged
service/mongodb-service created
```

lets list the internal service now

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl get svc 
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP     5h15m
mongodb-service   ClusterIP   10.99.249.132   <none>        27017/TCP   30s
```

Lets Describe our Service and list our POD and compare our Endpoint service with our pod IP address

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl describe svc mongodb-service 
Name:              mongodb-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=mongodb
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.99.249.132
IPs:               10.99.249.132
Port:              <unset>  27017/TCP
TargetPort:        27017/TCP
Endpoints:         172.17.0.3:27017
Session Affinity:  None
Events:            <none>


degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl get po mongodb-deployment-67dcfb9c9f-cjnf4 -o wide
NAME                                  READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
mongodb-deployment-67dcfb9c9f-cjnf4   1/1     Running   0          9m10s   172.17.0.3   minikube   <none>           <none>
```

As we see the Endopoint Ip address is pointing to our POD IP



# Creating our ConfigMap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service
```

Lets now apply

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl apply -f mongo-configmap.yaml 
configmap/mongodb-configmap created
```

# Creating our Mongo-Express 

Now that we have our configmap and secret created we can create our mongo-express and reference those two on yaml file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url

```

Now we create our mongo-express by applying our mongo-express.yaml file

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl apply -f mongo-express.yaml 
deployment.apps/mongo-express created
```

## Checking mongodb and mongo-express running

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl get po
NAME                                  READY   STATUS    RESTARTS   AGE
mongo-express-98c6ff4b4-q5dq5         1/1     Running   0          2m3s
mongodb-deployment-67dcfb9c9f-cjnf4   1/1     Running   0          16h
```

Now we can check the mongo-express log

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl logs mongo-express-98c6ff4b4-q5dq5
Welcome to mongo-express
------------------------


(node:7) [MONGODB DRIVER] Warning: Current Server Discovery and Monitoring engine is deprecated, and will be removed in a future version. To use the new Server Discover and Monitoring engine, pass option { useUnifiedTopology: true } to the MongoClient constructor.
Mongo Express server listening at http://0.0.0.0:8081
Server is open to allow connections from anyone (0.0.0.0)
basicAuth credentials are "admin:pass", it is recommended you change this in your config.js!
```

As we see the Mongo-Express is listening on por 8081


# Creating an external service to expose the mongo-express pod

We will keep this service yaml code within the mongo-express.yaml file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url

---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer  
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000

```

Note that the External service has a type set to LoadBalancer and there is another element `nodePort` which both will allow this external service to receive external request on port 30000 of the node. We can use nodePort from 30000 to 32767

Now we can apply the new configuration with the external service added into the yaml file

```
degutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ kubectl apply -f mongo-express.yaml 
deployment.apps/mongo-express unchanged
service/mongo-express-service created
```

Realize that the external service mongo-express-service has the type set to LoadBalancer and the the internal IP is set `10.97.162.166` and the external IP which is the IP where external customer will connect to and that is pending. This is a special feature of minikube platform wher we will need to run a minikube command to get service exposed and working


```
egutos@pop-os:~/git/kube-deployment-lab/demo-kube-yaml$ minikube service mongo-express-service
|-----------|-----------------------|-------------|---------------------------|
| NAMESPACE |         NAME          | TARGET PORT |            URL            |
|-----------|-----------------------|-------------|---------------------------|
| default   | mongo-express-service |        8081 | http://192.168.49.2:30000 |
|-----------|-----------------------|-------------|---------------------------|
🎉  Opening service default/mongo-express-service in default browser...
```

Now we open our browser on that port http://192.168.49.2:30000/ and we will get the mongo express UI running 



