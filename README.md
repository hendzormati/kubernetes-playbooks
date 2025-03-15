# kubernetes-playbooks

Ansible playbooks that automate Kubernetes deployment on Ubuntu 24.04 LTS.
## Repository Traffic Overview

Here's the traffic overview for this repository:

- 👁️ **Total Views** Since Creation: **0** views
- 🔄 **Total Clones** Since Creation: **10** clones
- 📈 **Recent Views** (Last 14 days): **0** views
- 📊 **Recent Clones** (Last 14 days): **5** clones

---

Last traffic data update: **Sat Mar 15 2025 14:49:16 CET**

---
## 📂 Playbook Structure

- **Single-Master Deployment** (Basic setup)
  - Playbooks are in the root directory.
  - Deploys a single control-plane node.
- **Multi-Master Deployment (HA)** (Advanced setup)
  - Playbooks are inside `playbooks/multi-master/`
  - Uses HAProxy and multiple control-plane nodes.

---

## **🔹 Prerequisites**
- Ansible and Python3 installed on the local machine
- SSH access to your Ubuntu 24.04 nodes
- The nodes should be provisioned (OpenStack, other cloud providers, or on-premise)
- Unique **hostnames** for each node (`master-1`, `master-2`, etc.)

---

## **🔹 Single-Master Deployment**  

### **📝 Inventory Setup**
The `hosts.ini` file defines the master and worker nodes:

```ini
[master]
master1 ansible_host=192.168.1.159

[workers]
worker1 ansible_host=192.168.1.203
worker2 ansible_host=192.168.1.171

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=/path/to/your/key.pem
ansible_user=ubuntu
```

### **🚀 Deploy the Cluster**
1. Install dependencies:
   ```bash
   ansible-playbook -i hosts.ini dependencies.yaml
   ```
2. Initialize master node:
   ```bash
   ansible-playbook -i hosts.ini master.yaml
   ```
3. Join worker nodes:
   ```bash
   ansible-playbook -i hosts.ini worker.yaml
   ```
4. Verify cluster:
   ```bash
   ssh -i /path/to/your/key.pem ubuntu@<master_ip>
   kubectl get nodes
   ```

---

## **🔹 Multi-Master HA Deployment**  

📁 **Playbooks are under `playbooks/multi-master/`**  

### **📝 Inventory Setup for Multi-Master**
The `multi-hosts.ini` file defines the **multi-master** and **HAProxy** setup:

```ini
[master]
master1 ansible_host=192.168.1.203
master2 ansible_host=192.168.1.249
master3 ansible_host=192.168.1.198
[workers]
worker1 ansible_host=192.168.1.153
worker2 ansible_host=192.168.1.239
[haproxy]
server ansible_host=192.168.1.212
[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_extra_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
ansible_ssh_private_key_file=/path/to/your/key.pem
ansible_user=ubuntu
```

## Deploying the Cluster

1. Install Kubernetes dependencies on all nodes:
```bash
ansible-playbook -i hosts.ini dependencies.yml
```
This playbook:
- Disables swap
- Loads necessary kernel modules
- Configures system parameters
- Installs containerd runtime
- Installs Kubernetes components (kubelet, kubeadm)

2. Initialize the master node:
```bash
ansible-playbook -i hosts.ini master.yml
```
This playbook:
- Initializes the Kubernetes control plane
- Sets up pod networking (Flannel)
- Configures kubeconfig

3. Join worker nodes:
```bash
ansible-playbook -i hosts.ini worker.yml
```
This playbook:
- Retrieves the join command from the master
- Joins worker nodes to the cluster

## Verify the Cluster

SSH into the master node and check the cluster status:

```bash
ssh -i /path/to/your/key.pem ubuntu@<master_ip>
kubectl get nodes
# Expected output:
NAME                STATUS   ROLES           AGE   VERSION
master-controller   Ready    control-plane   10d   v1.30.9
worker-controller   Ready    <none>          10d   v1.30.9
```

All nodes should show "Ready" status.

## Testing External Access with Nginx

To test external access to your cluster, deploy a sample Nginx application:

1. Create a file named `nginx-test.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  externalIPs: []
```

2. Apply the configuration:
```bash
kubectl apply -f nginx-test.yaml
# Expected output:
deployment.apps/nginx-deployment created
service/nginx-service created
```

3. Verify the deployment:
```bash
kubectl get deployments
# Expected output:
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           6s

kubectl get pods
# Expected output:
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-576c6b7b6-88c2r   1/1     Running   0          11s
nginx-deployment-576c6b7b6-jwndk   1/1     Running   0          11s

kubectl get services
# Expected output:
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        43s
nginx-service   NodePort    10.111.243.240   <none>        80:30080/TCP   14s
```

4. Access Nginx:
- Through NodePort: `http://<any-node-ip>:30080`
- You should see the default Nginx welcome page

To verify everything is working:
```bash
# Check pods are running
kubectl get pods -o wide

# Check service details
kubectl describe service nginx-service

# Check logs of a pod (replace <pod-name> with actual pod name)
kubectl logs <pod-name>
```

---

## **🛠 Troubleshooting**
- **Multi-master not joining?**
  - Check etcd members:  
    ```bash
    ETCDCTL_API=3 etcdctl member list --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key
    ```
  - Remove unstarted members:
    ```bash
    ETCDCTL_API=3 etcdctl member remove <MEMBER_ID>
    ```
- **HAProxy not balancing?**
  - Check logs:  
    ```bash
    sudo journalctl -u haproxy --no-pager
    ```
  - Ensure all masters are **reachable** via port `6443`.

---

## **📜 Credits**
- Based on [kubernetes-playbooks](https://github.com/torgeirl/kubernetes-playbooks) by torgeirl, adapted for Ubuntu 24.04 LTS
---

## **📜 License**
See the LICENSE file for license rights and limitations (MIT).
