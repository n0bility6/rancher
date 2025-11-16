## Provision RKE2 cluster for Rancher

Login to the server

run `sudo bash` pass same password that you used for login to the server.

Create Directory for rke2

```bash
mkdir -p /etc/rancher/rke2
```

Create config file for first Server, create directory if not exist `/etc/rancher/rke2/config.yaml`

``` bash
echo "token: rke2-token" > /etc/rancher/rke2/config.yaml
```

Run the installer

```bash
curl -sfL https://get.rke2.io | sudo sh -

[INFO]  finding release for channel stable
[INFO]  using v1.33.5+rke2r1 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.33.5%2Brke2r1/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.33.5%2Brke2r1/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local

```

Enable the rke2-server service

```bash
sudo systemctl enable --now rke2-server

Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /usr/local/lib/systemd/system/rke2-server.service.

```

Follow the logs, if you like

```bash
journalctl -u rke2-server -f
```

### Setup kubectl

```bash
cd ~
echo "PATH=$PATH:/usr/local/bin:/var/lib/rancher/rke2/bin" >> .bashrc
PATH=$PATH:/usr/local/bin:/var/lib/rancher/rke2/bin
mkdir .kube
sudo cat /etc/rancher/rke2/rke2.yaml > .kube/config
```

Check the status of kubernetes nodes

```bash
kubectl get nodes
NAME       STATUS     ROLES                       AGE     VERSION
lab0-vm1   NotReady   control-plane,etcd,master   3m42s   v1.33.5+rke2r1

```
wait till the controlplane is up and running, in b/w you can run `kubectl get pods -A`

```bash
kubectl get nodes
NAME       STATUS   ROLES                       AGE     VERSION
lab0-vm1   Ready    control-plane,etcd,master   4m29s   v1.33.5+rke2r1

```
you should be able to see the control plane node up and running.
After successful installation, we can start provisioning Rancher into the RKE2 cluster.

## Setting up Rancher

To Setup rancher we need to install Helm CLI

### Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh


Helm v3.19.0 is already latest

```

Once helm is installed, we need to add helm repos
Install the Rancher Helm Chart

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

"rancher-stable" has been added to your repositories

```bash
helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "rancher-stable" chart repository
Update Complete. ⎈Happy Helming!⎈

```

### Create a Namespace for Rancher

```bash
kubectl create namespace cattle-system

namespace/cattle-system created

```

### Install cert-manager

#### Add the Jetstack Helm repository

```bash
helm repo add jetstack https://charts.jetstack.io

"jetstack" has been added to your repositories

```

### Update your local Helm chart repository cache

```bash
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "rancher-stable" chart repository
Update Complete. ⎈Happy Helming!⎈

```

#### Install the cert-manager Helm chart

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

NAME: cert-manager
LAST DEPLOYED: Tue Sep 30 07:57:22 2025
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
⚠️  WARNING: New default private key rotation policy for Certificate resources.
The default private key rotation policy for Certificate resources was
changed to `Always` in cert-manager >= v1.18.0.
Learn more in the [1.18 release notes](https://cert-manager.io/docs/releases/rel                                                                                            ease-notes/release-notes-1.18).

cert-manager v1.18.2 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/

```

Wait till the Cert Manager pods are up and running
```bash
watch kubectl get pods -n cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-58dd99f969-4mwnr              1/1     Running   0          3m29s
cert-manager-cainjector-55cd9f77b5-gf5r2   1/1     Running   0          3m29s
cert-manager-webhook-7987476d56-c9lh7      1/1     Running   0          3m29s

```

Once all the pods are up and running, we can start deploying Rancher

Check your ip adress

```bash
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:a5:43:19 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.230.171/24 brd 192.168.230.255 scope global ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fea5:4319/64 scope link
       valid_lft forever preferred_lft forever
```
check DNS lookups

```bash
nslookup $YOUR_IP.sslip.io
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   192.168.230.171.sslip.io
Address: 192.168.230.171

```

### Create self signed certificate

```bash
openssl req -newkey rsa:2048 -x509 -sha256 -days 365 -nodes \
  -subj "/CN=$YOUR_IP.sslip.io" \
  -keyout tls.key -out tls.crt \
  -addext "subjectAltName=DNS:$YOUR_IP.sslip.io"
```
### Creare k8s secret for ingress tls certs

```bash
kubectl -n cattle-system create secret tls tls-rancher-ingress   --cert=tls.crt   --key=tls.key
```

```bash
 kubectl get secrets -n cattle-system
NAME                  TYPE                DATA   AGE
tls-rancher-ingress   kubernetes.io/tls   2      49s
```

```bash
helm upgrade --install rancher rancher-stable/rancher   --namespace cattle-system   --set hostname=$YOUR_IP.sslip.io   --set bootstrapPassword=admin   --set ingress.tls.source=secret --set replicas=1
 
Release "rancher" does not exist. Installing it now.
NAME: rancher
LAST DEPLOYED: Tue Sep 30 08:17:55 2025
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

## First Time Login

If you provided your own bootstrap password during installation, browse to https://192.168.230.171.sslip.io to get started.
If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:

```
echo https://192.168.230.171.sslip.io/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

To get just the bootstrap password on its own, run:

```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```


Happy Containering!


```

Wait for Rancher to be rolled out:

```bash
kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 1 updated replicas are available...
deployment "rancher" successfully rolled out
```

You should be able to access Rancher UI using the FQDN.

### Adding New Masternodes

1. On the new master node

   Create or edit the RKE2 configuration file **/etc/rancher/rke2/config.yaml** to point to the existing cluster's main server URL and include the node token used for cluster authentication. The file should look like this:
```
server: https://<existing-master-node-ip>:9345
token: <node-token-from-existing-master>
```
   You can get the current node token from an existing master node by retrieving the content of **/var/lib/rancher/rke2/server/node-token**.
   
2. Clean up any previous RKE2 installation or remnants on the new node:

   Run the RKE2 kill script to stop all RKE2 processes:

```
/usr/local/bin/rke2-killall.sh
```
   Delete the old database if any:

```
sudo rm -rf /var/lib/rancher/rke2/server/db
```

3. Install RKE2 on the new master node if not already installed:
```
curl -sfL https://get.rke2.io | sh -
```
4. Enable and start the RKE2 server service:
```
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

5. Setup kubectl
   
```
cd ~
echo "PATH=$PATH:/usr/local/bin:/var/lib/rancher/rke2/bin" >> .bashrc
PATH=$PATH:/usr/local/bin:/var/lib/rancher/rke2/bin
mkdir .kube

sudo cat /etc/rancher/rke2/rke2.yaml > .kube/config
Or
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml #(required every login)
```
6. Verify the new master node has joined the cluster by checking nodes from any existing master node with:

```
kubectl get nodes

NAME                  STATUS   ROLES                       AGE     VERSION
rke2-master1-ubuntu   Ready    control-plane,etcd,master   50m     v1.33.5+rke2r1
rke2-master2-ubuntu   Ready    control-plane,etcd,master   7m58s   v1.33.5+rke2r1
```
