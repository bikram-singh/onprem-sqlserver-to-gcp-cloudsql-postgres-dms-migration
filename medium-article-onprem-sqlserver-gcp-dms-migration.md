# A Real DMS Migration, Built From a Home Network: On-Prem SQL Server to GCP Cloud SQL for PostgreSQL

Most migration tutorials show you the happy path — click here, click there, done in ten minutes. This isn't that. This is what actually happens when you build a real on-prem-to-cloud migration from a home network: two Hyper-V VMs, a hand-rolled Cloud VPN connection, a subtle networking bug that took hours to trace, a missing BGP route that silently broke Database Migration Service, and finally, a working pipeline moving four SQL Server databases into Cloud SQL for PostgreSQL.

If you're planning something similar — or just want to see what real troubleshooting looks like instead of a sanitized console walkthrough — this is the whole thing, start to finish, with nothing smoothed over.

---

## Why I Built This

There's a specific kind of learning that only happens when nothing is handed to you pre-configured. Anyone can follow a quickstart guide where the source database is already sitting in a neighboring GCP project with connectivity already solved. Almost nobody documents what happens when your source is a single Windows machine on a home Wi-Fi connection with a dynamic public IP, no static addressing, no business-grade router, and no existing VPN infrastructure of any kind.

That gap is exactly why this project exists: to build a **genuine** on-prem-to-GCP migration, from the actual physical and network constraints most home labs and small businesses start with, using nothing but free and open-source software on the on-prem side, and Google's managed services on the cloud side.

---

## Architecture Overview

```
[Home Network — Wi-Fi, dynamic public IP]
        │
        ├── Ubuntu-strongswan-vm (Hyper-V, Default Switch)
        │     strongSwan (IPsec) + FRR (BGP)
        │     → HA VPN tunnels to GCP
        │
        └── Ubuntu-sqlserver-vm (Hyper-V, Default Switch)
              SQL Server 2025 on Linux
              5 databases: MyDatabase, LibraryDB,
              HospitalDB, InventoryDB, SchoolDB
                    │
                    │ (same NAT'd subnet as strongswan-gw,
                    │  advertised to GCP via BGP)
                    ▼
[GCP: Cloud VPN — HA VPN Gateway] ── [VPC: my-vpc] ── [Cloud SQL for PostgreSQL: my-postgres]
                              │
                              └── [DMS: Private Connectivity → Conversion Workspace → Migration Jobs]
```

Two deliberate decisions shaped everything downstream, and they're worth stating up front because the rest of the article keeps referring back to them:

1. **Two separate VMs, not one.** A VPN gateway and a database server are different roles with different failure modes. Keeping them apart matches real production architecture, and — as you'll see — it sidesteps a genuinely ugly double-NAT/port-forwarding problem.
2. **SQL Server on Linux, not Windows.** The natural instinct is to just point the migration at whatever's already running. But the "obvious" source — SQL Server on the Windows Hyper-V host itself — sits on a different network path than the VPN gateway, which would have meant DNAT rules and real complexity. Running SQL Server 2025 natively on Ubuntu, inside a VM on the *same* subnet as the gateway, made it directly reachable with zero extra routing tricks.

---

## Part 1: Building the Two VMs in Hyper-V

### The Wi-Fi trap

The first VM, `Ubuntu-strongswan-vm` (hostname later set to `strongswan-gw`), was created the standard way: Hyper-V Manager → New → Virtual Machine, Generation 1, 2048 MB of Dynamic Memory, a 40 GB dynamically-expanding VHDX, Ubuntu Server ISO attached as the install source.

For networking, the obvious first instinct was an **External Switch** — bridge the VM directly onto the physical network, so it looks like just another device on the LAN. It failed immediately. Inside the VM, `eth0` showed "not connected," and DHCP autoconfiguration never completed.

The root cause took a moment to place: OPTIPLEX-PC (the physical host) connects to the internet over **Wi-Fi**, not the wired Ethernet port. The External Switch had been dutifully bound to a physical Ethernet adapter that had no cable plugged into it at all — a completely dead link. And even if a cable *had* been available, Hyper-V's Wi-Fi bridging support is separately known to be unreliable: Microsoft ships a "Wi-Fi Direct Virtual Adapter" mechanism meant to make this work, but it frequently fails to hand the guest VM a working IP, and in some cases can disrupt the *host's own* Wi-Fi connection mid-attempt.

The fix was switching to **Default Switch** instead — Hyper-V's built-in NAT-based virtual switch. Unlike External Switch, it doesn't try to bridge a physical adapter at all; Windows manages the NAT translation itself, which means it works identically whether the host is on Wi-Fi or Ethernet. The moment the VM's network adapter was reassigned from `ExternalSwitch` to `Default Switch`, `eth0` picked up a real DHCP-assigned address in the `172.24.x.x/20` range, with working internet access.

### VM 1 — `strongswan-gw`

With networking sorted, the rest of the Ubuntu Server install followed a standard flow: English language, default keyboard layout, "Use an entire disk" for storage with LVM enabled and **LUKS encryption deliberately left unchecked** — a LUKS passphrase prompt on every boot would block headless, unattended startup, which is fatal for a server VM you can't always sit in front of.

Profile setup used username `bikram` and server name `strongswan-gw`. The single most important checkbox in the entire install was easy to miss: **"Install OpenSSH server."** Without it, there's no way to manage a VM with no attached display once the install finishes — you'd be stuck working exclusively through the (much clunkier) Hyper-V console window for everything that follows.

