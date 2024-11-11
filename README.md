# k8s-advanced-scheduling-techniques: 

![arch](https://github.com/user-attachments/assets/b72875ad-44e5-42ca-93de-369d50919785)


This repository demonstrates advanced scheduling techniques in Kubernetes, including Node Selector, Node Affinity, Taints & Tolerations, and Pod Affinity & Anti-Affinity. These techniques allow you to fine-tune how your workloads are distributed across your cluster, ensuring optimal performance and resource utilization.

## Topics Covered

### 1. Node Selector

**Node Selector** is a simple way to ensure that your pods run on specific nodes by assigning labels. This is useful when you have nodes with special hardware or software configurations and you want certain workloads to run only on those nodes.

- **Example Use Case:** Targeting a node with high-performance CPUs for compute-intensive tasks.

```sh
# Label a node with a custom label
kubectl label node <node-name> high-perf-cpu=yes

# Deploy a pod targeting the node with the specified label
kubectl apply -f <your-deployment-file>.yaml

# Verify that the pod is running on the correct node
kubectl get pods -o wide --no-headers | awk -F" " '{print $1, $8}'
```



# Practical exp:

**--->** Lets configure the **Node Selector** by giving the worker node **hostname** through below manifest:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp01
  name: myapp01
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp01
  template:
    metadata:
      labels:
        app: myapp01
    spec:
      containers:
      - image: rajpractise/kubegame:v2
        name: kubegame
      nodeSelector:
        kubernetes.io/hostname: i-018107e4b5f29e169
```


![1-nodeselexp1](https://github.com/user-attachments/assets/88cdeb37-9b60-4559-bec5-7233d1413a88)




--> We could see that all pods got scheduled on only worker node with hostname i-018107e4b5f29e169 which we specified.




 # **--->** Another use-case is by giving specific **Node Labels** through below manifest:
 

-->  first label the worker nodes as follows:

 ```bash
 kubectl label node i-018107e4b5f29e169 high-perf-cpu=true
 kubectl label node i-0b9b9ac4ff4e06ab6 low-perf-cpu=true
```


--> Now configure the **Node Selector** by specifying the **label** : **low-perf-cpu=true** through below manifest:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp02
  name: myapp02
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp02
  template:
    metadata:
      labels:
        app: myapp02
    spec:
      containers:
      - image: rajpractise/kubegame:v2
        name: kubegame
      nodeSelector:
        low-perf-cpu: "true"
```


![2-nodeselexp2](https://github.com/user-attachments/assets/e8c33eb1-7b2f-48ef-b501-4e2a6f6f9eef)





--> We could see that all pods got scheduled on only that worker node who had label as **low-perf-cpu: "true"** i.e **Node i-0b9b9ac4ff4e06ab6**




# Now lets scale the pods and see the result:


![3-nodeselexp2](https://github.com/user-attachments/assets/e6f5bf42-64c9-4429-a859-e23a93aa021d)




--> We could see that even when we scaled the deployment, all pods got scheduled only on that worker node who had label as **low-perf-cpu: "true"** i.e **Node i-0b9b9ac4ff4e06ab6**




 
##########################################################################################################################




### 2. Node Affinity

**Node Affinity** allows more complex scheduling decisions based on node labels. Unlike Node Selector, Node Affinity supports multiple label expressions and can differentiate between preferred and required conditions.

- **Example Use Case:** Ensuring that pods are deployed only on nodes with specific environmental settings or hardware.

```sh
# Label nodes with different environment labels
kubectl label node <node1-id> env=one
kubectl label node <node2-id> env=two
kubectl label node <node3-id> env=three

# Deploy a workload with Node Affinity rules
kubectl apply -f <your-deployment-file>.yaml

# Scale the deployment to observe how pods are distributed
kubectl scale deployment <deployment-name> --replicas=8
```



# Practical exp:


--> Lets see the first type in Node Affinity i.e **requiredDuringSchedulingIgnoredDuringExecution**

--> Lets label both the worker nodes and configure the manifest file below:


```bash
kubectl label node i-018107e4b5f29e169 env=one
kubectl label node i-0b9b9ac4ff4e06ab6 env=two

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: required-hard
  name: required-hard
spec:
  replicas: 4
  selector:
    matchLabels:
      run: required-hard
  template:
    metadata:
      labels:
        run: required-hard
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - one
                - two
      containers:
      - image: nginx:latest
        name: required-hard
```



![4-nodesaffreq](https://github.com/user-attachments/assets/fe953600-8701-40f0-9b96-682e00dd6d75)





--> We could see that all pods got scheduled on only those worker nodes whose label we specified i.e env=one, env=two.





 --> Lets see the Second type in Node Affinity i.e **preferredDuringSchedulingIgnoredDuringExecution**:


 ---> First we will give more priority to a particular worker node as compared to another node and see the result i.e **90::10**

 ```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: preferred-soft
  name: preferred-soft
spec:
  replicas: 4
  selector:
    matchLabels:
      run: preferred-soft
  template:
    metadata:
      labels:
        run: preferred-soft
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 90
              preference:
                matchExpressions:
                  - key: env
                    operator: In
                    values:
                      - one
            - weight: 10
              preference:
                matchExpressions:
                  - key: env
                    operator: In
                    values:
                      - two
      containers:
        - image: nginx
          name: deploy01
```




![5-nodesaffpreff90-10](https://github.com/user-attachments/assets/c1e58a06-8a52-4c49-826d-7462d49e2616)




--> We could see that all pods got scheduled on only one worker node because we gave **high priority** to that node.



--> Now Lets give the equal priority to both the worker nodes in the mainfest i.e **50::50** and see the result:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: preferred-soft
  name: preferred-soft
spec:
  replicas: 4
  selector:
    matchLabels:
      run: preferred-soft
  template:
    metadata:
      labels:
        run: preferred-soft
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              preference:
                matchExpressions:
                  - key: env
                    operator: In
                    values:
                      - one
            - weight: 50
              preference:
                matchExpressions:
                  - key: env
                    operator: In
                    values:
                      - two
      containers:
        - image: nginx
          name: deploy01
```



![5-nodesaffpreff50-50](https://github.com/user-attachments/assets/bcb36809-3b73-40c4-8142-1a17d1bfb15a)




---> We could see that all pods got scheduled equally on both worker nodes since we gave equal weightage priority to both nodes i.e **50:50**





#################################################################################################################################################





### 3. Taints & Tolerations

**Taints** are applied to nodes to prevent pods that don't tolerate the taint from being scheduled on them. **Tolerations** are applied to pods to allow them to be scheduled on tainted nodes. This feature is essential for keeping certain workloads separate or ensuring critical workloads are not disrupted.

- **Example Use Case:** Ensuring that only specific pods can run on a node reserved for special tasks.

```sh
# Taint nodes to control scheduling
kubectl taint node <node-id> high-cpu=yes:NoSchedule
kubectl taint node <node-id> med-cpu=yes:NoExecute

# Describe the node to see its taints
kubectl describe node <node-id> | grep -i high

# Deploy a pod to see if it gets scheduled based on tolerations
kubectl apply -f <your-deployment-file>.yaml

# Remove taints from nodes if needed
kubectl taint node <node-id> high-cpu-
kubectl taint node <node-id> med-cpu-
```


# Practical exp:

--> Lets first taint both of the worker nodes and try to schedule any pod without **toleration** and see the result:


  ```bash
kubectl taint node i-018107e4b5f29e169 high-cpu=yes:NoSchedule
kubectl taint node i-0b9b9ac4ff4e06ab6 med-cpu=yes:NoSchedule
```


![6-taint-tolnosched1](https://github.com/user-attachments/assets/1ba5019e-5f8f-4831-be52-179939ac2421)

![6-taint-tolnosched2](https://github.com/user-attachments/assets/707e25a3-6356-4ce4-a2e8-a57a40b6543b)




We could clearly see that pod was not scheduled on any worker node since its not **tolerating** any of the **taints**.






---> Now lets tolerate the pods for one of the taints applied on worker node with **effect: "NoSchedule"**:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  labels:
    app: app1
spec:
  replicas: 6
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - name: nginx
          image: nginx:latest
      tolerations:
        - key: "high-cpu"
          operator: "Equal"
          value: "yes"
          effect: "NoSchedule"
```


![6-taint-tolnosched3](https://github.com/user-attachments/assets/5cc57253-28f6-4bd8-949e-7b1510bc855d)





--> We could see that all pods got scheduled on that particular worker node whose **taint** it is **tolerating**.





--> Now lets apply tolerating the pods for one of the taints applied on worker node with **effect: "NoExecute"**


for that lets first taint all the worker nodes appropriately:

```bash
kubectl taint node i-018107e4b5f29e169 high-cpu=yes:NoExecute
kubectl taint node i-0b9b9ac4ff4e06ab6 med-cpu=yes:NoExecute
```


![7-taint-tol-noexec1](https://github.com/user-attachments/assets/b9a50ce5-96e2-46fd-ba5b-dcecfbe04610)




--> We could see that as soon as all worker nodes was tainted with **effect as NoExecute** , all pods which were running previously got halted as none of them are tolerating any taints.




# Now lets configure the manifest file and apply it:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app2
  name: app2
spec:
  replicas: 6
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - image: nginx:latest
        name: nginx
      tolerations:
      - key: "high-cpu"
        operator: "Equal"
        value: "yes"
        effect: "NoExecute"
      - key: "med-cpu"
        operator: "Equal"
        value: "yes"
        effect: "NoExecute"
```



![8-taint-tol-noexec](https://github.com/user-attachments/assets/c47933f3-1dff-4e46-aef7-d235ff2d0662)





---> We colud see that all pods got scheduled equally on both worker nodes as they are tolerating the taints for both of the worker nodes.





##################################################################################################################################




### 4. Pod Affinity & Anti-Affinity

**Pod Affinity** allows you to schedule pods together on the same node, while **Pod Anti-Affinity** ensures that pods are placed on separate nodes. This is useful for reducing latency between pods or ensuring high availability by spreading pods across nodes.

- **Example Use Case:** Grouping frontend and backend pods together to reduce network latency or ensuring that replicas of the same service are spread out to avoid single points of failure.

```sh
# Use pod affinity or anti-affinity in your deployment YAML
kubectl apply -f <your-deployment-file>.yaml

# Scale the deployment and observe the behavior
kubectl scale deployment <deployment-name> --replicas=2

# Drain a node to see how pods are rescheduled
kubectl drain <node-id>
kubectl uncordon <node-id>
```




# Practical Exp:


--> Lets deploy two deployments one for the **web app and other for the redis app**.

-->We will apply both both affinity and anti-affinity rules:
**Pod Affinity** allows you to schedule pods together on the same node, \
while **Pod Anti-Affinity** ensures that pods are placed on separate nodes.



--> Lets apply the manifest file below:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis
  name: redis
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: "topology.kubernetes.io/zone"
      containers:
      - image: redis:latest
        name: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        perf: high
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: "topology.kubernetes.io/zone"
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: "topology.kubernetes.io/zone"
      containers:
      - image: nginx:latest
        name: nginx
```


--> We have applied affinity rule for the **web app** such that it should always be scheduled in the **same zone or worker node** as the **redis app**.


---> Lets see the result:



![9-podaff-antiaff-1](https://github.com/user-attachments/assets/3589818c-9b80-4d47-a347-9a42397a47ff)





# Now lets scale the deployment to see the results more accurately:



![9-podaff-antiaff-2](https://github.com/user-attachments/assets/a454147a-0f18-4cf1-8cb1-16e4d57ba524)




---> We can clearly see that even when we scaled the deployments, **web app pod** is still scheduled on the same worker node as  **redis app pod**



####################################################################################################################



## Commands Overview

Here’s a summary of the key commands used for the topics covered:

```sh
# Label a node
kubectl label node <node-name> high-perf-cpu=yes

# Apply a deployment file
kubectl apply -f <your-deployment-file>.yaml

# Verify pod placement
kubectl get pods -o wide --no-headers | awk -F" " '{print $1, $8}'

# Label nodes for Node Affinity
kubectl label node <node1-id> env=one
kubectl label node <node2-id> env=two
kubectl label node <node3-id> env=three

# Scale a deployment
kubectl scale deployment <deployment-name> --replicas=8

# Taint nodes
kubectl taint node <node-id> high-cpu=yes:NoSchedule
kubectl taint node <node-id> med-cpu=yes:NoExecute

# Remove taints from nodes
kubectl taint node <node-id> high-cpu-
kubectl taint node <node-id> med-cpu-

# Drain and uncordon a node
kubectl drain <node-id>
kubectl uncordon <node-id>
```

## Conclusion

This repository provides practical examples and commands to help you understand and implement advanced scheduling techniques in Kubernetes. By mastering these concepts, you can ensure that your applications run efficiently and reliably in a Kubernetes environment.

---
