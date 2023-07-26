# Setup RKE2 cluster with Kube-VIP

## Definition

RKE2, also known as RKE Government, is Rancher's next-generation Kubernetes distribution.

It is a fully conformant Kubernetes distribution that focuses on security and compliance within the U.S. Federal Government sector.

To meet these goals, RKE2 does the following:


1. Provides defaults and configuration options that allow clusters to pass the CIS Kubernetes Benchmark v1.5 or v1.6with minimal operator intervention
2. Enables FIPS 140-2 compliance
3. Regularly scans components for CVEs using trivy in our build pipeline

## What is kube-vip

The kube-vip project provides High-Availability and load-balancing for both inside and outside a Kubernetes cluster

## Introduction

### Prereqs

In order to proceed with this guide, you will need the following:

DNS server or modification of /etc/hosts with the node hostnames and rke2 master HA hostname firewall turned off

### Assumptions

| Host | IP | VIP | Notes |
|----|----|----|----|
| master1 | 10.166.9.11 | 10.166.9.10 | control-plane,etcd,master |
| master2 | 10.166.9.12 | 10.166.9.10 | control-plane,etcd,master |
| master3 | 10.166.9.13 | 10.166.9.10 | control-plane,etcd,master |

If you do not have a DNS server available/configured, the `/etc/hosts` file on each node will need to include the following.

```javascript
10.166.9.11 master1 master1.sre.local
10.166.9.12 master2 master2.sre.local
10.166.9.13 master3 master3.sre.local
10.166.9.10 rke2-ha.sre.local
```

## Install

**On master1, execute the following commands**

```bash
master1$ mkdir -p /etc/rancher/rke2/

# Create tls-san for kubernetes
master1$ cat > /etc/rancher/rke2/config.yaml << HERE
write-kubeconfig-mode: "0644"
tls-san:
- master1
- master2
- master3
- master1.sre.local
- master2.sre.local
- master3.sre.local
- rke2-ha.sre.local
- 10.166.9.10
node-taint:
- "CriticalAddonsOnly=true:NoExecute"
node-label:
- "teams=sre"
- "environment=product"
- "location=ttepz"
- "site=sdn"
cni:
- calico
HERE

# Setup environment
export VIP=10.166.9.10
export TAG=v0.5.12
export INTERFACE=ens160
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
alias k=kubectl

# Run bootstrap to install rke2
master1$ curl -sfL https://get.rke2.io | sh -

# (Optional) If the master behind the proxy
master$ cat > /etc/default/rke2-server << HERE
CONTAINERD_HTTP_PROXY=http://10.10.41.254:3128
CONTAINERD_HTTPS_PROXY=http://10.10.41.254:3128
CONTAINERD_NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local
HTTP_PROXY=http://10.10.41.254:3128
HTTPS_PROXY=http://10.10.41.254:3128
NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local
HERE

systemctl enable rke2-server
systemctl start rke2-server # Wait about 2-3 minutes for rke2 to be ready

# Install kube-vip installation

# Create kube-vip rbac manifest
curl -s https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml

# Pull image with container runtime
ctr image pull docker.io/plndr/kube-vip:$TAG

# Lệnh này chạy mỗi khi mở lại IP máy unbutu
alias kube-vip="ctr run --rm --net-host docker.io/plndr/kube-vip:$TAG vip /kube-vip"

# generate manifest
kube-vip manifest daemonset \
    --arp \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --leaderElection \
    --taint \
    --services \
    --inCluster | tee /var/lib/rancher/rke2/server/manifests/kube-vip.yaml

# check kube-vip pod and daemonset
master1$ k -n kube-system get ds #( kubectl -n kube-system get ds )



#master1$ k -n kube-system get pod -l name=kube-vip-ds
master1$ k -n kube-system get pod
NAME                READY   STATUS    RESTARTS   AGE
kube-vip-ds-2vjkz   1/1     Running   0          2m1s



# check vip
master1$ ip a s | grep -B 5 ens160
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:ad:fa:e5 brd ff:ff:ff:ff:ff:ff
    inet 10.166.9.11/24 brd 10.166.9.255 scope global ens160
       valid_lft forever preferred_lft forever
    inet 10.166.9.10/32 scope global ens160

# Because the master set taints value,so the coredns is not working. Add tolorations for coredns deployment

cat > /var/lib/rancher/rke2/server/manifests/rke2-coredns-config.yaml << HERE
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-coredns
  namespace: kube-system
spec:
  valuesContent: |-
    nodeSelector:
      node-role.kubernetes.io/master: "true"
    tolerations:
    - key: CriticalAddonsOnly
      effect: NoExecute
    servers:
    - zones:
      - zone: .
      port: 53
      plugins:
      - name: errors
      - name: health
        configBlock: |-
          lameduck 5s
      - name: ready
      - name: kubernetes
        parameters: cluster.local in-addr.arpa ip6.arpa
        configBlock: |-
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
      - name: prometheus
        parameters: 0.0.0.0:9153
      - name: forward
        parameters: . /etc/resolv.conf
      - name: cache
        parameters: 30
      - name: reload
      - name: loop
      - name: loadbalance
HERE

# Retrieve the token
master1$  cat /var/lib/rancher/rke2/server/token
####hidden_string#####

# Đã làm xong tới đây ------------------------------------------------------------------------------ >
```


