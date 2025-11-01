## kubernetes-command

### create cluster using yaml


```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30001
    hostPort: 30001
- role: worker
- role: worker
```
apply yaml
```
  kind create cluster --name demo-cluster --config kind-cluster.yaml

```
### create cluster using command
```
  kind create cluster --name demo-cluster
```
## Pod
A Pod is the smallest deployable unit in Kubernetes.
It runs one or more containers that share the same network namespace and storage.
Pods have their own IP address and run on a single node.
Kubernetes schedules, manages, and restarts Pods, not individual containers.

### create pod using yaml
```
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
```

### create pod using command
```
  kubectl run nginx-pod --image=nginx --port=80
```
## Replicas
A replica is one instance of a Pod.
Replica count ensures the desired number of identical Pods are always running.
Kubernetes uses a ReplicaSet/Deployment to maintain these replicas.
If a Pod dies, the ReplicaSet automatically creates a new one to keep the count constant.
## ReplicaSet
A ReplicaSet ensures a specified number of identical Pods are always running.
It continuously monitors Pods and replaces any that fail.
It uses selectors to identify which Pods it manages.
Deployments internally use ReplicaSets to handle scaling and rolling updates.

### create replicas using yaml

```
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

```

Scale replicas by command 
```
kubectl scale rs nginx-rs --replicas=2
```
Scale replicas by editing yaml
```
  kubectl edit rs nginx-rs
```
Then update:
<pre>
  spec:
  replicas: 5

</pre>

## Deployment 
A Deployment is a higher-level Kubernetes object that manages ReplicaSets and Pods.
It provides rolling updates, rollbacks, and version control for your application.
It ensures the desired number of replicas always run with the correct Pod template.
Deployments make updating and scaling applications safe and automatic.

```
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

```
apply deployment
```
  kubectl apply -f deployment.yaml
```
change the version of image
```
  kubectl set image deployment/nginx-deploy frontend=nginx:1.25
```
check the version
```
  kubectl describe deployment nginx-deploy
```
rollback 
```
  kubectl rollout undo deployment/nginx-deploy
```

## NodePort
A NodePort is a Kubernetes Service type that exposes your application on a port of every worker node.
It opens a fixed port (30000–32767) on each node and forwards traffic to the target Pods.
You can access the app using NodeIP:NodePort from outside the cluster.
It is the simplest way to make a Pod reachable externally.

Step 1
If you use a Kind cluster, you must perform the port mapping to expose the container port. Use the below config to create a new Kind cluster

**We use NodePort in Kubernetes to expose a pod or Deployment outside the cluster, so it can be accessed from outside the Kubernetes network.**

Your Deployment runs Nginx pods inside the cluster.

By default, those pods are not accessible from your browser because they only have a ClusterIP.

By creating a NodePort Service, Kubernetes opens a port on the node (e.g., 30001) that forwards traffic to your pods.

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30001
    hostPort: 30001
- role: worker
- role: worker
```

Step 2: Create Deployment
```
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
```

Step 3: Create NodePort
```
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```

To check the Services
```
kubectl get svc
```
To Describe the NodePort Services
```
  kubectl describe svc nodeport-svc
```

Check in the browser EC2 Ip address with port (30001)
<img width="1886" height="788" alt="image" src="https://github.com/user-attachments/assets/40a365e4-5e90-4e37-8344-f12280cb1e94" />

## Custome html page
**Step 1.** Create a file index.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My Custom Nginx Page</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            background-color: #f2f2f2;
            color: #333;
            margin-top: 50px;
        }
        h1 {
            color: #ff6600;
        }
        p {
            font-size: 18px;
        }
    </style>
</head>
<body>
    <h1>Welcome to My Nginx Server!</h1>
    <p>This page is served from a Kubernetes NodePort service.</p>
</body>
</html>
```

**Step 2.** Create a ConfigMap from this file
```
  kubectl create configmap nginx-html --from-file=index.html
```

**Step 3:** Update your Deployment to mount the ConfigMap

```
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
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: nginx-html

```
Apply it:
```
  kubectl apply -f nginx-deploy.yaml
```

Step 4: Use port-forwarding or nodePort or LoadBalancer or ClusterIp

**By port-forwarding**
```
  kubectl port-forward --address 0.0.0.0 deployment/nginx-deploy 8080:80
```
Access using ec2-ip:8080
```
  http://65.1.65.70:8080/
```
**By nodePort**
Cluster should be created using extraPortMapping

