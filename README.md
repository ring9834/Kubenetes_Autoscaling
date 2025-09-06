# Kubernetes with Pods Autoscaling/Rolling Updates
In this example, we use Minikube to demonstrate core Kubernetes functionalities with Docker, covering both basic and advanced usage. The environment is an Ubuntu Linux virtual machine running Kubernetes.

## Basic: Set Up the Kubernetes Environment on Ubuntu
Update the System
```sh
sudo apt update && sudo apt upgrade -y
```

Install Docker (Minikube uses Docker as the default container runtime):
```sh
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER // add your current user to the Docker group so that you can run Docker commands without needing to use sudo every time
```

Install Minikube:

// download the latest version of Minikube for Linux (64-bit) using curl

// -L: Follow redirects (in case the URL points to another location). 

// -O: Write output to a file with the same name as in the URL (in this case, minikube-linux-amd64).

```sh
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Install kubectl:
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
// +x: Adds execute permission to the file.
chmod +x kubectl 
sudo mv kubectl /usr/local/bin/
```

Install conntrack (required for Minikube networking):
```sh
sudo apt install -y conntrack
```

Start Minikube Cluster:

visudo // to edit the sudoers file securely

hh ALL=(ALL:ALL) ALL // Add this line to grant hh sudo privileges

// start a local Kubernetes cluster using Minikube with the Docker driver as the underlying virtualization method.

// --driver=docker: Tells Minikube to use Docker as the container runtime (instead of VirtualBox, KVM, Hyper-V, etc.)

```sh
minikube start --driver=docker
```
This creates a single-node Kubernetes cluster.
Verify with:
```
minikube status
kubectl cluster-info
```

Enable Metrics Server (for autoscaling later):
```sh
minikube addons enable metrics-server
```
Verification:
```sh
kubectl get nodes
```
Now our environment is ready. Let's proceed with Kubernetes core functions.

## Basic: Deploy a Simple Pod
A Pod is the smallest deployable unit in Kubernetes, running one or more containers.

Example: Deploy a single Nginx container as a Pod.

Create a Pod Manifest:
```sh
nano pod.yaml
```

Add the following:
```sh
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
- containerPort: 80
```

Apply the Pod:
```sh
kubectl apply -f pod.yaml
```

Check Pod Status:
```sh
kubectl get pods
kubectl describe pod nginx-pod
```

## Basic: Expose the Pod with a Service
A Service provides a stable endpoint to access Pods, enabling load balancing and discovery.

Example: Expose the Nginx Pod using a ClusterIP Service.

Create a Service Manifest:
```sh
nano service.yaml
```

Add the following:
```sh
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
    targetPort: 80 // Maps Service port 80 to container port 80.
  type: ClusterIP // ClusterIP makes the Service accessible only within the cluster.
  ```

Apply the Service:
```sh
kubectl apply -f service.yaml
```

Check Service:
```sh
kubectl get services
```

Test Access (from within the cluster):

// create and run a temporary pod in a Kubernetes cluster, using the curlimages/curl image, and opens an interactive shell session inside it.
```sh
kubectl run curl-pod --image=curlimages/curl --rm -it -- sh
```

Inside the pod, run:
```sh
curl nginx-service
```
You should see the Nginx welcome page HTML.

## Intermediate: Scale with a Deployment
A Deployment manages multiple Pod replicas, ensuring scalability and self-healing.

Example: Replace the single Pod with a Deployment running 3 Nginx replicas.

Create a Deployment Manifest:
```sh
nano deployment.yaml
```

Add the following:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec: // Defines the desired state of the Deployment.
  replicas: 3 // Specifies that three instances (pods) of the application should be running. Kubernetes will maintain this number, restarting or creating new pods if needed.
  selector: // Defines how the Deployment identifies the pods it manages.
    matchLabels:
      app: nginx // The Deployment manages pods with the label app: nginx. This links the Deployment to its pods.
  template: // Describes the pod template used to create new pods.
    metadata:
      labels:
        app: nginx // Assigns the label app: nginx to pods created by this Deployment. This matches the selector above, ensuring the Deployment controls these pods.
    spec: // Defines the pod’s configuration.
      containers: // Lists the containers to run in each pod.
      - name: nginx // Names the container nginx (arbitrary, but useful for identification).
        image: nginx:latest
        ports:
        - containerPort: 80 // Indicates the container listens on port 80 (standard HTTP port for NGINX). This is informational for Kubernetes; it doesn’t configure networking on its own.
