# Setup MicroK8S

## Grant auth to `admin`

```
sudo usermod -a -G microk8s admin
sudo chown -f -R admin ~/.kube

newgrp microk8s
```

Check microk8s status
`microk8s status --wait-ready`

## Create permanent alias

Edit
`nano ~/.bashrc`

```
alias kubectl='microk8s.kubectl'
alias helm='microk8s.helm3'
```

## Enable Add-ons

```
microk8s enable dns ingress helm3 hostpath-storage host-access
```

## Configure Private Registry

For internal registries where TLS with a custom CA is used (e.g. in enterprise environments), containerd will fail to fetch images unless the CA is explicitly specified.

In our previous example, if the registry was instead at https://server.demo.com:8482, the configuration should be changed to the following:

`sudo mkdir /var/snap/microk8s/current/args/certs.d/server.demo.com:8482`

`sudo nano /var/snap/microk8s/current/args/certs.d/server.demo.com:8482/hosts.toml`

```
server = "https://server.demo.com:8482"

[host."https://server.demo.com:8482"]
capabilities = ["pull", "resolve"]
ca = "/var/snap/microk8s/current/args/certs.d/server.demo.com:8482/ca.crt"
```

Add the CA certificate under /var/snap/microk8s/current/args/certs.d/server.demo.com:8482/ca.crt:
`sudo cp ~/tmp/ca.crt /var/snap/microk8s/current/args/certs.d/server.demo.com:8482/ca.crt`

Add the below section to bottom of /var/snap/microk8s/current/args/containerd-template.toml.

`sudo nano /var/snap/microk8s/current/args/containerd-template.toml`

```
[plugins."io.containerd.grpc.v1.cri".registry.configs."server.demo.com:8482".auth]
username = "admin"
password = "admin@123"
```

`sudo snap restart microk8s`

## Setup NFS Server for PV

`sudo apt-get install nfs-kernel-server`

```
sudo mkdir -p /srv/nfs
sudo chown nobody:nogroup /srv/nfs
sudo chmod 0777 /srv/nfs
```

Edit `/etc/exports` file, making sure that the IP addresses of all your MicroK8s nodes are able to mount this share

```
sudo mv /etc/exports /etc/exports.bak
echo '/srv/nfs *(rw,sync,no_subtree_check)' | sudo tee /etc/exports
sudo exportfs -rav
```

Restart nfs server
`sudo systemctl restart nfs-kernel-server`

## Install CSI driver for NFS

```
microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
microk8s helm3 repo update
microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace kube-system \
    --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
```

Wait for csi controller pod to startup.
`microk8s kubectl wait pod --selector app.kubernetes.io/name=csi-driver-nfs --for condition=ready --namespace kube-system`

List available csi drivers.
`microk8s kubectl get csidrivers`

```
NAME             ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
nfs.csi.k8s.io   false            false            false             <unset>         false               Persistent   66s
```

## Create a StorageClass for NFS

Edit `nfs-csi.yaml`

```
mkdir /ktcmaai/nfs
cd /ktcmaai/nfs
nano nfs-csi.yaml
```

Paste content

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.30
  share: /srv/nfs
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
```

Apply on kubernetes
`microk8s kubectl apply -f - < nfs-csi.yaml`
