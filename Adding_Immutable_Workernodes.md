
## Adding Suse Micro 6.1 Immutable OS as Worker Node

1. On the new worker node

Login as root and shell into transactional update
```
transactional-update shell
```
  Create or edit the RKE2 configuration file **/etc/rancher/rke2/config.yaml** to point to the existing cluster's main server URL and include the node token used for cluster authentication. The file should look like this:
```
server: https://<rke2-server-or-loadbalancer-ip>:9345
token: <node-token-from-existing-cluster>
```
   You can get the current node token from an existing master node by retrieving the content of **/var/lib/rancher/rke2/server/node-token**.
   
2. Install RKE2 on the new worker node if not already installed:
```
curl -sfL https://get.rke2.io | sudo sh -

[INFO]  finding release for channel stable
[INFO]  using v1.33.5+rke2r1 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.33.5%2Brke2r1/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.33.5%2Brke2r1/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local
```

3. Enable and start the RKE2 agent service (not server):
```
sudo systemctl enable --now rke2-agent.service

Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /usr/local/lib/systemd/system/rke2-server.service.

#Bug fix for the symbolic link above pointing to the correct systemd path of Suse Micro.
mv /usr/local/lib/systemd/system/rke2* /etc/systemd/system

sudo systemctl reenable rke2-agent.service
sudo systemctl restart rke2-agent.service
```
4. Confirm the new worker node joined the cluster:


5. Reboot to enable the changes
```
sudo systemctl reboot
```