```

Delete the Previous Pod:
```sh
kubectl delete pod nginx-pod
```

Apply the Deployment:
```sh
kubectl apply -f deployment.yaml
```

Check Deployment and Pods:
```sh
kubectl get deployments
kubectl get pods
```

Scale the Deployment:
```sh
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods
```

## Intermediate: Expose Deployment Externally with NodePort
Since Minikube doesn't support LoadBalancer services natively, we'll use a NodePort Service to expose the Deployment externally.

Example: Expose the Nginx Deployment to external traffic.

Create a NodePort Service Manifest:
```sh
nano nodeport.yaml
```

Add the following:
```sh
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80 // The port the Service exposes internally in the cluster. Clients within the cluster can access the Service at this port.
    targetPort: 80 // The port on the pods where traffic is sent. This matches the containerPort: 80 in the NGINX pods from the Deployment, where NGINX listens.
    nodePort: 30080
  type: NodePort // Specifies an external port on each cluster node (in the range 30000–32767) where the Service is accessible.
  ```

Apply the Service:
```sh
kubectl apply -f nodeport.yaml
```

Check Service:
```sh
kubectl get services
```

Access the Service:
```sh
minikube service nginx-nodeport --url
```
This outputs a URL: http://192.168.49.2:30080

Open the URL in a browser or use:
```sh
curl http://192.168.49.2:30080
```

## Advanced: Configure Persistent Storage with PVC and PV
A PersistentVolume (PV) and PersistentVolumeClaim (PVC) provide storage for stateful applications.

Example: Add persistent storage to a Pod running Nginx.

Create a PersistentVolume Manifest:
```sh
nano pv.yaml
```

Add the following:
```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
path: "/mnt/data"
```
Create a PersistentVolumeClaim Manifest:
```sh
nano pvc.yaml
```

Add the following:
```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Create a Pod with PVC:
```sh
nano pod-with-pvc.yaml
```

Add the following:
```sh
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pvc-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: storage
      mountPath: "/usr/share/nginx/html"
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: nginx-pvc
```

Apply the Resources:
```sh
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod-with-pvc.yaml
```

Verify:
```sh
kubectl get pvc
kubectl get pv
kubectl get pods
```

PersistentVolume (PV): A PV is a cluster-wide resource representing a piece of storage in the cluster, provisioned either manually by an administrator or dynamically by a storage class. It defines the storage's properties, such as capacity, access modes (e.g., ReadWriteOnce, ReadOnlyMany), and the underlying storage type (e.g., NFS, cloud storage like AWS EBS).

PersistentVolumeClaim (PVC): A PVC is a request for storage by a user or application. It specifies the storage requirements, such as size and access modes, without needing to know the details of the underlying storage. A PVC binds to a PV that meets its requirements.

Binding: A PVC "claims" a PV that matches its requirements (e.g., size, access mode). Once bound, the PV is reserved exclusively for that PVC, and no other PVC can use it until it's released.

Abstraction: PVs are low-level storage resources managed by cluster admins, while PVCs provide a user-friendly abstraction, allowing developers to request storage without worrying about the underlying infrastructure.

A PV exists independently and can be pre-provisioned or dynamically created.
A PVC is created by a user and binds to an available PV. If no suitable PV exists, dynamic provisioning (via a StorageClass) may create one.
When a PVC is deleted, the bound PV's fate depends on its reclaim policy (e.g., Retain, Delete, or Recycle).

One-to-One Mapping: A PVC typically binds to one PV, forming a one-to-one relationship during its lifecycle.

Example:
A PV is created with 10Gi of storage and ReadWriteOnce access.
A PVC is created requesting 5Gi with ReadWriteOnce. Kubernetes binds the PVC to the PV, and the pod using the PVC can access the storage.
If the PVC is deleted, the PV might be retained or deleted based on its reclaim policy.

## Advanced: Implement Autoscaling with HPA
A HorizontalPodAutoscaler (HPA) scales Pods based on resource usage.

Example: Autoscale the Nginx Deployment based on CPU usage.

Update Deployment with Resource Limits:
```sh
nano deployment.yaml
```

Update to include resource limits:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources: // Defines resource constraints for the container. The container is guaranteed 100m CPU and capped at 200m CPU.
          requests:
            cpu: "100m" // Requests a minimum of 100 milliCPU (0.1 CPU core) for the container.
          limits:
            cpu: "200m" // Limits the container to a maximum of 200 milliCPU (0.2 CPU core) to prevent overuse.
