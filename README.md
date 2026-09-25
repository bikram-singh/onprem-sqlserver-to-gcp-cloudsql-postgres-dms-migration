<div align="center">

# 🗄️ On-Prem SQL Server → GCP Cloud SQL for PostgreSQL

### A Real DMS Migration, Built From a Home Network

[![SQL Server](https://img.shields.io/badge/SQL_Server-2025_on_Linux-CC2927?logo=microsoftsqlserver&logoColor=white)](docs/ubuntu-sqlserver-setup-guide.md)
[![Cloud SQL](https://img.shields.io/badge/Cloud_SQL-PostgreSQL-4285F4?logo=googlecloud&logoColor=white)](#-architecture)
[![Database Migration Service](https://img.shields.io/badge/GCP-Database_Migration_Service-4285F4?logo=googlecloud&logoColor=white)](#-migrating-each-database)
[![strongSwan](https://img.shields.io/badge/strongSwan-IPsec_VPN-DA291C?logo=linux&logoColor=white)](docs/ubuntu-strongswan-setup-guide.md)
[![FRR](https://img.shields.io/badge/FRR-Dynamic_BGP-0052CC)](docs/ubuntu-strongswan-setup-guide.md)
[![Connectivity](https://img.shields.io/badge/Connectivity-Private_Only-2ECC71)](#-architecture)
[![Setup](https://img.shields.io/badge/Setup-Console--Only%2C_No_CI%2FCD-777777)](#-repository-structure)

---

*Most migration tutorials show you the happy path — click here, click
there, done in ten minutes. This isn't that. This is what actually
happens when you build a real on-prem-to-cloud migration from a home
network: two Hyper-V VMs, a hand-rolled Cloud VPN connection, a subtle
networking bug that took hours to trace, a missing route policy that
silently broke BGP, and finally, a working pipeline moving four SQL
Server databases into Cloud SQL for PostgreSQL — fully privately, with
no public IPs anywhere.*

</div>

---

## 🔗 Quick Links

📄 [**strongSwan / VPN Gateway Setup Guide**](docs/ubuntu-strongswan-setup-guide.md)

📄 [**SQL Server VM Setup Guide**](docs/ubuntu-sqlserver-setup-guide.md)

📸 [**146 Snapshots**](docs/snapshots)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [The Problem, In Plain Terms](#-the-problem-in-plain-terms)
- [Architecture](#-architecture)
- [Repository Structure](#-repository-structure)
- [Prerequisites](#-prerequisites)
- [Build Guide — The Two VMs](#-build-guide--the-two-vms)
- [Database Migration Service Setup](#-database-migration-service-setup)
- [Migrating Each Database](#-migrating-each-database)
- [Real Problems Found & Fixed](#-real-problems-found--fixed)
- [Verification](#-verification)
- [Screenshots](#-screenshots)
- [Repository](#-repository)
- [Result](#-result)

---

## 🌐 Overview

A genuine on-prem-to-GCP database migration, built entirely on a home
network — no lab-provided VPC, no pre-solved connectivity, no public IPs
at any point in the path. Four SQL Server databases running on a Linux
VM at home are migrated into **Cloud SQL for PostgreSQL** using GCP's
managed **Database Migration Service (DMS)**, reached over a
self-built, redundant **HA Cloud VPN** connection with dynamic **BGP**
routing.

### 🔑 Key Facts

| Property | Value |
|---|---|
| ☁️ **Cloud Platform** | Google Cloud Platform |
| 🔁 **Migration Service** | GCP Database Migration Service (DMS) — schema conversion + data migration |
| 🗄️ **Source** | SQL Server 2025 (Express) on Ubuntu Linux |
| 🐘 **Destination** | Cloud SQL for PostgreSQL (`my-postgres`, private IP only) |
| 🔐 **Connectivity** | HA Cloud VPN (2 tunnels) + dynamic BGP, DMS private connectivity, PSC — **zero public IPs** |
| 🖥️ **On-prem infrastructure** | 2 × Hyper-V Ubuntu Server VMs on a home network (Wi-Fi, dynamic public IP) |
| 🧱 **VPN software** | strongSwan (IPsec, route-based/VTI) + FRR (BGP) |
| 🏗️ **Deployment style** | Manual, GCP Console-driven — **no Terraform, no CI/CD** |
| 🗃️ **Databases migrated** | `MyDatabase`, `HospitalDB`, `InventoryDB`, `LibraryDB` |
| 📸 **Screenshots** | 146, organized by phase under [`docs/snapshots/`](docs/snapshots) |

---

## 🏭 The Problem, In Plain Terms

There's a specific kind of learning that only happens when nothing is
handed to you pre-configured. Anyone can follow a quickstart guide where
the source database already sits in a neighboring GCP project with
connectivity already solved. Almost nobody documents what happens when
the source is a single Windows machine on a home Wi-Fi connection, with
a dynamic public IP, no static addressing, no business-grade router, and
no existing VPN infrastructure of any kind.

That gap is exactly why this project exists: to build a **genuine**
on-prem-to-GCP migration, from the actual physical and network
constraints most home labs and small businesses start with, using free
and open-source software on the on-prem side and Google's managed
services on the cloud side — with private connectivity as a hard
requirement, not an afterthought.

---

## 🏛️ Architecture

```
[Home Network — Wi-Fi, dynamic public IP]
        │
        ├── strongswan-gw (Hyper-V, Default Switch)
        │     strongSwan (IPsec) + FRR (BGP)
        │     → HA VPN tunnels to GCP
        │
        └── ubuntu-sqlserver (Hyper-V, Default Switch)
              SQL Server 2025 on Linux
              5 databases: MyDatabase, LibraryDB,
              HospitalDB, InventoryDB, SchoolDB
                    │
                    │ (same NAT'd subnet as strongswan-gw,
                    │  advertised to GCP via BGP)
                    ▼
[GCP: Cloud VPN — HA VPN Gateway] ── [VPC: my-vpc] ── [Cloud SQL for PostgreSQL: my-postgres]
                              │
                              └── [DMS: Private Connectivity → Connection Profiles →
                                       Conversion Workspace → Migration Jobs]
```

### 🔄 Layer Breakdown

| Layer | Components |
|---|---|
| 🖥️ **On-prem compute** | Two Hyper-V VMs on `Default Switch` (NAT), same `172.24.96.0/20` subnet |
| 🔐 **VPN gateway** | `strongswan-gw` — strongSwan (route-based IPsec via VTI interfaces) + FRR (eBGP) |
| 🗄️ **Database host** | `ubuntu-sqlserver` — SQL Server 2025 Express on Ubuntu, same subnet as the gateway (no DNAT needed) |
| ☁️ **GCP connectivity** | HA VPN gateway (2 tunnels, 2 external IPs), Cloud Router, dynamic BGP |
| 🎯 **Destination** | Cloud SQL for PostgreSQL, private IP only, reached via Private Service Connect |
| 🔁 **Migration engine** | DMS: private connectivity configuration → source/destination connection profiles → conversion workspace (schema translation) → migration job (data copy) |

**Two deliberate decisions shaped everything downstream:**

1. **Two separate VMs, not one.** A VPN gateway and a database server are
   different roles with different failure modes. Keeping them apart
   matches real production architecture and sidesteps a genuinely ugly
   double-NAT/port-forwarding problem.
2. **SQL Server on Linux, not on the Windows host.** The "obvious"
   source — SQL Server running on the Windows Hyper-V host itself — sits
   on a different network path than the VPN gateway, which would have
   meant DNAT rules and real complexity. Running SQL Server 2025
   natively on Ubuntu, inside a VM on the *same* subnet as the gateway,
   made it directly reachable with zero extra routing tricks.

---

## 📁 Repository Structure

```
onprem-sqlserver-to-gcp-cloudsql-postgres-dms-migration/
│
├── README.md
│
└── docs/
    ├── ubuntu-strongswan-setup-guide.md   # VPN gateway VM — full build + troubleshooting
    ├── ubuntu-sqlserver-setup-guide.md    # SQL Server VM — full build + dataset creation
    │
    └── snapshots/                         # 146 screenshots, organized by phase
        ├── README.md
        ├── 01-vpn-networking-setup/              (51)
        ├── 02-dms-private-connectivity-setup/     (3)
        ├── 03-mydatabase/                        (33)
        ├── 04-hospitaldb/                        (18)
        ├── 05-inventorydb/                       (20)
        └── 06-librarydb/                         (21)
```

---

## ✅ Prerequisites

| Requirement | Details |
|---|---|
| 🖥️ **Hyper-V host** | Windows machine with Hyper-V enabled, internet via Wi-Fi or Ethernet |
| ☁️ **GCP Project** | Billing enabled, Database Migration Service + Cloud SQL Admin APIs enabled |
| 🌐 **A routable home public IP** | Dynamic is fine — found at setup time via `curl -4 ifconfig.me` |
| 🔐 **VPC + Cloud Router** | A VPC (`my-vpc`) in the target region, ready for HA VPN + BGP |
| 🐘 **Cloud SQL for PostgreSQL instance** | Private IP only, in the same region as the VPC |

---

## 🏗️ Build Guide — The Two VMs

The full, step-by-step build for both on-prem VMs — every command run,
every wrong turn taken, and the reasoning behind each fix — lives in two
companion guides under [`docs/`](docs):

### 📄 [`docs/ubuntu-strongswan-setup-guide.md`](docs/ubuntu-strongswan-setup-guide.md)
The VPN gateway VM (`strongswan-gw`). Covers:
- Hyper-V VM creation and the **Wi-Fi/External-Switch failure** that forced a switch to `Default Switch`
- strongSwan + FRR install, route-based IPsec via VTI interfaces
- Building the first HA VPN tunnel end-to-end (GCP Console + `swanctl` config) and verifying the BGP session
- Adding the **second tunnel** to complete the HA pair
- The real root-cause investigation into a silent connection timeout (link-local source addresses being dropped by GCP PSC) and the SNAT fix that resolved it
- Persisting the fix across reboots

### 📄 [`docs/ubuntu-sqlserver-setup-guide.md`](docs/ubuntu-sqlserver-setup-guide.md)
The database VM (`ubuntu-sqlserver`). Covers:
- Hyper-V VM creation on the same `Default Switch` subnet as the gateway
- Installing SQL Server 2025 (Express) natively on Ubuntu, plus `sqlcmd` tooling
- Recreating the full 5-database sample dataset (`MyDatabase`, `LibraryDB`, `HospitalDB`, `InventoryDB`, `SchoolDB`)
- Verifying every table's row count against the original dataset
- Confirming SQL Server is reachable **across the network**, not just from `localhost` — the same access path DMS needs

---

## 🔁 Database Migration Service Setup

Before any single database could be migrated, three one-time,
account-wide pieces were set up in the GCP Console (screenshots in
[`docs/snapshots/02-dms-private-connectivity-setup/`](docs/snapshots/02-dms-private-connectivity-setup)):

1. **Empty target databases** created on `my-postgres` — `mydatabase`, `hospitaldb`, `inventorydb`, `librarydb` (matching the source names)
2. **DMS private connectivity configuration** (`home-network-pc`) — VPC peering between the DMS service network and `my-vpc`, with a dedicated `10.250.0.0/29` IP range
3. Confirmed the configuration listed as active before creating any connection profiles

Every migration job below reuses this same private connectivity
configuration — it only needs to be created once.

---

## 🗃️ Migrating Each Database

Each database went through the same four-stage DMS flow: **connection
profile → conversion workspace (schema conversion) → migration job →
post-migration verification.** Screenshots for each stage are in that
database's folder under [`docs/snapshots/`](docs/snapshots).

| Database | Tables | Screenshots |
|---|---|---|
| **MyDatabase** — HR/sales dataset | `Departments`, `Employees`, `Projects`, `EmployeeProjects`, `Customers`, `Products`, `Orders`, `OrderDetails` | [`03-mydatabase/`](docs/snapshots/03-mydatabase) (33) |
| **HospitalDB** | `Doctors`, `Patients`, `Appointments` | [`04-hospitaldb/`](docs/snapshots/04-hospitaldb) (18) |
| **InventoryDB** | `Warehouses`, `Items`, `StockMovements` | [`05-inventorydb/`](docs/snapshots/05-inventorydb) (20) |
| **LibraryDB** | `Books`, `Members`, `Loans` | [`06-librarydb/`](docs/snapshots/06-librarydb) (21) |

**MyDatabase went first** and used the generic connection-profile /
conversion-workspace names (`sqlserver-source`, `postgres-destination`,
`sqlserver-to-postgres-cw`) as the template run. Each subsequent
database used its own explicitly-named profiles and workspace (e.g.
`sqlserver-librarydb`, `postgres-librarydb`, `librarydb-topostgres-cw`,
`librarydb-migration`) so all four could coexist in the same DMS project
without collisions.

> `SchoolDB` was created on-prem alongside the other four as part of the
> shared sample dataset but was not migrated in this pass.

---

## 🔧 Real Problems Found & Fixed

Most tutorials skip this part. This project didn't:

| Problem | Symptom | Fix |
|---|---|---|
| 🌐 **Wi-Fi + External Switch** | `eth0` "not connected," DHCP never completes on the gateway VM | Use Hyper-V's **`Default Switch`** (NAT-based) instead — works regardless of the host's Wi-Fi/Ethernet connection |
| 🔁 **HA VPN gateway needs both interfaces used** | GCP flags the tunnel as "not properly configured" for HA VPN | Add a **second tunnel** on the gateway's other external interface, to the same peer |
| 🧭 **FRR silently drops eBGP routes** | BGP session up, but no routes exchanged, despite no filters configured | `no bgp ebgp-requires-policy` — modern FRR enforces RFC 8212 by default and needs this explicitly disabled (or a real route-map, in production) |
| 👻 **DMS connection timed out despite "REACHABLE" connectivity tests** | `psql` to the Cloud SQL instance hangs, even with both tunnels and BGP fully established | `tcpdump` showed outbound packets sourced from the VTI interface's **link-local address** — GCP's PSC infrastructure silently drops these. Fixed with `iptables … -j SNAT --to-source <VM's real subnet IP>` on both VTI interfaces |
| 💾 **SNAT rule lost on reboot** | Connectivity breaks again after a VM restart | `iptables-persistent`, saved to `/etc/iptables/rules.v4`, auto-loaded via `netfilter-persistent` |

Full detail, exact commands, and the diagnostic steps for each are in
[`docs/ubuntu-strongswan-setup-guide.md`](docs/ubuntu-strongswan-setup-guide.md).

---

## ✅ Verification

Every table's row count was confirmed against the original dataset
before and after migration. Source-side counts (Ubuntu SQL Server),
confirmed in [`docs/ubuntu-sqlserver-setup-guide.md`](docs/ubuntu-sqlserver-setup-guide.md):

| Database | Table | Rows |
|---|---|---|
| MyDatabase | Departments | 5 |
| MyDatabase | Employees | 30 |
| MyDatabase | Projects | 10 |
| MyDatabase | Customers | 20 |
| MyDatabase | EmployeeProjects | 40 |
| MyDatabase | Products | 15 |
| MyDatabase | Orders | 30 |
| MyDatabase | OrderDetails | 62 |
| LibraryDB | Books | 12 |
| LibraryDB | Members | 8 |
| LibraryDB | Loans | 12 |
| HospitalDB | Doctors | 6 |
| HospitalDB | Patients | 10 |
| HospitalDB | Appointments | 12 |
| InventoryDB | Warehouses | 4 |
| InventoryDB | Items | 10 |
| InventoryDB | StockMovements | 20 |

Matching row counts were re-confirmed on the Cloud SQL for PostgreSQL
side after each migration job completed — see the verification
screenshots at the end of each database's folder under
[`docs/snapshots/`](docs/snapshots).

---

## 📸 Screenshots

Full step-by-step screenshots (146 total) are under
[`docs/snapshots/`](docs/snapshots), organized by phase:

- **[`docs/snapshots/01-vpn-networking-setup/`](docs/snapshots/01-vpn-networking-setup)** — Home ↔ GCP Cloud VPN (2 HA tunnels), strongSwan/FRR setup on the on-prem gateway VM
- **[`docs/snapshots/02-dms-private-connectivity-setup/`](docs/snapshots/02-dms-private-connectivity-setup)** — Target Cloud SQL for PostgreSQL databases and DMS private connectivity configuration
- **[`docs/snapshots/03-mydatabase/`](docs/snapshots/03-mydatabase)** — MyDatabase migration (HR/sales dataset: `Departments`, `Employees`, `Projects`, `EmployeeProjects`, `Customers`, `Products`, `Orders`, `OrderDetails`)
- **[`docs/snapshots/04-hospitaldb/`](docs/snapshots/04-hospitaldb)** — HospitalDB migration (`Doctors`, `Patients`, `Appointments`)
- **[`docs/snapshots/05-inventorydb/`](docs/snapshots/05-inventorydb)** — InventoryDB migration (`Warehouses`, `Items`, `StockMovements`)
- **[`docs/snapshots/06-librarydb/`](docs/snapshots/06-librarydb)** — LibraryDB migration (`Books`, `Members`, `Loans`)

Each database folder walks through: connection profile creation (source
+ destination) → conversion workspace / schema conversion → migration
job creation and start → post-migration row-count verification.

---

## 🔗 Repository

| Repository | Purpose |
|---|---|
| [`onprem-sqlserver-to-gcp-cloudsql-postgres-dms-migration`](https://github.com/bikram-singh/onprem-sqlserver-to-gcp-cloudsql-postgres-dms-migration) | This project — on-prem SQL Server to Cloud SQL for PostgreSQL, via DMS, over a self-built private HA VPN |

---

## 🏁 Result

A fully working, privately-connected migration pipeline: two SQL Server
databases' worth of infrastructure — a redundant HA VPN gateway and a
SQL Server host — built from scratch on a home network with no
static IP and no business-grade hardware, feeding four databases through
GCP's Database Migration Service into Cloud SQL for PostgreSQL with
**zero public IP exposure** at any hop. Every row count verified,
every real networking failure diagnosed and fixed rather than routed
around, and all 146 steps captured in screenshots for anyone rebuilding
this from scratch.

<div align="center">

**Maintained by Bikram Singh**

*Built with strongSwan · FRR · GCP Database Migration Service · Cloud SQL for PostgreSQL*

</div>
