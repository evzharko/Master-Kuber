# Встановлення та конфігурування Kub

## 1. Створюємо необіхдін директорії

```bash
   sudo mkdir -p ./kubebuilder/bin
   sudo mkdir -p /etc/cni/net.d
   sudo mkdir -p /var/lib/kubelet
   sudo mkdir -p /var/lib/kubelet/pki
   sudo mkdir -p /etc/kubernetes/manifests
   sudo mkdir -p /var/log/kubernetes
   sudo mkdir -p /etc/containerd/
   sudo mkdir -p /run/containerd
   sudo mkdir -p /opt/cni
```

## 2. Качаємо компоненти ядра

```bash
# Download kubebuilder tools (includes etcd, kubectl, etc)

curl -L https://storage.googleapis.com/kubebuilder-tools/kubebuilder-tools-1.30.0-linux-amd64.tar.gz -o /tmp/kubebuilder-tools.tar.gz
sudo tar -C ./kubebuilder --strip-components=1 -zxf /tmp/kubebuilder-tools.tar.gz
rm /tmp/kubebuilder-tools.tar.gz
sudo chmod -R 755 ./kubebuilder/bin

# Download kubelet
sudo curl -L "https://dl.k8s.io/v1.30.0/bin/linux/amd64/kubelet" -o kubebuilder/bin/kubelet
sudo chmod 755 kubebuilder/bin/kubelet
```

## 3. Встановлюємо containerd
```bash
# Download and install containerd
wget https://github.com/containerd/containerd/releases/download/v2.0.5/containerd-static-2.0.5-linux-amd64.tar.gz -O /tmp/containerd.tar.gz
sudo tar zxf /tmp/containerd.tar.gz -C /opt/cni/
rm /tmp/containerd.tar.gz

# Install runc
sudo curl -L "https://github.com/opencontainers/runc/releases/download/v1.2.6/runc.amd64" -o /opt/cni/bin/runc
sudo chmod +x /opt/cni/bin/runc

# Install CNI plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz -O /tmp/cni-plugins.tgz
sudo tar zxf /tmp/cni-plugins.tgz -C /opt/cni/bin/
rm /tmp/cni-plugins.tgz
```

## 4. Встановлюємо додаткові компоненти, та виcтавляємо права
```bash
# Download controller manager and scheduler
sudo curl -L "https://dl.k8s.io/v1.30.0/bin/linux/amd64/kube-controller-manager" -o kubebuilder/bin/kube-controller-manager
sudo curl -L "https://dl.k8s.io/v1.30.0/bin/linux/amd64/kube-scheduler" -o kubebuilder/bin/kube-scheduler
sudo curl -L "https://dl.k8s.io/v1.30.0/bin/linux/amd64/cloud-controller-manager" -o kubebuilder/bin/cloud-controller-manager

# Set permissions
sudo chmod 755 kubebuilder/bin/kube-controller-manager
sudo chmod 755 kubebuilder/bin/kube-scheduler
sudo chmod 755 kubebuilder/bin/cloud-controller-manager
```

## 5. Генеруємо сертифікати та токени
```bash
# Generate service account key pair
openssl genrsa -out /tmp/sa.key 2048
openssl rsa -in /tmp/sa.key -pubout -out /tmp/sa.pub

# Generate token file
TOKEN="1234567890"
echo "${TOKEN},admin,admin,system:masters" > /tmp/token.csv

# Generate CA certificate
openssl genrsa -out /tmp/ca.key 2048
openssl req -x509 -new -nodes -key /tmp/ca.key -subj "/CN=kubelet-ca" -days 365 -out /tmp/ca.crt
sudo cp /tmp/ca.crt /var/lib/kubelet/ca.crt
sudo cp /tmp/ca.crt /var/lib/kubelet/pki/ca.crt
```

## 6. Конфігурім kubectl
Опціонально (echo "alias k=kubectl" >> ~/.zshrc && source ~/.zshrc)
```bash
   sudo kubebuilder/bin/kubectl config set-credentials test-user --token=1234567890
   sudo kubebuilder/bin/kubectl config set-cluster test-env --server=https://127.0.0.1:6443 --insecure-skip-tls-verify
   sudo kubebuilder/bin/kubectl config set-context test-context --cluster=test-env --user=test-user --namespace=default
   sudo kubebuilder/bin/kubectl config use-context test-context
```

## 7. Конфігурим CNI файл. Описуємо як саме піднімати мережу для Pod’ів.
```bash
   cat <<EOF | sudo tee /etc/cni/net.d/10-mynet.conf
{
    "cniVersion": "0.3.1",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
EOF
```

## 8. Конфігуремо containerd
```bash
   cat <<EOF | sudo tee /etc/containerd/config.toml
   version = 3

[grpc]
address = "/run/containerd/containerd.sock"

[plugins.'io.containerd.cri.v1.runtime']
enable_selinux = false
enable_unprivileged_ports = true
enable_unprivileged_icmp = true
device_ownership_from_security_context = false

[plugins.'io.containerd.cri.v1.images']
snapshotter = "native"
disable_snapshot_annotations = true

[plugins.'io.containerd.cri.v1.runtime'.cni]
bin_dir = "/opt/cni/bin"
conf_dir = "/etc/cni/net.d"

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
runtime_type = "io.containerd.runc.v2"

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
SystemdCgroup = false
EOF
```

