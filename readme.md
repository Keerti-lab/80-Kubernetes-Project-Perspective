## Container Orchestration

- To overcome above disadvantages, we can use an orchestrator
- We need to someone to give instructions to the container based on the live traffic
- E.g. a manager at a company who handles different teams and instructs each team what should they work on i.e. distributes work to the corresponding teams for a better productivity and manages us
- Similarly we need some orchestration tool to monitor the containers or give instructions at runtime based on the live traffic
- Out of all available tools, Kubernetes is popular. Docker Swarm is also present but has lot of challenges and is not scalable
- We still build our images using Docker and hand over the container management lifecycle to the Kubernetes
- Kubernetes (K8s) offers better networking and storage solutions
- Every cloud offers K8s as solution but not docker swarm

## Kubernetes installation

- Kubernetes is a cluster and offers PaaS (Platform as a Service) model
- Kubernetes is a cloud agnostic service i.e. it can be used on any cloud platform, on-prem etc
- Every clustering solution has a Master and Worker nodes
  - Master node: Assign instructions to worker nodes
  - Worker nodes: Performs task based on the instructions from Master
- If there are no worker nodes, Master node needs to perform all the tasks
- In production grade or high-end application, we have more than one replica of master node (control plane). Usually 3 is a good number
- **Minikube**: Installing all the Master and worker components within **one** server
- But this minikube is only for learning purpose and not production ready
- Manifest files are written in yaml and is a way to inform K8s to run the image
- `kubectl` is the command line that connects with the Kubernetes cluster and pushes the manifest files
- We pass on the manifest file to control plane and it will ask its worker to perform the task
- We only talk to control through the API to the control plane and don't talk to the worker nodes directly
- Everything will only happen on the worker nodes
- K8 cluster pulls the images from Dockerhub and creates containers which are referred as **Pods** in Kubernetes
- A pod is the smallest unit in K8s
- If Minikube is successfully installed, a file with name **kubeconfig** will be created in our home directory
- We should create a directory called `.kube` and copy `kubeconfig` file as `config` to `.kube` directory
- After that if we run `kubectl get nodes` command, we should see a node listed in the output
- Everything in Kubernetes is referred as a resource

## Kubernetes Resources

### Namespace

- An isolated project space
- With in this, we create project resources
- To view all the namespaces: `kubectl get namespaces`
- If we don't specify our own namespace, K8s will use the default namespace
- The other namespaces are used by Kubernetes cluster internally

#### Creating a new namespace

`namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: roboshop
```

- To create a namespace: `kubectl create -f 01-namespace.yaml`
  - `-f` - filename
- With the above command, kubectl connects to the cluster and creates the namespace
- If we try to apply the same command once again, we get an error
- Difference between `kubectl create` and `kubectl apply`:
  - `kubectl create`: If a resource doesn't exist, it creates and if exists we get an error
  - `kubectl apply`: When resource doesn't exist, it will create and if exists it updates or doesn't do anything
- Therefore using `kubectl apply -f <filename>.yaml` is better than `kubectl create`
- To delete a namespace: `kubectl delete -f <filename>.yaml`
- `apiVersion` is used for categorizing the resources in kubernetes
- We can get the list of resources using: `kubectl api-resources`

# Kubernetes

## Namespace (Contd.)

- In AWS, there are VPC level and non-vpc level resources
- VPC level resources i.e. attached to VPC are:
  - subnet
  - security group
  - Load balancer
- E.g. for non vpc level resources is Route53
- Even in K8s, we have namespace level and non-namespace level resources
- Few resources are attached to the namespace such as project level or cluster level resources and few are not such as nodes
- We can get the list of resources: `kubectl api-resources`
- We can identify them using the true/false value set to the resource under Namespaced column

## Pods

- Pods are smallest deployment units in K8s
- Every resource in Kubernetes has: `apiVersion`, `kind` and `metadata`

  ```yaml
  apiVersion: v1
  kind: Pod # What kind of resource is it?
  metadata:
    name: nginx # Name of the pod
    namespace: roboshop # To attach it to roboshop namespace
  ```

- All the other configuration information is specified under spec
- For e.g. when we run nginx as a docker container: `docker run -d -p 80:8080 nginx:latest`
  - Here `-p 80:8080` information is specifed under spec
- We use K8s to run the images but not docker, therefore we need `manifest.yaml` files to run the pods

  `01-pod.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod # What kind of resource is it?
  metadata:
    name: nginx # Name of the pod
    namespace: roboshop
  spec:
    containers:
    - name: simple-pod # Name of the container
      image: nginx:1.14.2
      ports:
      - containerPort: 80
  ```

**How to create pod?**

- First ensure that roboshop namespace is present
  - If not, create it using `kubectl apply -f 01-namespace.yaml`
- Then run `kubectl apply -f 01-pod.yaml`
- To get the list of pods in **roboshop** namespace: `kubectl get pods -n roboshop`
- To delete a pod: `kubectl delete -f 01-pod.yaml`

**How to login to a pod?**

- `kubectl exec -it <name-of-the-pod> -n roboshop -- bash`

### Difference between Pods and container

- 1 pod can have multiple containers
- All containers inside pod can share same storage and n/w

**Why Kubernetes offers multi-containers?**

- In ELK, we installed an agent such as filebeat on the component server which monitors `/var/log/messages` file and pushing the messages to ELK Cluster
- Similarly inside Pod, we can run multiple containers. For e.g. 1 NGiNX pod, 1 filebeat pod
- NGiNXs stores the logs to `/var/log/nginx/access.log` file and as both the containers are inside the same pod, filebeat can access this file as they share common storage and push the messages to ELK cluster
- Here **filebeat** container is referred as **side car** container i.e. offers extra functionality to the main container
- Multi containers can be used in:
  - side car
  - proxy patterns
  - init containers

**How can we create multiple containers inside a pod?**

  `02-multi-container.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod # What kind of resource is it?
  metadata:
    name: multi-container # Name of the pod
    namespace: roboshop
  spec:
    containers:
    - name: nginx
      image: nginx
    - name: sidecar
      image: almalinux:8
      command: ["sleep", "200"]
  ```

- As almalinux doesn't have a command to run for infinite time, therefore we need to explicitly specify a command on our own
- The pods can created with the command: `kubectl apply -f 02-multi-container.yaml`
- Once they are created, we can login in to the pods using: `kubectl exec -it <name-of-the-pod> -c <name-of-the-container> -- bash`.
  - For e.g. `kubectl exec -it multi-container -c sidecar -- bash`
- As both these containers share the same network, we can run `curl localhost` to see the default homepage that is served by NGiNX
- To access the logs generated by NGiNX inside sidecar container, we need to perform **Volume mapping**

**How to add labels to the pods?**

  `03-labels.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx # Name of the pod
    namespace: roboshop
    labels:
      environment: production
      app: nginx
  spec:
    containers:
    - name: nginx
      image: nginx
    - name: sidecar
      image: almalinux:8
      command: ["sleep", "200"]
  ```

- Labels are used in selectors and it has some functional advantage apart from filtering
- Like `docker inspect`, we can inspect a pod using: `kubectl describe pod <name-of-the-pod>`

**Difference between Labels and annotations:**

- labels can't have special char in key names, annotations can have
- labels key have some length restrictions, annotations can have more length compared to labels
- labels are used for selecting internal kubernetes resources selectors where as annotations are used in selecting external resources

  `04-annotations.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx # Name of the pod
    namespace: roboshop
    labels:
      environment: production
      app: nginx
    annotations:
      com.project.name: roboshop
      com.component.name: frontend
  spec:
    containers:
    - name: nginx
      image: nginx
    - name: sidecar
      image: almalinux:8
      command: ["sleep", "200"]
  ```

**Difference between VM and Docker:**

- In VM, the resources are blocked when we create and run it even if we don't use them where as with Docker containers, it can dynamically consume te system resources
- But the disadvantage with dynamic consumption is that, it can also end up occupying complete system resources based on the task it is performing such application logs
- To overcome this, we can define resource limits i.e. soft limit and/or hardlimit:
  - Soft limit using `requests`
  - hard limit using `limits`
