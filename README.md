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

## 5. Apply all resources

### Kube router
```
k apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```

### Metal LB
 https://metallb.universe.tf/installation/
```
k apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
k apply -f deployments/metallb/ipaddress_pools.yaml
```

### Cert Manager
```
k apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.3/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --values=helm/cert-manager/values.yaml --version v1.14.3
```

### Traefik
```
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik --values=helm/traefik/traefik-values.yaml
k apply -f helm/traefik/ingress.yaml
```

### Further Traefik
k apply -f helm/traefik/skip-ssl.yaml
k apply -f helm/traefik/reverse-proxy/nas-ingress.yaml
k apply -f helm/traefik/reverse-proxy/proxmox-hs-ingress.yaml
k apply -f helm/traefik/reverse-proxy/proxmox-n1-ingress.yaml
k apply -f helm/traefik/reverse-proxy/proxmox-n2-ingress.yaml
k apply -f helm/traefik/reverse-proxy/zigbee-controller-ingress.yaml

### Longhorn
```
k apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
k apply -f deployments/longhorn/ingress.yaml
```

### Traefik SSL cert usage

### SSL / Let's Encrypt
 https://www.youtube.com/watch?v=G4CmbYL9UPg
```
k apply -f secrets/cf-token.secret.yaml
```
### Staging
```
k apply -f secrets/certificate-issuers/letsencrypt-staging.insecure.yaml
k apply -f helm/cert-manager/certificates/staging/umhomelab-com.yaml
k apply -f helm/cert-manager/certificates/staging/traefik-default-tls.yaml
```

Wait for propagation: find the right pod by checking logs and wait till it says propagated.
The following command should output: `The certificate has been successfully issued` (when propagated).

```
k describe certificate umhomelab-com
```

If the cert does not appear:
  - the issue might be on the name used in traefik-default-tls.yaml
  - try deleting traefik pod or CTRL+F5

### Prod
```
k delete secret umhomelab-com-staging
k apply -f secrets/certificate-issuers/letsencrypt-production.insecure.yaml
k apply -f helm/cert-manager/certificates/production/umhomelab-com.yaml
k apply -f helm/cert-manager/certificates/production/traefik-default-tls.yaml
```

### Postgres
```
k apply -f deployments/postgres/namespace.yaml

k apply -f secrets/pg.secret.yaml
k apply -f secrets/pgadmin.secret.yaml

k apply -f deployments/postgres/configmaps.yaml
k apply -f deployments/postgres/secret.yaml
k apply -f deployments/postgres/postgres.yaml
k apply -f deployments/postgres/postgres14.yaml
```

### Authentik
```
k apply -f helm/authentik/namespace.yaml
k apply -f helm/authentik/authentik-volume+claim.yaml
helm repo add goauthentik https://charts.goauthentik.io
helm repo update
helm upgrade --install authentik goauthentik/authentik -f secrets/authentik-values.insecure.yaml -n auth --version 2024.2.2
k apply -f helm/traefik/middlewares/
k apply -f helm/authentik/ingress.yaml
```

### K8s dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
helm upgrade --install dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace dashboard -f helm/k8s-dashboard/values.yaml --version 7.0.0-alpha3

k apply -f secrets/create-service-account.secret.yaml
k apply -f secrets/create-cluster_role_binding.secret.yaml
k apply -f secrets/get-bearer.secret.yaml

k apply -f helm/k8s-dashboard/ingress.yaml
kubectl get secret jac -n dashboard -o jsonpath={".data.token"} | base64 -d
```

### Mosquitto
```
k apply -f deployments/mosquitto/
```

### Dev plugin
```
k apply -f deployments/generic-device-plugin/
```

### Zigbee
```
k apply -f deployments/zigbee/
```

### Multus
Usual install (do not use now due to issue)
```
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git
cd multus-cni/
cat ./deployments/multus-daemonset-thick.yml | k apply -f -

cd ~/homelab/k8s-templates/
k apply -f deployments/multus/multus-lan.yaml
```
Sometimes multus fails due to: https://github.com/k8snetworkplumbingwg/multus-cni/issues/1221
Current temporary fix in:
```
k apply -f deployments/multus/multus-lan.yaml
k apply -f deployments/multus/multus-daemonset-thick.yaml
```

### Home Assistant
```
k apply -f deployments/hass/
```

### Code
```
k apply -f deployments/code/
```

### Jellyfin
```
k apply -f deployments/jellyfin/
```

### RR
```
k apply -f deployments/rr/namespace.yaml
k apply -f deployments/rr/rr-media-volume+claim.yaml
```

### Homepage
```
k apply -f deployments/homepage/
```

### Homarr
```
k apply -f deployments/homarr/
```

### Bazarr
```
k apply -f deployments/rr/bazarr/
k apply -f secrets/bazarr-pg14.secret.yaml
```

### Jellyseerr
```
k apply -f deployments/rr/jellyseerr/
```

### Prowlarr
```
k apply -f deployments/rr/prowlarr/
```

### Radarr
```
k apply -f deployments/rr/radarr/
```

### Sonarr
```
k apply -f deployments/rr/sonarr/
```

### Qbittorrent
```
k apply -f deployments/qbittorrent/
```

### Monitoring
```
k create ns monitoring
```
### Prometheus
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install -n monitoring prometheus prometheus-community/kube-prometheus-stack -f deployments/monitoring/prometheus.yaml
```

### Redis
```
k apply -f helm/redis/volume.yaml
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install -n db redis bitnami/redis -f secrets/redis.insecure.yaml
```

### Immich
```
k create ns immich
k apply -f deployments/immich/photos-volume.yaml
k apply -f secrets/immich.insecure.yaml
k apply -f deployments/immich/ingress.yaml
```

### Apache Guacamole
```
k create ns guaca
k apply -f secrets/guaca.secret.yaml
k apply -f deployments/guaca/guacd.yaml
k apply -f deployments/guaca/guaca.yaml
```