create file and apply this file
```
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```


Step 5: Access your page

Open your browser and go to:
<pre>
  http://NODE-IP:30001
</pre>
or
<pre>
  http://EC2-public-IP:30001
</pre>
<img width="2652" height="612" alt="image" src="https://github.com/user-attachments/assets/6201cd50-2276-4264-ac7e-b5891fa71f2f" />


### ClusterIP
ClusterIP is the default Kubernetes Service type used for internal communication.
It gives a stable internal IP address so Pods can talk to each other inside the cluster.
It cannot be accessed from outside the cluster.
Used for internal services like backend-to-database communication.
It does not expose the Service outside of the cluster (so you can’t access it from your laptop/browser directly).

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
apply 
```
  kubectl apply -f cluster.yaml 
```
To describe
```
  kubectl describe svc/cluster-svc
```
<pre>
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-clusterip   ClusterIP   10.96.85.123    <none>        80/TCP    2m
</pre> 
To delete
```
  kubectl delete svc/cluster-svc
```

If another Pod tries to connect to http://10.96.85.123:80, it will reach your Nginx app.

**🔹 Use cases**

  Useful for internal communication between microservices (backend ↔ database, frontend ↔ backend).

  Not for external browser access (for that you need NodePort, LoadBalancer, Ingress, or port-forward).

<pre>
Difference from other types:

ClusterIP (default): Accessible only inside the cluster.

NodePort: Exposes app on each node’s IP at a fixed port (not cloud friendly, manual).

LoadBalancer: Cloud provider gives you a public IP/DNS → easiest way for external users.

Ingress: Adds advanced routing on top of LoadBalancer/NodePort.
</pre>
### LoadBalancer
```
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
```

### ExternalName
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.api.example.com
```

## Multi-container
create a pod which do some work and depend on other work
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    env:
    - name: FIRSTNAME
      value: "Piyush"
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c'] #command to run
    args: ['until nslookup myservice.default.svc.cluster.local; do echo waiting for myservice; sleep 2; done']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c']
    args: ['until nslookup mydb.default.svc.cluster.local; do echo waiting for mydb; sleep 2; done']
```
apply the pod and check the status
```
  kubectl apply -f pod.yaml
  kubectl get pods
```
It will show init 0/1

Now check the logs

```
  kubectl logs myapp-pod -c init-myservice
```
it will diplay like can't resolved
<img width="1446" height="342" alt="image" src="https://github.com/user-attachments/assets/885ef551-8aeb-4961-8ec3-3fb128b077ac" />

Now create a deployment and apply the deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myservice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myservice
  template:
    metadata:
      labels:
        app: myservice
    spec:
      containers:
      - name: myservice
        image: busybox:1.28
        command: ['sh', '-c', 'echo myservice is ready && sleep 3600']
```

Create a service and expose the service
```
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  selector:
    app: myservice
  ports:
    - port: 80
      targetPort: 80
```
check the pod now it should be running
```
  kubectl get pod myapp-pod
```
<pre>
  NAME         READY   STATUS    RESTARTS   AGE
  myapp-pod    1/1     Running   0          2m

</pre>
Now describe the pod
```
  kubectl logs myapp-pod -c init-myservice
```
it should be deplay like
<img width="1128" height="154" alt="image" src="https://github.com/user-attachments/assets/76f2585f-2097-4c23-b8bc-7fbcc840a732" />

## DaemonSet
A DaemonSet ensures that one Pod runs on every node (or on selected nodes) in the cluster.
When new nodes are added, the DaemonSet automatically adds Pods to them.
Commonly used for node-level agents like log collectors and monitoring tools.
If a node is removed, the DaemonSet removes the Pod from it.

**create a file and appply this file**
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemon
spec:
  selector:
    matchLabels:
      app: my-daemon
  template:
    metadata:
      labels:
        app: my-daemon
    spec:
      containers:
      - name: my-daemon
        image: busybox
        command: ["sh", "-c", "while true; do echo Running on $(hostname); sleep 10; done"]

```
run this command to check pod running
```
  kubectl get pods -o wide
```
output should be like this
<pre>
NAME             READY   STATUS    NODE
my-daemon-xxx1   1/1     Running   worker-node-1
my-daemon-xxx2   1/1     Running   worker-node-2

