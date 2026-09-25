# Ubuntu-sqlserver-vm: SQL Server 2025 on Linux — Full Setup Guide

This documents the complete process of building a second Hyper-V VM (`Ubuntu-sqlserver-vm`), installing SQL Server 2025 natively on Ubuntu, and recreating the same 5-database sample dataset used on the original Windows SQL Server instance — with every command and the reasoning behind each choice.

**Why this VM exists:** the original SQL Server ran on Windows (OPTIPLEX-PC itself), which meant DMS traffic from GCP would have to reach a *different* machine than the VPN gateway VM — requiring extra DNAT/port-forwarding. Running SQL Server directly inside a VM that's already on the same NAT'd network as the gateway VM (`strongswan-gw`) eliminates that complexity entirely: this VM's IP already falls within the `172.24.96.0/20` range already advertised to GCP via BGP.

---

## Part 1 — Create the VM in Hyper-V Manager

1. **Hyper-V Manager → Action → New → Virtual Machine**
2. **Name:** `Ubuntu-sqlserver-vm`
3. **Generation:** Generation 1 (matches `strongswan-gw`; simpler BIOS-based boot, no UEFI complications)
4. **Assign Memory:** `4096` MB, check **"Use Dynamic Memory for this virtual machine."**
   - SQL Server needs meaningfully more memory than the lightweight gateway VM (which only runs strongSwan + FRR). 4GB gives headroom for SQL Server's default memory management without starving the host.
5. **Configure Networking:** select **`Default Switch`**
   - Not `ExternalSwitch` — this host connects via Wi-Fi, and Hyper-V's External Switch bridging is unreliable on Wi-Fi adapters (a known Hyper-V limitation). Default Switch uses Windows' own NAT, which works regardless of the host's network type.
6. **Connect Virtual Hard Disk:** Create a new virtual hard disk, size **127 GB** (dynamically expanding VHDX — only consumes actual space used, no harm in a larger max size)
7. **Installation Options:** Install an operating system from a bootable image file (.iso) — point to the downloaded Ubuntu Server ISO
8. **Finish** — this creates the VM

### Verify/fix boot order
- Right-click the VM → Settings → **BIOS** → confirm **CD** is first in the Startup order (above IDE/Hard Drive), so it boots the installer.

---

## Part 2 — Install Ubuntu Server

Boot the VM (Hyper-V Manager → double-click → Start). Walk through the installer:

1. **Language** — English (default)
2. **Keyboard layout** — confirm auto-detected default
3. **Installation type** — Ubuntu Server (not minimized)
4. **Network configuration** — the installer auto-detects DHCP via Default Switch's built-in NAT. Confirm the assigned IP falls within `172.24.96.0/20` (this VM got `172.24.100.130/20` — good, matches).
5. **Proxy configuration** — leave blank
6. **Ubuntu archive mirror** — leave default (auto-selects nearest, e.g. `in.archive.ubuntu.com` for India)
7. **Storage configuration** — **"Use an entire disk,"** check **"Set up this disk as an LVM group,"** leave **"Encrypt the LVM group with LUKS" unchecked**
   - LUKS encryption would prompt for a passphrase on every boot, blocking headless/automatic startup — not wanted for a server VM.
