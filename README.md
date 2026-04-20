# Kubernetes NFS Shared-Storage Application

## Core Logic: The Shared Storage Pattern

The fundamental logic of this architecture is the **Shared Volume Pattern**. In Kubernetes, standard block storage usually follows the ReadWriteOnce (RWO) constraint, meaning only one Pod can mount the volume at a time. By implementing an NFS (Network File System) provisioner, we unlock the ReadWriteMany (RWX) access mode. This allows a decoupled architecture where a producer (the Job) and a consumer (the Deployment) interact with the same data lifecycle simultaneously.

## Prerequisites

- Minikube (Running with Docker driver)
- Helm v3+
- kubectl

## Step-by-Step Execution Commands

### 1. Initialize Cluster
Bypass hardware virtualization constraints by using the Docker driver:

```bash
minikube start --driver=docker
```

### 2. Deploy Infrastructure (Helm)
Add the repository and install the NFS Ganesha Provisioner. This creates the "Storage Factory" that dynamically fulfills volume requests:

```bash
helm repo add nfs-test https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/
helm install nfs-server nfs-test/nfs-server-provisioner \
  --set storageClass.name=nfs-storage \
  --set storageClass.defaultClass=false
```

### 3. Deploy Application Manifests
Apply the single declarative file containing the PVC, Job, Deployment, and Service:

```bash
kubectl apply -f assignment.yaml
```

### 4. Verification & Access
Wait for the Job to reach Completed status, then expose the service:

```bash
kubectl get jobs --watch
minikube service web-service
```

## Application Architecture Report

### Short Description
The application executes a **Provision-Inject-Serve** lifecycle:

1. **Provisioning**: The PersistentVolumeClaim (PVC) requests storage from the nfs-storage class. The NFS Pod dynamically creates a folder on its local disk to satisfy this request.

2. **Injection**: The web-content-job (using a busybox image) mounts the shared volume and writes a custom index.html.

3. **Serving**: The web-server-deploy (using an nginx image) mounts the exact same volume at its web root (`/usr/share/nginx/html`), allowing it to serve the data injected by the Job.

### Component Roles

| Component | Role | Mental Trigger |
|-----------|------|----------------|
| NFS Provisioner (Pod) | The Dynamic Factory | Watches the API for PVCs and creates physical folders on demand. |
| StorageClass (nfs-storage) | The Matchmaker | Bridges the gap between the PVC request and the NFS Provisioner. |
| PVC (nfs-pvc) | The Storage Voucher | Claims 1Gi of space with ReadWriteMany access. |
| Job (web-content-job) | The Data Producer | A one-time worker that seeds the shared volume with data. |
| Deployment (web-server) | The Data Consumer | A persistent server that renders the shared volume content. |
| Service (NodePort) | The Gateway | Provides a stable entry point to access the web server from the host. |