</pre>
Check logs of one pod
```
  kubectl logs my-daemon-aaa1
```
## JOB
A Job runs a task once (or a set number of times) until it successfully completes.
It ensures Pods run to completion instead of running continuously.
If a Pod fails, the Job creates a new one to finish the task.
Used for batch tasks like backups, data processing, or one-time scripts.

**create file and apply**
```
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Hello from Kubernetes Job!"]
      restartPolicy: Never
  backoffLimit: 2
```
Run it:
```
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/hello-job
```
You’ll see output:
<pre>
  Hello from Kubernetes Job!
</pre>

## CronJob
A CronJob runs Jobs on a schedule, similar to Linux cron.
It uses a cron expression to decide when to run.
Each scheduled run creates a Job that runs to completion.
Used for periodic tasks like daily backups or cleanup scripts.

**Create file and run file**
<pre>
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "*/1 * * * *"   # every 1 minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["sh", "-c", "date; echo Hello from Kubernetes CronJob!"]
          restartPolicy: Never

</pre>
Run it:
<pre>
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get jobs
kubectl logs <job-pod-name>

</pre>
You’ll see logs like:
<pre>
Sat Aug 24 11:45:01 UTC 2025
Hello from Kubernetes CronJob!

</pre>

<pre>
✅ Why two different versions?

Because Kubernetes organizes resources into API groups.

Resource Type	              API Group	        apiVersion
Pod, Service, ConfigMap	    core	            v1
Deployment, StatefulSet	    apps	            apps/v1
Ingress	                    networking.k8s.io	networking.k8s.io/v1
CronJob	                    batch	            batch/v1
</pre>

## Taint & Tolerant & nodeSelector
Taints and tolerations are used to control which Pods can run on which Nodes.
A taint on a node prevents regular Pods from being scheduled there.
A toleration on a Pod allows it to run on a tainted node.
They are used for dedicated nodes, GPU nodes, system workloads, and maintenance scenarios.
This ensures only the right Pods run on specific nodes.

In Kubernetes, taints and tolerations work together to control which pods can be scheduled onto which nodes.
### Taint
 1. Applied on a node.
 2. Means “do not schedule pods here unless they tolerate this taint.”
```
  kubectl taint nodes <node-name> key=value:effect
```
Effects:

**NoSchedule** → Pods that don’t tolerate the taint will not be scheduled.

**PreferNoSchedule** → Tries to avoid placing pods here, but not strict.

**NoExecute** → Evicts existing pods that don’t tolerate it, and stops new ones from being scheduled.
```
  kubectl taint nodes worker1 dedicated=database:NoSchedule
```
### Toleration
Applied on a pod.

Tells the scheduler: “I can tolerate this taint.”

Defined inside the Pod spec:
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx

```
**How they work together:**
If a node has a taint, only pods with a matching toleration can be scheduled there.
If no toleration exists, the pod is kept away from that node.

### nodeSelector
NodeSelector is the simplest way to schedule a Pod on specific nodes.
You add labels to nodes, and the Pod’s nodeSelector must match those labels.
Only nodes with the matching label will run that Pod.
Used when you want to place a workload on particular nodes without taints or tolerations.

<p>NodeSelector schedules a pod only on nodes with specific labels.</p>
<pre>
  kubectl label nodes node-name dedicated=database
</pre>
<pre>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-nodeselector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        dedicated: database
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

</pre> 


<pre>
✅ Summary

Mechanism	Node       Requirement	                Pod Spec Requirement

NodeSelector	       Node must have label	        Pod must have nodeSelector

Toleration	         Node must have taint	        Pod must have toleration
</pre>

### If you are using nodeSelector with labels, but the target node has a taint, the pod will not run unless it also has a matching toleration.


## Affinity
Affinity is an advanced way to control where Pods are scheduled, more flexible than nodeSelector.
It lets you place Pods based on node labels (Node Affinity) or other Pods (Pod Affinity/Anti-Affinity).
You can make rules that are required (must follow) or preferred (best effort).
Used for grouping Pods together, separating Pods, or placing them on specific types of nodes.

**Create pod or deployment**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

```
**Step 1.** Label your node with disktype=ssd
```
  kubectl label nodes <your-node-name> disktype=ssd
```
**Step 2.** Apply the YAML:
```
  kubectl apply -f nginx-deploy.yaml
```
**Step 3.** Check Pods:
```
  kubectl get pods -o wide
```

