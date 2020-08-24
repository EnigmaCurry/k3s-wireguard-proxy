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
mkdir -p ${HOME}/.kube
k3sup install --ip <IP_ADDRESS> --k3s-extra-args '--no-deploy traefik' --local-path ${HOME}/.kube/config
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
wg genkey | tee /tmp/k3s-wireguard-key | wg pubkey > /tmp/k3s-wireguard-pubkey
```

Create the peer entry to allow your key access to the cluster:
```
cat <<EOF | kubectl apply -f -
apiVersion: kilo.squat.ai/v1alpha1
kind: Peer
metadata:
  name: ${USER}-${HOSTNAME}
spec:
  allowedIPs:
  - 10.5.0.1/32
  publicKey: $(cat /tmp/k3s-wireguard-pubkey)
  persistentKeepalive: 10
EOF
 ```

Install `kgctl` command line tool:
 * Install [go]() and put `$HOME/go/bin` in your `$PATH`
 * Install `kgctl`:
```
go get github.com/squat/kilo/cmd/kgctl
go install github.com/squat/kilo/cmd/kgctl
```

Get the peer configuration:
```
kgctl --kubeconfig ${HOME}/.kube/config showconf peer ${USER}-${HOSTNAME} > /tmp/k3s-wireguard.ini
```

Setup [systemd-networkd](https://wiki.archlinux.org/index.php/Systemd-networkd) on your workstation

Setup systemd-networkd devices:

```
cat <<EOF | sudo tee /etc/systemd/network/40-k3s-wireguard.netdev
[NetDev]
Name = wg-k3s
Kind = wireguard
Description = k3s -> kilo WireGuard VPN

[WireGuard]
PrivateKey = $(cat /tmp/k3s-wireguard-key)

$(cat /tmp/k3s-wireguard.ini | sed 's/\[Peer\]/\[WireGuardPeer\]/')
EOF
```

Setup systemd-networkd network:

```
cat <<EOF | sudo tee /etc/systemd/network/45-k3s-wireguard.network
[Match]
Name=wg-k3s

[Network]
Address = 10.5.0.1/32

$(for ip in $(grep AllowedIPs /tmp/k3s-wireguard.ini | cut -f 3- -d ' ' | tr -d ','); do
   cat <<OOF
[Route]
Destination = $ip

OOF
done)
EOF
```

Now restart systemd-networkd:

```
sudo systemctl restart systemd-networkd
```

Check to see the interface *and* peer are connected:

```
sudo wg
```

You should see output similar to:

```
interface: wg-k3s
  public key: XXXXXXXXXXXXXXXXXXXX
  private key: (hidden)
  listening port: 38051

peer: XXXXXXXXXXXXXXXXXXXXXXXXXXX
  endpoint: X.X.X.X:51820
  allowed ips: 10.4.0.1/32, 10.48.0.5/32, 10.42.0.0/24
  latest handshake: 55 seconds ago
  transfer: 348 B received, 596 B sent
  persistent keepalive: every 10 seconds
```


Check the interface acquired the ip address:

```
ip addr show dev wg-k3s
```

Debug any systemd-networkd errors that might be in the log:

```
journalctl --unit systemd-networkd
```
