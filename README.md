# Overview

Documentation of how to use Pfsense to utilize Policy Based Routing (PBR) to a remote cloud virtual private server. This will allow you to setup custom routes for specific hosts in your network to route out of a VPS public IP.

> [!WARNING]
> I am using nftables in this setup. Installing nftables over iptables "might" override your iptables ruleset.

# VPS setup:

## Install required packages:

Install all the required packages on your VPS.

```bash
sudo apt update; sudo apt upgrade -y; \
sudo apt install ssh wireguard nftables coreutils -y
```

## Add kernel parameters to allow IP forwarding:

Add the IP forwarding kernel module:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

Apply kernel modules:

```bash
sudo sysctl -p
```

## Nftables ruleset:

Place the config below in the following file `/etc/nftables.conf`.

> [!IMPORTANT]
> 
> Change the following parameters to meet your needs:.
> 
> - `DEV_WORLD`: The WAN interface of the VPS. You will need to find what interface on your VPS has a public IP on it.
> - `Local_Networks`: The ranges of your local network behind your Pfsense router. You can seperate multiple ranges with a comma.
> - `SSH_In_Port`: SSH port to allow through nftables ruleset. Not adding this means you won't be able to SSH into your VPS instance.
> - `WireGuard_In_Port`: The WireGuard port to listen on from WAN. I recommend changing this to anything other than the default port (default port is 51820).
> - `IPv4_DEV_WireGuard_MSS`: This is the Maximum Segment Size (MSS) value that will be set for outbound TCP connections.

Nftables configuration:

```bash
#!/usr/sbin/nft -f
flush ruleset

## Change the values below to meet your needs:
# VPN Interface
define DEV_WireGuard = wg0
# WAN Interface
define DEV_WORLD = enp1s0
# Local networks from the wireguard tunnel
define Local_Networks = { 10.0.0.0/8 }
# SSH port
define SSH_In_Port = 22
# WAN inbound Wireguard port
define WireGuard_In_Port = 51820
# MSS override
# Note: This is assuming that you are using an MTU value of 1420 on the WireGuard tunnel interface.
define IPv4_DEV_WireGuard_MSS = 1380
##########################################
#### Don't edit anything below this line.#
##########################################

table inet global {
    chain inbound_world {
        # allow SSH connections from internet host.
        tcp dport $SSH_In_Port accept
        # allow wireguard connections from the internet.
        udp dport $WireGuard_In_Port accept
    }
    chain inbound_private_lan {
        # accepting ping (icmp-echo-request) for diagnostic purposes. I prefer allowing ping from inside the tunnel only.
        icmp type echo-request limit rate 10/second accept
        # allow SSH connections from the wireguard tunnel.
        tcp dport $SSH_In_Port accept
    }
    chain inbound {
        type filter hook input priority 0; policy drop;
        # Allow traffic from established and related packets, drop invalid
        ct state vmap { established : accept, related : accept, invalid : drop }
        # allow loopback traffic, anything else jump to chain for further evaluation
        iifname vmap { lo : accept, $DEV_WORLD : jump inbound_world, $DEV_WireGuard : jump inbound_private_lan }
        # the rest is dropped by the above policy
    }
    chain forward {
        type filter hook forward priority 0; policy drop;
        # MSS clamping
        iifname $DEV_WORLD oifname $DEV_WireGuard ip daddr $Local_Networks tcp flags syn tcp option maxseg size set $IPv4_DEV_WireGuard_MSS counter
        # Allow traffic from established and related packets, drop invalid
        ct state vmap { established : accept, related : accept, invalid : drop }
        # connections from the internal net to the internet: Malicious to WAN allowed; WAN to Malicious not allowed
        meta iifname . meta oifname { $DEV_WireGuard . $DEV_WORLD, $DEV_WORLD . $DEV_WireGuard } accept
        # the rest is dropped by the above policy
    }
    chain output {
        type filter hook output priority filter; policy accept;
        # Accept and counter outbound traffic
        counter accept
    }
    chain input {
        type nat hook prerouting priority -100; policy accept;
        # Destination NAT
    }
    chain outgoing {
        type nat hook postrouting priority 100; policy accept;
        # NAT Masquerade private IP addresses when forwarding traffic to the internet
        ip saddr $Local_Networks meta oifname $DEV_WORLD counter masquerade random,persistent
    }
}
```

Apply the nftables ruleset:

> [!NOTE]
> You might face an error where nftables is unable to restart. This can happen if the kernel module for NAT masquerade is not loaded.
> You just need to reboot your instance `sudo reboot now` and that should fix the issue.

```bash
sudo systemctl enable --now nftables; sudo systemctl reload nftables
```

## WireGuard setup:

### Generate WireGuard keys:

You can use this baby script that I made to generate all the WireGuard keys for your VPS and Pfsense router. Run the commands below in a bash shell to generate all the required WireGuard keys.

WireGuard keys generation baby script:

> [!NOTE]
> You will need the values from the output of these commands later. Paste them in an empty file.

```bash
SERV_PRIV_KEY=$(wg genkey); \
SERV_PUB_KEY=$(echo "${SERV_PRIV_KEY}" | wg pubkey); \
CLIENT_PRIV_KEY=$(wg genkey); \
CLIENT_PUB_KEY=$(echo "${CLIENT_PRIV_KEY}" | wg pubkey); \
PRESHARED_KEY=$(wg genpsk); \
echo "VPS private key: ${SERV_PRIV_KEY}"; \
echo "VPS public Key: ${SERV_PUB_KEY}"; \
echo "Pfsense private key: ${CLIENT_PRIV_KEY}"; \
echo "Pfsense public Key: ${CLIENT_PUB_KEY}"; \
echo "Preshared Key: ${PRESHARED_KEY}"
```

### VPS WireGuard configuration:

> [!IMPORTANT]
> Things to change in the WireGuard configuration of the VPS:
> 
> - Be sure to change the `ListenPort` value to match the `WireGuard_In_Port` value in the nftables rulebase configuration.
> - Replace the `localPrivateKeyAbcAbcAbc=` value with the `VPS private key` value from the script in the previous section.
> - Replace the `remotePublicKeyAbcAbcAbc=` value with the `Pfsense public key` value from the script in the previous section.
> - Replace the `presharedkeyabcabcabc=` value with the `Preshared key` value from the script in the previous section.
> - The `AllowedIPs` field needs to include all the IPs that will be forwarded up the tunnel as well as the Pfsense WireGuard tunnel interface IP.

VPS WireGuard configuration:

```bash
[Interface]
# Name = VPS interface settings
Address = 198.51.100.1/30
ListenPort = 51820
PrivateKey = localPrivateKeyAbcAbcAbc= 
MTU = 1420 

[Peer] 
# Name = site2.example.local
AllowedIPs = 198.51.100.2/32, 10.0.0.0/8
PublicKey = remotePublicKeyAbcAbcAbc=
PresharedKey = presharedkeyabcabcabc=
```

Place the configuration above in the following file: `/etc/wireguard/wg0.conf`.

Make sure to change the permission of the WireGuard folder after you place your config in the `wg0.conf` file.

```bash
sudo chown root:root -R /etc/wireguard; sudo chmod 600 -R /etc/wireguard
```

Enable and start the WireGuard tunnel:

```bash
sudo systemctl enable --now wg-quick@wg0
```

### Pfsense configuration:

> [!NOTE]
> I am still working on this section 🙂.