**1. nodeSelector**

Simplest way to tell which node(s) a Pod can run on.

Pod requires nodes to have exact label match.

Hard rule → If no node has the label, Pod stays Pending.

**2. Node Affinity**

Advanced version of nodeSelector.

**Two modes:**

requiredDuringScheduling → Hard rule (like nodeSelector). Pod stays Pending if no match.

preferredDuringScheduling → Soft rule. Scheduler tries to honor it, but if no match, Pod will still run elsewhere.

Allows operators (In, NotIn, Exists, etc.) and preferences.

**3. Taints & Tolerations**

Node-side mechanism: a taint repels Pods unless they explicitly tolerate it.

Without toleration, Pod cannot run on a tainted node.

Controls where Pods cannot be scheduled unless allowed.

Effects:

NoSchedule → Pod won’t schedule.

PreferNoSchedule → Avoid scheduling if possible.

NoExecute → Evicts running Pods if they don’t tolerate.

**✅ One-liner difference:**

nodeSelector → Pod chooses nodes by exact label.

nodeAffinity → Pod chooses nodes with flexible/advanced rules (hard or soft).

Taints & Tolerations → Node repels Pods unless Pods tolerate the taint.

## Kubernetes Requests and Limits

**Requests:**
Minimum resources (CPU/Memory) guaranteed for a container.
Scheduler uses this value to decide which node the pod can run on.
Example: requests.memory: "50Mi" → Pod will be placed on a node with at least 50Mi available.

**Limit:**
Maximum resources (CPU/Memory) a container can use.
If memory usage > limit → Container gets OOMKilled.
If CPU usage > limit → Container is throttled (not killed).