```

If there some errors for pods, use the following command to restart pods:
```sh
kubectl delete pod nginx-deployment-96b9d695-2kzlr -n default
```

Apply Updated Deployment:
```sh
kubectl apply -f deployment.yaml
```

Create an HPA:

// The HPA scales between 2 and 10 replicas if CPU usage exceeds 50%.
```sh
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=2 --max=10
```

Verify HPA:
```sh
kubectl get hpa
```

Simulate Load:

// The load generator simulates traffic to trigger scaling.

// --image=busybox:Specifies the container image to use for the pod. Here, busybox is a lightweight image that includes a minimal set of Unix utilities. It’s commonly used for debugging or running simple commands in Kubernetes.

// --rm:Automatically deletes the pod when the command or session exits. This ensures the pod is temporary and doesn’t persist in the cluster after you’re done, cleaning up resources.

```sh
kubectl run load-generator --image=busybox --rm -it -- sh
```

Inside the pod, run:

// wget:A command-line tool to retrieve content from web servers via HTTP, HTTPS, or FTP. In this case, it’s used to make HTTP requests.
```sh
while true; do wget -q -O- http://nginx-nodeport; done
```

Monitor Scaling:

// The metrics server (enabled earlier) provides CPU usage data.

// -w (or --watch):Enables "watch" mode, which keeps the command running and updates the output in real-time whenever changes occur to the pods in the specified namespace.

```sh
kubectl get pods -w
kubectl get hpa
```

## Advanced: Manage Configuration with ConfigMap and Secrets
ConfigMaps and Secrets manage configuration and sensitive data.

Example: Use a ConfigMap for Nginx configuration and a Secret for a password.

Create a ConfigMap:
```sh
nano configmap.yaml
```

Add the following:

// This ConfigMap, named nginx-config, stores an Nginx configuration file (nginx.conf) that can be mounted into a pod (nginx-deployment pods) or used as environment variables.

// It allows you to decouple the Nginx configuration from the pod’s container image, making it easier to update the configuration without rebuilding the image.

```sh
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: | // The key nginx.conf represents the name of the configuration file or data entry. The | symbol indicates a multi-line string (literal block scalar) in YAML, preserving line breaks in the value.
    server {
      listen 80;
      server_name example.com; // Specifies the domain name (example.com) for this server block. Requests to this domain will be handled by this configuration.
      root /usr/share/nginx/html; // Sets the document root directory to /usr/share/nginx/html, where Nginx will serve files (e.g., index.html) from.
}
```

Create a Secret:
```sh
nano secret.yaml
```

Add the following:
```sh
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  password: cGFzc3dvcmQ= # base64 encoded "password"
```

Create a Deployment with ConfigMap and Secret:
```sh
nano deployment-with-config.yaml
```

Add the following:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-config-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-config
  template:
    metadata:
      labels:
        app: nginx-config
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        env:
        - name: APP_PASSWORD
          valueFrom:
            secretKeyRef: // Secret stores a base64-encoded password, accessible as an environment variable.
              name: nginx-secret
              key: password
        volumeMounts:
        - name: config
          mountPath: "/etc/nginx/conf.d" // ConfigMap provides an Nginx configuration file mounted at /etc/nginx/conf.d.
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

Apply Resources:
```sh
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment-with-config.yaml
```

Verify:
```sh
kubectl get configmaps
kubectl get secrets
kubectl get pods
```

## Advanced: Rolling Updates and Rollbacks
Deployments support rolling updates for zero-downtime updates and rollbacks to revert changes.

Example: Update the Nginx image and rollback if needed.

Update the Deployment:

// --record saves the command in the rollout history.

```sh
kubectl set image deployment/nginx-deployment nginx=nginx:1.21 --record
```

Check Rollout Status:
```sh
kubectl rollout status deployment/nginx-deployment
```

View Rollout History:
```sh
kubectl rollout history deployment/nginx-deployment
```

Simulate a Bad Update:

// A bad update (nginx:bad) causes Pods to fail.

```sh
kubectl set image deployment/nginx-deployment nginx=nginx:bad --record
kubectl rollout status deployment/nginx-deployment
```

Rollback to Previous Version:

// rollout undo reverts to the previous working version.

```sh
kubectl rollout undo deployment/nginx-deployment
kubectl get pods
```















