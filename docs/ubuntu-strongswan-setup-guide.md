# Ubuntu-strongswan-vm (strongswan-gw): On-Prem-to-GCP VPN Gateway — Full Setup Guide

This documents the complete build of the home-network VPN gateway VM — from Hyper-V creation through a working, redundant HA VPN + BGP connection into GCP, including the real troubleshooting encountered along the way (kept in, since it's the most instructive part).

**Purpose of this VM:** it's the on-prem side of a genuine site-to-site connection between a home network and a GCP VPC — running `strongSwan` (IPsec tunnels) and `FRR` (BGP dynamic routing), entirely software-based, avoiding the need for hardware VPN gear or router-level IPsec support.

---

## Part 1 — Create the VM in Hyper-V Manager

1. **Hyper-V Manager → Action → New → Virtual Machine**
2. **Name:** `Ubuntu-vm` (hostname later set to `strongswan-gw`)
3. **Generation:** Generation 1
4. **Assign Memory:** `2048` MB, check **"Use Dynamic Memory for this virtual machine."**
5. **Configure Networking:** initially attempted **External Switch** bound to the host's physical Ethernet adapter — this failed (see Troubleshooting below); ultimately used **`Default Switch`**.
6. **Connect Virtual Hard Disk:** new VHDX, 40 GB, dynamically expanding
7. **Installation Options:** Install from a bootable image file (.iso) — Ubuntu Server ISO
8. **Finish**

### Troubleshooting: networking didn't work at first

**Attempt 1 — External Switch:** Created a Virtual Switch Manager entry bound to the host's Ethernet adapter and connected the VM to it. Result: `eth0` inside the VM showed **"not connected," DHCP autoconfiguration failed.**

**Root cause:** the host (OPTIPLEX-PC) connects to the internet via **Wi-Fi**, not the wired Ethernet port — so the External Switch was bound to a physical adapter with no actual link. Hyper-V's External Switch bridging to Wi-Fi adapters is also known to be unreliable even when attempted (Microsoft's own Wi-Fi-bridging feature frequently fails to hand the VM a working IP, and can disrupt the host's own Wi-Fi).

**Fix — switched to Default Switch:** Hyper-V's built-in NAT-based switch, which works regardless of whether the host uses Wi-Fi or Ethernet, since Windows manages the NAT itself.
- Settings → Network Adapter → changed **Virtual switch** from `ExternalSwitch` to **`Default Switch`**
- Boot confirmed: `eth0` got a real DHCP-assigned IP (`172.24.x.x/20` range) with working internet access

This is the same reason `Ubuntu-sqlserver-vm` was built on Default Switch from the start.

---

## Part 2 — Install Ubuntu Server

Standard install flow (same as documented for the SQL Server VM): language, keyboard, "Use an entire disk" with LVM and **no LUKS**, profile setup (username `bikram`, hostname `strongswan-gw`), **enable OpenSSH server**, skip snaps, reboot, eject ISO.

Confirm networking post-install:
```bash
ip a show eth0
```
```bash
ping -c 4 google.com
```

---

## Part 3 — Install strongSwan and FRR

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y strongswan strongswan-pki libcharon-extra-plugins
sudo apt install -y frr
```

Note: the installed strongSwan build uses the modern `swanctl`/`charon-systemd` configuration system, **not** the legacy `ipsec.conf`/`ipsec` command-line tool — config lives under `/etc/swanctl/conf.d/`, and the `swanctl` command (not `ipsec`) manages it.

---

## Part 4 — GCP side: create the HA VPN gateway (Console)

1. **Console → Network Connectivity → VPN → Create a VPN → High-availability (HA) VPN**
2. **VPN gateway name:** `home-vpn-gateway`, **Network:** `my-vpc`, **Region:** `us-central1`, **IP version:** IPv4, **IP stack type:** IPv4 (single-stack)
3. GCP auto-allocates two external interface IPs:
   - Interface 0: `35.242.125.164`
   - Interface 1: `34.184.18.124`
4. **Peer VPN gateway:** "On-prem or Non Google Cloud" → new peer gateway named `home-strongswan-gw`, one interface, IP = home network's public IP (found via `curl -4 ifconfig.me` on the VM): `106.219.121.170`
5. **Add VPN tunnel (tunnel 1):**
   - High availability: **"Create a single VPN tunnel"**
   - Routing: Dynamic (BGP), new Cloud Router `home-vpn-router`, ASN `65001`, BGP identifier auto-assigned, advertise all subnets (default)
   - Associated Cloud VPN gateway interface: `0 : 35.242.125.164`
   - Associated peer VPN gateway interface: `0 : 106.219.121.170`
   - Name: `home-tunnel-1`, IKE version: IKEv2, pre-shared key: generated/entered
6. **Configure BGP session:**
   - Type: IPv4, Name: `home-bgp-session`, **Peer ASN: `65002`** (must differ from the Cloud Router's own ASN `65001`)
   - Allocate BGP IPv4 address: Automatically
   - Result: Cloud Router BGP IP `169.254.193.169`, Peer BGP IP `169.254.193.170`

---

## Part 5 — VM side: configure tunnel 1

**Disable strongSwan's automatic route installation** (critical — its default route-injection conflicts with the manual VTI-based routing used here):
```bash
sudo tee -a /etc/strongswan.d/charon.conf << 'EOF'
charon {
    install_routes = no
}
EOF
```

**Tunnel 1 connection config:**
```bash
sudo mkdir -p /etc/swanctl/conf.d
sudo tee /etc/swanctl/conf.d/gcp-tunnel1.conf << 'EOF'
connections {
    gcp-tunnel1 {
        version = 2
        local_addrs = %any
        remote_addrs = 35.242.125.164
        local {
            auth = psk
            id = 106.219.121.170
        }
        remote {
            auth = psk
            id = 35.242.125.164
        }
        children {
            gcp-tunnel1-child {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                mode = tunnel
                start_action = start
                dpd_action = restart
                updown = /etc/ipsec-vti.sh
                mark_in = 42
                mark_out = 42
            }
        }
        proposals = aes256-sha256-modp2048
    }
}
EOF
```

**Pre-shared key:**
```bash
sudo tee /etc/swanctl/conf.d/gcp-secrets.conf << 'EOF'
secrets {
    ike-1 {
        id-1 = 106.219.121.170
        id-2 = 35.242.125.164
        secret = "on-prem-gcp"
    }
}
EOF
```

**VTI (route-based tunnel) interface script** — creates a virtual tunnel interface carrying the BGP link-local IPs:
```bash
sudo tee /etc/ipsec-vti.sh << 'EOF'
#!/bin/bash
PLUTO_MARK_OUT_ARR=(${PLUTO_MARK_OUT//// })
PLUTO_MARK_IN_ARR=(${PLUTO_MARK_IN//// })
case "$PLUTO_VERB" in
    up-client)
        ip link add vti1 type vti local $PLUTO_ME remote $PLUTO_PEER okey ${PLUTO_MARK_OUT_ARR[0]} ikey ${PLUTO_MARK_IN_ARR[0]} 2>/dev/null
        ip addr add 169.254.193.170/30 remote 169.254.193.169/30 dev vti1 2>/dev/null
        ip link set vti1 up mtu 1436
        sysctl -w net.ipv4.conf.vti1.disable_policy=1
        sysctl -w net.ipv4.conf.vti1.rp_filter=0
        ;;
    down-client)
        ip link del vti1 2>/dev/null
        ;;
esac
EOF
sudo chmod +x /etc/ipsec-vti.sh
```

**Load and bring up:**
```bash
sudo systemctl restart strongswan
sudo swanctl --load-all
sudo swanctl --list-sas
```
Result: `gcp-tunnel1: ESTABLISHED`, child SA `INSTALLED`.

---

## Part 6 — Configure FRR (BGP)

```bash
sudo sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo systemctl restart frr
sudo vtysh
```

Inside `vtysh`:
```
configure terminal
router bgp 65002
neighbor 169.254.193.169 remote-as 65001
no bgp ebgp-requires-policy
address-family ipv4 unicast
neighbor 169.254.193.169 activate
redistribute connected
exit-address-family
exit
write memory
exit
```

**Note on `no bgp ebgp-requires-policy`:** modern FRR enforces RFC 8212 by default, silently discarding all inbound/outbound routes on an eBGP session unless an explicit route policy exists — even with no filters configured. This command disables that strict requirement (appropriate for a lab; production would instead define explicit route-maps).

**Verify:**
```bash
sudo vtysh -c "show ip bgp summary"
sudo vtysh -c "show ip bgp"
sudo vtysh -c "show ip route bgp"
```
Confirmed: BGP session `Established`, GCP's `10.0.0.0/24` subnet received and installed in the kernel routing table via `vti1`.

---

## Part 7 — Add a second tunnel (HA pair)

GCP flagged `home-tunnel-1` as *"not properly configured"* for HA VPN, since a full HA gateway expects tunnels on both of its interfaces. A second tunnel was added on interface `1 : 34.184.18.124`, to the same peer.

**GCP Console:** VPN → `home-vpn-gateway` → Add VPN tunnel → select existing peer gateway `home-strongswan-gw` → **Associated Cloud VPN gateway interface: `1 : 34.184.18.124`** → same Cloud Router `home-vpn-router` → name `home-tunnel-2` → IKEv2 → same pre-shared key.

BGP session for tunnel 2: Peer ASN `65002`, resulting link-local IPs: Cloud Router BGP IP `169.254.99.253`, Peer BGP IP `169.254.99.254`.

**VM side — tunnel 2 config:**
```bash
sudo tee /etc/swanctl/conf.d/gcp-tunnel2.conf << 'EOF'
connections {
    gcp-tunnel2 {
        version = 2
        local_addrs = %any
        remote_addrs = 34.184.18.124
        local {
            auth = psk
            id = 106.219.121.170
        }
        remote {
            auth = psk
            id = 34.184.18.124
        }
        children {
            gcp-tunnel2-child {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                mode = tunnel
                start_action = start
                dpd_action = restart
                updown = /etc/ipsec-vti2.sh
                mark_in = 43
                mark_out = 43
            }
        }
        proposals = aes256-sha256-modp2048
    }
}
EOF

sudo tee /etc/swanctl/conf.d/gcp-secrets2.conf << 'EOF'
secrets {
    ike-2 {
        id-1 = 106.219.121.170
        id-2 = 34.184.18.124
        secret = "on-prem-gcp"
    }
}
EOF

sudo tee /etc/ipsec-vti2.sh << 'EOF'
#!/bin/bash
PLUTO_MARK_OUT_ARR=(${PLUTO_MARK_OUT//// })
PLUTO_MARK_IN_ARR=(${PLUTO_MARK_IN//// })
case "$PLUTO_VERB" in
    up-client)
        ip link add vti2 type vti local $PLUTO_ME remote $PLUTO_PEER okey ${PLUTO_MARK_OUT_ARR[0]} ikey ${PLUTO_MARK_IN_ARR[0]} 2>/dev/null
        ip addr add 169.254.99.254/30 remote 169.254.99.253/30 dev vti2 2>/dev/null
        ip link set vti2 up mtu 1436
        sysctl -w net.ipv4.conf.vti2.disable_policy=1
        sysctl -w net.ipv4.conf.vti2.rp_filter=0
        ;;
    down-client)
        ip link del vti2 2>/dev/null
        ;;
esac
EOF
sudo chmod +x /etc/ipsec-vti2.sh

sudo swanctl --load-all
sudo swanctl --list-sas
```
Result: both `gcp-tunnel1` and `gcp-tunnel2` `ESTABLISHED`.

**Add the second BGP neighbor in FRR:**
```bash
sudo vtysh
```
```
configure terminal
router bgp 65002
neighbor 169.254.99.253 remote-as 65001
address-family ipv4 unicast
neighbor 169.254.99.253 activate
exit-address-family
exit
write memory
exit
```

**Verify both sessions:**
```bash
sudo vtysh -c "show ip bgp summary"
```
Both neighbors `Established`.

---

## Part 8 — The real root cause: link-local source addresses dropped by GCP

Even with both tunnels and BGP fully established, an actual `psql` connection attempt to the Cloud SQL PostgreSQL instance (`10.0.0.2`) still timed out. Extensive diagnosis (GCP Connectivity Test tool showing "REACHABLE" both ways, firewall rules confirmed, PSC endpoint confirmed `ACCEPTED`, both tunnels confirmed carrying encrypted traffic in both directions) ruled out every obvious cause.

**Root cause, found via `tcpdump -i vti1 -n port 5432`:** outbound TCP packets were sourced from **`169.254.193.170`** — the VTI interface's own link-local address (RFC 3927) — because Linux's routing selects the source address based on the outgoing interface. GCP's Private Service Connect infrastructure silently drops packets carrying a link-local source address, since it cannot build a meaningful connection-tracking/return-routing entry for a non-globally-routable source.

**Fix — SNAT the VTI interfaces' outbound traffic to the VM's real subnet IP:**
```bash
sudo iptables -t nat -A POSTROUTING -o vti1 -j SNAT --to-source 172.24.107.2
sudo iptables -t nat -A POSTROUTING -o vti2 -j SNAT --to-source 172.24.107.2
```
(replace `172.24.107.2` with the VM's actual current `eth0` IP — Default Switch NAT can reassign this on reboot)

This immediately resolved the issue — `psql` connected successfully.

---

## Part 9 — Persist the SNAT rules across reboot

```bash
sudo apt install -y iptables-persistent
```
When prompted **"Save current IPv4 rules?"** — select **Yes**. This saves the rules to `/etc/iptables/rules.v4`, auto-loaded on every boot via the `netfilter-persistent` systemd service.

Verify:
```bash
sudo cat /etc/iptables/rules.v4
```
Confirms both `SNAT --to-source 172.24.107.2` rules for `vti1` and `vti2` are saved.

---

## Result

A fully working, redundant (2-tunnel) HA VPN connection from a home network to GCP, using entirely free/open-source software (`strongSwan` + `FRR`) on a Hyper-V VM, with dynamic BGP routing exchanging routes automatically in both directions, and a non-obvious PSC/link-local-address issue correctly diagnosed and fixed. Verified end-to-end with a live, authenticated `psql` session against a Cloud SQL for PostgreSQL instance — proving genuine private connectivity, not just a configured-but-unverified tunnel.