Create a pod which stress the memory 

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```
output
<pre>
kubectl get pods -n mem-example
  
NAME            READY   STATUS      RESTARTS      AGE
memory-demo-2   0/1     OOMKilled   2 (18s ago)   26s
</pre>
OOM (Out of memory)

If a container tries to use more memory than its limit, the Kubernetes kubelet kills the container with status OOMKilled, even though the node might have free memory.

The Below pod will be scheduled
```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```
<pre>
kubectl get pods -n mem-example
  
NAME            READY   STATUS      RESTARTS      AGE
memory-demo     1/1     Running     0             18s
memory-demo-2   0/1     OOMKilled   4 (53s ago)   103s
root@ip-172-31-32-126:~# 
</pre>

## Health Probes

**What are probes?**

To investigate or monitor something and to take necessary actions

**What are health probes in Kubernetes?**

Health probes monitor your Kubernetes applications and take necessary actions to recover from failure
To ensure your application is highly available and self-healing

**Type of health probes in Kubernetes**

Readiness ( Ensure application is ready)
Liveness ( Restart the application if health checks fail)
Startup ( Probes for legacy applications that need a lot of time to start)

**Types of health checks they perform?**

HTTP/TCP/command

liveness-http and readiness-http

<pre>
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    args:
    - liveness
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
</pre>

liveness command
<pre>
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat 
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
</pre>

liveness-tcp
<pre>
apiVersion: v1
kind: Pod
metadata:
  name: tcp-pod
  labels:
    app: tcp-pod
spec:
  containers:
  - name: goproxy
    image: registry.k8s.io/goproxy:0.1
    ports:
    - containerPort: 8080
    livenessProbe:
      tcpSocket:
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
</pre>

### RBAC (Role Based Access Control Kubernetes)

<pre>
  kubernetes auth can-i get pod
</pre>
<pre>
  kubernetes auth whoami
</pre>
<pre>
  kubernets auth can-i get pod as krish
</pre>

# Certificate TLS

### Kubernetes CertificateSigningRequest
1. Generate a private key and CSR:
   <pre>
      openssl genrsa -out my-user.key 2048
      openssl req -new -key my-user.key -subj "/CN=my-user" -out my-user.csr
   </pre>


2. Create csr.yaml file

<pre>
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: my-user-csr
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRV...   # base64-encoded CSR
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # (optional) 1 year = 365*24*60*60
  usages:
  - digital signature
  - key encipherment
  - client auth
</pre>

3. Base64 encode the CSR (remove newlines):
<pre>
  cat my-user.csr | base64 | tr -d '\n'
</pre>

4. Put that string under spec.request in the YAML.

5. Apply the CSR:
   <pre>
     kubectl apply -f csr.yaml

   </pre>
  
6. Approve certificate
   <pre> kubectl certificate approve my-user-csr </pre>

   <pre>kubectl get csr </pre>
   <pre> kubectl get csr adam -o yaml > issuecert.yaml </pre>

## Resource (pod) accessing using custom user

 ###Create Role yaml
   
  A Role defines what actions are allowed on which resources within a namespace

<pre>
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: demo-ns
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

</pre>

This role allows get, list, and watch access to Pods in the demo-ns namespace.

###RoleBinding YAML

A RoleBinding attaches the above Role to a user, group, or ServiceAccount.
<pre>
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: demo-ns
subjects:
- kind: User
  name: my-user       # Must match CN in the certificate
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

</pre>

<p>
###Key Points

subjects.kind can be:

User → Maps to a client certificate CN (like my-user from your CSR).

Group → Maps to the CSR’s O (organization field).

ServiceAccount → Maps to a namespace + ServiceAccount.

roleRef must exactly reference the Role you created.
</p>

## Custom context
### To create a context in Kubernetes for a user named adam, you need three parts:
1. A cluster (already defined in kubeconfig or you must create one).
2. A user (here adam).
3. The context that ties them together.

### Steps
1. Add the user adam to kubeconfig
   (Example: using a client certificate for authentication)
   <pre>
     kubectl config set-credentials adam --client-certificate=/path/to/adam.crt --client-key=/path/to/adam.key
   </pre>
2. Create a context that uses adam
   Suppose the cluster name is my-cluster and namespace is default:
   <pre>
     kubectl config set-context adam-context --cluster=my-cluster --namespace=default --user=adam
   </pre>
3. Switch to the new context
   <pre>
     kubectl config use-context adam-context
   </pre>
4. Verify
   <pre>
     kubectl config get-contexts
     kubectl config current-context
   </pre>

<img width="998" height="738" alt="image" src="https://github.com/user-attachments/assets/677edfb2-f05f-4b81-92b4-34c27771497e" />

## ClusterRoleBinding

### Example: Give Adam admin access across the cluster
1. Create a role 
<pre>
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: adam-cluster-role
  rules:
  - apiGroups: [""]   # "" means the core API group
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]

</pre>
<pre>
  kubectl apply -f role.yaml
</pre>
This role lets Adam manage pods, services, and configmaps.

2. Bind the role to Adam
<pre>
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: adam-binding
  subjects:
  - kind: User
    name: adam           
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: adam-cluster-role  
    apiGroup: rbac.authorization.k8s.io

   </pre>
   <pre>
     kubectl apply -f adam-binding.yaml
   </pre>

3. Test with Adam’s context
   <pre>
       kubectl config use-context adam-context
       kubectl get pods -A

   </pre>
   
# Network Policies
### What is a network policy
Network policy allows you to control the Inbound and Outbound traffic to and from the cluster. For example, you can specify a deny-all network policy that restricts all incoming traffic to the cluster, or you can create an allow-network policy that will only allow certain services to be accessed by certain pods on a specific port.
<img width="1407" height="605" alt="image" src="https://github.com/user-attachments/assets/1acb64e7-99ca-436b-a413-3adb2740a72f" />

1. Create a cluster

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
  networking:
    disableDefaultCNI: true
    podSubnet: 192.168.0.0/16
</pre>
<pre>
  kubectl get nodes -o wide
</pre> 
return like this NotReady
<pre>
  NAME                STATUS      ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
  dev-control-plane   NotReady    control-plane   4m     v1.25.0   172.18.0.2    <none>        Ubuntu 22.04.1 LTS   5.10.0-17-amd64   containerd://1.6.7
  dev-worker          NotReady    <none>          4m     v1.25.0   172.18.0.4    <none>        Ubuntu 22.04.1 LTS   5.10.0-17-amd64   containerd://1.6.7
  dev-worker2         NotReady    <none>          4m     v1.25.0   172.18.0.3    <none>        Ubuntu 22.04.1 LTS   5.10.0-17-amd64   containerd://1.6.7
</pre>

2.  install calico CNI
<pre>
       kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
</pre>
3. Verify Calico installation
<pre>
       NAMESPACE     NAME                READY   STATUS    RESTARTS   AGE
       kube-system   calico-node-2xcf4   1/1     Running   0          57s
       kube-system   calico-node-gkqkg   1/1     Running   0          57s
       kube-system   calico-node-j44hp   1/1     Running   0          57s 
</pre>
4. Create 3 pods and 3 services
<pre>
      apiVersion: v1
      kind: Pod
      metadata:
        name: frontend
        labels:
          role: frontend
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - name: http
            containerPort: 80
            protocol: TCP
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: frontend
        labels:
          role: frontend
      spec:
        selector:
          role: frontend
        ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: backend
        labels:
          role: backend
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - name: http
            containerPort: 80
            protocol: TCP
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: backend
        labels:
          role: backend
      spec:
        selector:
          role: backend
        ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: db
        labels:
          name: mysql
      spec:
        selector:
          name: mysql
        ports:
        - protocol: TCP
          port: 3306
          targetPort: 3306
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: mysql
        labels:
          name: mysql
      spec:
        containers:
          - name: mysql
            image: mysql:latest
            env:
              - name: "MYSQL_USER"
                value: "mysql"
              - name: "MYSQL_PASSWORD"
                value: "mysql"
              - name: "MYSQL_DATABASE"
                value: "testdb"
              - name: "MYSQL_ROOT_PASSWORD"
                value: "verysecure"
            ports:
              - name: http
                containerPort: 3306
                protocol: TCP
</pre>
Now exec in pods and try to make connection with other pods
<pre>
     kubectl exec -it frontend -- bash
</pre>
<pre>
     curl backend 
</pre>
<pre>
     apt-get update && apt-get install telnet -y
     telnet db 3306
</pre>
It will ablne to make the connection

### Now apply the network policy which only backend will able to make connection with db

apply this np
<pre>
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: db-test
    spec:
      podSelector:
        matchLabels:
          name: mysql
      policyTypes:
      - Ingress
      ingress:
      - from:
        - podSelector:
            matchLabels:
              role: backend
        ports:
        - port: 3306
</pre>
Now again exec into the pods and try to make connection, now only backend pod will able to make connection with db, not frontend pod

# Setup a Multi Node (EC2) Kubernetes Cluster Using Kubeadm

1. Create 2 Security groups 1st for master node and second for worker node
for master node SG inblund rule
<pre>	
  TCP 6443
  TCP 10250
  TCP 10259
  TCP 30000-32767
  TCP 10257
  SSH 22
  TCP 2379-2380
</pre>
for worker node SG inbound rule
<pre>	
  TCP 6443
  TCP 10250
  TCP 10259
  TCP 30000-32767
  SSH 22
  TCP 2379-2380
</pre>

2. Create 3 EC2 instance attach master SG for master EC2 and attach worker SG for worker node(EC2)
3. Update (All Nodes)
<pre>
  sudo apt update && sudo apt upgrade -y
</pre>
4. Prepare Linux kernel settings & disable swap (ALL NODES)
```
  # disable swap immediately and persistently
  swapoff -a
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Forwarding IPv4 and letting iptables see bridged traffic
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```
5. Install containerd (ALL NODES)
```
# install prerequisites
sudo apt install -y ca-certificates curl gnupg lsb-release

