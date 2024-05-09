# Home Lab
My homelab

# Submodules

## Cloning this repo

```
git clone --recurse-submodules  git@github.com:UnconventionalMindset/homelab.git
```

## Updating the submodules

```
git submodule update --remote
```

# Notes

Nextcloud has been dropped for being too slow.
TODO: automate the following provisioning even more and use GitOps
TODO: make insecure files secure. They are currently left out of this repo for security reasons

# Provisioning Guide

## 0. Cleanup

### Remove info of old cluster
```
ssh-keygen -f "/home/jac/.ssh/known_hosts" -R "192.168.31.190"
ssh-keygen -f "/home/jac/.ssh/known_hosts" -R "192.168.31.191"
ssh-keygen -f "/home/jac/.ssh/known_hosts" -R "192.168.31.192"
rm -f ~/.kube/config
```

## 1. Terraform automations
```
terraform init
terraform apply
```

## 2. Ansible automations
```
cd ~/homelab/ansible-automations/
ansible-galaxy install -r requirements.yaml
ansible-playbook playbooks/coreos-packages.yaml
ansible-playbook playbooks/k8s-init.yaml
ansible-playbook playbooks/k8s-tools-setup.yaml
ansible-playbook playbooks/cni-plugins.yaml
export ANSIBLE_VAULT_PASSWORD_FILE="password.secret"
```

## 3. Local setup

### Copy kubecfg locally

```
ssh -i ~/.ssh/coreos core@192.168.31.190
sudo su
cp ~/.kube/config /home/core/config
chmod 777 /home/core/config
```

In another terminal tab execute:

```
mkdir -p ~/.kube
scp -i ~/.ssh/coreos core@192.168.31.190:/home/core/config ~/.kube/config
chmod 600 ~/.kube/config
cd ~/homelab/k8s-templates
```

In the other tab, delete the k8s config:
```
rm /home/core/config
```

## 4. Proxmox setup

### USB amd PCI passthrough:
Add in the worker node that has the zigbee USB:

USB Device - Vendor/Device ID: Use USB Vendor/Device ID

In the workers node add GPU passthrough, selecting:
PCI Device - RAW

## 5. Apply all K8s resources

Continue the setup in the k8s-templates submodule