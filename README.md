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