# install containerd
sudo apt update
sudo apt install -y containerd

# create config and set systemd cgroup
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# set SystemdCgroup = true (ensures containerd uses systemd cgroup driver)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml || true

# restart & enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```
6. Install runc (ALL NODES)
```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
7. install cni plugin (ALL NODES)
```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```
8. Install kubeadm, kubelet, kubectl (ALL NODES)
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages
sudo apt-mark hold kubelet kubeadm kubectl

kubeadm version
kubelet --version
kubectl version --client
```
9. Configure crictl to work with containerd (ALL NODES)
```
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```
10. initialize control plane (Master Node only)
```
kubeadm init --apiserver-advertise-address=<master-private-ip> --pod-network-cidr=192.168.0.0/16
```
11. Prepare kubeconfig (Master Node only)
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
12. Install calico (Master Node only)
```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```
13. Run this command to generate the Token in master node (Master Node only)
```
kubeadm token create --print-join-command 
```
14. Copy token command from master node and run in worker node (worker node)
15. Final verification (Master node)
```
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
```
# Volume
<img width="797" height="560" alt="image" src="https://github.com/user-attachments/assets/97a3dd03-c7c1-4e84-b27f-bf15b423af51" />

### Docker volume
1. install docker and clone any github project
sample project
```
git clone https://github.com/docker/getting-started-app.git
```
2. create Dockerfile
```
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```
3. Create image
```
docker build -t myapp .
```
4. Craete docker volume
```
docker volume create vol_data
docker volume ls
```
5. Create container from docker image
```
docker run -d -p3000:3000 -vol_data:/app --name=todo myapp
```
All volume data is available in this folder
```
/var/lib/docker/volumes/vol_data/_data/
```
6. list container
```
docker ps
```
7. Exec into container
```
docker exec -it <container_id> sh
```
8. Create any file or anything just to check that data is available or not after container delete or stop,
sample here i created one folder
```
mkdir myfolder
```
9. exit from container
```
exit
```
11. Delete the container
```
docker rm -f <container_id>
```
12. again create a new container and check sample data is availble or not
```
docker run -d -p3000:3000 -vol_data:/app --name=newtodo myapp
docker exec -it <container_id> sh
ls
```
Data will be there which i created a sample folder

