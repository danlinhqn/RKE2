# High Available RKE2


### Assumptions

| Host | IP | Notes|
|:----|:----:|:-----|
|master1|172.16.34.16|etcd|
|master2|172.16.34.11|etcd|
|worker1|172.16.34.6||


If you do not have a DNS server available/configured, the `/etc/hosts` file on each node will need to include the following.

```
172.16.34.16 master1 master1.demo.local
172.16.34.11 master2 master2.demo.local
172.16.34.6 worker01 worker01.demo.local
```

## Install

### RKE2 installation

#### On master1, excute the following commands

``` shell
master1$ mkdir -p /etc/rancher/rke2/

# Create tls-san for kubernetes
master1$ cat > /etc/rancher/rke2/config.yaml << HERE
tls-san:
- master1
- master2
- master1.demo.local
- master2.demo.local
HERE

# Setup environment
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
alias k=kubectl


# Run bootstrap to install rke2
master1$ export https_proxy=http://172.16.31.201:3128
master1$ export http_proxy=http://172.16.31.201:3128
master1$ export no_proxy=127.0.0.1,localhost,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local

# OR
master1$ source ~/proxy

master1$ curl -sfL https://get.rke2.io | sh -

# (Optional) If the master behind the proxy
master1$ cat > /etc/default/rke2-server << HERE
CONTAINERD_HTTP_PROXY=http://172.16.31.201:3128
CONTAINERD_HTTPS_PROXY=http://172.16.31.201:3128
CONTAINERD_NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
HTTP_PROXY=http://172.16.31.201:3128
HTTPS_PROXY=http://172.16.31.201:3128
NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
HERE

master1$ systemctl enable rke2-server
master1$ systemctl start rke2-server # Wait about 2-3 minutes for rke2 to be ready

# Check status rke2-server on master1
journalctl -f -u rke2-server
```

#### Retrieve the token of cluster k8s
```
master1$  cat /var/lib/rancher/rke2/server/token
K103043ee68dfecaf979b32a5892257b11c8014fa314a5886160e82f0677e6404e1::server:f213955a6d2e3ccc0596de138cb5c2ec
```

#### On master2, execute the following commands

``` shell
master2$ mkdir -p /etc/rancher/rke2

master2$ cat > /etc/rancher/rke2/config.yaml << HERE
token: xxxx
server: https://172.16.34.16:9345
tls-san:
- master1
- master2
- master1.demo.local
- master2.demo.local
HERE

master2$ export https_proxy=http://172.16.31.201:3128
master2$ export http_proxy=http://172.16.31.201:3128
master2$ export no_proxy=127.0.0.1,localhost,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
master2$ curl -sfL https://get.rke2.io | sh -

master2$ cat > /etc/default/rke2-server << HERE
CONTAINERD_HTTP_PROXY=http://172.16.31.201:3128
CONTAINERD_HTTPS_PROXY=http://172.16.31.201:3128
CONTAINERD_NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
HTTP_PROXY=http://172.16.31.201:3128
HTTPS_PROXY=http://172.16.31.201:3128
NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
HERE

master2$ systemctl enable rke2-server
master2$ systemctl start rke2-server

# Check status rke2-server on master2
journalctl -f -u rke2-server

```

#### On worker1, execute the following commands

```shell
worker1$ cat > /etc/default/rke2-agent << HERE
CONTAINERD_HTTP_PROXY=http://172.16.31.201:3128
CONTAINERD_HTTPS_PROXY=http://172.16.31.201:3128
CONTAINERD_NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
HTTP_PROXY=http://172.16.31.201:3128
HTTPS_PROXY=http://172.16.31.201:3128
NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
HERE

mkdir -p /etc/rancher/rke2/

worker1$ export https_proxy=http://172.16.31.201:3128
worker1$ export http_proxy=http://172.16.31.201:3128
worker1$ export no_proxy=127.0.0.1,localhost,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.local
worker1$ curl -sfL https://get.rke2.io | sh -

cat > /etc/rancher/rke2/config.yaml << HERE
server: https://172.16.34.16:9345
token: ####hidden_string####
HERE

worker01$ systemctl enable rke2-agent
worker01$ systemctl start rke2-agent

# Check status rke2-agent on master2
journalctl -f -u rke2-agent
```
