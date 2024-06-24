### Use Case: Setting Up a Multi-Node Kubernetes Development Cluster on a Single VM

**Scenario**: You are tasked with setting up a Kubernetes development cluster with multiple worker nodes on a single VM to deploy and manage a web application.

### Architecture Overview

**Cluster Setup**:
- **Nodes**: A single VM will simulate multiple nodes.
- **Components**: Deploy Kubernetes control plane components (API server, controller manager, scheduler) and worker components (kubelet, kube-proxy).

### Prerequisites

- **Operating System**: RHEL 8.2
- **VM Configuration**: Ensure the VM has sufficient CPU and memory to simulate multiple nodes.
- **Docker**: Installed and running.
- **Kubeadm, Kubelet, Kubectl**: Installed on the VM.

### Kubernetes Components and Flow

#### Step 1: Install Docker

Ensure Docker is installed and running.

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

- **Explanation**:
  - `yum install -y yum-utils`: Installs yum-utils, a collection of utilities for yum package management.
  - `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`: Adds the Docker CE repository to your system.
  - `yum install -y docker-ce docker-ce-cli containerd.io`: Installs Docker Engine, CLI, and containerd.
  - `systemctl start docker`: Starts the Docker service.
  - `systemctl enable docker`: Enables Docker to start on boot.

#### Step 2: Initialize the Kubernetes Control Plane

Initialize the control plane with custom API server advertise address and port.

**Example Command**:
```bash
sudo kubeadm init --apiserver-advertise-address=192.168.1.100 --apiserver-bind-port=6443 --pod-network-cidr=192.168.0.0/16
```

- **Explanation**:
  - `kubeadm init`: Initializes the Kubernetes control plane.
  - `--apiserver-advertise-address=<VM_IP>`: Specifies the IP address that the API server will advertise to the nodes. Replace `<VM_IP>` with your VM's IP address (e.g., `192.168.1.100`).
  - `--apiserver-bind-port=<custom_port>`: Specifies the port for the API server to listen on (e.g., `6443`).
  - `--pod-network-cidr=192.168.0.0/16`: Specifies the CIDR for the Pod network. The value `192.168.0.0/16` is a common private network range for Pods.

**Purpose**:
- **Source**: The VM where you are setting up the Kubernetes cluster.
- **Action**: Initializes the control plane components.
- **Destination**: Configures the API server, controller manager, and scheduler.

Configure `kubectl` for the root user.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- **Explanation**:
  - `mkdir -p $HOME/.kube`: Creates the `.kube` directory in the user's home directory if it doesn't already exist.
  - `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`: Copies the admin configuration file to the user's `.kube` directory.
  - `sudo chown $(id -u):$(id -g) $HOME/.kube/config`: Changes the ownership of the configuration file to the current user. The `id -u` and `id -g` commands get the user and group IDs of the current user.

**Purpose**:
- **Source**: Copies the configuration from the Kubernetes admin configuration file.
- **Action**: Sets up the `kubectl` command-line tool.
- **Destination**: User's `.kube/config` directory for accessing the cluster.

#### Step 3: Install a Network Plugin

Deploy a network plugin such as Calico.

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

- **Explanation**:
  - `kubectl apply -f`: Applies the configuration from the specified file or URL.
  - `https://docs.projectcalico.org/manifests/calico.yaml`: URL for the Calico network plugin manifest.

**Purpose**:
- **Source**: Downloads the Calico manifest from the provided URL.
- **Action**: Configures the networking for the cluster.
- **Destination**: Deploys the Calico network plugin in the cluster.

#### Step 4: Create Worker Nodes (Simulated in Single VM)

Simulate multiple worker nodes by creating Docker containers to act as nodes.

```bash
# Create a docker network
docker network create --driver=bridge kube-network

# Create worker node containers
docker run -d --privileged --name kube-node1 --network kube-network --hostname kube-node1 --add-host=kube-node1:<VM_IP> docker:dind
docker run -d --privileged --name kube-node2 --network kube-network --hostname kube-node2 --add-host=kube-node2:<VM_IP> docker:dind
```

- **Explanation**:
  - `docker network create --driver=bridge kube-network`: Creates a Docker network named `kube-network` with the bridge driver.
  - `docker run -d --privileged --name kube-node1 --network kube-network --hostname kube-node1 --add-host=kube-node1:<VM_IP> docker:dind`: Runs a Docker container named `kube-node1` in detached mode with the Docker-in-Docker (DinD) image, simulating a worker node.
  - `--privileged`: Gives extended privileges to the container.
  - `--network=kube-network`: Connects the container to the `kube-network`.
  - `--hostname=kube-node1`: Sets the hostname of the container to `kube-node1`.
  - `--add-host=kube-node1:<VM_IP>`: Adds a custom host-to-IP mapping inside the container.

