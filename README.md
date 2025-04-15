# **Project Title: Introduction to Configuration Management in Kubernetes with Kustomize on AWS**

## **Overview**
This project provides a foundational guide for Kubernetes configuration management using **Kustomize**, with deployments on **Amazon Web Services (AWS)** using **Elastic Kubernetes Service (EKS)**. The goal is to enable users to grasp how to manage Kubernetes resources effectively using Kustomize and deploy them on a cloud infrastructure like AWS.

---

## **Lesson 2.1: Kustomize Structure and Concepts**

### **Objective**
To understand the core directory structure, primary files, and fundamental concepts of Kustomize.

### **Tasks & Steps**

1. **Understanding Kustomize Structure**
   - **Base Directory**: Contains common, reusable resources for the application, like base deployment, service definitions, etc.
   - **Overlays Directory**: Holds environment-specific configurations (e.g., dev, prod), allowing for easy adjustments for each environment.
   - **Kustomization File (kustomization.yaml)**: Declares resources, bases, and overlays to build the Kubernetes manifests.

2. **Key Concepts**
   - **Resources**: Kubernetes object manifests (e.g., Deployments, Services) that define the infrastructure and app configurations.
   - **Bases**: Generic configurations that serve as the foundational setup.
   - **Overlays**: Environment-specific adjustments applied on top of the bases for customization.

3. **Activity: Create Directory Structure**

```bash
myapp/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    │   └── replica_count.yaml
    │   └── service_nodeport.yaml
    └── prod/
        └── kustomization.yaml
        └── replica_count.yaml
```

- Populate `deployment.yaml` with a simple deployment manifest.

### **Screenshot Placeholder**  
_Add screenshots of the local directory structure and a few of the file contents here._

---

## **Lesson 2.2: Creating and Managing Resources**

### **Objective**
To define, include, and manage Kubernetes resources using Kustomize.

### **Tasks & Steps**

1. **Define Basic Deployment**

In `base/deployment.yaml`:

```yaml
apiVersion: apps/v1    # API version for the deployment object
kind: Deployment       # Specifies that this is a Deployment
metadata:
  name: nginx-deployment  # Name of the deployment
spec:                   # Specification of the deployment
  replicas: 2           # Number of replicas (pods running)
  selector:             # Selector to identify the pods
    matchLabels:
      app: nginx
  template:             # Template for the pod creation
    metadata:
      labels:
        app: nginx     # Label applied to the pod
    spec:
      containers:      
      - name: nginx    # Name of the container
        image: nginx:latest  # Docker image to use for the container
        ports:
        - containerPort: 80  # Port the container will expose 
```

2. **Reference in kustomization.yaml**

In `base/kustomization.yaml`:

```yaml
resources:
  - deployment.yaml # List of resource files to include in this configuration
  - service.yaml    # List of resource files to include in this configuration 
```

In `base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

3. **Manage Other Kubernetes Objects**
   - Extend this process to Services, Pods, ConfigMaps, and other Kubernetes objects as necessary.

### **Screenshot Placeholder**  
_Add screenshots of the Kubernetes resources, such as the service and deployment manifest._

---

## **Lesson 2.3: Basic Customization Techniques**

### **Objective**
To customize configurations for different environments using Kustomize.

### **Tasks & Steps**

1. **Create Environment Overlays**
   - `overlays/dev/kustomization.yaml`
   - `overlays/dev/replica_count.yaml`

2. **Customization Example**

**`kustomization.yaml` (in `overlays/dev/`)**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
  - target:
      kind: Deployment
      name: nginx-deployment
    path: replica_count.yaml
  - target:
      kind: Service
      name: nginx-service
    path: service_nodeport.yaml
```

