Homelab Kubernetes setup, using `kubeadmn` on Rapberry Pis. Used [official](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) docs as a reference and miscellaneous blog posts. For learning and function.

# High Level

- Compute: Raspberry Pi 5s
- CRI: containerd
- CNI: Cilium
- Helm: managing pods
- ArgoCD: [GitOps](https://about.gitlab.com/topics/gitops/) which essentially syncs the actual cluster to config files in GitHub to

# Setting Up New Raspberry Pi 5

## All Nodes

### Assign Static IP

This allows us to more reliably connect the nodes together. All it requires is defining this in the Wifi router (go to gateway IP and set).

### Disable Swap

Keeps processes isolated from each other, can only use allocated amount of ram and if going over won't bleed into system.

Turn off with these commands (from the man page `man swap.conf`)

```bash
$ sudo mkdir -p /etc/rpi/swap.conf.d
$ sudo echo -e "[Main]\nMechanism=none" | sudo tee /etc/rpi/swap.conf.d/disable-swap.conf
$ sudo reboot
```

### Containerd

This is the standard for containerizing k8s. Follow install instructions [here](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf). How I did it using the Debian docs:

Setup docker key stuff since it distributes containerd

```bash
# Add Docker's official GPG key:
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt-get update
```

Install containerd and add modules, most docs mention this but I don't see it in any k8s documentation.

```bash
$ sudo apt update && sudo apt install -y containerd.io
$ sudo mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml
$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
$ sudo modprobe overlay br_netfilter
$ sudo systemctl restart containerd
```

Next edit `/etc/containerd/config.toml` so that `SystemdCgroup = true`. Then run `sudo systemctl restart containerd`

### Networking

Mostly from [k8s docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

To manually enable IPv4 packet forwarding (needed because each pod has an IP that the host will forward to, I think):

```bash
# sysctl params required by setup, params persist across reboots
$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
$ sudo sysctl --system
```

### cgroups

cgroups allow partitioning of resources, obviously very important for k8s.

Easy way to enable cgroup on the pi.

```bash
$ sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt
```

### Persistent Storage

```bash
$ git clone https://github.com/rancher/local-path-provisioner.git
$ cd local-path-provisioner
$ helm install local-path-storage --namespace local-path-storage --create-namespace ./deploy/chart/local-path-provisioner
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Kubernetes Tooling

Followed [docs](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

```bash
$ sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
$ sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
$ sudo systemctl enable --now kubelet
```

## Control Plane Node

### Init

Skip kube-proxy for Cilium

```bash
$ sudo reboot # just in case
$ sudo kubeadm init --skip-phases=addon/kube-proxy

# follow instructions printed, something like
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Save output join command for later, 1Password works. Something like

```bash
$ kubeadm join 192.168.0.52:6443 --token w8m6hp.6t2tcjpjjkgoh9uo \
	--discovery-token-ca-cert-hash sha256...
```

### Helm

Install Helm (needed for cilium setup)

```bash
$ sudo apt-get install curl gpg apt-transport-https --yes
$ curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
$ echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt-get update
$ sudo apt-get install helm
```

### Cilium

Following [docs](https://docs.cilium.io/en/latest/network/kubernetes/kubeproxy-free/)

```bash
# add helm cilium repo
$ helm repo add cilium https://helm.cilium.io/
$ helm repo update

# install cilium to cluster
$ API_SERVER_IP=192.168.0.52 API_SERVER_PORT=6443 helm install cilium cilium/cilium \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=192.168.0.52 \
    --set k8sServicePort=6443 \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=shared \
    --set ingressController.service.loadBalancerIP=192.168.0.2 \
    --set l2announcements.enabled=true
```

### Remove Taint

Allows data plane (workers) to run on the control plane node.

```bash
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install ArgoCD

One off installation to manage everything!

```bash
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
```

### Secrets

Generate and apply the secret manually (never commit the real token):

openclaw-token

```bash
# Generate a secure token
TOKEN=$(openssl rand -hex 32)

# Create the secret
kubectl create secret generic openclaw-token \
  --from-literal=value=$TOKEN \
  --namespace=openclaw
# Save the token somewhere safe (1Password, etc.)
echo $TOKEN
```

To retrieve the token later:

```bash
kubectl get secret openclaw-token -n openclaw \
  -o jsonpath='{.data.value}' | base64 --decode; echo
```

Telegram Bot Token

```bash
  kubectl create secret generic telegram-token \
    --from-literal=value=$YOUR_BOT_TOKEN \
    --namespace=openclaw
```

Telegram Allow From

```bash
  kubectl create secret generic telegram-allow-from \
    --from-literal=value=$VALUE \
    --namespace=openclaw
```

Google API Key

This was too expensive so I commented it out, but this is how to add a secret.

```bash
  # kubectl create secret generic google-api-key \
  #  --from-literal=value=$YOUR_GOOGLE_API_KEY \
  #  --namespace=openclaw
```

## Local DNS And LAN App Access

Local app names use `home.arpa`, the standard home-network domain.

```text
https://calibre.home.arpa
https://argocd.home.arpa
https://pihole.home.arpa
...
```

### Router Range

Two LAN IPs are reserved by limiting the router DHCP range to `.4` - `.253`,
making these safe to hardcode:

```text
192.168.0.2 # shared web ingress for apps
192.168.0.3 # Pi-hole DNS
```

That keeps `.2` and `.3` out of the automatic device pool.

### Cilium

The GitOps `networking` app creates the Cilium IP pool and L2 announcement policy. L2 announcements means Cilium tells the LAN which node is currently serving those IPs. Announcements use `eth0` on any eligible node, so another Pi can take over if the current announcer goes down.

The Cilium IP pool only says which IPs are allowed. Each `LoadBalancer` service still
asks for the exact IP it wants:

If Cilium was installed before ingress and L2 announcements were enabled:

```bash
$ helm upgrade cilium cilium/cilium \
    --namespace kube-system \
    --reuse-values \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=shared \
    --set ingressController.service.loadBalancerIP=192.168.0.2 \
    --set l2announcements.enabled=true
```

Pi-hole uses the HelmForge chart. Before syncing it, create the admin password secret:

```bash
$ kubectl create namespace pihole
$ kubectl create secret generic pihole-admin-secret \
    --from-literal=password=$PIHOLE_ADMIN_PASSWORD \
    --namespace=pihole
```

Once Cilium assigns `192.168.0.3`,
set the router DNS server to `192.168.0.3`.

### Local HTTPS

Local HTTPS uses a self-signed wildcard cert. Browsers will warn until the cert is trusted on that device.

Create one cert:

```bash
$ openssl req -x509 -newkey rsa:2048 -nodes -days 825 \
    -keyout /private/tmp/home-arpa.key \
    -out /private/tmp/home-arpa.crt \
    -subj "/CN=*.home.arpa" \
    -addext "subjectAltName=DNS:*.home.arpa,DNS:home.arpa"
```

Install the same cert secret into each app namespace:

```bash
$ kubectl -n calibre create secret tls home-arpa-tls \
    --cert=/private/tmp/home-arpa.crt \
    --key=/private/tmp/home-arpa.key

$ kubectl -n argocd create secret tls home-arpa-tls \
    --cert=/private/tmp/home-arpa.crt \
    --key=/private/tmp/home-arpa.key

$ kubectl -n pihole create secret tls home-arpa-tls \
    --cert=/private/tmp/home-arpa.crt \
    --key=/private/tmp/home-arpa.key
```

Sync Argo after creating the secrets.

# Maintaining

### Connect to Control Node From Laptop

For running kubectl commands and maintaining, very useful for port forwarding. Note, to get this to work in Iterm2 on MacOS I had to give Iterm2 local network permissions, and to do that I had to restart Iterm2 and run `telnet <RPI_IP> 6443`. Not the most fun thing to figure out, also a transient fix so I recommend other terminals when possible.

```bash
$ ssh cobyforrester@192.168.0.52 "sudo cat /etc/kubernetes/admin.conf" > ~/.kube/config-pi
$ export KUBECONFIG=~/.kube/config-pi
```

### Connect to ArgoCD

Connect using port forwarding on personal computer.

```bash
$ kubectl -n argocd port-forward svc/argocd-server 8443:443 --address 127.0.0.1
```

Print argocd password with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

Then in the browser navigate to `https://localhost:8443` to login.

### Setup ArgoCD

Create SSH key and add to the repository section in the UI.

### Install K8s Operators

[CloudNativePG (Postgres)](https://cloudnative-pg.io/docs/1.28/installation_upgrade):

```bash
# Operator
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.28/releases/cnpg-1.28.0.yaml

# PG Image
kubectl apply -f \
  https://raw.githubusercontent.com/cloudnative-pg/artifacts/refs/heads/main/image-catalogs/catalog-minimal-trixie.yaml
```

### Snapshots

I have already had it where a power outage corrupts everything, this may help:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backup/snapshot.db \
  --endpoints=127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### In Case Of Pi-Hole Disruption

Since the DNS server is set to a local pod, if that pod is disrupted, then the DNS resolution for the local network will be disrupted. This will look like no webpages loading, but you can still load the router UI at http://192.168.0.1.

Resolve by either fixing the Pi-Hole device, or resetting the router to use the non-managed DNS server (default). Do this by:

1. Navigate to http://192.168.0.1
2. Login, username `admin` and password is the WiFi password.
3. Navigate to Connection -> Local IP Network -> IPv4. From here check `Enable IPv4/IPv6 DNS Relay`, and for `LAN DNS` click `Obtained automatically`. Then save.
4. Disconnect and reconnect to the WiFi, can take several minutes to reset.