### VM 2 — `Ubuntu-sqlserver-vm`

Same install flow, same Default Switch decision, but sized differently to match its heavier workload: **4096 MB** of memory (SQL Server needs real headroom that the lightweight gateway VM never will) and a **127 GB** virtual disk. Hostname: `ubuntu-sqlserver`.

Both VMs landed cleanly inside `172.24.96.0/20` — `strongswan-gw` at `172.24.107.2`, `ubuntu-sqlserver` at `172.24.100.130`. This shared subnet becomes the single most load-bearing detail in the entire architecture: it's the one range that eventually gets advertised to GCP over BGP, and it's precisely *why* putting SQL Server in its own VM on this subnet — rather than leaving it on the Windows host, which sits outside this range entirely — avoids needing any port-forwarding at all.

---

## Part 2: SQL Server 2025 on Linux

Installing SQL Server on Ubuntu turned out to be refreshingly close to installing any other Linux package — a world away from the Windows installer's VC++ Redistributable dance that had actually caused real friction earlier in the on-prem setup (a known SQL Server 2025 issue requiring a *repair*, not just an install, of the Visual C++ 2015–2022 Redistributable before setup would proceed).

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget curl gpg gnupg2 software-properties-common apt-transport-https lsb-release ca-certificates
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
curl -fsSL https://packages.microsoft.com/config/ubuntu/24.04/mssql-server-2025.list | sudo tee /etc/apt/sources.list.d/mssql-server-2025.list
sudo apt update
sudo apt install -y mssql-server
sudo /opt/mssql/bin/mssql-conf setup
```

The setup wizard prompts for an edition — **4 (Express)** was chosen specifically to match what the original Windows instance ran, sets the `sa` password, and starts the `mssql-server` systemd service automatically once license terms are accepted.

One detail worth flagging for anyone following along: this used Microsoft's Ubuntu **24.04** repository track — the officially GA-supported version — even though the VM itself was running Ubuntu **26.04**. Technically an unsupported combination. In practice, it installed and ran without a single compatibility hiccup, which is worth knowing if you're on a similarly bleeding-edge distro release and hesitant to try.

Command-line tools (`sqlcmd`) followed the identical repository-registration pattern:

```bash
curl -fsSL https://packages.microsoft.com/config/ubuntu/24.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
sudo apt update
sudo apt install -y mssql-tools18 unixodbc-dev
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
source ~/.bashrc
```

### Rebuilding five databases from scratch

The original Windows instance carried five sample databases with real relational structure — foreign keys, indexes, realistic row counts:

- **MyDatabase** — an HR/sales dataset: Departments, Employees, Projects, EmployeeProjects, Customers, Products, Orders, OrderDetails
- **LibraryDB** — Books, Members, Loans
- **HospitalDB** — Doctors, Patients, Appointments
- **InventoryDB** — Warehouses, Items, StockMovements
- **SchoolDB** — Students, Courses, Enrollments

Rather than transferring `.sql` files across the network, both setup scripts were recreated directly on the VM through an SSH session, using a `tee` heredoc:

```bash
cat > multi_db_setup.sql << 'SQLEOF'
-- full T-SQL: CREATE DATABASE, CREATE TABLE, INSERT statements for
-- LibraryDB, HospitalDB, InventoryDB, SchoolDB
SQLEOF
sqlcmd -S localhost -U sa -P 'YourSAPassword' -C -i multi_db_setup.sql
```

This choice wasn't arbitrary — it came directly out of a real, repeated annoyance: both `vi` and `nano` kept mangling multi-line pastes over SSH (missing characters, garbled indentation, occasional dropped lines mid-paste), while a `tee`-based heredoc handled every large paste cleanly, every single time, across dozens of similar operations throughout this project. If you're doing heavy SSH-based scripting work, this is a small habit worth adopting early.

A full row-count verification query across all fifteen tables confirmed an exact match against the original Windows dataset — 30 employees, 5 departments, 12 books, 6 doctors, 4 warehouses, and so on down the list, all fifteen numbers matching precisely.

### Proving remote reachability before GCP ever enters the picture

One check mattered more than it might initially seem: confirming SQL Server was actually listening on *all* network interfaces, not just `localhost`.

```bash
sudo ss -tlnp | grep 1433
# LISTEN 0.0.0.0:1433 ...
# LISTEN *:1433 ...
```

And then proving it from a genuinely separate machine — SSH'd into `strongswan-gw`, installed the same client tools there, and connected across the network to `172.24.100.130:1433`. A clean SQL Server version banner came back immediately. That's precisely the kind of cross-machine access DMS would eventually need to replicate from much further away (from inside Google's network, through a VPN tunnel) — confirming it worked at the shortest possible distance first, before adding layers of complexity on top, saved a lot of guesswork later when things did break.

---

## Part 3: Cloud VPN — Building the GCP Side

This is the part that deserves its own dedicated section, because it's easy to undersell just how much of this project's real complexity lives here.

### HA VPN vs. Classic VPN

Google Cloud offers two flavors of Cloud VPN. **Classic VPN** is the older option — supports only static routing, single-tunnel by default, and is generally considered legacy for new deployments. **HA VPN (High-Availability VPN)** is the modern, recommended choice: it supports dynamic routing via BGP, and — critically — it's architected from the ground up as a **two-interface resource**, specifically so that a properly configured HA VPN gateway maintains two independent tunnel paths for a 99.99% SLA.

That two-interface design is the single fact that shapes almost everything else in this section, including a warning message that initially looked like something had gone wrong.

### Creating the HA VPN Gateway

In the GCP Console: **Network Connectivity → VPN → Create a VPN → High-availability (HA) VPN.**

- **Name:** `home-vpn-gateway`
- **Network:** `my-vpc`
- **Region:** `us-central1`
- **IP version:** IPv4
- **IP stack type:** IPv4 (single-stack)

The moment this gateway is created, Google auto-allocates **two external IP addresses** — one per interface — regardless of how many tunnels you end up building. In this project, those came out to `35.242.125.164` (interface 0) and `34.184.18.124` (interface 1). Both of these IPs exist from the very first click, whether or not a tunnel is ever attached to the second one.

### Defining the peer gateway

GCP needs to know what it's tunneling *to*. Under **Peer VPN gateway**, the option **"On-prem or Non Google Cloud"** was selected, and a new peer gateway resource was created — named `home-strongswan-gw` — representing the home network's own endpoint. Since this is a genuine home network with a single public-facing connection (not a redundant multi-ISP setup), it was configured with **one interface**, pointed at the home network's actual public IP.

Finding that IP required nothing more elaborate than:

```bash
curl -4 ifconfig.me
```
```
106.219.121.170
```

That single IP address is what GCP's peer gateway resource is built around — it's the identity GCP uses to recognize which incoming connection attempts are legitimately from this home network's tunnel.

### The first tunnel

With both the local gateway and the peer gateway defined, the first VPN tunnel was created:

- **High availability:** "Create a single VPN tunnel" (deliberately — see below for why this changed)
- **Routing:** Dynamic (BGP)
- **Cloud Router:** a new router, `home-vpn-router`, ASN `65001`
- **Associated Cloud VPN gateway interface:** `0 : 35.242.125.164`
- **Associated peer VPN gateway interface:** `0 : 106.219.121.170`
- **Name:** `home-tunnel-1`
- **IKE version:** IKEv2
- **Pre-shared key:** generated via GCP's own key generator

That pre-shared key deserves its own callout: GCP shows it to you **exactly once**, with an explicit warning that it cannot be retrieved after the form closes. Losing it isn't catastrophic, but it does mean deleting and recreating the tunnel from scratch to set a new one — GCP doesn't support editing a tunnel's shared secret after creation, on the theory that a secret you can rotate through the UI isn't really a secret. (This happened during the build — the original key wasn't saved, and the tunnel had to be deleted and recreated with a fresh key that *was* properly recorded.)

Configuring the BGP session that rides on top of this tunnel produced a pair of **link-local addresses** (from the `169.254.0.0/16` range reserved for exactly this purpose) — `169.254.193.169` on GCP's Cloud Router side, `169.254.193.170` on the peer side — with the peer's ASN set to `65002`, deliberately different from the Cloud Router's own `65001`, since BGP fundamentally requires two distinct autonomous systems on either end of a session.

### The warning that wasn't actually a failure

Once `home-tunnel-1` was up and the VM-side configuration (covered in Part 4) had it fully `ESTABLISHED`, GCP's Console displayed a persistent, easy-to-misread warning:

> **"The following HA VPN tunnels are not properly configured: home-tunnel-1."**

This is worth being precise about, because it's the single easiest part of this whole project to describe wrong. **`home-tunnel-1` was never broken.** At no point did it stop working, drop its connection, or fail to carry traffic. The warning is Google's Console simply reminding you that an HA VPN gateway, by design, has two interfaces — and you've only attached a tunnel to one of them. It's a configuration-completeness warning, not a health alert.

The fix was adding a **second tunnel**, `home-tunnel-2`, to the *same* peer gateway, this time on the gateway's other interface:

- **Associated Cloud VPN gateway interface:** `1 : 34.184.18.124`
- **Associated peer VPN gateway interface:** `0 : 106.219.121.170` (same single home IP — one peer, tunneled from both of GCP's interfaces)
- Same Cloud Router (`home-vpn-router`), same pre-shared key for simplicity
- **Name:** `home-tunnel-2`

This produced its own independent BGP session, with its own link-local pair: `169.254.99.253` (Cloud Router) and `169.254.99.254` (peer). Once the matching VM-side configuration was added (again, Part 4), both tunnels showed `ESTABLISHED`, both BGP sessions showed `Established`, and the earlier warning cleared entirely.

Think of it this way: this was never "tunnel A failed, so we built tunnel B to replace it." It's "the bridge was designed with two lanes, and we'd only opened one." Both lanes belong to the same bridge, run to the same destination, and were open and usable simultaneously by the end of this step.

### What Cloud VPN actually provides — and what it doesn't

It's worth being explicit about the division of labor here, because it becomes important in Part 5. **Cloud VPN's HA VPN Gateway handles exactly one thing: encrypted tunnel endpoints.** It terminates IPsec on Google's side, matching whatever the on-prem side sends. It does **not**, by itself, know anything about routing, doesn't understand what's reachable through the tunnel, and doesn't move a single packet toward any actual destination inside your VPC on its own.

That's the Cloud Router's job — and by extension, BGP's job, and by extension, FRR's job on the VM side. Cloud VPN gets you an encrypted pipe. Something else has to tell both ends what belongs on the other side of that pipe. That's exactly where Part 4 picks up.

---

## Part 4: strongSwan + FRR — The On-Prem Side of the Tunnel

```bash
sudo apt install -y strongswan strongswan-pki libcharon-extra-plugins
sudo apt install -y frr
```

Worth flagging immediately for anyone following along with older strongSwan tutorials online: this install uses the modern `swanctl`/`charon-systemd` configuration system, **not** the legacy `ipsec.conf`/`ipsec` command-line tool most older guides assume. Configuration lives under `/etc/swanctl/conf.d/`, and the `swanctl` command — not `ipsec` — is what manages the running daemon. This distinction cost real early time: a perfectly well-formed `ipsec.conf` file was written and populated correctly, only for the running `charon` daemon to simply never read it, because it wasn't looking there at all.

### Disabling automatic route installation

Before writing any tunnel configuration, one setting had to be changed globally, and it's easy to skip because nothing in a typical strongSwan guide calls it out as mandatory for this specific setup:

```bash
sudo tee -a /etc/strongswan.d/charon.conf << 'EOF'
charon {
    install_routes = no
}
EOF
```

By default, strongSwan tries to manage kernel routing tables itself whenever a tunnel comes up — perfectly reasonable behavior for a simple policy-based VPN, and directly destructive here, where FRR is meant to be the sole authority over the routing table via BGP-learned routes. With both trying to manage the same routes, the result was a genuinely confusing failure mode discovered later: `ip route get <destination>` would resolve to the *wrong* interface (the regular default gateway instead of the VPN tunnel), even though the BGP session itself showed as perfectly healthy.

### Tunnel 1 configuration

```
connections {
    gcp-tunnel1 {
        version = 2
        local_addrs = %any
        remote_addrs = 35.242.125.164
        local  { auth = psk; id = 106.219.121.170 }
        remote { auth = psk; id = 35.242.125.164 }
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
```

With a matching `secrets` block holding the pre-shared key generated back in Part 3. The `local_ts`/`remote_ts` values of `0.0.0.0/0` mean this is deliberately a **route-based** VPN (traffic selectors covering "everything"), rather than a traditional **policy-based** VPN with specific subnet-to-subnet selectors — route-based is what makes dynamic BGP routing over the tunnel possible at all, since the actual routing decisions get delegated entirely to the kernel routing table rather than baked into the IPsec policy itself.

### The VTI interface — where IPsec meets BGP

The `updown` script is where this all becomes concrete. It creates a **VTI (Virtual Tunnel Interface)** — a genuine Linux network interface that the kernel can route through, carrying exactly the link-local IPs GCP's BGP session assigned:

```bash
#!/bin/bash
PLUTO_MARK_OUT_ARR=(${PLUTO_MARK_OUT//// })
PLUTO_MARK_IN_ARR=(${PLUTO_MARK_IN//// })
case "$PLUTO_VERB" in
    up-client)
        ip link add vti1 type vti local $PLUTO_ME remote $PLUTO_PEER \
            okey ${PLUTO_MARK_OUT_ARR[0]} ikey ${PLUTO_MARK_IN_ARR[0]} 2>/dev/null
        ip addr add 169.254.193.170/30 remote 169.254.193.169/30 dev vti1 2>/dev/null
        ip link set vti1 up mtu 1436
        sysctl -w net.ipv4.conf.vti1.disable_policy=1
        sysctl -w net.ipv4.conf.vti1.rp_filter=0
        ;;
    down-client)
        ip link del vti1 2>/dev/null
        ;;
esac
```

This is genuinely the architectural crux of the whole IPsec side: without a VTI interface, an IPsec tunnel is just an encrypted policy applied to matching traffic — there's no actual network interface for a routing protocol like BGP to run over. The VTI interface turns the tunnel into something that looks, to the rest of the Linux networking stack, like any other network card: something with an IP address, something `ip route` can point traffic at, something FRR can establish a BGP neighbor relationship across.

```bash
sudo systemctl restart strongswan
sudo swanctl --load-all
sudo swanctl --list-sas
```

Result: `gcp-tunnel1: ESTABLISHED`, with its child SA showing `INSTALLED` — genuine encrypted traffic now able to flow.

### FRR — teaching the VM to speak BGP

```bash
sudo sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo systemctl restart frr
sudo vtysh
```

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
```

The `no bgp ebgp-requires-policy` line is the single easiest thing to skip in this entire config, and among the most expensive to miss. Modern FRR enforces **RFC 8212** by default — a rule requiring an explicit inbound and outbound routing policy on any eBGP session (a session between two different autonomous systems, which this is) before *any* routes are accepted or advertised, even with zero actual filters configured anywhere. Without disabling this, the BGP session itself shows as cleanly `Established` — keepalives flowing, session healthy — while silently exchanging **zero routes** in either direction. `show ip bgp neighbor` eventually surfaced the actual giveaway, buried in its detailed output:

```
Inbound updates discarded due to missing policy
Outbound updates discarded due to missing policy
0 accepted, 0 sent prefixes
```

A perfectly healthy-looking BGP session that was, functionally, doing nothing. With `no bgp ebgp-requires-policy` applied, `show ip bgp` immediately began showing GCP's `10.0.0.0/24` subnet, correctly installed into the kernel routing table via `vti1`.

### Wiring up tunnel 2

The VM-side configuration for `home-tunnel-2` mirrors tunnel 1 almost exactly — a second `swanctl` connection block pointed at GCP's second interface (`34.184.18.124`), a second VTI script (`vti2`, carrying the new link-local pair `169.254.99.253`/`.254`), and a second FRR BGP neighbor statement. Both tunnels came up `ESTABLISHED`; both BGP sessions came up `Established`; FRR's routing table began showing genuine multipath entries (`*=`) for routes reachable via either tunnel — real redundancy, not just a second connection sitting idle.

---

## Part 5: The Bug That Ate an Afternoon — Link-Local Packets Dropped Silently

With two tunnels, dynamic BGP routing, and a Cloud SQL for PostgreSQL instance (`my-postgres`, reachable via Private Service Connect at `10.0.0.2`) all independently confirmed as correctly configured, a simple connection attempt —

```bash
psql -h 10.0.0.2 -U postgres -d postgres
```

— still timed out. Every single time.

What made this genuinely difficult wasn't a lack of tools — it was that **every layer checked out clean.** GCP's own Connectivity Test tool, run directly from Cloud Shell, reported `REACHABLE` in both directions, tracing the simulated packet's entire path all the way to *"packet delivered to Cloud SQL instance."* The relevant firewall rule allowed the right source range on the right port. The PSC endpoint's connection status showed `ACCEPTED`. `swanctl --list-sas` showed real, growing byte counters in *both* the `in` and `out` directions across both tunnels — genuine encrypted traffic was flowing, not just idle keepalives.

The eventual answer came from watching actual packets leave the interface, rather than trusting any tool's summary of what *should* happen:

```bash
sudo tcpdump -i vti1 -n port 5432
```

```
169.254.193.170.35680 > 10.0.0.2.5432: Flags [S], seq ...
```

There it was, sitting in plain sight the whole time. The outbound SYN packet's **source address** was `169.254.193.170` — the VTI interface's own link-local address, straight out of the `169.254.0.0/16` block reserved by RFC 3927 for exactly this kind of link-local, non-globally-routable use. Linux, by default, selects a packet's source address based on the interface it's leaving through — and since the route to `10.0.0.2` correctly pointed out `vti1`, that interface's own address became the packet's source, entirely automatically, with no explicit configuration choosing it.

GCP's Connectivity Test never caught this, and it's worth understanding exactly why: it's a **static configuration analyzer**. It checks whether the *path* — the routes, the firewall rules, the peering — is theoretically correct for a packet with the source address *you tell it to assume*. It was never told to assume a link-local source, because nobody realized that's what would actually happen.

The real underlying mechanism: **GCP's Private Service Connect infrastructure silently drops packets carrying a link-local source address.** PSC needs to build a connection-tracking and return-routing entry for every flow it handles — a record of "this specific source needs replies routed back this specific way." A link-local address, by its very nature, isn't something PSC can build a meaningful, globally-unambiguous entry for. So the SYN packet simply vanishes on arrival. No RST, no ICMP error, nothing — which is precisely why it looks, from every diagnostic tool above the packet level, identical to a generic network timeout with no discernible cause.

The fix, once the actual cause was identified, was two lines of `iptables`:

```bash
sudo iptables -t nat -A POSTROUTING -o vti1 -j SNAT --to-source 172.24.107.2
sudo iptables -t nat -A POSTROUTING -o vti2 -j SNAT --to-source 172.24.107.2
```

Rewriting the VTI interfaces' outbound source address to the VM's real, globally-meaningful subnet address (`172.24.107.2`, sitting inside `172.24.96.0/20` — the very range BGP was already correctly advertising) instead of the link-local tunnel address. `psql` connected successfully on the very next attempt, with no other change to anything.

```bash
sudo apt install -y iptables-persistent
```

— selecting "Yes" when prompted to save current IPv4 rules, so this fix survives every future reboot via the `netfilter-persistent` systemd service, rather than needing to be reapplied by hand.

This single discovery is, without much competition, the most broadly useful thing in this entire project for anyone doing similar work: **if you're running a route-based VPN (VTI interfaces) into GCP, and anything on the other end sits behind Private Service Connect, SNAT your tunnel-bound traffic to a real, routable subnet address before it leaves the tunnel interface. Never let it leave with a link-local source.**

---

## Part 6: A Second, Quieter Gap — DMS Couldn't Find the Route Either

Everything in Part 5 proved connectivity *from the gateway VM itself* into GCP. Weeks later, once Database Migration Service private connectivity was configured (see Part 8), an eerily familiar symptom reappeared: DMS's own connection tests to the SQL Server source kept failing with a plain **"Connect timed out"** — the exact same generic message, for what turned out to be an almost identical underlying cause, in a completely different part of the stack.

Tracing this one followed the same pattern that had worked before: `tcpdump` on `strongswan-gw`'s `vti1` interface showed DMS's SYN packets *arriving correctly* — sourced from `10.250.0.2`, the private connectivity peering range. So packets were reaching the tunnel. The problem was one hop further back: checking `ip route get 10.250.0.2` from `strongswan-gw` showed it resolving via the regular default gateway (`eth0`), not via either VPN tunnel at all.

The actual cause, found by cross-referencing GCP's route table:

```bash
gcloud compute routes list --project=migration-project-111 --filter="network:my-vpc" \
  --format="table(name,destRange,nextHopVpnTunnel)"
```

```
NAME                              DEST_RANGE      NEXT_HOP_VPN_TUNNEL
peering-route-c56af68ef660120e    10.250.0.0/29
```

That last column — `NEXT_HOP_VPN_TUNNEL` — was **empty**. GCP's Cloud Router had never actually advertised the DMS private-connectivity peering range (`10.250.0.0/29`) back over BGP to the home network at all. The reason: **VPC peering only automatically exchanges subnet routes by default — never custom, dynamically-learned routes.** The `10.250.0.0/29` range is exactly that kind of route: it exists only because of a peering relationship DMS itself created behind the scenes, not because it's a "real" subnet that belonged to `my-vpc` from the start. Cloud Router's default "advertise all subnets" behavior genuinely doesn't cover it.

The fix required switching the Cloud Router's advertisement mode from its default to **Custom**, and explicitly re-adding *both* pieces side by side — the existing subnet advertisement (so nothing already working broke) plus the new peering range:

```bash
gcloud compute routers update home-vpn-router \
  --region=us-central1 \
  --advertisement-mode=CUSTOM \
  --set-advertisement-groups=ALL_SUBNETS \
  --set-advertisement-ranges=10.250.0.0/29 \
  --project=migration-project-111
```

Once applied, `show ip bgp` on `strongswan-gw` immediately showed `10.250.0.0/29` alongside the existing `10.0.0.0/24`, and `ip route get 10.250.0.2` correctly resolved via `vti1`. DMS's connection test passed on the very next attempt.

The broader lesson here echoes Part 5 almost exactly, just one layer higher in the stack: **anything that introduces a new IP range into your VPC after the fact — a new peering, a new private connectivity configuration, a new PSC endpoint — needs to be explicitly checked against whatever's actually being advertised over your dynamic routing sessions.** "It's reachable inside the VPC" and "it's reachable *through your VPN* from outside the VPC" are two genuinely different questions, and GCP's default behaviors don't automatically make the second one true just because the first one is.

---

## Part 7: Forwarding Between the Two VMs

Even with tunnels, BGP, and the SNAT fix all working, one more gap remained specific to the two-VM split described back in the architecture overview. GCP's traffic reaches `strongswan-gw` — the actual tunnel endpoint — but the real destination, `ubuntu-sqlserver`, is a *different* machine on the shared subnet. Two more settings on `strongswan-gw` closed this gap:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo tee /etc/sysctl.d/99-ip-forward.conf << 'EOF'
net.ipv4.ip_forward=1
EOF
sudo iptables -t nat -A POSTROUTING -o eth0 -d 172.24.96.0/20 -j MASQUERADE
sudo netfilter-persistent save
```

The first line enables the Linux kernel's core packet-forwarding capability — disabled by default on a standard Ubuntu install, since most machines aren't meant to route traffic for other machines. The persisted `/etc/sysctl.d/99-ip-forward.conf` file matters more than it might look: a plain `sysctl -w` change is *not* persistent by itself, and this setting reverting silently across a reboot was, in fact, the exact cause of a full afternoon's worth of confusing "it worked yesterday" debugging later in the project, before the dedicated persistence file was added.

The second line — the MASQUERADE rule — ensures that when `ubuntu-sqlserver` replies to a request that arrived via the tunnel, that reply gets correctly routed back out through `strongswan-gw` and the tunnel, rather than attempting to go directly out `ubuntu-sqlserver`'s own default gateway (which has no idea what a GCP VPN tunnel even is).

This forwarding requirement is, again, precisely the cost the "two VMs" architectural decision knowingly took on in exchange for avoiding something worse. The alternative — SQL Server running directly on the Windows host — would have needed a genuine DNAT (destination NAT) setup bridging *two separate NAT layers* (the home router's NAT, plus Windows' own Default Switch NAT), which is a meaningfully messier problem than one clean, subnet-level IP forward and MASQUERADE rule between two VMs that already share an address space.

### A memory lesson, learned the hard way

Partway through this phase of testing, SQL Server started disappearing without warning:

```
Active: failed (Result: oom-kill)
```

Both VMs, sharing one physical host's RAM pool through Hyper-V's Dynamic Memory feature, had quietly ballooned down under host-level pressure — a `free -h` check on the SQL Server VM showed only **2.1 GiB total**, despite the VM being explicitly configured with a 4096 MB Startup memory setting. Dynamic Memory, it turns out, can shrink a running VM below what you configured as its starting point if the host genuinely doesn't have enough free physical RAM to sustain it — and a starved SQL Server process is exactly the kind of thing the Linux OOM killer terminates first.

The real fix required two separate changes working together:

1. **Explicit Minimum RAM floors** in each VM's Hyper-V Dynamic Memory settings — `3072` MB minimum for the SQL Server VM (guaranteeing Dynamic Memory could never shrink it below what SQL Server genuinely needs to stay alive), while trimming the lightweight gateway VM down to a leaner `1024` MB Startup / `512` MB Minimum, freeing up host headroom for the VM that actually needed it.
2. **An explicit memory ceiling inside SQL Server itself**, so the database engine would never try to claim more RAM than the VM could actually sustain in the first place:
   ```bash
   sudo /opt/mssql/bin/mssql-conf set memory.memorylimitmb 2800
   ```

The general lesson: a Hyper-V "Minimum RAM" setting isn't merely a performance-tuning knob — without one, Dynamic Memory can silently starve a VM's workload to the point of an out-of-memory kill, with symptoms (a service that "just crashes sometimes") that look nothing like a memory problem until you specifically go looking for one.

Static IP addresses, configured via netplan on both VMs, closed out the infrastructure-hardening work — removing a smaller but persistent source of friction where DHCP-assigned addresses occasionally shifted after a reboot, silently invalidating SNAT rules and connection-profile hostnames that had been hardcoded against the previous address.

---

## Part 8: Database Migration Service, End to End

With connectivity fully proven at every layer, the actual migration work — the part everything above existed to enable — could finally begin.

### Private Connectivity: giving DMS a way in

Database Migration Service doesn't run inside your VPC at all. It runs from a completely separate, Google-managed tenant project, and needs its own explicit bridge into `my-vpc` before it can reach anything on the other side of that bridge — including, eventually, everything reachable via the VPN tunnels built in Parts 3 and 4.

**Database Migration → Private connectivity → Create Private Connection**, with a genuinely important fork in the road: GCP offers two distinct connectivity methods here, **PSC Interfaces** and **VPC peering**, and they're built for opposite situations. PSC Interfaces is meant for when you *don't* already have VPC-level connectivity to your source and want DMS to reach it through a dedicated network attachment you create specifically for that purpose. **VPC peering** is meant for exactly this project's situation — an existing VPC that already has VPN or Interconnect connectivity to the source, which DMS can simply piggyback on rather than needing its own separate path built from scratch.

VPC peering was configured against `my-vpc`, with a peering range (`10.250.0.0/29`) deliberately chosen to avoid overlapping the existing `10.0.0.0/24` subnet. This is the exact peering relationship whose missing BGP advertisement caused the whole detour documented in Part 6.

### Connection Profiles: one pair per database

Each of the four databases ultimately migrated needed its own matched pair of connection profiles:

- A **source** profile — SQL Server engine, hostname `172.24.100.130`, port `1433`, the `sa` login, private connectivity via the configuration above, and critically, the **exact source database name** (`MyDatabase`, `LibraryDB`, `HospitalDB`, or `InventoryDB`).
- A **destination** profile — pointing at the existing `my-postgres` Cloud SQL instance, the `postgres` login, and the matching lowercase destination database name (`mydatabase`, `librarydb`, and so on — PostgreSQL lowercases unquoted identifiers by convention).

One early mistake here deserves a full explanation, because it produced a genuinely confusing symptom. The very first source connection profile had its "Database name" field left at the SQL Server default, `master`. DMS's schema-pull step *succeeded* — no error at all — because `master` is a perfectly real, connectable database. But `master` is SQL Server's own internal system database; it has no user tables. The conversion workspace consequently showed "0 schemas" with real user tables entirely absent, not because anything had failed, but because it had faithfully converted the schema of a database that was never meant to be migrated in the first place. The fix was creating a fresh connection profile with the actual target database name explicitly specified — after which the schema pull correctly surfaced all eight real tables.

### Conversion Workspace: translating SQL Server into PostgreSQL

DMS's Conversion Workspace exists because SQL Server (T-SQL) and PostgreSQL are genuinely different SQL dialects — this isn't a raw data copy, it's an actual schema translation. Creating a workspace (`sqlserver-to-postgres-cw`) and pulling a schema snapshot produced a clean, largely-automatic result on the first real database: **31 total objects, 23 with zero issues, 8 flagged "awaiting review," 0 requiring hard manual action.**

Every single flagged object shared the identical underlying cause: `CLUSTERED INDEX is not yet implemented`. SQL Server makes every primary key a clustered index by default — meaning the table's actual rows are physically stored on disk in that key's order. PostgreSQL has genuinely no equivalent concept at all; its closest analog, the `CLUSTER` command, is a one-time manual reordering operation rather than a maintained table property. There is, quite simply, nothing for the deterministic converter to map this onto. The underlying primary key constraint and its supporting index still get created correctly, as a standard (non-clustered-concept) index — functionally equivalent for every practical query pattern this project's sample data would ever exercise. Safe to proceed without modification.

**Applying to destination** committed this converted schema — every table, primary key, foreign key, and index — onto `my-postgres`. On the two databases with denser foreign-key relationships (`LibraryDB` and `InventoryDB`), a first application attempt occasionally collided with a partially-applied leftover from an earlier attempt (`cannot drop table dbo.items because other objects depend on it`), resolved cleanly with a manual `DROP TABLE ... CASCADE` via `psql` on the conflicting child and parent tables before re-running the apply step, which then completed with zero issues.

### The Migration Job: One-time vs. Continuous

This is the step that actually moves data, and it forces a real architectural decision: **One-time** migration versus **Continuous** migration.

Continuous migration relies on SQL Server's native **Change Data Capture (CDC)** mechanism to keep replicating ongoing changes after the initial load — which in turn requires **SQL Server Agent** to be installed and running, since Agent is what processes CDC's underlying capture and cleanup jobs. And SQL Server Agent has never been available on **Express Edition**, on Windows or on Linux — this is a longstanding, deliberate licensing restriction, not a bug or an oversight.

Since this build's SQL Server instance was deliberately Express Edition (to match the original Windows source), switching to Developer Edition purely to unlock Agent for a proof-of-concept migration would have been solving a problem this project didn't actually have. **One-time migration** — a single full-dump phase, with no CDC phase and no Agent dependency whatsoever — turned out to be exactly the right fit, and it's worth being clear that this is a fully legitimate, first-class DMS migration type in its own right, not a lesser workaround for when Continuous "isn't available."

One warning from the migration job's built-in test deserves genuine explanation rather than a quick dismissal: DMS flagged that foreign keys couldn't be strictly enforced during a bulk, non-transactional load — naming specific tables like `orders` and `employeeprojects` whose foreign keys reference other tables being loaded in parallel, with no guarantee the parent row lands before the child row does. The recommended, and correct, fix:

```sql
ALTER USER postgres WITH REPLICATION;
```

Granting the destination database user PostgreSQL's `REPLICATION` attribute allows it to bypass foreign-key enforcement specifically *during* the bulk-loading process. This isn't a security compromise or a permanent weakening of the schema — the foreign key constraints remain fully defined and fully enforced immediately afterward; they're simply not actively re-checked row-by-row while thousands of rows are being loaded in parallel across multiple tables at once, which is standard, expected, and necessary behavior for essentially any bulk migration tool moving relational data with foreign keys intact.

With that single fix applied once, every subsequent migration job's test passed cleanly, and each job — for `MyDatabase`, `HospitalDB`, `InventoryDB`, and `LibraryDB` — ran to completion: **Status: Completed. Every table: Promoted. 0 errors**, across the board.

### Verification: the only check that actually counts

Console status pages are reassuring, but the only verification that genuinely matters is querying the destination directly and comparing it against the known source:

```sql
SELECT 'employees' AS tbl, COUNT(*) FROM dbo.employees
UNION ALL SELECT 'departments', COUNT(*) FROM dbo.departments
UNION ALL SELECT 'projects', COUNT(*) FROM dbo.projects
UNION ALL SELECT 'customers', COUNT(*) FROM dbo.customers
UNION ALL SELECT 'employeeprojects', COUNT(*) FROM dbo.employeeprojects
UNION ALL SELECT 'orders', COUNT(*) FROM dbo.orders
UNION ALL SELECT 'orderdetails', COUNT(*) FROM dbo.orderdetails
UNION ALL SELECT 'products', COUNT(*) FROM dbo.products;
```

Every table, across every migrated database, came back an exact match against the SQL Server source:

| Database | Verified row counts |
|---|---|
| **MyDatabase** | Employees 30, Departments 5, Projects 10, Customers 20, EmployeeProjects 40, Orders 30, OrderDetails 62, Products 15 |
| **HospitalDB** | Doctors 6, Patients 10, Appointments 12 |
| **InventoryDB** | Warehouses 4, Items 10, StockMovements 20 |
| **LibraryDB** | Books 12, Members 8, Loans 12 |

`SchoolDB` was deliberately set aside — its source connection profile kept hitting the identical connectivity-test timeout that ultimately turned out (in the other four databases' case) to trace back to the missing BGP route from Part 6, but by the time it recurred specifically for `SchoolDB`'s profile, four out of five databases already stood as clean, fully row-count-verified migrations — a solid, legitimate stopping point rather than a gap worth forcing through at the cost of the article's honesty about what was actually completed.

---

## Lessons Learned

A few patterns emerged repeatedly enough across this project that they're worth naming explicitly, independent of any single specific bug:

**A green checkmark from a config validator is not proof of a working data path.** GCP's Connectivity Test tool said "REACHABLE" throughout the entire Part 5 saga, because it validates configuration, not runtime packet behavior. The gap between "the config permits this" and "a real packet actually gets there" is exactly where the two hardest bugs in this project — the link-local source address, and the missing BGP route advertisement — both lived.

**`tcpdump`, watched at the right layer, resolves ambiguity that no higher-level tool can.** Both of this project's genuinely hard bugs were solved the same way: capturing traffic simultaneously on multiple interfaces across multiple machines, and comparing exactly what left one interface against exactly what arrived at the next. Every other diagnostic — Console error messages, `gcloud` describe output, connectivity test results — narrowed the search space usefully, but never actually identified the root cause on its own.

**A generic error message ("Connect timed out") can hide entirely different root causes at completely different layers of the stack.** The exact same symptom appeared twice in this project — once from a link-local source address getting silently dropped by PSC, once from a missing BGP route advertisement for a peering range — and both times, the surface-level error text gave no hint which one it was.

**Infrastructure settings that look like performance tuning can silently become availability problems.** Hyper-V's Dynamic Memory Minimum RAM setting reads like a performance knob. In practice, without it, Dynamic Memory starved a running SQL Server process to the point of an OOM kill — a failure mode that looks nothing like "slow" and everything like "randomly crashes."

**Persistence is not automatic, and its absence produces the most confusing kind of bug: "it worked yesterday."** `sysctl -w`, ad-hoc `iptables` rules, and DHCP-assigned addresses all share the same trap — they work perfectly in the moment, and then silently vanish across the next reboot, undoing a fix with no error message anywhere pointing back at what changed.

---

## What's Next

With four of five databases fully migrated and independently verified, a few directions remain open: circling back to resolve `SchoolDB`'s connectivity profile (very likely the same missing-BGP-route class of issue, just not yet explicitly re-diagnosed for that specific profile); exploring a genuine Continuous/CDC migration path by switching to SQL Server Developer Edition, to demonstrate near-zero-downtime cutover rather than a one-time full dump; and documenting the teardown process for anyone using this as a learning exercise rather than a production build, since every VM, VPN tunnel, and Cloud SQL instance here continues to accrue real GCP cost while running.

---

*Full setup guides, exact command sequences, and 146 screenshots documenting every step of this build are available in the companion repository: [`onprem-sqlserver-to-gcp-cloudsql-postgres-dms-migration`](https://github.com/bikram-singh/onprem-sqlserver-to-gcp-cloudsql-postgres-dms-migration).*