8. Confirm the destructive-action warning (safe — fresh empty disk)
9. **Profile setup** — name: `bikram`, server's name: `ubuntu-sqlserver`, pick a username, set a strong password
10. **Upgrade to Ubuntu Pro** — Skip for now
11. **SSH Setup — check "Install OpenSSH server"** (critical — this is how you'll manage the VM after this)
12. **Featured Server Snaps** — leave unchecked, skip all
13. Installation runs (several minutes) → **Reboot Now**
14. **Remove installation medium** prompt — Media menu → DVD Drive → Eject, then press Enter to complete the reboot

---

## Part 3 — Connect and verify networking

From PowerShell on the Windows host:
```powershell
ssh bikram@172.24.100.130
```
(accept the host key fingerprint the first time with `yes`, then enter your password)

Confirm the IP and connectivity:
```bash
ip a show eth0
```
Look for an `inet` line showing an address within `172.24.96.0/20` (e.g. `172.24.100.130/20`).

---

## Part 4 — Install SQL Server 2025

**1. Update the system:**
```bash
sudo apt update && sudo apt upgrade -y
```

**2. Install prerequisite packages** (repo management tools SQL Server's installer needs):
```bash
sudo apt install -y wget curl gpg gnupg2 software-properties-common apt-transport-https lsb-release ca-certificates
```

**3. Import Microsoft's package-signing GPG key** (so `apt` trusts packages from Microsoft's repo):
```bash
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
```

**4. Register the SQL Server 2025 GA (production) repository:**
```bash
curl -fsSL https://packages.microsoft.com/config/ubuntu/24.04/mssql-server-2025.list | sudo tee /etc/apt/sources.list.d/mssql-server-2025.list
```
Note: this uses Microsoft's Ubuntu 24.04 repo track (the officially GA-supported version) even though this VM runs Ubuntu 26.04 — worked without issue in practice, but is technically an unsupported combination.

**5. Refresh package lists and install SQL Server:**
```bash
sudo apt update
sudo apt install -y mssql-server
```
This is a ~300MB download; takes a few minutes.

**6. Run the configuration wizard:**
```bash
sudo /opt/mssql/bin/mssql-conf setup
```
- **Edition:** entered `4` for **Express** (free, matches the original Windows instance's edition)
- **License terms:** typed `Yes` to accept
- **SA password:** set to a password meeting complexity requirements (8+ chars, mixed case/digits/symbols)

Setup completes and starts the `mssql-server` service automatically.

**7. Verify it's running:**
```bash
systemctl status mssql-server --no-pager
```
Should show `Active: active (running)`.

---

## Part 5 — Install `sqlcmd` (command-line client tools)

**1. Register the tools repository:**
```bash
curl -fsSL https://packages.microsoft.com/config/ubuntu/24.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
```

**2. Update and install:**
```bash
sudo apt update
sudo apt install -y mssql-tools18 unixodbc-dev
```
(accept the ODBC driver license terms if prompted — navigate the dialog with Tab/arrows to `<Yes>`, press Enter)

**3. Add `sqlcmd` to your shell's PATH permanently:**
```bash
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
source ~/.bashrc
```

**4. Test a local connection:**
```bash
sqlcmd -S localhost -U sa -P 'YourSAPassword' -C -Q "SELECT @@VERSION"
```
The `-C` flag trusts the self-signed TLS certificate SQL Server generates by default (required for `sqlcmd` v18's stricter encryption defaults). Confirmed output: `Microsoft SQL Server 2025 (RTM-CU9) ... Express Edition (64-bit) on Linux (Ubuntu 26.04.1 LTS)`.

---

## Part 6 — Create the databases

Two scripts recreate the exact same 5-database dataset used on the original Windows instance. Rather than transferring files, both were created directly on the VM using a heredoc (avoids clipboard/paste issues that occurred with `vi`):

**Script 1 — `multi_db_setup.sql`** (creates `LibraryDB`, `HospitalDB`, `InventoryDB`, `SchoolDB`):
```bash
cat > multi_db_setup.sql << 'SQLEOF'
-- [full script: see multi_db_setup.sql artifact from earlier in this conversation]
SQLEOF
```
Run it:
```bash
sqlcmd -S localhost -U sa -P 'YourSAPassword' -C -i multi_db_setup.sql
```
Result: 4 databases created, 12 books, 8 members, 12 loans, 6 doctors, 10 patients, 12 appointments, 4 warehouses, 10 items, 20 stock movements, 10 students, 8 courses, 20 enrollments.

**Script 2 — `sample_data_setup.sql`** (creates `MyDatabase` — Employees, Departments, Projects, Customers, Orders, etc.):
```bash
cat > sample_data_setup.sql << 'SQLEOF'
-- [full script: see sample_data_setup.sql artifact from earlier in this conversation]
SQLEOF
```
Run it:
```bash
sqlcmd -S localhost -U sa -P 'YourSAPassword' -C -i sample_data_setup.sql
```
Result: 5 departments, 30 employees, 10 projects, 40 project assignments, 20 customers, 15 products, 30 orders, 62 order line items.

---

## Part 7 — Verify everything

**List all databases:**
```bash
sqlcmd -S localhost -U sa -P 'YourSAPassword' -C -Q "SELECT name FROM sys.databases;"
```
Expected: `master`, `tempdb`, `model`, `msdb`, `MyDatabase`, `LibraryDB`, `HospitalDB`, `InventoryDB`, `SchoolDB`.

**Full row-count check across all 5 user databases:**
```bash
sqlcmd -S localhost -U sa -P 'YourSAPassword' -C -Q "
SELECT 'MyDatabase.Employees' AS TableName, COUNT(*) AS Rows FROM MyDatabase.dbo.Employees
UNION ALL SELECT 'MyDatabase.Departments', COUNT(*) FROM MyDatabase.dbo.Departments
UNION ALL SELECT 'MyDatabase.Orders', COUNT(*) FROM MyDatabase.dbo.Orders
UNION ALL SELECT 'LibraryDB.Books', COUNT(*) FROM LibraryDB.dbo.Books
UNION ALL SELECT 'LibraryDB.Members', COUNT(*) FROM LibraryDB.dbo.Members
UNION ALL SELECT 'LibraryDB.Loans', COUNT(*) FROM LibraryDB.dbo.Loans
UNION ALL SELECT 'HospitalDB.Doctors', COUNT(*) FROM HospitalDB.dbo.Doctors
UNION ALL SELECT 'HospitalDB.Patients', COUNT(*) FROM HospitalDB.dbo.Patients
UNION ALL SELECT 'HospitalDB.Appointments', COUNT(*) FROM HospitalDB.dbo.Appointments
UNION ALL SELECT 'InventoryDB.Warehouses', COUNT(*) FROM InventoryDB.dbo.Warehouses
UNION ALL SELECT 'InventoryDB.Items', COUNT(*) FROM InventoryDB.dbo.Items
UNION ALL SELECT 'InventoryDB.StockMovements', COUNT(*) FROM InventoryDB.dbo.StockMovements
UNION ALL SELECT 'SchoolDB.Students', COUNT(*) FROM SchoolDB.dbo.Students
UNION ALL SELECT 'SchoolDB.Courses', COUNT(*) FROM SchoolDB.dbo.Courses
UNION ALL SELECT 'SchoolDB.Enrollments', COUNT(*) FROM SchoolDB.dbo.Enrollments;
"
```
All 15 rows returned matching the expected counts, confirming an exact match with the original Windows dataset.

---

## Part 8 — Confirm remote (cross-network) reachability

This step matters because everything tested above was from `localhost` — DMS will connect from a *different* machine entirely.

**1. Confirm SQL Server is listening on all interfaces, not just localhost:**
```bash
sudo ss -tlnp | grep 1433
```
Expected output shows `0.0.0.0:1433` (IPv4, all interfaces) and `*:1433` (IPv6) — confirms SQL Server is not restricted to loopback-only connections.

**2. Test from a genuinely separate machine** — SSH into `strongswan-gw` (a different VM) and connect across the network:
```bash
ssh bikram@<strongswan-gw-ip>
```
On that VM, install the client tools the same way (steps from Part 5), then:
```bash
sqlcmd -S 172.24.100.130 -U sa -P 'YourSAPassword' -C -Q "SELECT @@VERSION"
```
Successful version banner returned — confirms SQL Server is reachable across the local network, which is the same kind of access DMS (arriving via the VPN + BGP path from GCP) will need.

---

## Result

A fully working SQL Server 2025 (Express) instance running natively on Ubuntu, with an exact replica of the original 5-database dataset (`MyDatabase`, `LibraryDB`, `HospitalDB`, `InventoryDB`, `SchoolDB`), confirmed reachable across the local network — sitting on a VM whose IP is already advertised to GCP via the working VPN/BGP setup from the earlier connectivity work. No DNAT or port-forwarding complexity required, unlike the original Windows-hosted SQL Server.
