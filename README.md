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

**We use NodePort in Kubernetes to expose a pod or Deployment outside the cluster, so it can be accessed from outside the Kubernetes network.**

Your Deployment runs Nginx pods inside the cluster.

By default, those pods are not accessible from your browser because they only have a ClusterIP.

By creating a NodePort Service, Kubernetes opens a port on the node (e.g., 30001) that forwards traffic to your pods.
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
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
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

## Custome html page or other page 
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
<pre>
  kubectl create configmap nginx-html --from-file=index.html
</pre>

**Step 3:** Update your Deployment to mount the ConfigMap

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
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: nginx-html

</pre>
Apply it:
<pre>
  kubectl apply -f nginx-deploy.yaml
</pre>

Step 4: Use port-forwarding or nodePort or LoadBalancer or ClusterIp

**By port-forwarding**
<pre>
  kubectl port-forward --address 0.0.0.0 deployment/nginx-deploy 8080:80
</pre> 
Access using ec2-ip:8080
<pre>
  http://65.1.65.70:8080/
</pre>
**By nodePort**
Cluster should be created using extraPortMapping

create file and apply this file
<pre>
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
</pre>


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

**🔹 What it does:**

It gives your Service a stable internal IP address inside the cluster.

That IP is only reachable within the cluster network (pods, nodes, other services).

It does not expose the Service outside of the cluster (so you can’t access it from your laptop/browser directly).

<pre>
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
</pre>
apply 
<pre>
  kubectl apply -f cluster.yaml 
</pre>
To describe
<pre>
  kubectl describe svc/cluster-svc
</pre>
<pre>
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-clusterip   ClusterIP   10.96.85.123    <none>        80/TCP    2m
</pre> 
To delete
<pre>
  kubectl delete svc/cluster-svc
</pre>

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

## Multi-container
create a pod which do some work and depend on other work
<pre>
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
</pre>
apply the pod and check the status
<pre>
  kubectl apply -f pod.yaml
  kubectl get pods
</pre>
It will show init 0/1

Now check the logs

<pre>
  kubectl logs myapp-pod -c init-myservice
</pre>
it will diplay like can't resolved
<img width="1446" height="342" alt="image" src="https://github.com/user-attachments/assets/885ef551-8aeb-4961-8ec3-3fb128b077ac" />

Now create a deployment and apply the deployment
<pre>
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
</pre>

Create a service and expose the service
<pre>
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
</pre>
check the pod now it should be running
<pre>
  kubectl get pod myapp-pod
</pre>
<pre>
  NAME         READY   STATUS    RESTARTS   AGE
  myapp-pod    1/1     Running   0          2m

</pre>
Now describe the pod
<pre>
  kubectl logs myapp-pod -c init-myservice
</pre>
it should be deplay like
<img width="1128" height="154" alt="image" src="https://github.com/user-attachments/assets/76f2585f-2097-4c23-b8bc-7fbcc840a732" />

## DaemonSet
create a file and appply this file
<pre>
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

</pre>
run this command to check pod running
<pre>
  kubectl get pods -o wide
</pre>
output should be like this
<pre>
NAME             READY   STATUS    NODE
my-daemon-xxx1   1/1     Running   worker-node-1
my-daemon-xxx2   1/1     Running   worker-node-2

</pre>
Check logs of one pod
<pre>
  kubectl logs my-daemon-aaa1
</pre>
## JOB
create file and apply
<pre>
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
</pre>
Run it:
<pre>
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/hello-job
</pre>
You’ll see output:
<pre>
  Hello from Kubernetes Job!
</pre>
## CronJob
Create file and run file
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

## Taint & Tolerant & nodeSelector

In Kubernetes, taints and tolerations work together to control which pods can be scheduled onto which nodes.
### Taint
 1. Applied on a node.
 2. Means “do not schedule pods here unless they tolerate this taint.”
<pre>
  kubectl taint nodes <node-name> key=value:effect
</pre>
Effects:

**NoSchedule** → Pods that don’t tolerate the taint will not be scheduled.

**PreferNoSchedule** → Tries to avoid placing pods here, but not strict.

**NoExecute** → Evicts existing pods that don’t tolerate it, and stops new ones from being scheduled.
<pre>
  kubectl taint nodes worker1 dedicated=database:NoSchedule
</pre>
### Toleration
Applied on a pod.

Tells the scheduler: “I can tolerate this taint.”

Defined inside the Pod spec:
<pre>
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

</pre>
**How they work together:**
If a node has a taint, only pods with a matching toleration can be scheduled there.

If no toleration exists, the pod is kept away from that node.
### nodeSelector
NodeSelector schedules a pod only on nodes with specific labels.
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
Create pod or deployment 
<pre>
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

</pre>
**Step 1.** Label your node with disktype=ssd
<pre>
  kubectl label nodes <your-node-name> disktype=ssd

</pre>
**Step 2.** Apply the YAML:
<pre>
  kubectl apply -f nginx-deploy.yaml
</pre>
**Step 3.** Check Pods:
<pre>
  kubectl get pods -o wide
</pre>

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

### Kubernetes Requests and Limits

**Requests:**
Minimum resources (CPU/Memory) guaranteed for a container.
Scheduler uses this value to decide which node the pod can run on.
Example: requests.memory: "50Mi" → Pod will be placed on a node with at least 50Mi available.

**Limit:**
Maximum resources (CPU/Memory) a container can use.
If memory usage > limit → Container gets OOMKilled.
If CPU usage > limit → Container is throttled (not killed).

Create a pod which stress the memory 

<pre>
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
</pre>
output
<pre>
kubectl get pods -n mem-example
  
NAME            READY   STATUS      RESTARTS      AGE
memory-demo-2   0/1     OOMKilled   2 (18s ago)   26s
</pre>
OOM (Out of memory)

If a container tries to use more memory than its limit, the Kubernetes kubelet kills the container with status OOMKilled, even though the node might have free memory.

The Below pod will be scheduled
<pre>
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
</pre>
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
  sudo swapoff -a
  sudo sed -i '/ swap / s/^/#/' /etc/fstab
  
  # load required kernel modules on boot
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF
  sudo modprobe overlay
  sudo modprobe br_netfilter
  
  # sysctl params required by kubelet/container runtimes
  cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  sudo sysctl --system

</pre>
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
13. Run this command to generate the Token in master node
```
kubeadm token create --print-join-command (Master Node only)
```
14. Copy token command from master node and run in worker node (worker node)
15. Final verification (Master node)
```
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
```