**`replica_count.yaml` (in `overlays/dev/`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3  # Updating the number of replicas for the dev environment
```

**`service_nodeport.yaml` (in `overlays/dev/`)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080  # Optional: Explicitly assign a port (range: 30000-32767)
```

3. **Variables and Placeholders**
   - Kustomize allows for environment-specific substitution using variables (refer to official documentation for advanced usage).

### **Screenshot Placeholder**  
_Add screenshots of the customized `kustomization.yaml` and any files reflecting environment-specific changes._

---

## **Test the Service (from inside the cluster)**

Because `nginx-service` is a **ClusterIP** type, it’s only accessible **inside the cluster**, so we’ll use a temporary Pod to curl the service.

### **Step 1: Create a Test Pod**

Run this command to start a debugging Pod with `curl` installed:

```bash
kubectl run test-pod --image=busybox --restart=Never -it --rm --command -- sh
```

Inside the shell that pops up, type this:

```sh
wget -qO- nginx-service
```

If everything’s working, you’ll get HTML output from NGINX.

---

### **If it doesn’t work:**

Try:

```sh
wget -qO- nginx-service.default.svc.cluster.local
```

Or check if DNS is working inside the pod:

```sh
nslookup nginx-service
```

---

### **Next Recommended Step: Add a `NodePort` to Dev Overlay**

Right now, your `nginx-service` is a `ClusterIP`, which isn’t reachable outside the cluster.

Let’s expose it via `NodePort` in the `dev` overlay so you can test it in the browser from your host machine.

---

### **Step-by-Step: Patch `nginx-service` to NodePort**

1. **Create a patch file** in your `overlays/dev/` directory:

```bash
touch service_nodeport.yaml
```

2. **Edit the `service_nodeport.yaml`** to override the service type:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080  # Optional: Explicitly assign a port (range: 30000-32767)
```

3. **Update `kustomization.yaml` in `overlays/dev/`** to include the new patch:

```yaml
resources:
  - ../../base

patches:
  - replica_count.yaml
  - service_nodeport.yaml
```

4. **Apply the overlay again:**

```bash
kubectl apply -k myapp/overlays/dev/
```

5. **Check the updated service:**

```bash
kubectl get svc nginx-service
```

---

## **Leveraging AWS: Deploying with Amazon EKS**

### **Step 1: AWS Account & CLI Setup**
- Create an AWS account if you don't have one.
- Install and configure the AWS CLI:

```bash
aws configure
```

---

### **Step 2: Install and Configure eksctl**
- Install `eksctl` from the [GitHub repository](https://github.com/weaveworks/eksctl):

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

---

### **Step 3: Create EKS Cluster**

In this step, you will create an EKS cluster using `eksctl`.

```bash
eksctl create cluster \
  --name my-kustomize-cluster \
  --version 1.18 \
  --region us-west-2 \
  --nodegroup-name my-nodes \
  --node-type t2.medium \
  --nodes 3
```

---

### **Step 4: Deploy Kustomize to EKS**

Deploy your application to EKS using the `kubectl` command:

```bash
kubectl apply -k overlays/dev/
```

---

### **Step 5: Verify Deployment**

Check the status of the deployment:

```bash
kubectl get all
kubectl describe deployment nginx-deployment
kubectl logs <pod-name>
```

---

### **Step 6: Clean Up**

Once you're done, clean up the resources to avoid incurring unnecessary costs:

```bash
kubectl delete -k overlays/dev/
eksctl delete cluster --name my-kustomize-cluster
```

---

### **Additional Considerations**
- **AWS Costs**: Monitor AWS costs by using AWS Cost Explorer and ensuring that resources are cleaned up after usage.
- **Security**: Make sure to follow best practices for Kubernetes security on AWS, including using IAM roles, security groups, and Kubernetes RBAC (Role-Based Access Control) to secure your resources.
- **Documentation**: Always refer to the latest AWS and Kustomize documentation for updates and best practices.

---

## **Conclusion**
This project provides hands-on experience in configuring Kubernetes resources using **Kustomize** and deploying them on **Amazon EKS**. By following these steps, I was able to manage environment-specific configurations and deploy them successfully to the cloud, providing a scalable solution for managing Kubernetes workloads.