- To ensure that the resources are not blocking the memory and ram rather they're used by K8s to monitor the pods so that they do not end-up consuming more resources than mentioned.

  `05-resources.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: resources
  spec:
    containers:
    - name: nginx
      image: nginx
      resources:
        requests: # soft limit
          memory: "64Mi"
          cpu: "250m"
        limits: # hard limit i.e. max. limit
          memory: "128Mi"
          cpu: "500m"
  ```

- 1 CPU is 1000 milli cores, 250m -> 0.25 CPU

### Image Pull Policy

- We're publishing our images to dockerhub
- By default K8s only pulls the image from docker hub at the time of creation of the pod
- If there are any updates to the image, it will not fetch them by default
- To fetch the updates, we need to set: `imagePullPolicy: Always`

  `05-resources.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: resources
  spec:
    containers:
    - name: nginx
      image: nginx
      imagePullPolicy: Always
      resources:
        requests: # soft limit
          memory: "64Mi"
          cpu: "250m"
        limits: # hard limit i.e. max. limit
          memory: "128Mi"
          cpu: "500m"
  ```

## Environment Variables

- Defining environment variables to access them inside the container

  `06-environment.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: env-var-demo
  spec:
    containers:
    - name: env-var-demo
      image: nginx
      env:
      - name: DEMO
        value: "Hello Environment variable"
  ```

## Config map

- It is used for keeping the configuration information separate from the container as its a best practice

  `07-configmap.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: course-config
  data:
    COURSE: DevOps
    DURATION: 120HRS
    TRAINER: SIVAKUMAR
  ```

- To use the above defined configmap inside a pod, we need to first create it using `kubectl apply -f 07-configmap.yaml`
- To list all the configmaps: `kubectl get configmaps`
- To inspect the config map: `kubectl describe configmap <name-of-the-configmap>`. For e.g. `kubectl describe configmap course-config`
- Once the configuration is created, we can attach it to a pod by fetching the values from a configmap using:

  ```yaml
  - name: COURSE
    valueFrom:
      configMapKeyRef:
        name: course-config
        key: COURSE
  ```

  `08-config-pod.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: config-pod
  spec:
    containers:
    - name: config-pod
      image: nginx
      env:
      - name: COURSE
        valueFrom:
          configMapKeyRef:
            name: course-config
            key: COURSE
      - name: PERSON
        valueFrom:
          configMapKeyRef:
            name: course-config
            key: TRAINER
      - name: DURATION
        valueFrom:
          configMapKeyRef:
            name: course-config
            key: DURATION
  ```

- In future, if we need to change the values, we just modify it inside the ConfigMap and restart the pod i.e. delete and create the pod
- But this approach is not scalable if we have more configuration variables like 10 or even more

### How to import all configuration variables from configmap

- Instead of hardcoding all the values and assigning them to variables in the pod manifest, we can import all of them at one go
- This way we can have the pod manifest file as small as possible

  `09-import-all.yaml`

  ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: import-all
    spec:
      containers:
      - name: import-all
        image: nginx
        envFrom:
        - configMapRef:
          name: course-config
  ```

## Secret

- We can store URL information in ConfigMap but not sensitive information such as username and password
- For this purpose, we can use a `Secret` resource type
- The information that we store in Secret resource should be base64 encoded
- To encode, we can use: `echo "pavan" | base64` and to decode `echo "cGF2YW4K" | base64 --decode`
- We can define multiple resources in the same yaml file with `---` separated

  `10-secret.yaml`

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: secret-basic-auth
  type: Opaque # default
  data:
    username: c2l2YWt1bWFyCg==
    password: YWJjMTIzCg==
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: secret-pod
  spec:
    containers:
    - name: secret-pod
      image: nginx
      envFrom:
      - secretRef:
          name: secret-basic-auth
  ```

- Now that the pods are running, how can we allow users to access the application running inside the pod as we don't have any port mappings enabled ?
- For this purpose, we can use **Services** in K8s i.e. we can expose the deployed pods to the external users
- The secrets information can be managed using the service available on the cloud platform
- For e.g. SecretsManager on AWS can be used to manage these credentials

## K8s Services

- There are 3 kinds of services
  1. Cluster IP: Internal to Kubernetes
  2. NodePort: Expose to outside world
  3. Load Balancer: Only for Cloud related kubernetes
- **With service, we can create names as DNS records**
- NodePort and LoadBalancer services are used to expose the application to external users
- For NodePort service and Load Balancer service, minikube environment is not sufficient. We need to provision an actual K8s cluster

### Advantages of using a service

1. To expose the application to outside world
2. To balance the load --> at the time of deployment and replicaset
3. Serves as a **Service Mesh** as well i.e. even though the IP address changes, it gets mapped to the DNS

### Cluster IP

- Internal to cluster, not exposed to outside world, only exposed within the cluster
- We need to first create a pod and then attach it to a service
- Its important to provide the port information as well or else the service cannot attach to the pod

  `services/01-cluster-ip.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels: # These are used for selecting and filtering the pod
      environment: dev
      app: frontend
      project: roboshop
  spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: http-web-svc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: http-web-svc # this port belongs to container
  ```

- Once the service is provisioned using `kubectl apply -f 01-cluster-ip.yaml`, a cluster IP is associated with the service
- We can get the service information using: `kubectl get service`
- To test the same, we can proivison a new pod using `kubectl apply -f pods/02-multi-container.yaml` and test the same using:
  - Cluster IP address: `curl <cluster-ip>` or
  - Service name: `curl nginx-service`

  # Kubernetes



## Provisioning EKS on AWS

- For that, we need a tool from AWS i.e. **eksctl** which is a command line tool
- When we push the manifest files from Workstation to EKS cluster, a node group called for e.g. SpotNG will contain list of servers that will pull the image from Dockerhub
- First we should create an EKS cluster
  - For this, we provision a t2.micro instance with in the default VPC, 30GB of EBS volume and call it as workstation
  - With in the workstation, we install `kubectl`, `eksctl`, `docker` and `kubens`
- We have one master node that is present on EKS and group of worker nodes, we call the group as Spot NG
- The OS that is present on Worker nodes is the choice of AWS. Therefore it chose AWS Linux 2 AMI
- Using `kubens` we can select the K8s namespace
- We can use the following commands to install **kubens**

  ```bash
  sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
  sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
  ```