**On master2, execute the following commands**


```bash
master2$ mkdir -p /etc/rancher/rke2

master2$ cat > /etc/rancher/rke2/config.yaml << HERE
server: https://rke2-ha.sre.local:9345
token: ####hidden_string####
tls-san:
- master1
- master2
- master3
- master1.sre.local
- master2.sre.local
- master3.sre.local
- rke2-ha.sre.local
- 10.166.9.10
node-taint:
- "CriticalAddonsOnly=true:NoExecute"
node-label:
- "teams=sre"
- "environment=product"
- "location=ttepz"
- "site=sdn"
cni:
- calico
HERE

# (Optional) Config proxy
cat > /etc/default/rke2-server <<HERE
CONTAINERD_HTTP_PROXY=http://10.10.41.254:3128
CONTAINERD_HTTPS_PROXY=http://10.10.41.254:3128
CONTAINERD_NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local
HTTP_PROXY=http://10.10.41.254:3128
HTTPS_PROXY=http://10.10.41.254:3128
NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local
HERE

master2$ curl -sfL https://get.rke2.io | sh -
master2$ systemctl enable rke2-server
master2$ systemctl start rke2-server
```


On `master 3`, same step of `master 2`

### Join worker in cluster

| Host | IP | Roles |
|----|----|----|
| worker01 | 10.166.9.21 | worker |
| worker02 | 10.166.9.22 | worker |


**On worker01**

```bash
echo "10.166.9.11 master1 master1.sre.local
10.166.9.12 master2 master2.sre.local
10.166.9.13 master3 master3.sre.local
10.166.9.10 rke2-ha.sre.local" >> /etc/hosts


export http_proxy=http://10.166.9.199:3128
export https_proxy=http://10.166.9.199:3128
export no_proxy=10.0.0.0/8

curl -sfL https://get.rke2.io | sh -

# (optional) Config environemnt http proxy
cat > /etc/default/rke2-agent << HERE
CONTAINERD_HTTP_PROXY=http://10.166.9.199:3128
CONTAINERD_HTTPS_PROXY=http://10.166.9.199:3128
CONTAINERD_NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local
HTTP_PROXY=http://10.166.9.199:3128
HTTPS_PROXY=http://10.166.9.199:3128
NO_PROXY=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local
HERE

mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml << HERE
server: https://rke2-ha.sre.local:9345
token: ####hidden_string####
node-label:
- "teams=sre"
- "environment=product"
- "location=ttepz"
- "site=sdn"
HERE

worker01$ systemctl enable rke2-agent
worker01$ systemctl start rke2-agent
```

### Troubleshoot

CoreDNS loop: <https://fossies.org/linux/coredns/man/coredns-loop.7>
