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



# Namespaces

- Organise resources in namespaces
- Virtual cluster inside of a cluster
- 4 Namespaces per Default


```
degutos@pop-os:~/git/wikis.wiki$ kubectl get ns
NAME              STATUS   AGE
default           Active   22h
kube-node-lease   Active   22h
kube-public       Active   22h
kube-system       Active   22h
```


- Most of resources can not be accessed from another namespace. Each NS must have its own ConfigMap and Secret.
- If you have ConfigMap in NS-1 appointing to a resource you can not use this ConfigMap in NS-2 which will need to have its own ConfigMap
- Some resources can NOT be created within a Namespace. Ex. Volume and Node they are out of the Namespace


## Namespace use cases

1. Structure your components like separating DEV, Test, Staging, Production, Database, monitoring, Elastic Stack, Nginx-ingress ...

2. Avoid conflicts between teams, imagine two teams decide create two different deployments with the same name my-app deployment. Once resource will destroy the other one.

3. Share services between different environments. Ex. when we have staging and production namespace, we could also have Elastic Stack and Nginx-ingress namespaces. We could also use Production Blue and Production green, two levels of maturety for production

4. Access and Resource Limition. We can give access to a team to access only on namespace. We can also limit cpu, ram storage per NS


## Creating a namespace

```
degutos@pop-os:~/git/wikis.wiki$ kubectl create ns my-namespace
namespace/my-namespace created

degutos@pop-os:~/git/wikis.wiki$ kubectl get ns
NAME              STATUS   AGE
default           Active   22h
kube-node-lease   Active   22h
kube-public       Active   22h
kube-system       Active   22h
my-namespace      Active   5s
```


# Ingress 


Ingress is another layer to expose our service outside. 
External service expose our service through external IP and port over 30000, but to a end product this is not what we want. We want to have some domain name and https secure protocol. For that we can use Ingress 

Ingress will receive customer request (from outside world) and send the request to the our `INTERNAL` service exposed for the pod. Eventually the external service will send the request to the pod internally

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: app.com
      http:
        paths:
          - path: /
            backend:
              serviceName: my-service
              servicePort: 8080

```

## Ingress Controller 

- Ingress controller is a pod or group of pods to 
- evaluate all the rules, 
- manage redirections
- Entrypoint to cluster
- Many options of third-party implementaions 
- k8s Nginx Ingress Controller - kube solution for ingress controller


### LoadBalancer

There are many different ways of configure our ingress. Most of Cloud provider will provide you an external LoadBalancer as an external entry point which will receive the external requests and forward to Ingress Controller to apply and evaluate all the rules
If you are configuring your cluster manually with a baremetal, you will need to provide a solution of LoadBalancer on your own.


### Installing ingress controller on Minikube

```
$ minikube addons enable ingress
```

```
degutos@pop-os:~/git/wikis.wiki$ minikube addons enable ingress
💡  ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
    ▪ Using image k8s.gcr.io/ingress-nginx/controller:v1.2.1
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```

Lets check that we have a new NS called ingress-nginx

```
degutos@pop-os:~/git/wikis.wiki$ kubectl get ns
NAME              STATUS   AGE
default           Active   28h
ingress-nginx     Active   5m
kube-node-lease   Active   28h
kube-public       Active   28h
kube-system       Active   28h
```

Lets check what pods we have within ingress-nginx ns 

```
degutos@pop-os:~/git/wikis.wiki$ kubectl get po -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-6crg5        0/1     Completed   0          8m18s
ingress-nginx-admission-patch-248n8         0/1     Completed   1          8m18s
ingress-nginx-controller-755dfbfc65-mm8nw   1/1     Running     0          8m18s
```

Now we can see that we have installed the Ingress Controller on our Cluster




### Creating a Dashboard-ingress

```
-> minikube addons enable dashboard        ~/git/kube-deployment-lab/demo-kube-yaml(main✗)@pop-os
💡  dashboard is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
    ▪ Using image kubernetesui/dashboard:v2.6.0
    ▪ Using image kubernetesui/metrics-scraper:v1.0.8
💡  Some dashboard features require the metrics-server addon. To enable all features please run:

        minikube addons enable metrics-server


🌟  The 'dashboard' addon is enabled
-> 
```

Now lets check what we have inside of kubernetes-dashboard namespace

```
-> kubectl get all -n kubernetes-dashboard ~/git/kube-deployment-lab/demo-kube-yaml(main✗)@pop-os
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-78dbd9dbf5-62v59   1/1     Running   0          30m
pod/kubernetes-dashboard-5fd5574d9f-8d8cr        1/1     Running   0          30m

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/dashboard-metrics-scraper   ClusterIP   10.102.252.81   <none>        8000/TCP   30m
service/kubernetes-dashboard        ClusterIP   10.109.68.205   <none>        80/TCP     30m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           30m
deployment.apps/kubernetes-dashboard        1/1     1            1           30m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-78dbd9dbf5   1         1         1       30m
replicaset.apps/kubernetes-dashboard-5fd5574d9f        1         1         1       30m
->    
```

Realize that we have `kubernetes-dashboard ` pod and internal service already created. Now we can create our Ingress rules in order to access Dashboard from outside by using our Host IP 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: dashboard.com
    http:
      paths:
      - path: /
        pathType: Exact  
        backend:
          service:
            name: kubernetes-dashboard
            port: 
              number: 80

```


Lets now apply our new yaml file created

```
-> kubectl apply -f dashboard-ingress.yaml ~/git/kube-deployment-lab/demo-kube-yaml(main✗)@pop-os
ingress.networking.k8s.io/dashboard-ingress created
```


Now lets check our ingress dashboard resource created 

```
-> kubectl get ingress -n kubernetes-dashboard
NAME                CLASS    HOSTS           ADDRESS        PORTS   AGE
dashboard-ingress   <none>   dashboard.com   192.168.49.2   80      77s
```

Realize that the dashboard ingress reflects the same IP of the Node `192.168.49.2`

```
> kubectl get no -o wide                  ~/git/kube-deployment-lab/demo-kube-yaml(main✗)@pop-os
NAME       STATUS   ROLES           AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION             CONTAINER-RUNTIME
minikube   Ready    control-plane   3d1h   v1.24.3   192.168.49.2   <none>        Ubuntu 20.04.4 LTS   5.18.10-76051810-generic   docker://20.10.17
```


