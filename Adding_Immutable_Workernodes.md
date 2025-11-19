
## Adding Suse Micro 6.1 Immutable OS as Worker Node
Summary:
 - Register the OS
 - install packages iptables open-iscsi nfs-client
 - update packages  openssh curl
 - sudo systemctl enable --now nfs-client.target
 - sudo systemctl enable --now rpcbind.service
 - sudo systemctl enable --now iscsid.service
 - create RKE2 folder in /etc/rancher/rke2
 - create config file /etc/rancher/rke2/config.yaml
 - create folder /etc/rancher/rke2/config.yaml.d/
 - create 50-rancher in /etc/rancher/rke2/config.yaml.d/50-rancher.yaml
 - sudo systemctl reboot
 - curl -sfL https://get.rke2.io | sudo sh -
 - sudo systemctl enable --now rke2-agent.service
 - if the node has no Role type "kubectl label node <nodename> node-role.kubernetes.io/worker=true"

## Steps
### On the new worker node

1. Register the OS
```
sudo transactional-update register -r abcdefg1234567
```
2. Login as root and shell into transactional update
```
transactional-update shell
zypper install iptables open-iscsi nfs-client
```
 Or Install packages via transactional-update command
```
transactional-update pkg install iptables open-iscsi nfs-client
```
3. Run the services of the installed packages.
```
sudo systemctl enable --now nfs-client.target or sudo systemctl enable --now nfs-client.service
sudo systemctl enable --now rpcbind.service
sudo systemctl enable --now iscsid.service
```
3. Create RKE2 folder create RKE2 folder in /etc/rancher/rke2
```
mkdir -p /etc/rancher/rke2
```
4. Prepare the RKE2 configuration file

   A. Create or edit the RKE2 configuration file **/etc/rancher/rke2/config.yaml** to point to the existing cluster's main server URL and include the node token used for cluster authentication. The file should look like this:
```
server: https://<rke2-server-or-loadbalancer-ip>:9345
token: <node-token-from-existing-cluster>
```
 Note: You can get the current node token from an existing master node by retrieving the content of **/var/lib/rancher/rke2/server/node-token**.

   B. **Alternative** Create or edit the RKE2 configuration file **/etc/rancher/rke2/config.yaml** to point to the existing cluster's main server URL and include the node token used for cluster authentication. The file should look like this:
```
cni: calico
resolv-conf: /etc/resolv.conf
```
  Create or edit 50-rancher.yaml file in **/etc/rancher/rke2/config.yaml.d/50-rancher.yaml** 
```
{
  "node-label": [
    "rke.cattle.io/machine=<machine id>",
    "node-role.kubernetes.io/worker=true"
  ],
  "private-registry": "/etc/rancher/rke2/registries.yaml",
  "protect-kernel-defaults": false,
  "server": "https://<masternode ip or loadbalancer ip>:9345"
  "token": "<masternode token>"
}
```
  Note: You can get the current node token from an existing master node by retrieving the content of **/var/lib/rancher/rke2/server/node-token**.

Reboot the **Suse Linux Micro OS** to save the changes
```
sudo systemctl reboot or systemctl reboot 
```
   
5. Install RKE2 on the new worker node if not already installed:
```
curl -sfL https://get.rke2.io | sudo sh - 

[INFO]  finding release for channel stable
[INFO]  using v1.33.5+rke2r1 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.33.5%2Brke2r1/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.33.5%2Brke2r1/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local
```
For specific version and agent only
```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION=v1.33.3-rke2r1 sh -
```

6. Enable and start the RKE2 agent service (not server):
```
sudo systemctl enable --now rke2-agent.service

Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /usr/local/lib/systemd/system/rke2-server.service.
```
To view logs and troubleshoot.
```
journalctl -xeu rke2-agent.service
```
#Bug fix for the symbolic link above pointing to the correct systemd path of Suse Micro.
mv /usr/local/lib/systemd/system/rke2* /etc/systemd/system
```
sudo systemctl reenable rke2-agent.service
sudo systemctl restart rke2-agent.service
```
7. Confirm the new worker node joined the cluster. Run on the masternode.
```
kubectl get nodes
```
8. Set workernode role to worker if **None**. 
```
kubectl label nodename node-role.kubernetes.io/worker=true
```
