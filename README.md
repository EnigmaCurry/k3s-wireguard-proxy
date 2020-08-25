# k3s-wireguard-proxy

Access your private kubernetes resources through wireguard (VPN) and/or expose
your local services to the cluster and/or internet.

You can use this like a self-hosted ngrok.

## How it works

Following this guide you will:

 * Configure and create a new single-node k3s cluster on DigitalOcean ($5/mo
   droplet) utilizing [k3sup](https://github.com/alexellis/k3sup). You can also
   use any existing kubernetes cluster.
 * Install [kilo](https://github.com/squat/kilo), which creates a wireguard
   service inside your k3s cluster network.
 * Create a wireguard network interface on your local workstation.
 * Maintain your connection through
   [systemd-networkd](https://wiki.archlinux.org/index.php/Systemd-networkd)

## Clone this repository on your workstation/laptop

```
DIR=$HOME/git/vendor/enigmacurry/k3s-wireguard-proxy
git clone https://github.com/EnigmaCurry/k3s-wireguard-proxy.git $DIR
cd $DIR
```

## Create a single node k3s cluster on DigitalOcean

### From DigitalOcean console:

 * Create a $5 Droplet on Ubuntu 20.04(LTS) (1GB RAM)
 * Checkmark the `User data` box and then paste the following text into the user data textarea:
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
apt-get install -y wireguard wireguard-tools
``` 
 * Assign an SSH pubkey from your workstation/laptop. 
  * If you haven't got one yet, click `New SSH Key` and follow the on-screen
    directions.
 * For this example, set the hostname to `k3s-wireguard`.
 * Finish creating the droplet.
 * Once created, assign a floating ip address to the droplet. (optional.)
 * Create domain name(s) and DNS that points to your droplet's (floating) ip address.
 * From your workstation, login to watch the cloud-init progress complete:
```
ssh root@<IP-address> cloud-init status -w
```
 * Wait for cloud-init to report `status: done`
 
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

## Connect local wireguard client to cluster

First make sure you are running a firewall locally, in order to to deny incoming
connections by default. Enabling the default `ufw` settings will do this for
you, but may have to install `ufw` first, and you must `enable` it:

```
sudo ufw enable
```

The default settings will block most incoming ports, but will still allow SSH.
If you don't need SSH, run `sudo ufw deny ssh`. For any ports you need to open,
you could run `sudo ufw allow [PORT]`, or to restrict based upon origin, you
could run `sudo ufw allow [PORT] from [ADDRESS]`

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
 * Install [go](https://golang.org/doc/install) and make sure `$HOME/go/bin` is
   in your `PATH` environment variable.
   * In your `$HOME/.bash_profile` (or maybe `.bashrc` depending) put: `export PATH=$HOME/go/bin:$PATH`
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

Check to see both the interface *and* the peer are connected:

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

Ping the peer to test (the first ip of the allowed ips from `sudo wg`):
```
ping -c3 10.4.0.1
```

Test it works the other direction by pinging your local workstation from the k3s node:
```
ssh root@<Public_IP_Address> ping -c3 10.5.0.1
```

If it didn't work, check that the interface acquired the ip address (`10.5.0.1`):

```
ip addr show dev wg-k3s
```

Debug any systemd-networkd errors that might be in the log:

```
journalctl --unit systemd-networkd
```

Check the kilo logs:

```
kubectl logs -n kube-system -l app.kubernetes.io/name=kilo
```

## Create the traefik reverse proxy

Make sure to change to the directory where you cloned this repository

```
cd $DIR
```

Make a copy of the included `prod-template` directory and call it `prod`:

```
cp -a src/prod-template/ src/prod
```

(`src/prod` is in the `.gitignore` file, so your changes in this directory are
not stored in git. If you wish to commit them, remove this line from
`.gitignore`)

The `src/prod` directory is now your directory to make configuration changes. 

Edit `src/prod/030-traefik-daemonset-patch.yaml`:

 * Choose the Lets Encrypt CA server for staging or prod. Use the
   `acme-staging-v02` until you are finished testing, but when you want to
   install permanently, change it to `acme-v02` to use the production Lets
   Encrypt server.
 * Edit your email address for Lets Encrypt certificates

Now apply the configuration to the cluster:

```
kubectl apply -k src/prod/traefik
```
