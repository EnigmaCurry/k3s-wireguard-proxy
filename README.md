# k3s-wireguard-proxy

Public traefik reverse proxy + wireguard to NAT/firewalled services. 

A small public kubernetes cluster runs traefik and wireguard, in order to
connect a second private kubernetes cluster to expose select private services
publicly.

## Create a single node k3s cluster on DigitalOcean

### From DigitalOcean console:

 * Create a $5 Droplet on Ubuntu 20.04(LTS) (1GB RAM)
 * Select `User data` and paste the following text into the user data textarea:
```
#!/bin/bash
ufw allow 22
ufw allow 80
ufw allow 443
ufw allow 6443
ufw allow 51820
ufw enable
ufw status

apt-get update -y
apt-get install -y wireguard
``` 
 * Assign an SSH pubkey from your workstation/laptop.
 * For this example, set the hostname to `k3s-wireguard`.
 * Create the droplet.
 * Once created, assign a floating ip address to the droplet. (optional.)
 * Create domain name(s) and DNS that points to your droplet's (floating) ip address.

### Administer cluster from your workstation/laptop:

Install kubectl:
 * Prefer your os packages:
   * Arch: `sudo pacman -S kubectl`
   * Ubuntu: [See docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-using-native-package-management)
 * Or this way:
```
TMP=$(mktemp)
curl -L "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl" > $TMP
sudo install $TMP /usr/local/bin/kubectl
rm $TMP
```
   
[Install k3sup](https://github.com/alexellis/k3sup#download-k3sup-tldr)
`curl -sLS https://raw.githubusercontent.com/alexellis/k3sup/master/get.sh | sudo sh`

Create the cluster and config, run:
```
mkdir -p ~/.kube
k3sup install --ip <IP_ADDRESS> --k3s-extra-args '--no-deploy traefik' --local-path ~/.kube/config
```

Test access with kubectl:
```
kubectl get nodes -o wide
```

Install [kilo](https://github.com/squat/kilo):
```
kubectl annotate node k3s-wireguard kilo.squat.ai/location="digitalocean"
kubectl apply -f https://raw.githubusercontent.com/squat/kilo/master/manifests/kilo-k3s.yaml
```

Clone this repository to apply more config:
```
git clone https://github.com/EnigmaCurry/k3s-wireguard.git $HOME/git/vendor/enigmacurry/k3s-wireguard
cd $HOME/git/vendor/enigmacurry/k3s-wireguard
```

Create a wireguard keypair for your workstation/laptop:
```
wg genkey | tee ~/.k3s-wireguard-key | wg pubkey > ~/.k3s-wireguard-pubkey
```
 
Create the peer entry to allow your key access to the cluster:
```
 cat <<EOF | kubectl apply -f -
apiVersion: kilo.squat.ai/v1alpha1
kind: Peer
metadata:
  name: ${USER}
spec:
  allowedIPs:
  - 10.5.0.1/32
  publicKey: $(cat ${HOME}/.k3s-wireguard-pubkey)
  persistentKeepalive: 10
EOF
 ```