## 9. Конфігуремо kubelet
```bash
   cat << EOF | sudo tee /var/lib/kubelet/config.yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   authentication:
   anonymous:
   enabled: true
   webhook:
   enabled: true
   x509:
   clientCAFile: "/var/lib/kubelet/ca.crt"
   authorization:
   mode: AlwaysAllow
   clusterDomain: "cluster.local"
   clusterDNS:
     - "10.0.0.10"
  resolvConf: "/etc/resolv.conf"
  runtimeRequestTimeout: "15m"
  failSwapOn: false
  seccompDefault: true
  serverTLSBootstrap: false
  containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
  staticPodPath: "/etc/kubernetes/manifests"
EOF
```

## 10. Стартуємо компоненти
```bash
# Get the host IP:
HOST_IP=$(hostname -I | awk '{print $1}')

# Start etcd:
sudo kubebuilder/bin/etcd \
--advertise-client-urls http://$HOST_IP:2379 \
--listen-client-urls http://0.0.0.0:2379 \
--data-dir ./etcd \
--listen-peer-urls http://0.0.0.0:2380 \
--initial-cluster default=http://$HOST_IP:2380 \
--initial-advertise-peer-urls http://$HOST_IP:2380 \
--initial-cluster-state new \
--initial-cluster-token test-token &

# Start kube-apiserver:
sudo kubebuilder/bin/kube-apiserver \
--etcd-servers=http://$HOST_IP:2379 \
--service-cluster-ip-range=10.0.0.0/24 \
--bind-address=0.0.0.0 \
--secure-port=6443 \
--advertise-address=$HOST_IP \
--authorization-mode=AlwaysAllow \
--token-auth-file=/tmp/token.csv \
--enable-priority-and-fairness=false \
--allow-privileged=true \
--profiling=false \
--storage-backend=etcd3 \
--storage-media-type=application/json \
--v=0 \
--service-account-issuer=https://kubernetes.default.svc.cluster.local \
--service-account-key-file=/tmp/sa.pub \
--service-account-signing-key-file=/tmp/sa.key &

# Start containerd:
export PATH=$PATH:/opt/cni/bin:kubebuilder/bin
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin /opt/cni/bin/containerd -c /etc/containerd/config.toml &

# Start kube-scheduler:
sudo kubebuilder/bin/kube-scheduler \
--kubeconfig=/root/.kube/config \
--leader-elect=false \
--v=2 \
--bind-address=0.0.0.0 &

# Prepare for kubelet:
# Copy kubeconfig
sudo cp /root/.kube/config /var/lib/kubelet/kubeconfig
export KUBECONFIG=~/.kube/config
cp /tmp/sa.pub /tmp/ca.crt

# Create service account and configmap
sudo kubebuilder/bin/kubectl create sa default
sudo kubebuilder/bin/kubectl create configmap kube-root-ca.crt --from-file=ca.crt=/tmp/ca.crt -n default

# Start kubelet:
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin kubebuilder/bin/kubelet \
--kubeconfig=/var/lib/kubelet/kubeconfig \
--config=/var/lib/kubelet/config.yaml \
--root-dir=/var/lib/kubelet \
--cert-dir=/var/lib/kubelet/pki \
--hostname-override=$(hostname) \
--pod-infra-container-image=registry.k8s.io/pause:3.10 \
--node-ip=$HOST_IP \
--cgroup-driver=cgroupfs \
--max-pods=4  \
--v=1 &

# Label the node:
NODE_NAME=$(hostname)
sudo kubebuilder/bin/kubectl label node "$NODE_NAME" node-role.kubernetes.io/master="" --overwrite

# Start kube-controller-manager:
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin kubebuilder/bin/kube-controller-manager \
--kubeconfig=/var/lib/kubelet/kubeconfig \
--leader-elect=false \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-name=kubernetes \
--root-ca-file=/var/lib/kubelet/ca.crt \
--service-account-private-key-file=/tmp/sa.key \
--use-service-account-credentials=true \
--v=2 &
```

## 11. Перевірка роботи служб
```bash
# Check node status
sudo kubebuilder/bin/kubectl get nodes
# Check component status
sudo kubebuilder/bin/kubectl get componentstatuses
# Check API server health
sudo kubebuilder/bin/kubectl get --raw='/readyz?verbose'
# Create Deployment
sudo  kubebuilder/bin/kubectl create deploy demo-nginx --image nginx
# Check all resources
sudo kubebuilder/bin/kubectl get all -A
# Debug pods
sudo kubebuilder/bin/kubectl describe pod demo-677cfb9d49-td8qx
```
## Приклад помилки яз за якою не запускався pod.
```bash
Warning  FailedScheduling  2m16s  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node.cloudprovider.kubernetes.io/uninitialized: true}. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

## Запуск середовища за допомгою скрипта
```bash
sudo bash ./init.sh start
```