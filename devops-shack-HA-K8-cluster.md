# HA K8 Cluster Implementation

To set up a highly available Kubernetes cluster with two master nodes and three worker nodes without using a cloud load balancer, you can use a virtual machine to act as a load balancer for the API server. Here are the detailed steps for setting up such a cluster:

### Prerequisites
- 3 master nodes
- 3 worker nodes
- 1 load balancer node (Master HAProxy)
- 1 load balancer node (Backup HAProxy) Optional
- All nodes should be running a Linux distribution like Ubuntu 

### Step 1: Prepare the Load Balancer Node
1. **Install HAProxy:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y haproxy
   ```

2. **Configure HAProxy:**
   Edit the HAProxy configuration file (`/etc/haproxy/haproxy.cfg`):
   ```bash
   sudo nano /etc/haproxy/haproxy.cfg
   ```

   Add the following configuration:
   ```haproxy
   frontend kubernetes-frontend
       bind *:6443
       option tcplog
       mode tcp
       default_backend kubernetes-backend

   backend kubernetes-backend
       mode tcp
       balance roundrobin
       option tcp-check
       server master1 <MASTER1_IP>:6443 check
       server master2 <MASTER2_IP>:6443 check
       server master3 <MASTER2_IP>:6443 check
   ```

3. **Restart HAProxy:**
   ```bash
   sudo systemctl restart haproxy
   ```

4. **Install Keepalived(Optional: to provide high availability to HAProxy):**
   ```bash
   sudo apt update
   sudo apt install keepalived -y
   ```
5. **Configure Keepalived:**

   **Primary Server**
   vi /etc/keepalived/keepalived.conf

   ```bash
   vrrp_instance VI_1 {
    state MASTER
    interface eth0  # network interface
    virtual_router_id 51
    priority 100  # Higher priority for MASTER
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 12345  
    }

    virtual_ipaddress {
        192.168.10.10  
    }
    }
   ```

**Backend Server**
   vi /etc/keepalived/keepalived.conf

   ```bash
   vrrp_instance VI_1 {
    state MASTER
    interface eth0  # network interface
    virtual_router_id 51
    priority 90  
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 12345  
    }

    virtual_ipaddress {
        192.168.10.10  
    }
}

   ```

Restart Keepalived and verify VIP Configuration (on both servers)

```bash
sudo systemctl restart keepalived
sudo ip address
```

Stop Keepalive on master and then see if VIP has been assigned to the backup node
```bash
sudo systemctl stop keepalived
sudo ip address
```


### Step 2: Prepare All Nodes (Masters and Workers)

**Prerequisites:**
- Required ports should be opened as mentioned in this linke (https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- Unique hostname for each machine
- disable swap, sudo swapoff -a (temporarily), To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab, systemd.swap, depending how it was configured on your system.
- All the nodes should be reachable to each other


1. **Install Docker, kubeadm, kubelet, and kubectl:**
```bash
sudo apt-get update
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable net.bridge.bridge-nf-call-iptables
sudo sysctl net.bridge.bridge-nf-call-iptables=1

# Download and install cri-dockerd
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd-0.3.14.amd64.tgz
tar -xvf cri-dockerd-0.3.14.amd64.tgz
cd cri-dockerd
sudo install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd

# Download and set up cri-dockerd systemd service
cd ..
wget https://github.com/Mirantis/cri-dockerd/archive/refs/tags/v0.3.14.tar.gz
tar -xvf v0.3.14.tar.gz
cd cri-dockerd-0.3.14/
sudo cp packaging/systemd/* /etc/systemd/system
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

# Enable and start cri-docker service
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket
sudo systemctl enable cri-docker
sudo systemctl start cri-docker
sudo systemctl status cri-docker
```

### Step 3: Initialize the First Master Node
1. **Initialize the first master node:**
   ```bash
   sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_IP:6443" --upload-certs --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock
   ```

2. **Set up kubeconfig for the first master node:**
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. **Install Calico network plugin:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
   ```

4. **Install Ingress-NGINX Controller:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
   ```

### Step 4: Join the Second & third Master Node
1. **Get the join command and certificate key from the first master node:**
   ```bash
   kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
   ```

2. **Run the join command as a root user on the second & third master node:**
   ```bash
   sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <certificate-key> --cri-socket unix:///var/run/cri-dockerd.sock
   ```

3. **Set up kubeconfig for the second & third master node as a non-root user:**
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

### Step 5: Join the Worker Nodes
1. **Get the join command from the first master node:**
   ```bash
   kubeadm token create --print-join-command
   ```

2. **Run the join command as a root user on each worker node:**
   ```bash
   sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket unix:///var/run/cri-dockerd.sock
   ```

### Step 6: Verify the Cluster
1. **Check the status of all nodes:**
   ```bash
   kubectl get nodes
   ```

2. **Check the status of all pods:**
   ```bash
   kubectl get pods --all-namespaces
   ```

By following these steps, you will have a highly available Kubernetes cluster with two master nodes and three worker nodes, and a load balancer distributing traffic between the master nodes. This setup ensures that if one master node fails, the other will continue to serve the API requests.



# Verification

### Step 1: Install etcdctl
1. **Install etcdctl using apt:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y etcd-client
   ```

### Step 2: Verify Etcd Cluster Health
1. **Check the health of the etcd cluster:**
   ```bash
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key endpoint health
   ```

2. **Check the cluster membership:**
   ```bash
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key member list
   ```
3. **Check the leader master node:**
   ```bash
   sudo ETCDCTL_API=3 etcdctl \
    --endpoints=127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint status --write-out=table
   ```
### Step 3: Verify HAProxy Configuration and Functionality
1. **Configure HAProxy Stats:**
   - Add the stats configuration to `/etc/haproxy/haproxy.cfg`:
     ```haproxy
     listen stats
         bind *:8404
         mode http
         stats enable
         stats uri /
         stats refresh 10s
         stats admin if LOCALHOST
     ```

2. **Restart HAProxy:**
   ```bash
   sudo systemctl restart haproxy
   ```

3. **Check HAProxy Stats:**
   - Access the stats page at `http://<LOAD_BALANCER_IP>:8404`.

### Step 4: Test High Availability
1. **Simulate Master Node Failure:**
   - Stop the kubelet service and Docker containers on one of the master nodes to simulate a failure:
     ```bash
     sudo systemctl stop kubelet
     sudo docker stop $(sudo docker ps -q)
     ```

2. **Verify Cluster Functionality:**
   - Check the status of the cluster from a worker node or the remaining master node:
     ```bash
     kubectl get nodes
     kubectl get pods --all-namespaces
     ```

   - The cluster should still show the remaining nodes as Ready, and the Kubernetes API should be accessible.

3. **HAProxy Routing:**
   - Ensure that HAProxy is routing traffic to the remaining master node. Check the stats page or use curl to test:
     ```bash
     curl -k https://<LOAD_BALANCER_IP>:6443/version
     ```

### Summary
By installing `etcdctl` and using it to check the health and membership of the etcd cluster, you can ensure that your HA setup is working correctly. Additionally, configuring HAProxy to route traffic properly and simulating master node failures will help verify the resilience and high availability of your Kubernetes cluster.
