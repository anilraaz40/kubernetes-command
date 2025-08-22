## kubernetes-command

### create cluster using yaml


<pre>
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
</pre>
apply yaml
<pre>
  kind create cluster --name demo-cluster --config kind-cluster.yaml

</pre>
### create cluster using command
<pre>
  kind create cluster --name demo-cluster

</pre>
### create pod using yaml
<pre>
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
</pre>

### create pod using command
<pre>
  kubectl run nginx-pod --image=nginx --port=80
</pre> 

### create replicas using yaml

<pre>
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: frontend
          image: nginx:latest
          ports:
            - containerPort: 80

</pre>

Scale replicas by command 
<pre>
  kubectl scale rs nginx-rs --replicas=2
</pre>
Scale replicas by editing yaml
<pre>
  kubectl edit rs nginx-rs
</pre>
Then update:
<pre>
  spec:
  replicas: 5

</pre>

### Deployment 

<pre>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:              
        - name: frontend        
          image: nginx:latest   
          ports:
            - containerPort: 80

</pre>
apply deployment
<pre>
  kubectl apply -f deployment.yaml
</pre>
change the version of image
<pre>
  kubectl set image deployment/nginx-deploy frontend=nginx:1.25
</pre>
check the version
<pre>
  kubectl describe deployment nginx-deploy
</pre>
rollback 
<pre>
  kubectl rollout undo deployment/nginx-deploy
</pre>

## NodePort
Step 1
If you use a Kind cluster, you must perform the port mapping to expose the container port. Use the below config to create a new Kind cluster
<pre>
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30001
    hostPort: 30001
- role: worker
- role: worker
</pre>
Step 2: Create Deployment
<pre>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:              
        - name: frontend        
          image: nginx:latest   
          ports:
            - containerPort: 80
</pre>
Step 3: Create NodePort
<pre>
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
  labels:
    env: demo
spec:
  type: NodePort
  ports:
  - nodePort: 30001
    port: 80
    targetPort: 80
  selector:
    env: demo
</pre>
To check the Services
<pre>
kubectl get svc
</pre>
To Describe the NodePort Services
<pre>
  kubectl describe svc nodeport-svc
</pre>
Check in the browser EC2 Ip address with port (30001)
<img width="1886" height="788" alt="image" src="https://github.com/user-attachments/assets/40a365e4-5e90-4e37-8344-f12280cb1e94" />

### ClusterIP
<pre>
apiVersion: v1
kind: Service
metadata:
  name: cluster-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  selector:
    app: nginx
</pre>
apply 
<pre>
  kubectl apply -f cluster.yaml 
</pre>
To describe
<pre>
  kubectl describe svc/cluster-svc
</pre>
To delete
<pre>
  kubectl delete svc/cluster-svc
</pre>

### LoadBalancer
<pre>
apiVersion: v1
kind: Service
metadata:
  name: lb-svc
  labels:
    env: demo
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    env: demo
</pre>

### ExternalName
<pre>
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.api.example.com
</pre>
