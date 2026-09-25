# Screenshots

146 screenshots documenting the full build, organized by phase. General
networking/DMS-setup steps are shared across all four databases; the last
four folders hold only the steps specific to migrating that database
(connection profile → conversion workspace → migration job → verification).

| Folder | Covers | Screenshots |
|---|---|---|
| [`01-vpn-networking-setup/`](./01-vpn-networking-setup) | Home ↔ GCP Cloud VPN (2 HA tunnels), strongSwan/FRR config on the on-prem gateway VM | 51 |
| [`02-dms-private-connectivity-setup/`](./02-dms-private-connectivity-setup) | Empty target databases on Cloud SQL for PostgreSQL, DMS private connectivity configuration | 3 |
| [`03-mydatabase/`](./03-mydatabase) | MyDatabase migration (connection profiles, conversion workspace, migration job, verification) | 33 |
| [`04-hospitaldb/`](./04-hospitaldb) | HospitalDB migration | 18 |
| [`05-inventorydb/`](./05-inventorydb) | InventoryDB migration | 20 |
| [`06-librarydb/`](./06-librarydb) | LibraryDB migration | 21 |

## Databases migrated

### MyDatabase — HR/sales dataset
`Departments`, `Employees`, `Projects`, `EmployeeProjects`, `Customers`, `Products`, `Orders`, `OrderDetails`

### LibraryDB
`Books`, `Members`, `Loans`

### HospitalDB
`Doctors`, `Patients`, `Appointments`

### InventoryDB
`Warehouses`, `Items`, `StockMovements`

---

**Note on the two Cloud VPN tunnels:** two tunnels were created for the
home-to-GCP connection (`gcp-tunnel1`, `gcp-tunnel2`). Screenshots for
building and troubleshooting both are in `01-vpn-networking-setup/`, in the
order they were configured — check the BGP session status shots near the
end of that folder to see which tunnel was confirmed working.