**Purpose**:
- **Source**: Uses the Docker-in-Docker image.
- **Action**: Creates Docker containers to simulate worker nodes.
- **Destination**: The `kube-network` Docker network.

#### Step 5: Join Worker Nodes to the Cluster

Retrieve the join command from the output of the `kubeadm init` command or generate a new one.

```bash
kubeadm token create --print-join-command
```

- **Explanation**:
  - `kubeadm token create --print-join-command`: Creates a token and prints the command to join a node to the cluster.

**Purpose**:
- **Source**: Kubernetes control plane.
- **Action**: Generates a join command for worker nodes.
- **Destination**: Output to be used for joining nodes.

Inside each Docker container (worker node):

```bash
docker exec -it kube-node1 /bin/sh
# Install kubeadm, kubelet, kubectl inside the container
apk add --no-cache curl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubeadm
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubelet
chmod +x kubectl kubeadm kubelet
mv kubectl kubeadm kubelet /usr/local/bin/

# Join the cluster
kubeadm join <VM_IP>:<custom_port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

- **Explanation**:
  - `docker exec -it kube-node1 /bin/sh`: Opens an interactive shell inside the `kube-node1` container.
  - `apk add --no-cache curl`: Installs curl inside the container (assuming Alpine Linux).
  - `curl -LO`: Downloads the specified file.
  - `chmod +x`: Makes the downloaded files executable.
  - `mv kubectl kubeadm kubelet /usr/local/bin/`: Moves the files to `/usr/local/bin` for easier execution.
  - `kubeadm join <VM_IP>:<custom_port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>`: Joins the node to the cluster using the provided token and CA certificate hash.

Repeat for `kube-node2`.

**Purpose**:
- **Source**: The Docker containers simulating worker nodes.
- **Action**: Joins the simulated worker nodes to the cluster.
- **Destination**: The Kubernetes control plane.

### Verification

Check the status of the nodes and Pods.

```bash
kubectl get nodes
kubectl get pods -A
```

- **Explanation**:
  - `kubectl get nodes`: Lists all the nodes in the cluster.
  - `kubectl get pods -A`: Lists all the Pods across all namespaces.

**Purpose**:


- **Source**: Kubernetes API server.
- **Action**: Retrieves the status of nodes and Pods.
- **Destination**: Output to the user.

### Uninstallation

To uninstall the Kubernetes components:

1. **Reset the Cluster**:
   ```bash
   sudo kubeadm reset
   ```

   - **Explanation**:
     - `kubeadm reset`: Resets the Kubernetes cluster, removing all configurations and state.

2. **Remove Kubernetes Packages**:
   ```bash
   sudo yum remove -y kubelet kubeadm kubectl
   sudo rm -rf /etc/kubernetes/
   sudo rm -rf /var/lib/kubelet/
   ```

   - **Explanation**:
     - `yum remove -y kubelet kubeadm kubectl`: Removes the Kubernetes packages.
     - `rm -rf /etc/kubernetes/`: Deletes the Kubernetes configuration directory.
     - `rm -rf /var/lib/kubelet/`: Deletes the `kubelet` state directory.

3. **Remove Docker Containers**:
   ```bash
   docker rm -f kube-node1 kube-node2
   ```

   - **Explanation**:
     - `docker rm -f`: Forces the removal of the specified Docker containers.

### Notes

1. **kubeadm** initializes the Kubernetes control plane on the specified node.
2. **Custom Ports**: Ensure custom ports do not conflict with other services and are open in firewall settings.

### Minikube Installation and Verification

Install Minikube:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
```

- **Explanation**:
  - `curl -Lo minikube`: Downloads the latest Minikube binary.
  - `chmod +x minikube`: Makes the Minikube binary executable.
  - `sudo mv minikube /usr/local/bin/`: Moves the Minikube binary to `/usr/local/bin`.

Start Minikube:

```bash
minikube start
```

- **Explanation**:
  - `minikube start`: Starts a local Kubernetes cluster using Minikube.

Get Cluster Details:

```bash
kubectl config view
kubectl cluster-info
```

- **Explanation**:
  - `kubectl config view`: Displays the kubeconfig configuration.
  - `kubectl cluster-info`: Displays the cluster information.