- We can install all these components using the following script: `curl -O https://raw.githubusercontent.com/sivadevopsdaws74s/kubernetes-eksctl/master/workstation.sh` and use `sudo sh workstation.sh`
- After that we need to configure our workstation instance with a CLI user that has admin rights inorder to provision EKS cluster on AWS
- EKS: Elastic Kubernetes Service from AWS
- We **cannot** have SSH access to the master node and it is managed by AWS itself
- `eksctl` is a simple way to provision an EKS clutser on AWS
- Sample example configs to provision EKS cluster using eksctl can be found [here](https://github.com/eksctl-io/eksctl/tree/main/examples)
- Along with it, we have `eks.yaml` file in which we specify the configuration we would like to have

  `eks.yaml`

  ```yaml
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig

  metadata:
    name: eks-spot-cluster
    region: us-east-1

  # this is also completely managed by AWS
  managedNodeGroups:
    - name: spot
      instanceType: m5.large
      # your K8 node can be anytime taken back by AWS
      spot: true
      desiredCapacity: 3
      ssh:
        publicKeyName: kubernetes # replace this with your keypair that exists in AWS.
  ```

- We specify the number of worker nodes that needs to be a part of the EKS Cluster under `managedNodeGroups`
- Spot instances are not used in production environments rather in development environments and non-crucial workloads
- In case if the spot instance is taken back by AWS, K8s can provision another one immediately
- By default the EKS cluster is provisioned in the Default VPC
- We should generate a keypair with the name kubernetes using `ssh-keygen -f kubernetes` and import the public key on to AWS
- Once everything is ready, then we should run: `eksctl create cluster --config-file=eks.yaml`
- This will take around **15-20** mins of time to provision
- Once the EKS cluster is successfully provisioned, we can run `kubectl get nodes` command on the workstation where eksctl is installeed

## Node port

- To create and attach NodePort service to a pod, we just need to set `type` to `NodePort` under `spec`

  `services/02-nodeport.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels: # These are used for selecting and filtering the pod
      environment: dev
      app: frontend
      project: roboshop
  spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: http-web-svc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    type: NodePort
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: http-web-svc # this port belongs to container
  ```

- Run the command: `kubectl apply -f 02-nodeport.yaml`
- To view all the services: `kubectl get service` or `kubectl get svc`
- Once the service is successfully created, a cluster IP is allocated and random port is also exposed
- A port number in the ephemeral range (30000 - 32767) at random (it will be same for all the nodes) is choosen by K8s on each and every node
- In addition to that a **security group** is also created for all the nodes that in the cluster by EKS on its own
- We should allow the port in the security group that has **remoteAccess** in its name
- Once the port is enabled, we will be able to access the application that is running on the pod using: `<Node-public-ip>:<random-port-number-assigned-by-EKS>`
- **Cluster IP is a subset of Node port** i.e. when we provision a node port service, a cluster IP service is also provisioned in the background
- We can also specify which port to expose using: `nodePort: 30007`

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    type: NodePort
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: 80 # this port belongs to container
      nodePort: 30007 # this port belongs to host
  ```

## Loadbalancer

- This service works only for cloud based K8s clusters
- By setting type to LoadBalancer, a loadbalancer service is provisioned
- Similar to Node Port, Load Balancer is created, a random ephemeral port is exposed and cluster IP is also present
- In Node Port, we need to expose the ports on our own for all the nodes that are part of the cluster group
- Also each node has unique public IP linked to it.
- This might be a security concern when we open a range of ports on our server
- To avoid this, we can use a load balancer service from K8s which exposes only the open port, creates a DNS record and routes the traffic to the nodes using Round Robin algorithm

  `services/03-loadbalancer.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels: # These are used for selecting and filtering the pod
      environment: dev
      app: frontend
      project: roboshop
  spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: http-web-svc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    type: LoadBalancer
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: http-web-svc # this port belongs to container
  ```

- All the node group instances are added as a part of the LB under Target Instances

### Flow of a request

- Once a user sends a request using the load balancer address
  - Step 1: LB transfers it to any of the node in the cluster
  - Step 2: Then its forwarded to the Cluster IP
  - Step 3: Finally it reaches the pod where the App is hosted
- We can also create an Alias record on Route53 to the Load Balancer to keep its address short

## Sets in K8s

1. Replica set
2. Deployment set
3. Daemon set
4. Stateful set

### 1. Replica set

- Suppose if the traffic is very high, we need to change the name of the pod manually and provision it manually which is a painful process
- Instead we can handover this job to K8s using ReplicaSet and it provisions the pods on our behalf

  `sets/01-replicaset.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: frontend
    labels: # these labels are replica set labels, every k8 resource can have labels.
      app: nginx
      tier: frontend
  spec:
    # modify replicas according to your case
    replicas: 3
    selector:
      matchLabels: # these labels should match with pod labels
        app: nginx
        tier: frontend
        environment: dev
    template: # this one is nothing but pod definition.
      metadata:
        labels: # these are the pod labels.
          app: nginx
          tier: frontend
          environment: dev
      spec:
        containers:
        - name: nginx
          image: nginx:1.14.2
  ```

- ReplicaSet is present in `apps/v1` resource type. Therefore, we set `apiVersion` to `apps/v1`
- `kubectl get rs` to check the status of the replicasets
- A replicaset creates pods internally

### 2. Deployment Set

- **If there is an update in the image, it will not update the exisiting pods in the replicaset** since its main purpose is to provide the no: of replicas mentioned in the manifest file
- Therefore we go for **deployment set**
- This is the most commonly used among others

  `sets/02-deployment.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels: # these labels belong to deployment
      app: nginx
  spec:
    replicas: 3
    selector:
      matchLabels: # these labels should match with pod
        app: nginx
        project: roboshop
        component: frontend
    template: # this is the pod definition
      metadata:
        labels:
          app: nginx
          project: roboshop
          component: frontend
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
  ```

- We can get all the deployments using: `kubectl get deploy`
- When we create a Deployment, it creates a Replicaset with the `replicaSet_name-random_ID`
- Deployment created another replicaset, therefore Replicaset is a subset of deployment
- Therefore, when we check: `kubectl get rs` to get the list of ReplicaSets, we can see it has the name: `deployment_name-random_ID`
- Using this, **we can acheive Zero Downtime** of our application

### How does it achieve it?

- When an update takes place, Deployment creates a new replicaset and uses the updated image when creating the pods
- Once a pod is provisioned, it terminates the pod from previous replicaset that is outdated as it needs to maintain the number of replicas

## Roboshop on EKS

- Broadly there are 2 steps that are involved:
  1. Image should exist i.e. build the image
  2. Configure the image using manifest files

### Common errors when implementing project with K8s

1. Either image exists or not ?
2. Is the image properly built ?
3. Did we specify the version of the image correctly?
4. Does the K8s cluster has access to pull the images from docker hub?
5. If everything is proper, is the configuration information specified is correct or not?

### Setup flow

1. Develop Dockerfiles and Manifest files locally and push them to GitHub
2. Pull them into Workstation and build the Dockerfile
3. Push the images to Docker Hub
4. Push the manifests to EKS Cluster

### MongoDB

- We start by creating a namespace

  `namespace.yaml`

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: roboshop
  ```

- Within the manifest files, we specify our deployments

  `mongodb/manifest.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mongodb
    namespace: roboshop
    labels: # these labels belong to deployment
      app: mongodb
      tier: db
      project: roboshop
  spec:
    replicas: 1
    selector:
      matchLabels: # these labels should match with pod
        app: mongodb
        tier: db
        project: roboshop
    template: # this is the pod definition
      metadata:
        namespace: roboshop
        labels:
          app: mongodb
          tier: db
          project: roboshop
      spec:
        containers:
        - name: mongodb
          image: joindevops/mongodb:1.0.0
  ```

#### Commands

1. `kubectl apply -f namespace.yaml`
2. `docker build -t joindevops/mongodb:1.0.0 .`
3. `docker login`
4. `docker push joindevops/mongodb:1.0.0`
5. `kubectl apply -f manifest.yaml`
6. `kubens roboshop`

- Using the 6th command, we don't need to mention `-n roboshop` everytime to fetch the resources provisioned in that namespace
- To confirm if the pood is created or not, we can run: `kubectl get pods -o wide`
- Now catalogue should connect with the MongoDB
- Connecting to it using IP address is not safe
- Therefore we use a service and as we don't need to expose the MongoDB port to the outside world, we can just use: **Cluster IP** service
- **Note**: A pod should always have a service attached to it in order to enable other pods to communicate with it
- **Attaching Cluster IP service to MongoDB**

  `mongodb/manifest.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mongodb
    namespace: roboshop
    labels: # these labels belong to deployment
      app: mongodb
      tier: db
      project: roboshop
  spec:
    replicas: 1
    selector:
      matchLabels: # these labels should match with pod
        app: mongodb
        tier: db
        project: roboshop
    template: # this is the pod definition
      metadata:
        labels:
          app: mongodb
          tier: db
          project: roboshop
      spec:
        containers:
        - name: mongodb
          image: joindevops/mongodb:1.0.0
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: mongodb
    namespace: roboshop
  spec:
    selector:
      app: mongodb
      tier: db
      project: roboshop
    ports:
    - name: mongodb-port
      protocol: TCP
      port: 27017 # this port belongs to service
      targetPort: 27017 # this port belongs to container
  ```

### Catalogue

- As we don't need to expose the catalogue port to external users, we can use **ClusterIP** service
- When using Docker with K8s, we should avoid storing variables as environment variables at the docker image layer
- Rather, we can define these environment variables using **ConfigMap**, therefore removing it from the Dockerfile
- Whenever there is a change in the application code, only then we should go for CICD process
- Therefore, we should maintain the configuration information outside the application code

  `catalogue/manifest.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: catalogue
    namespace: roboshop
  data:
    MONGO: "true" # keep true in double quotes
    MONGO_URL: "mongodb://mongodb:27017/catalogue" # protocol//service-name:service-port/schema
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: catalogue
    namespace: roboshop
    labels: # these labels belong to deployment
      app: catalogue
      tier: app
      project: roboshop
  spec:
    replicas: 1
    selector:
      matchLabels: # these labels should match with pod
        app: catalogue
        tier: app
        project: roboshop
    template: # this is the pod definition
      metadata:
        namespace: roboshop
        labels:
          app: catalogue
          tier: app
          project: roboshop
      spec:
        containers:
        - name: catalogue
          image: joindevops/catalogue:1.0.0
          envFrom:
          - configMapRef:
              name: catalogue
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: catalogue
    namespace: roboshop
  spec:
    selector:
      app: catalogue
      tier: app
      project: roboshop
    ports:
    - name: catalogue-port
      protocol: TCP
      port: 8080 # this port belongs to service
      targetPort: 8080 # this port belongs to container
  ```

- To provision the resources: `kubectl apply -f manifest.yaml`
- To check the logs:

  ```bash
  kubectl get pods
  kubectl logs <Name-of-the-pod>
  ```

- To debug the pods that are created, we can use a sample container with an image for e.g. almalinux that has basic commands such as telnet
- There can also be a case where the pods are being created in a different node i.e. mongo in one node, catalogue in another node
- Due to this, catalogue pod will not be able to communicate with the mongo pod

### Web Component

- Build is a costly operation, therefore we should:
  - Perform build and restart only when we have a code change
  - Perform restart only when there is a configuration
- Therefore we store the config information inside the ConfigMap as it allows to import config files as well
- Once we are done with our practice, we should clean up all the resources using:
  - `kubectl delete namespace roboshop` which also deletes the resources provisioned in that namespace
  - `kubens default`: Switching back to the default namespace
  - Or delete the whole cluster: `eksctl delete cluster --config-file=eks.yaml`
- `eksctl` uses cloudformation in the background which is native to AWS

# Kubernetes



## Provisioning EKS on AWS

- For that, we need a tool from AWS i.e. **eksctl** which is a command line tool
- When we push the manifest files from Workstation to EKS cluster, a node group called for e.g. SpotNG will contain list of servers that will pull the image from Dockerhub
- First we should create an EKS cluster
  - For this, we provision a t2.micro instance with in the default VPC, 30GB of EBS volume and call it as workstation
  - With in the workstation, we install `kubectl`, `eksctl`, `docker` and `kubens`
- We have one master node that is present on EKS and group of worker nodes, we call the group as Spot NG
- The OS that is present on Worker nodes is the choice of AWS. Therefore it chose AWS Linux 2 AMI
- Using `kubens` we can select the K8s namespace
- We can use the following commands to install **kubens**

  ```bash
  sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
  sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
  ```

- We can install all these components using the following script: `curl -O https://raw.githubusercontent.com/sivadevopsdaws74s/kubernetes-eksctl/master/workstation.sh` and use `sudo sh workstation.sh`
- After that we need to configure our workstation instance with a CLI user that has admin rights inorder to provision EKS cluster on AWS
- EKS: Elastic Kubernetes Service from AWS
- We **cannot** have SSH access to the master node and it is managed by AWS itself
- `eksctl` is a simple way to provision an EKS clutser on AWS
- Sample example configs to provision EKS cluster using eksctl can be found [here](https://github.com/eksctl-io/eksctl/tree/main/examples)
- Along with it, we have `eks.yaml` file in which we specify the configuration we would like to have

  `eks.yaml`

  ```yaml
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig

  metadata:
    name: eks-spot-cluster
    region: us-east-1

  # this is also completely managed by AWS
  managedNodeGroups:
    - name: spot
      instanceType: m5.large
      # your K8 node can be anytime taken back by AWS
      spot: true
      desiredCapacity: 3
      ssh:
        publicKeyName: kubernetes # replace this with your keypair that exists in AWS.
  ```

- We specify the number of worker nodes that needs to be a part of the EKS Cluster under `managedNodeGroups`
- Spot instances are not used in production environments rather in development environments and non-crucial workloads
- In case if the spot instance is taken back by AWS, K8s can provision another one immediately
- By default the EKS cluster is provisioned in the Default VPC
- We should generate a keypair with the name kubernetes using `ssh-keygen -f kubernetes` and import the public key on to AWS
- Once everything is ready, then we should run: `eksctl create cluster --config-file=eks.yaml`
- This will take around **15-20** mins of time to provision
- Once the EKS cluster is successfully provisioned, we can run `kubectl get nodes` command on the workstation where eksctl is installeed

## Node port

- To create and attach NodePort service to a pod, we just need to set `type` to `NodePort` under `spec`

  `services/02-nodeport.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels: # These are used for selecting and filtering the pod
      environment: dev
      app: frontend
      project: roboshop
  spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: http-web-svc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    type: NodePort
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: http-web-svc # this port belongs to container
  ```

- Run the command: `kubectl apply -f 02-nodeport.yaml`
- To view all the services: `kubectl get service` or `kubectl get svc`
- Once the service is successfully created, a cluster IP is allocated and random port is also exposed
- A port number in the ephemeral range (30000 - 32767) at random (it will be same for all the nodes) is choosen by K8s on each and every node
- In addition to that a **security group** is also created for all the nodes that in the cluster by EKS on its own
- We should allow the port in the security group that has **remoteAccess** in its name
- Once the port is enabled, we will be able to access the application that is running on the pod using: `<Node-public-ip>:<random-port-number-assigned-by-EKS>`
- **Cluster IP is a subset of Node port** i.e. when we provision a node port service, a cluster IP service is also provisioned in the background
- We can also specify which port to expose using: `nodePort: 30007`

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    type: NodePort
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: 80 # this port belongs to container
      nodePort: 30007 # this port belongs to host
  ```

## Loadbalancer

- This service works only for cloud based K8s clusters
- By setting type to LoadBalancer, a loadbalancer service is provisioned
- Similar to Node Port, Load Balancer is created, a random ephemeral port is exposed and cluster IP is also present
- In Node Port, we need to expose the ports on our own for all the nodes that are part of the cluster group
- Also each node has unique public IP linked to it.
- This might be a security concern when we open a range of ports on our server
- To avoid this, we can use a load balancer service from K8s which exposes only the open port, creates a DNS record and routes the traffic to the nodes using Round Robin algorithm

  `services/03-loadbalancer.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels: # These are used for selecting and filtering the pod
      environment: dev
      app: frontend
      project: roboshop
  spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: http-web-svc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
  spec:
    type: LoadBalancer
    # search for pods that matches the given labels and attach to it
    selector:
      environment: dev
      app: frontend
      project: roboshop
    ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80 # this port belongs to service
      targetPort: http-web-svc # this port belongs to container
  ```

- All the node group instances are added as a part of the LB under Target Instances

### Flow of a request

- Once a user sends a request using the load balancer address
  - Step 1: LB transfers it to any of the node in the cluster
  - Step 2: Then its forwarded to the Cluster IP
  - Step 3: Finally it reaches the pod where the App is hosted
- We can also create an Alias record on Route53 to the Load Balancer to keep its address short

## Sets in K8s

1. Replica set
2. Deployment set
3. Daemon set
4. Stateful set

### 1. Replica set

- Suppose if the traffic is very high, we need to change the name of the pod manually and provision it manually which is a painful process
- Instead we can handover this job to K8s using ReplicaSet and it provisions the pods on our behalf

  `sets/01-replicaset.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: frontend
    labels: # these labels are replica set labels, every k8 resource can have labels.
      app: nginx
      tier: frontend
  spec:
    # modify replicas according to your case
    replicas: 3
    selector:
      matchLabels: # these labels should match with pod labels
        app: nginx
        tier: frontend
        environment: dev
    template: # this one is nothing but pod definition.
      metadata:
        labels: # these are the pod labels.
          app: nginx
          tier: frontend
          environment: dev
      spec:
        containers:
        - name: nginx
          image: nginx:1.14.2
  ```

- ReplicaSet is present in `apps/v1` resource type. Therefore, we set `apiVersion` to `apps/v1`
- `kubectl get rs` to check the status of the replicasets
- A replicaset creates pods internally

### 2. Deployment Set

- **If there is an update in the image, it will not update the exisiting pods in the replicaset** since its main purpose is to provide the no: of replicas mentioned in the manifest file
- Therefore we go for **deployment set**
- This is the most commonly used among others

  `sets/02-deployment.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels: # these labels belong to deployment
      app: nginx
  spec:
    replicas: 3
    selector:
      matchLabels: # these labels should match with pod
        app: nginx
        project: roboshop
        component: frontend
    template: # this is the pod definition
      metadata:
        labels:
          app: nginx
          project: roboshop
          component: frontend
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
  ```

- We can get all the deployments using: `kubectl get deploy`
- When we create a Deployment, it creates a Replicaset with the `replicaSet_name-random_ID`
- Deployment created another replicaset, therefore Replicaset is a subset of deployment
- Therefore, when we check: `kubectl get rs` to get the list of ReplicaSets, we can see it has the name: `deployment_name-random_ID`
- Using this, **we can acheive Zero Downtime** of our application

### How does it achieve it?

- When an update takes place, Deployment creates a new replicaset and uses the updated image when creating the pods
- Once a pod is provisioned, it terminates the pod from previous replicaset that is outdated as it needs to maintain the number of replicas

## Roboshop on EKS

- Broadly there are 2 steps that are involved:
  1. Image should exist i.e. build the image
  2. Configure the image using manifest files

### Common errors when implementing project with K8s

1. Either image exists or not ?
2. Is the image properly built ?
3. Did we specify the version of the image correctly?
4. Does the K8s cluster has access to pull the images from docker hub?
5. If everything is proper, is the configuration information specified is correct or not?

### Setup flow

1. Develop Dockerfiles and Manifest files locally and push them to GitHub
2. Pull them into Workstation and build the Dockerfile
3. Push the images to Docker Hub
4. Push the manifests to EKS Cluster

### MongoDB

- We start by creating a namespace

  `namespace.yaml`

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: roboshop
  ```

- Within the manifest files, we specify our deployments

  `mongodb/manifest.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mongodb
    namespace: roboshop
    labels: # these labels belong to deployment
      app: mongodb
      tier: db
      project: roboshop
  spec:
    replicas: 1
    selector:
      matchLabels: # these labels should match with pod
        app: mongodb
        tier: db
        project: roboshop
    template: # this is the pod definition
      metadata:
        namespace: roboshop
        labels:
          app: mongodb
          tier: db
          project: roboshop
      spec:
        containers:
        - name: mongodb
          image: joindevops/mongodb:1.0.0
  ```

#### Commands

1. `kubectl apply -f namespace.yaml`
2. `docker build -t joindevops/mongodb:1.0.0 .`
3. `docker login`
4. `docker push joindevops/mongodb:1.0.0`
5. `kubectl apply -f manifest.yaml`
6. `kubens roboshop`

- Using the 6th command, we don't need to mention `-n roboshop` everytime to fetch the resources provisioned in that namespace
- To confirm if the pood is created or not, we can run: `kubectl get pods -o wide`
- Now catalogue should connect with the MongoDB
- Connecting to it using IP address is not safe
- Therefore we use a service and as we don't need to expose the MongoDB port to the outside world, we can just use: **Cluster IP** service
- **Note**: A pod should always have a service attached to it in order to enable other pods to communicate with it
- **Attaching Cluster IP service to MongoDB**

  `mongodb/manifest.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mongodb
    namespace: roboshop
    labels: # these labels belong to deployment
      app: mongodb
      tier: db
      project: roboshop
  spec:
    replicas: 1
    selector:
      matchLabels: # these labels should match with pod
        app: mongodb
        tier: db
        project: roboshop
    template: # this is the pod definition
      metadata:
        labels:
          app: mongodb
          tier: db
          project: roboshop
      spec:
        containers:
        - name: mongodb
          image: joindevops/mongodb:1.0.0
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: mongodb
    namespace: roboshop
  spec:
    selector:
      app: mongodb
      tier: db
      project: roboshop
    ports:
    - name: mongodb-port
      protocol: TCP
      port: 27017 # this port belongs to service
      targetPort: 27017 # this port belongs to container
  ```

### Catalogue

- As we don't need to expose the catalogue port to external users, we can use **ClusterIP** service
- When using Docker with K8s, we should avoid storing variables as environment variables at the docker image layer
- Rather, we can define these environment variables using **ConfigMap**, therefore removing it from the Dockerfile
- Whenever there is a change in the application code, only then we should go for CICD process
- Therefore, we should maintain the configuration information outside the application code

  `catalogue/manifest.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: catalogue
    namespace: roboshop
  data:
    MONGO: "true" # keep true in double quotes
    MONGO_URL: "mongodb://mongodb:27017/catalogue" # protocol//service-name:service-port/schema
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: catalogue
    namespace: roboshop
    labels: # these labels belong to deployment
      app: catalogue
      tier: app
      project: roboshop
  spec:
    replicas: 1
    selector:
      matchLabels: # these labels should match with pod
        app: catalogue
        tier: app
        project: roboshop
    template: # this is the pod definition
      metadata:
        namespace: roboshop
        labels:
          app: catalogue
          tier: app
          project: roboshop
      spec:
        containers:
        - name: catalogue
          image: joindevops/catalogue:1.0.0
          envFrom:
          - configMapRef:
              name: catalogue
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: catalogue
    namespace: roboshop
  spec:
    selector:
      app: catalogue
      tier: app
      project: roboshop
    ports:
    - name: catalogue-port
      protocol: TCP
      port: 8080 # this port belongs to service
      targetPort: 8080 # this port belongs to container
  ```

- To provision the resources: `kubectl apply -f manifest.yaml`
- To check the logs:

  ```bash
  kubectl get pods
  kubectl logs <Name-of-the-pod>
  ```

- To debug the pods that are created, we can use a sample container with an image for e.g. almalinux that has basic commands such as telnet
- There can also be a case where the pods are being created in a different node i.e. mongo in one node, catalogue in another node
- Due to this, catalogue pod will not be able to communicate with the mongo pod

### Web Component

- Build is a costly operation, therefore we should:
  - Perform build and restart only when we have a code change
  - Perform restart only when there is a configuration
- Therefore we store the config information inside the ConfigMap as it allows to import config files as well
- Once we are done with our practice, we should clean up all the resources using:
  - `kubectl delete namespace roboshop` which also deletes the resources provisioned in that namespace
  - `kubens default`: Switching back to the default namespace
  - Or delete the whole cluster: `eksctl delete cluster --config-file=eks.yaml`
- `eksctl` uses cloudformation in the background which is native to AWS

## Storage

- Storage is all about performing CRUD operations on the data that is residing in the storage
- The data can be of any type i.e. file, excel, logs, RDBMS, NoSQL etc

### Stateful vs Stateless

- Stateful applications are those which does operations on the data directly i.e. data is very crucial here
- Stateless applications are those which doesn't have any database on its own i.e. no storage is necessary
- If Stateless applications are down, we can restore them easily without impacting the business
- In our Roboshop project:
  - MongoDB, redis, rabbitmq and MySQL components are stateful applications
  - Web, Catalogue, User, Cart, Shipping, Payment and Dispatch are stateless applications

### Current roboshop project scenario

- We clearly know that the pods are ephemeral similarly nodes are also ephemeral
- Therefore the data is being stored inside the pods but is not persistent
- Also at any point we increase or decrease the worker nodes, anytime it can be be deleted
- Therefore we shouldn't store the crucial data of stateful applications inside the pods or nodes
- We should store our data inside an external storage and mount it in the nodes and pods
- On AWS, we have Elastic Block Storage (EBS) and Elastic File Storage (EFS) service for the same use which we can mount to any upcoming or existing instances

# Different storage types on Kubernetes

## Storage on K8s or K8s volumes

- The storage types can be broadly classified into 2 types:
  1. Internal
  2. External

1. emptyDir - ephemeral, as it is **inside** pod
2. hostPath - ephemeral, as it is **inside** server
3. Static provisioning
4. Dynamic provisioning

- Static and Dynamic provisioning are external storages, therefore the data on it is permanent

### 1. emptyDir

- As long as the pod is alive, we can use sidecar to read the logs and push them to Elasticsearch for further analysis
- If the pod is deleted, the logs are also deleted with this approach
- In case if they're pushed to Elasticsearch, the logs are still persistent
- emptyDir is the default K8s ephemeral volume type that is widely used in sidecar patterns and is mounted in both main and sidecar container
- emptyDir and sidecar patterns is for shipping the pod logs
- Filebeat is popular model to ship the logs from K8s to Elasticsearch
- Filebeat is a public image from Elasticsearch
- Therefore it needs some configuration information:
  1. Where to ship ? Elastic search server IP address
  2. What are the files to ship?
- We can provide this configuration through **ConfigMap**

### 2. hostPath

- Using emptyDir, we can only ship the logs of the pod to the external system
- Sometimes we have to ship the logs of the servers for e.g. all the worker nodes for further analysis
- For this purpose, we use hostPath for shipping the server logs
- As we wanted to analyse the logs of more than one worker node, we use **DaemonSet**

#### DaemonSet

- If we run a DaemonSet, kubernetes will create a pod in each and every worker node. Also it deletes those pods when the node is deleted
- hostPath access is only given to administrators
- The purpose is to ship the server logs to Elasticsearch even though it is has potential risks involved
- The only purpose of the pods created by Daemonset is to access the server logs
- For this purpose, we can use **fluentd** from ELK Stack which will be deployed as a Daemonset that can access the underlying host logs through hostPath and send them to Elasticsearch
- Fluentd continuosly monitors the `/var/log/` directory and ships them to Elasticsearch. But we can also modify it to access the complete filesystem which is dangerous. Therefore we should restrict it by only providing a read-only access
- **Name of the admin Namespace: kube-system**
- [Source Code](https://github.com/daws-76s/k8-resources/blob/main/storage/02-hostpath.yaml)

### 3. Static provisioning

- Storing logs in an external storage such as Elastic Block store (EBS) is safe and secure
- As a part of static provisioning, create an EBS volume and give access to the EKS cluster
- From now on, EKS pods will store data in EBS volumes so that even though pod is deleted, the logs data are still there
- Similarly we also have EFS (Elastic File System) on AWS
- The harddisks i.e. EBS should be created in the same AZ's as the location of servers
- To access the EBS volume inside K8s, we need to install [EBS CSI drivers](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md#deploy-driver)
- A proper role should be attached to EC2 instance i.e. Node Group in order to access the EBS i.e. `AmazonEC2FullAccess`

#### Steps

1. Create an EBS volume: Either storage admin or K8s admin will create this
2. Install EBS CSI drivers in K8s using: `kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.29"`
3. Attach `AmazonEC2FullAccess` policy to the role that is attached to the EC2 instance

#### K8s volumes and resources

1. Persistent Volume (PV)
2. Persistent Volume Claim (PVC)

- We use **Persistent volume (PV)** object which is a representation of the underlying external storage i.e. a wrapper for the external storage in K8s as the administrators need not to understand the underlying architecture of the EBS Harddisk. Therefore K8s came up with this PV object which we can perform operations on the underlying storage
- To use this, we create an equivalent PV object to represent EBS disk and map this to a EBS volume
- To use the EBS storage inside K8s pods, we need to have **PersistentVolumeClaim (PVC)** i.e. Pods should request volume through PVC
- PVC is not a physical representation rather its just a request to the PV
- When creating a PV object, twe should set the access mode to **ReadWriteOnce** for the EBS volume
- Using PVC, the pod can claim a certain amount of space from the PV.
- In Static provisioning, the storageClassName should be set to `""` as we're creating it manually
- After claiming, we should attach it to the Pod.
- To see if its working fine or not, we can attach a loadBalancer service to the pod
- When a pod gets deployed on to the nodes, there is chance that it might get deployed in the node in another AZ in which the created EBS volume is not present
- To restrict this, we can use: **nodeSelector** and assign labels to the nodes and specify in which nodes the pod can be deployed based on the labels
- We can see all the current labels of the nodes using: `kubectl get nodes --show-labels`
- To label a node, `kubectl label nodes <NAME-OF-THE-NODE> <LABEL>`
- Once the pods that are writing to EBS volume are deleted, the EBS volume will be unmounted as well i.e. changed from In-use to available state on AWS
- [Source Code](https://github.com/sivadevopsdaws74s/k8-resources/blob/master/storage/03-ebs-static.yaml)

#### Steps involved

1. Create an EBS volume: Either storage admin or K8s admin will create this
2. Install EBS CSI drivers in K8s using: `kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.29"`
3. EBS PV
4. Add necessary permissions to the worker nodes to access EBS
5. create PVC --> a way of claiming storage
6. create volume out of PVC
7. mount to container

### 4. Dynamic Provisioning

- Volume should be created automatically
- There is another object called **storageClass** that can create storage dynamically based on the request
- If we have created volume manually, we should also create PV and PVC objects manually and mount it in the pod
- As storageClass creates the external volume, it is the responsibility of **storageClass** to create the PV object as well

  `04-storage/ebs-sc.yaml`

  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: ebs-sc
  provisioner: ebs.csi.aws.com
  volumeBindingMode: WaitForFirstConsumer
  ```

- We can get the list of storageClasses using: `kubectl get sc`
- The driver that is installed by default (i.e. `kubernetes.io/aws-ebs`) is not good to us, therefore we should our own customized drivers such as `ebs.csi.aws.com`
- All the steps involved in static provisioning are manual whereas in dynamic provisioning, the disk is created automatically and mounted too using the storageClassName parameter
- Since storageClass is a ClusterLevel resource, admins should create this and hence EBS will have one storageClass cluster wide.
- Once the storageClass is created by admins, they will inform other teams and they will clam the space as per the need.

  `04-storage/04-ebs-dynamic.yaml`

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: ebs-dynamic
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: ebs-sc
    resources:
      requests:
        storage: 4Gi
  ```

- Attach the PVC to a pod:

  `04-storage/04-ebs-dynamic.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: app
    labels:
      demo: ebs-dynamic
  spec:
    containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - name: nginx-data
        mountPath: /usr/share/nginx/html
    nodeSelector:
      zone: 1b
    volumes:
    - name: nginx-data
      persistentVolumeClaim:
        claimName: ebs-dynamic
  ```

- Attach a Loadbalancer service to the pod:

  `04-storage/04-ebs-dynamic.yaml`

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: ebs-dynamic
  spec:
    type: LoadBalancer
    selector:
      demo: ebs-static
    ports:
    - protocol: TCP
      port: 80 #service-port
      targetPort: 80 #container-port
  ```

- PV is not a namespaced resource whereas PVC is a namespaced one.
- To get all the PV and PVC information, we can use: `kubectl get pv`, `kubectl get pvc`
- If the pod is deleted, the underlying volume will also be deleted as the `RECLAIMPOLICY` is set to delete which can be observed from: `kubectl get sc` command output
- We can set to `Retain` when defining the StorageClass object i.e. `reclaimPolicy: Retain`
- **The data from the databases should be stored in the external storage but not on the server**

## Elastic File Storage (EFS)

- It is an implementation of NFS (Network File Storage)
- NFS works on TCP Port 2049

# Different storage types on Kubernetes

## Dynamic Provisioning Storage on K8s

- Creation of disks can be handled by Kubernetes.
- StorageClass (SC) can be used for creating the underlying disks and PersistentVolumes (PV) automatically

### Steps for setup

1. Install drivers, because SC object use drivers to create volumes.
    - [EBS CSI drivers](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md#deploy-driver)
2. Claim the necessary volume using PersistentVolumeClaim
3. Deploy the pods and attach volume to it

## EFS

### EBS vs EFS

- EBS is blockstore, EFS is network storage
- EBS can't scale automatically, whereas EFS can get more space automatically
- As EFS is dynamic in nature, one whole company can have one EFS
- Each folder gets an individual access point that is specific to it

### Static Provisioning using EFS

1. Create EFS filesystem
2. Allow traffic to Port 2049 in the security group of the EFS from worker nodes
3. Install necessary EFS drivers: `kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.7" > private-ecr-driver.yaml`
4. Add AmazonElasticFileSystemFullAccess policy to any of the Worker Node to access EFS
5. Create PV, PVC and mount it to the pod

### Dynamic Provisioning using EFS

- We need to have StorageClass for Dynamic Provisioning

1. Create EFS filesystem
2. Allow traffic in the security group of the EFS
    - Install necessary EFS drivers
3. Add AmazonElasticFileSystemFullAccess policy to any of the Worker Node to access EFS

- There is a folder structure that is created by K8s inside the access point which is not visible

## Statefulset

- So far, we have discussed:
  - ReplicaSet
  - DeploymentSet --> stateless applications i.e. data is not at all important i.e. cart, catalogue
  - DeamonSet
- Statefulset: used for DB related applications such as MySQL, Redis, MongoDB, Rabbimq

### Deploymentset vs Statefulset

- Let's consider a use-case where a user creation, updation or deletion request is received by the Master node, it assigns that task to any of the work nodes in the cluster
- The node that gets assigned **will also inform all the other nodes** about this request so that the data is persistent across all nodes in the cluster
- Stateful applications should have the same static hostname for all the worker nodes whereas with Deploymentset the nodes get assigned with a random name
- Statefulset will create pods in orderly manner i.e. nginx-0, nginx-1, nginx-2 will be created in an order.
  - If nginx-0 is successfully up and running only then it creates nginx-1 similarly nginx-2
  - This is to ensure that the cluster is successfully created
- Stateful applications should follow orderly provisioning and terminating
  - Provisioning happens in forward direction and termination happens in reverse direction
- In StatefulSet each Replica should have its own database
- Deploymentset can create many pods at a time
- Statefulset keeps the pod identity same for the communication.
- Statefulset preserves network identity such as pod names
- For e.g. when deploying MySQL database in Kubernetes, MySQL takes care of the communication in background but we should provision the storage using PV and PVC

#### How does statefulSet updates all the other nodes about the request?

- It uses a **Headless service** for this purpose i.e. it performs `nslookup <service>` to fetches all the IP address of the other nodes that are part of it
- Then it updates one-by-one with this information
- Whereas with Deployment uses a service and when we perform `nslookup`, it returns the IP address of service that redirects the traffic to any-of-the node
- **In statefulset, clusterIP should be set to None**
- When the pods with type Statefulsets are deleted, the storage associated with them are not deleted
- This is to ensure that the volumes are attached back when they're created once again

#### Steps to implement

1. Install drivers
2. Attach **AmazonEBSCSIDriverPolicy** policy to the IAM user attached to worker nodes
3. Install storageClass

- It is not suggested to store end user data inside K8s as it becomes very difficult to manage for the DB team
- We can store data such as logs inside Elasticsearch, prometheus etc with in the K8s

# Different storage types on Kubernetes

## Dynamic Provisioning Storage on K8s

- Creation of disks can be handled by Kubernetes.
- StorageClass (SC) can be used for creating the underlying disks and PersistentVolumes (PV) automatically

### Steps for setup

1. Install drivers, because SC object use drivers to create volumes.
    - [EBS CSI drivers](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md#deploy-driver)
2. Claim the necessary volume using PersistentVolumeClaim
3. Deploy the pods and attach volume to it

## EFS

### EBS vs EFS

- EBS is blockstore, EFS is network storage
- EBS can't scale automatically, whereas EFS can get more space automatically
- As EFS is dynamic in nature, one whole company can have one EFS
- Each folder gets an individual access point that is specific to it

### Static Provisioning using EFS

1. Create EFS filesystem
2. Allow traffic to Port 2049 in the security group of the EFS from worker nodes
3. Install necessary EFS drivers: `kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.7" > private-ecr-driver.yaml`
4. Add AmazonElasticFileSystemFullAccess policy to any of the Worker Node to access EFS
5. Create PV, PVC and mount it to the pod

### Dynamic Provisioning using EFS

- We need to have StorageClass for Dynamic Provisioning

1. Create EFS filesystem
2. Allow traffic in the security group of the EFS
    - Install necessary EFS drivers
3. Add AmazonElasticFileSystemFullAccess policy to any of the Worker Node to access EFS

- There is a folder structure that is created by K8s inside the access point which is not visible

## Statefulset

- So far, we have discussed:
  - ReplicaSet
  - DeploymentSet --> stateless applications i.e. data is not at all important i.e. cart, catalogue
  - DeamonSet
- Statefulset: used for DB related applications such as MySQL, Redis, MongoDB, Rabbimq

### Deploymentset vs Statefulset

- Let's consider a use-case where a user creation, updation or deletion request is received by the Master node, it assigns that task to any of the work nodes in the cluster
- The node that gets assigned **will also inform all the other nodes** about this request so that the data is persistent across all nodes in the cluster
- Stateful applications should have the same static hostname for all the worker nodes whereas with Deploymentset the nodes get assigned with a random name
- Statefulset will create pods in orderly manner i.e. nginx-0, nginx-1, nginx-2 will be created in an order.
  - If nginx-0 is successfully up and running only then it creates nginx-1 similarly nginx-2
  - This is to ensure that the cluster is successfully created
- Stateful applications should follow orderly provisioning and terminating
  - Provisioning happens in forward direction and termination happens in reverse direction
- In StatefulSet each Replica should have its own database
- Deploymentset can create many pods at a time
- Statefulset keeps the pod identity same for the communication.
- Statefulset preserves network identity such as pod names
- For e.g. when deploying MySQL database in Kubernetes, MySQL takes care of the communication in background but we should provision the storage using PV and PVC

#### How does statefulSet updates all the other nodes about the request?

- It uses a **Headless service** for this purpose i.e. it performs `nslookup <service>` to fetches all the IP address of the other nodes that are part of it
- Then it updates one-by-one with this information
- Whereas with Deployment uses a service and when we perform `nslookup`, it returns the IP address of service that redirects the traffic to any-of-the node
- **In statefulset, clusterIP should be set to None**
- When the pods with type Statefulsets are deleted, the storage associated with them are not deleted
- This is to ensure that the volumes are attached back when they're created once again

#### Steps to implement

1. Install drivers
2. Attach **AmazonEBSCSIDriverPolicy** policy to the IAM user attached to worker nodes
3. Install storageClass

- It is not suggested to store end user data inside K8s as it becomes very difficult to manage for the DB team
- We can store data such as logs inside Elasticsearch, prometheus etc with in the K8s

# Different storage types on Kubernetes

## Dynamic Provisioning Storage on K8s

- Creation of disks can be handled by Kubernetes.
- StorageClass (SC) can be used for creating the underlying disks and PersistentVolumes (PV) automatically

### Steps for setup

1. Install drivers, because SC object use drivers to create volumes.
    - [EBS CSI drivers](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md#deploy-driver)
2. Claim the necessary volume using PersistentVolumeClaim
3. Deploy the pods and attach volume to it

## EFS

### EBS vs EFS

- EBS is blockstore, EFS is network storage
- EBS can't scale automatically, whereas EFS can get more space automatically
- As EFS is dynamic in nature, one whole company can have one EFS
- Each folder gets an individual access point that is specific to it

### Static Provisioning using EFS

1. Create EFS filesystem
2. Allow traffic to Port 2049 in the security group of the EFS from worker nodes
3. Install necessary EFS drivers: `kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.7" > private-ecr-driver.yaml`
4. Add AmazonElasticFileSystemFullAccess policy to any of the Worker Node to access EFS
5. Create PV, PVC and mount it to the pod

### Dynamic Provisioning using EFS

- We need to have StorageClass for Dynamic Provisioning

1. Create EFS filesystem
2. Allow traffic in the security group of the EFS
    - Install necessary EFS drivers
3. Add AmazonElasticFileSystemFullAccess policy to any of the Worker Node to access EFS

- There is a folder structure that is created by K8s inside the access point which is not visible

## Statefulset

- So far, we have discussed:
  - ReplicaSet
  - DeploymentSet --> stateless applications i.e. data is not at all important i.e. cart, catalogue
  - DeamonSet
- Statefulset: used for DB related applications such as MySQL, Redis, MongoDB, Rabbimq

### Deploymentset vs Statefulset

- Let's consider a use-case where a user creation, updation or deletion request is received by the Master node, it assigns that task to any of the work nodes in the cluster
- The node that gets assigned **will also inform all the other nodes** about this request so that the data is persistent across all nodes in the cluster
- Stateful applications should have the same static hostname for all the worker nodes whereas with Deploymentset the nodes get assigned with a random name
- Statefulset will create pods in orderly manner i.e. nginx-0, nginx-1, nginx-2 will be created in an order.
  - If nginx-0 is successfully up and running only then it creates nginx-1 similarly nginx-2
  - This is to ensure that the cluster is successfully created
- Stateful applications should follow orderly provisioning and terminating
  - Provisioning happens in forward direction and termination happens in reverse direction
- In StatefulSet each Replica should have its own database
- Deploymentset can create many pods at a time
- Statefulset keeps the pod identity same for the communication.
- Statefulset preserves network identity such as pod names
- For e.g. when deploying MySQL database in Kubernetes, MySQL takes care of the communication in background but we should provision the storage using PV and PVC

#### How does statefulSet updates all the other nodes about the request?

- It uses a **Headless service** for this purpose i.e. it performs `nslookup <service>` to fetches all the IP address of the other nodes that are part of it
- Then it updates one-by-one with this information
- Whereas with Deployment uses a service and when we perform `nslookup`, it returns the IP address of service that redirects the traffic to any-of-the node
- **In statefulset, clusterIP should be set to None**
- When the pods with type Statefulsets are deleted, the storage associated with them are not deleted
- This is to ensure that the volumes are attached back when they're created once again

#### Steps to implement

1. Install drivers
2. Attach **AmazonEBSCSIDriverPolicy** policy to the IAM user attached to worker nodes
3. Install storageClass

- It is not suggested to store end user data inside K8s as it becomes very difficult to manage for the DB team
- We can store data such as logs inside Elasticsearch, prometheus etc with in the K8s

# Further Topics in K8s

## Provision EKS cluster using Terraform

- EKS cluster is a VPC based resource
- Load Balancing is not a part of the EKS cluster
- **Each and every resource that is a part of VPC has a Security group associated with it**
- An IAM user is also created when creating the cluster with `eksctl` and we modified it according to our needs
- But when provisioning it using Terraform, we will automate it including the security groups that are linked with Node instances and Control plane
- All traffic should be enabled b/w control plane and worker node instances and vice-versa as it is completely secured and present inside a VPC
- The worker node instances should accept traffic on Ephemeral range from the loadbalancer as node port is created with in the range by EKS
- We can place the whole EKS clutser and worker nodes inside a Private subnet and Load balancer inside a public subnet
- **cluster_node**: Cluster accepting connections from Node
- After provisioning it using Terraform, we should update `kubeconfig` information in the workstation which can be done using: `aws eks update-kubeconfig --region us-east-1 --name roboshop-tf`

## Taints and Tolerations

### Taints

- Taint is nothing but paint i.e. polluting it
- When a node is tainted, the default scheduler can't schedule pod on that node
- But the exisiting pods in the node will continue running on it
- In organisation, few projects add their nodes to EKS cluster and would like to restrict other projects not to provision their pods inside their nodes.
- For this purpose, they taint their nodes
- We can do this using: `kubectl taint nodes <name of the node> key1=value1:NoSchedule`
  - `NoSchedule`: It specifies the scheduler not to schedule any new pods on that node
  - `NoExecute`: It deletes all the pods that are running on that node
  - `PreferNoSchedule`: K8s tries not to schedule new pods on the tained node but can't guarantee it
  - E.g. `kubectl taint nodes ip-10-0-11-33.ec2.internal project=roboshop:NoSchedule`
- To untaint a node, we can use: `kubectl taint nodes <name of the node> key1=value1:NoSchedule-`. For e.g. `kubectl taint nodes ip-10-0-11-33.ec2.internal project=roboshop:NoSchedule-`

### Tolerations

- A project can apply tolerations i.e. excuses inorder to schedule the pods in a specific node only
- We can specify the tolerations at the container level in the sepcifications of the resource

  `selectors/01-taint-toleration.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: taint-pod
  spec:
    # list of containers
    containers:
    - name: hello-pod
      image: nginx
      ports:
      - containerPort: 80
    tolerations:
    - key: "project"
      operator: "Exists"
      effect: "NoSchedule"
  ```

- Once a toleration is specified, the scheduler can provision the pods inside the tainted nodes that matches as well

## Affinity and anti-affinity

- Earlier when attaching an external EBS storage to a Pod, we used a PV object and we mentioned that both the EBS volume and nodes should be in the same region for the volume to be attached
- For this, we used `nodeSelector` attribute to specify the region and which nodes can be selected
- With `nodeSelector`, we don't have much options at hand to filter the nodes whereas with Affinity and anti-affinity, we have more options to select the nodes
- There are 2 types of node affinity:
  1. `requiredDuringSchedulingIgnoredDuringExecution`: While scheduling the pods, the labels should match and its a hardrule. But these settings are ignored not at the time of pods execution
  2. `preferredDuringSchedulingIgnoredDuringExecution`: While scheduling the pods, the scheduler tries to match the labels and if doesn't match, still the scheduler schedules the pod. These settings are ignored not at the time of pods execution
- Before applying affinity or anti-affinity, the nodes should be labeled. It can be done using: `kubectl label nodes ip-10-0-11-33.ec2.internal project=roboshop`

  `selectors/02-node-affinity.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: with-node-affinity
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: project
              operator: In
              values:
              - roboshop
    containers:
    - name: with-node-affinity
      image: nginx
  ```

- Node affinity weight will only work for nodes with `preferredDuringSchedulingIgnoredDuringExecution` option
- Anti affinity is nothing but setting the value of operator to `NotIn`
- Similar to node affinity, we also have pod affinity. For e.g. lets say we have pod1 that is running in node1. Now when we set pod affinity to pod2, it would also like to run inside the same node where pod1 is running.

## Cluster Upgrade

- Before upgrade, you can announce downtime i.e. no Create, Update of applications, No releases or Changes
- Also modify the security group settings so that no one is able to access the cluster using `kubectl`

1. Create a separate node group called green
2. We need to taint new worker nodes i.e. `kubectl taint nodes ip-10-0-11-157.ec2.internal project=roboshop:NoExecute` so that any running pods in those will be terminated
3. Upgrade control plane to 1.28 i.e. manually through console so that the existing blue worker node group will not be affected
4. Upgrade green node group to 1.28
5. Taint blue nodes so that new pods will not be scheduled
6. untaint green
7. cordon/drain the blue nodes using: `kubectl drain --ignore-daemonsets <node-name> --force --delete-emptydir-data`

- Now in terraform code, we should comment the blue node group and change the version to 1.28 manually

## Liveness and Readiness Probe

- With Liveness probe, K8s checks whether a container is running or not for every specific duration that is specified for e.g. every 10 seconds
  - initialDelaySeconds: K8s waits for these many seconds (for e.g. 15 seconds) after the container starts before checking the status of the container
  - periodSeconds: After every 10 seconds for e.g. K8s will check the whether the container is running or not. If its not healthy, K8s will restart that container
- With Readiness probe, K8s checks whether the application is ready to serve traffic or not for every specific duration that is specified for e.g. every 10 seconds
- Using these two, kubernetes monitors our application 24x7 without any manual effort