### Kubernetes Volume
<img width="1073" height="763" alt="image" src="https://github.com/user-attachments/assets/515eabec-cec9-4094-a851-4b755c783bcf" />

1. create a cluster with extraMount
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /mnt/data/pv1   # EC2 host path
        containerPath: /mnt/data/pv1   # inside kind node
  - role: worker
    extraMounts:
      - hostPath: /mnt/data/pv1
        containerPath: /mnt/data/pv1
  - role: worker
    extraMounts:
      - hostPath: /mnt/data/pv1
        containerPath: /mnt/data/pv1
```
2. create PV
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/pv1"
  persistentVolumeReclaimPolicy: Retain
```
3. Create PVC
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: ""
```
4. create pod
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: mydata
  volumes:
    - name: mydata
      persistentVolumeClaim:
        claimName: local-pvc
```
5. exec into pod
```
kubectl exec -it nginx-pod -- bash

```
6. change something in data
```
echo "Hello EC2" > /usr/share/nginx/html/index.html
```
7. exit from pod
```
exit
```
8. Directly check on EC2
```
cat /mnt/data/pv1/index.html
```
✅ You’ll see:
<pre>
  Hello EC2
</pre>

How data is going from pod to EC2 local storage
<pre>
  -> Pod container → Volume mount (/usr/share/nginx/html)
  → PVC (local-pvc)
  → PV (local-pv)
  → hostPath (/mnt/data/pv1 inside kind node)
  → bind mount
  → /mnt/data/pv1 on EC2 host

</pre>


# Ingress

vi kind-config.yaml
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80   # expose http
        hostPort: 80
      - containerPort: 443  # expose https
        hostPort: 443
```
apply file
```
kind create cluster --name demo --config kind-config.yaml
kubectl get nodes

```

```
kubectl label node <your-node-name> ingress-ready=true

```
Install Ingress Controller (NGINX)
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/kind/deploy.yaml

```
### Create Simple Web Apps

We’ll use two deployments + services.

app1.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: app1-html
---
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-html
data:
  index.html: |
    <h1>Welcome to App1</h1>
```
Create app2.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: app2-html
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  selector:
    app: app2
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app2-html
data:
  index.html: |
    <h1>Welcome to App2</h1>
```
apply file
```
kubectl apply -f app1.yaml
kubectl apply -f app2.yaml
```
### Create Ingress Rule path-Based-Routing
Create ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
```

apply
```
kubectl apply -f ingress.yaml

```


### Test Website
Check ingress:
```
kubectl get ingress
```
### Now open browser:
```
http://<EC2_PUBLIC_IP>/app1
```
Welcome to App1
```
http://<EC2_PUBLIC_IP>/app2
```
Welcome to App2

## OR 

### Ingress for Host-Based Routing
Create ingress-host.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress-host
spec:
  ingressClassName: nginx
  rules:
  - host: app1.flipkart.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.flipkart.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
```
apply file
```
kubectl apply -f ingress-host.yaml
```
### DNS Resolution for Testing

Since you don’t own real domains yet, you can trick your local machine (or EC2 itself) to resolve these hostnames.

On your local laptop/PC, edit the /etc/hosts file:
```
<EC2_PUBLIC_IP> app1.flipkart.com
<EC2_PUBLIC_IP> app2.flipkart.com
```
### Test in Browser
```
http://app1.flipkart.com → shows Welcome to App1

http://app2.flipkart.com → shows Welcome to App2)
```

