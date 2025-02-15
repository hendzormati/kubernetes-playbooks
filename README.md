# kubernetes-playbooks

Ansible playbooks that create a single-master Kubernetes 1.30 cluster on Ubuntu 24.04 LTS nodes.

## Prerequisites

- Ansible and Python3 installed on the local machine
- SSH access to your Ubuntu 24.04 nodes
- The nodes should be provisioned (either through OpenStack, other cloud providers, or on-premise)

## Inventory Setup

The playbooks use an inventory file (`hosts.ini`) to define the master and worker nodes:

```ini
[master]
master1 ansible_host=192.168.1.159

[workers]
worker1 ansible_host=192.168.1.203
worker2 ansible_host=192.168.1.171

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

## Troubleshooting

If you can't access the Nginx service:
1. Verify your security group/firewall allows traffic to port 30080
2. Check pod status: `kubectl get pods`
3. Check service status: `kubectl get svc`
4. Check pod logs: `kubectl logs -l app=nginx`

## Credits

- Based on [kubernetes-playbooks](https://github.com/torgeirl/kubernetes-playbooks) by torgeirl, adapted for Ubuntu 24.04 LTS
- Modified for single master deployment with external access testing

## License

See the LICENSE file for license rights and limitations (MIT).
