# k3s-wireguard-proxy

A script to automatically create a k3s cluster on DigitalOcean, and connect your
local workstation to it through wireguard (VPN). This allows you to access
private cluster IP addresses directly from your workstation, as well as to
expose select services from your workstation to the cluster and/or the internet
through traefik (reverse proxy).

You can use this like a self-hosted ngrok.

## How it works

This is a self-contained bash script that will do all of the hard work for you
and walks you through the rest of the easy config:

 * Configures and creates a single-node k3s cluster on DigitalOcean ($5/mo
   droplet) utilizing [k3sup](https://github.com/alexellis/k3sup).
 * Installs [kilo](https://github.com/squat/kilo), which creates a wireguard
   service inside your k3s cluster network.
 * Creates a wireguard interface on your local workstation.
 * Maintains your connection through
   [systemd-networkd](https://wiki.archlinux.org/index.php/Systemd-networkd)

You will need bash, git, wireguard, kubectl, kgctl, etc. But don't worry about
that, the script walks you through all of it, so just get started by downloading
the script:

curl -sLS https:// | sh
