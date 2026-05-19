# Order 50: Manual Database Restore

This step is NOT automated via OADP Restore CR. It requires manual intervention to restore database data from the backup PVCs using GaleraRestore CRs.

## Prerequisites

- Order 00-40 restore completed
- OpenStackControlPlane is deployed with `deployment-stage: infrastructure-only` annotation
- Galera database pods are running but contain empty databases
- Backup PVCs are restored and mounted
- GaleraBackup CRs are restored (from order 40)

## Steps

### 1. Identify GaleraBackup CRs

List the restored GaleraBackup CRs to find the backup sources:

```bash
oc get galerabackup -n openstack
```

Expected output shows backup CR names (e.g., `openstack`, `openstack-cell1`)

### 2. Create GaleraRestore CRs

For each GaleraBackup, create a corresponding GaleraRestore CR:

```bash
# Main cell restore
cat <<EOF | oc apply -f -
apiVersion: mariadb.openstack.org/v1beta1
kind: GaleraRestore
metadata:
  name: openstack
  namespace: openstack
spec:
  backupSource: openstack
EOF

# Cell1 restore (if multi-cell)
cat <<EOF | oc apply -f -
apiVersion: mariadb.openstack.org/v1beta1
kind: GaleraRestore
metadata:
  name: openstack-cell1
  namespace: openstack
spec:
  backupSource: openstack-cell1
EOF
```

### 3. Wait for restore pods to be ready

The mariadb-operator will create restore pods. You can identify all restore
pods using the `galerarestore/name` label:

```bash
oc get pod -l galerarestore/name -n openstack
NAME                      READY   STATUS    RESTARTS   AGE
restore-openstack         1/1     Running   0          5m
restore-openstack-cell1   1/1     Running   0          5m
```

### 4. Verify backup files exist

List the backup dump files matching your backup timestamp on each restore pod.
Replace `<BACKUP_TIMESTAMP>` with the timestamp used during backup
(e.g., `20260311-081234`):

```bash
BACKUP_TIMESTAMP=<BACKUP_TIMESTAMP>

# Main cell
oc exec -n openstack restore-openstack -- \
  ls -1 /backup/data/*_${BACKUP_TIMESTAMP}.sql.gz

# Cell1 (if multi-cell)
oc exec -n openstack restore-openstack-cell1 -- \
  ls -1 /backup/data/*_${BACKUP_TIMESTAMP}.sql.gz
```

You should see two files per cell: a database dump and a grants file.

### 5. Execute the database restore

Run the `restore_galera` script inside each restore pod. The `--yes` flag
skips the confirmation prompt. The `--content data` flag restores only
database content (tables, rows) without grants or users, since those are
recreated by the operators:

```bash
BACKUP_TIMESTAMP=<BACKUP_TIMESTAMP>

# Main cell
oc exec -n openstack restore-openstack -- \
  /var/lib/backup-scripts/restore_galera --yes --content data \
  "/backup/data/*_${BACKUP_TIMESTAMP}.sql.gz"

# Cell1 (if multi-cell)
oc exec -n openstack restore-openstack-cell1 -- \
  /var/lib/backup-scripts/restore_galera --yes --content data \
  "/backup/data/*_${BACKUP_TIMESTAMP}.sql.gz"
```

### 6. Verify database restore

```bash
# Check databases are restored
oc exec -it galera-0 -n openstack -- mysql -e "SHOW DATABASES;"
```

### 7. Clean up GaleraRestore CRs

Delete the GaleraRestore CRs to terminate the restore pods:

```bash
oc delete galerarestore openstack -n openstack
# Cell1 (if multi-cell):
oc delete galerarestore openstack-cell1 -n openstack
```

## Next Steps

After database restore, return to the main restore procedure in
[README.md](README.md#step-6b-remove-deployment-stage-annotation) to
remove the deployment-stage annotation and resume full deployment.
