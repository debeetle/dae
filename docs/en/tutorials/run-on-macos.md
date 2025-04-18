# Run on macOS

## Install brew

### For x86

You can install brew referring to official docs <https://docs.brew.sh/Installation>:

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

### For ARM64

To install ARM64 architecture packages, homebrew should be installed in `/opt/homebrew`:

```shell
cd /opt
sudo mkdir homebrew
sudo chown $(whoami):admin homebrew
curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
```

## Lima

### Setup

This section intruduces how to use [lima](https://github.com/lima-vm/lima) virtual machine to run dae, and proxy whole macOS host network.

First, we should install `lima` and `socket_vmnet`. And install `socket_vmnet` [from binary](https://github.com/lima-vm/socket_vmnet#from-binary) is recommended to avoid the 'not owned by "root"' error.

```shell
# Install lima for VM.
brew install lima

# Install socket_vmnet from binary for bridge
VERSION="$(curl -fsSL https://api.github.com/repos/lima-vm/socket_vmnet/releases/latest | jq -r .tag_name)"
FILE="socket_vmnet-${VERSION:1}-$(uname -m).tar.gz"

# Download the binary archive
curl -OSL "https://github.com/lima-vm/socket_vmnet/releases/download/${VERSION}/${FILE}"

# Install /opt/socket_vmnet from the binary archive
sudo tar Cxzvf / "${FILE}" opt/socket_vmnet

# Set up the sudoers file for launching socket_vmnet from Lima
limactl sudoers >etc_sudoers.d_lima
sudo install -o root etc_sudoers.d_lima /etc/sudoers.d/lima
```

Then, configure lima configuration and dae VM configuration.

```shell
# Configure lima networks.
socket_vmnet_bin=/opt/socket_vmnet/bin/socket_vmnet
sed -ir "s#^ *socketVMNet:.*#  socketVMNet: \"${socket_vmnet_bin}\"#" ~/.lima/_config/networks.yaml
```

```shell
# Configure dae vm.
mkdir ~/.lima/dae/
cat << 'EOF' | tee ~/.lima/dae/lima.yaml
images:
### amd64 or arm64
# - location: "https://cloud.debian.org/images/cloud/bookworm/daily/latest/debian-12-generic-amd64-daily.qcow2"
#   arch: "x86_64"
#   digest: "sha512: Paste the SHA512SUMS here and uncomment this line if you want security check"
- location: "https://cloud.debian.org/images/cloud/bookworm/daily/latest/debian-12-generic-arm64-daily.qcow2"
  arch: "aarch64"
# digest: "sha512: Paste the SHA512SUMS here and uncomment this line if you want security check"
mounts:
networks:
- lima: bridged
  interface: "lima0"
memory: "1GB"
disk: "3GiB"
EOF
```

Start dae VM and configure it.

```shell
# Start dae VM.
limactl start dae
```

```shell
# Enter the dae VM.
limactl shell dae

# Manually configure network.
cat << 'EOF' | sudo tee /etc/netplan/99-override.yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            dhcp4-overrides:
                route-metric: 200
        lima0:
            dhcp4: true
            dhcp6: true
    version: 2
EOF

# Apply netplan.
sudo netplan apply

# Install requirements.
sudo apt-get install jq

# Install dae.
sudo bash -c "$(wget -qO- https://github.com/daeuniverse/dae-installer/raw/main/installer.sh)" @ install

# If you have difficulty accessing GitHub, you can use this command instead.
sudo bash -c "$(curl -sL https://cdn.jsdelivr.net/gh/daeuniverse/dae-installer/installer.sh)" @ install use-cdn

# Configure config.dae.
cat << 'EOF' | sudo tee /usr/local/etc/dae/config.dae
global {
  lan_interface: lima0
  wan_interface: lima0

  log_level: info
  allow_insecure: false
  auto_config_kernel_parameter: true
}

subscription {
  # Fill in your subscription links here.
}

# See https://github.com/daeuniverse/dae/blob/main/docs/en/configuration/dns.md for full examples.
dns {
  upstream {
    googledns: 'tcp+udp://dns.google:53'
    alidns: 'udp://dns.alidns.com:53'
  }
  routing {
    request {
      fallback: alidns
    }
    response {
      upstream(googledns) -> accept
      ip(geoip:private) && !qname(geosite:cn) -> googledns
      fallback: accept
    }
  }
}

group {
  proxy {
    #filter: name(keyword: HK, keyword: SG)
    policy: min_moving_avg
  }
}

# See https://github.com/daeuniverse/dae/blob/main/docs/en/configuration/routing.md for full examples.
routing {
  pname(NetworkManager) -> direct
  dip(224.0.0.0/3, 'ff00::/8') -> direct

  ### Write your rules below.

  dip(geoip:private) -> direct
  dip(geoip:cn) -> direct
  domain(geosite:cn) -> direct

  fallback: proxy
}
EOF
sudo chmod 0600 /usr/local/etc/dae/config.dae

# Do not forget to add your subscriptions and nodes.
sudo vim /usr/local/etc/dae/config.dae

# Enable and start dae.
sudo systemctl enable --now dae.service

# Exit dae vm.
exit
```

Set default route of macOS to dae VM.

> **Note**
> You may need to execute this command every time you connect to network.
>
> Refer to [Auto set route and DNS](#auto-set-route-and-dns) if you want to auto execute it.

```shell
# Get IP of dae VM.
dae_ip=$(limactl shell dae ip --json addr | limactl shell dae jq -cr '.[] | select( .ifname == "lima0" ).addr_info | .[] | select( .family == "inet" ).local')
# Set gateway of macOS host to dae VM.
sudo route delete default; sudo route add default $dae_ip
# Set DNS of macOS host to dae VM.
networksetup -setdnsservers Wi-Fi $dae_ip
```

Verify that we were successful.

```shell
# Verify.
curl -v ipinfo.io
```

### Auto set route and DNS

Write a script to execute.

```shell
# The script to execute.
mkdir -p /Users/Shared/bin
cat << 'EOF' > /Users/Shared/bin/dae-network-update.sh
#!/bin/sh
set -ex
export PATH=$PATH:/opt/local/bin/:/opt/homebrew/bin/
dae_ip=$(limactl shell dae ip --json addr | limactl shell dae jq -cr '.[] | select( .ifname == "lima0" ).addr_info | .[] | select( .family == "inet" ).local')
current_gateway=$(route -n get default|grep gateway|rev|cut -d' ' -f1|rev)

# Take notice of the network interface, which you can find by `networksetup -listallnetworkservices'
networksetup -getdnsservers Wi-Fi | cut -d" " -f1 | grep -E '\.|:' && dns_override=1
[ ! -z "$dae_ip" ] && ping -c 1 -t 1 -n "$dae_ip" && dae_ready=1
[ -z "$dae_ready" ] && [ ! -z "$dns_override" ] && (networksetup -setmanual Wi-Fi 1.1.1.1 1.1.1.1/32 1.1.1.1; networksetup -setdhcp Wi-Fi; networksetup -setdnsservers Wi-Fi "Empty"; exit 1)
[ "$current_gateway" != "$dae_ip" ] && (sudo route delete default; sudo route add default $dae_ip)
networksetup -setdnsservers Wi-Fi $dae_ip
exit 0
EOF

# Give executable permission.
chmod +x /Users/Shared/bin/dae-network-update.sh
```

Give no-password permission for route.

```shell
if [ $(id -u) -eq "0" ]; then echo 'Do not use root!!'; else echo "$(whoami) ALL=(ALL) NOPASSWD: $(which route)" | sudo tee /etc/sudoers.d/"$(whoami)"-route; fi
```

Write a plist service file.

```shell
cat << 'EOF' > ~/Library/LaunchAgents/org.v2raya.dae.networkchanging.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" \
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>org.v2raya.dae.networkchanging</string>

  <key>LowPriorityIO</key>
  <true/>

  <key>ProgramArguments</key>
  <array>
    <string>/Users/Shared/bin/dae-network-update.sh</string>
  </array>

  <key>WatchPaths</key>
  <array>
    <string>/etc/resolv.conf</string>
    <string>/Library/Preferences/SystemConfiguration/NetworkInterfaces.plist</string>
    <string>/Library/Preferences/SystemConfiguration/com.apple.airport.preferences.plist</string>
  </array>

  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
EOF
```

Load the plist service.

```shell
launchctl load ~/Library/LaunchAgents/org.v2raya.dae.networkchanging.plist
```
