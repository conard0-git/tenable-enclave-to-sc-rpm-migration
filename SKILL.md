# Tenable.sc RHEL Migration and Recovery Skill

## Purpose
Use this skill when migrating or recovering Tenable Security Center / Tenable Enclave Security data onto a standalone RHEL 8 Tenable.sc host, especially when the source used an external PostgreSQL database (for example AWS RDS) and the source filesystem came from a Kubernetes/container deployment.

The objective is to preserve the authoritative database and persistent Tenable data while keeping the fresh RHEL RPM runtime intact.

## Core Principles

1. Treat the external PostgreSQL database as authoritative application state.
2. Take an RDS/DB snapshot before installation, migration, or upgrade work.
3. Preserve the original source `/opt/sc` backup unchanged as a rollback copy.
4. Do not replace the entire fresh RHEL `/opt/sc` tree with a Kubernetes/container `/opt/sc` tree.
5. Preserve the fresh RHEL host's `.pgvars` and `data/enc.key`.
6. Restore persistent data selectively while excluding application/runtime directories.
7. Validate feed/job processing before performing another Tenable.sc upgrade.
8. When `Operation not permitted` appears despite correct Unix permissions, investigate `fapolicyd` before changing file modes or ownership broadly.
9. Do not use blanket recursive chmod changes on `/opt/sc`.
10. Prefer normal RPM upgrade paths; do not use `--force` unless vendor support explicitly directs it.

## Workflow

### 1. Establish rollback points

Before making changes:

- Create an AWS RDS/cluster snapshot of the external PostgreSQL database.
- Preserve the original `/opt/sc` filesystem backup or failed migration tree.
- Before upgrades, take a stopped-service tar backup of the working RHEL `/opt/sc` tree.

Session-safe tar backup example:

```bash
systemctl stop SecurityCenter
sudo bash -lc 'cd / && nohup tar -pzcf /opt/sc_pre_upgrade_$(date +%F_%H%M)_backup.tar.gz --one-file-system opt/sc > /opt/sc_pre_upgrade_$(date +%F_%H%M).log 2>&1 & echo "Backup PID: $!"'
```

Verify completion:

```bash
ps -ef | grep 'tar -pzcf' | grep -v grep
```

Verify archive integrity:

```bash
tar -tzf /opt/sc_pre_upgrade_<timestamp>_backup.tar.gz > /dev/null
echo $?
```

Exit code `0` indicates that tar successfully read the complete archive.

### 2. Build a clean standalone RHEL runtime

Remove a failed package installation if needed, and move the failed `/opt/sc` tree aside instead of deleting it when space permits.

```bash
systemctl stop SecurityCenter 2>/dev/null || true
rpm -e --noscripts SecurityCenter 2>/dev/null || true
mv /opt/sc /opt/sc.bad.$(date +%F_%H%M) 2>/dev/null || true
```

Confirm:

```bash
rpm -q SecurityCenter
ls -ld /opt/sc
```

Expected clean state: SecurityCenter is not installed and `/opt/sc` does not exist.

### 3. Recover external PostgreSQL connection settings

If the old filesystem contains `.pgvars`, preserve it before rebuilding:

```bash
find /opt/sc.bad* -maxdepth 2 -name '.pgvars' -ls
cp -a /opt/sc.bad.<timestamp>/.pgvars /root/sc-source.pgvars
chmod 600 /root/sc-source.pgvars
```

Do not expose the database password in chat/logs.

Typical variables include:

```bash
export SC_PG_HOST='<rds-endpoint>'
export SC_PG_PORT='5432'
export SC_PG_USER='<db-user>'
export SC_PG_PASSWORD='<password>'
export SC_PG_DATABASE='<database>'
export SC_PG_CA_PATH=''
export SC_PG_REQUIRE_TLS='prefer'
```

Load/export these values in a root login shell before installing Tenable.sc.

### 4. Fresh-install Tenable.sc against the existing external database

Use a root login shell so the `SC_PG_*` variables are present for the installer.

Install the matching standalone RHEL RPM. If the local RPM is not signed by a key trusted by the host, use the organization's approved package-validation method; where explicitly accepted, `dnf --nogpgcheck` may be used for that local package.

Example:

```bash
dnf install --nogpgcheck ./SecurityCenter-<version>-el8.x86_64.rpm
```

After installation, confirm `/opt/sc/.pgvars` points to the expected external database.

Do not paste the password.

### 5. Validate RDS connectivity using Tenable's bundled psql

If system `psql` is absent, Tenable may provide `/opt/sc/support/bin/psql`.

```bash
source /opt/sc/.pgvars
PGPASSWORD="$SC_PG_PASSWORD" /opt/sc/support/bin/psql \
  -h "$SC_PG_HOST" \
  -p "$SC_PG_PORT" \
  -U "$SC_PG_USER" \
  -d "$SC_PG_DATABASE" \
  -c 'select current_database(), current_user, version();'
```

Also check whether `/opt/sc/data/postgresql` is empty. An empty directory is expected when the instance uses external PostgreSQL.

```bash
du -sh /opt/sc/data/postgresql 2>/dev/null
```

### 6. Prove the clean RHEL runtime can start before restoring data

```bash
systemctl daemon-reload
systemctl start SecurityCenter
systemctl status SecurityCenter --no-pager -l
```

Do not proceed to data overlay until the fresh runtime starts successfully against RDS.

### 7. Stop SecurityCenter and preserve RHEL-specific secrets

```bash
systemctl stop SecurityCenter
ps -ef | grep '/opt/sc' | grep -v grep
```

Preserve:

```bash
cp -a /opt/sc/.pgvars /root/rhel-sc.pgvars
cp -a /opt/sc/data/enc.key /root/rhel-sc.enc.key
chmod 600 /root/rhel-sc.pgvars /root/rhel-sc.enc.key
```

### 8. Selectively overlay persistent data from the old tree

Use `rsync` from the old filesystem tree into the new `/opt/sc`, excluding runtime/application directories and preserving the new RHEL `.pgvars` and `enc.key`.

Example:

```bash
rsync -aHAX --info=progress2 \
  --exclude='/saml/***' \
  --exclude='/support/bin/***' \
  --exclude='/support/include/***' \
  --exclude='/support/lib/***' \
  --exclude='/support/modules/***' \
  --exclude='/support/openssl/***' \
  --exclude='/support/php/***' \
  --exclude='/support/var/***' \
  --exclude='/src/***' \
  --exclude='/www/***' \
  --exclude='/data/plugins/***' \
  --exclude='/data/fips.cnf' \
  --exclude='/bin/***' \
  --exclude='/customer-tools/***' \
  --exclude='/fop/***' \
  --exclude='/.pgvars' \
  --exclude='/data/enc.key' \
  /opt/sc.bad.<timestamp>/ /opt/sc/
```

For unstable terminal sessions, run rsync under `nohup` and log to `/root/sc-migration-rsync.log`.

After the copy, confirm the protected RHEL files were not changed:

```bash
cmp /root/rhel-sc.pgvars /opt/sc/.pgvars && echo '.pgvars OK'
cmp /root/rhel-sc.enc.key /opt/sc/data/enc.key && echo 'enc.key OK'
```

### 9. Validate migrated data and runtime before start

Useful checks:

```bash
du -sh /opt/sc/orgs /opt/sc/repositories /opt/sc/data
ls -ld /opt/sc/support/bin /opt/sc/support/lib /opt/sc/support/php /opt/sc/src /opt/sc/www /opt/sc/bin
```

The runtime directories should reflect the fresh RHEL RPM, not the old container tree.

Then start:

```bash
systemctl daemon-reload
systemctl start SecurityCenter
systemctl status SecurityCenter --no-pager -l
```

### 10. Troubleshoot jobs/feed updates before upgrading

Primary log:

```bash
tail -n 0 -f /opt/sc/admin/logs/sc-error.log
```

If errors show:

```text
Failed to open stream: Operation not permitted
```

for readable files such as `/opt/sc/src/pcntl.inc`, test as `tns`:

```bash
sudo -u tns head -n 2 /opt/sc/src/pcntl.inc
```

If this fails with `Operation not permitted` despite correct mode/ownership, check `fapolicyd`.

```bash
systemctl status fapolicyd --no-pager -l
```

Temporarily stopping `fapolicyd` is a useful diagnostic test. If access immediately works with it stopped, the root cause is application allowlisting/trust policy rather than PHP or normal Unix permissions.

Do not recursively trust all of `/opt/sc`: the tree contains sockets and large dynamic data. A recursive `fapolicyd-cli --file add /opt/sc` may be slow and can fail on objects such as `/opt/sc/data/enc.sock`.

Prefer a targeted fapolicyd policy/trust configuration for Tenable runtime and dynamic feed-update access. Example targeted runtime paths include:

- `/opt/sc/bin`
- `/opt/sc/src`
- `/opt/sc/support`
- `/opt/sc/saml`
- `/opt/sc/www`

Dynamic files under `/opt/sc/data/feed.*` change during feed updates, so static checksum trust is not ideal for that area. Use a narrow allow rule when required by the host's fapolicyd policy.

Example rule used during troubleshooting:

```text
allow perm=open exe=/opt/sc/support/bin/php : dir=/opt/sc/data/
```

Always validate rule ordering with:

```bash
fapolicyd-cli --list
```

### 11. PHP feed memory troubleshooting

If plugin feed updates fail with PHP memory exhaustion, check:

```bash
grep -i 'Allowed memory size' /opt/sc/admin/logs/sc-error.log
```

Back up the PHP configuration before editing:

```bash
cp /opt/sc/support/etc/php.ini /opt/sc/support/etc/php.ini_bak
```

Locate active values:

```bash
grep -nE 'memory_limit|post_max_size' /opt/sc/support/etc/php.ini
```

For environments where Tenable guidance requires it, increase active settings, for example:

```ini
memory_limit = 4G
post_max_size = 4G
```

Restart SecurityCenter and retry the feed update.

### 12. Feed cleanup/storage troubleshooting

If `/opt/sc` becomes unexpectedly large, inspect:

```bash
du -xh /opt/sc --max-depth=2 | sort -h | tail -n 30
```

Large numbers of historical `/opt/sc/data/feed.*` directories can consume substantial storage when feed updates repeatedly fail. Do not blindly delete active feed content. Confirm the environment's current feed state and vendor guidance before pruning old feed directories.

### 13. Upgrade preparation

Before upgrading Tenable.sc:

- RDS snapshot exists.
- `/opt/sc` backup exists and passes `tar -tzf` with exit code `0`.
- Plugin/security feed is current enough for the target upgrade.
- Jobs/feed updates complete normally.
- External PostgreSQL connectivity is verified.
- Adequate disk and memory are available.

If SELinux/fapolicyd are suspected of interfering with the upgrade, a controlled diagnostic upgrade may be performed with SELinux permissive and fapolicyd stopped, provided rollback points exist and organizational policy permits it. Re-enable and validate each control separately afterward.

Use a normal upgrade command unless vendor support instructs otherwise:

```bash
rpm -Uvh SecurityCenter-<target-version>-el8.x86_64.rpm
```

Do not use `--force` by default.

### 14. Post-upgrade validation

Verify:

```bash
rpm -q SecurityCenter
systemctl status SecurityCenter --no-pager -l
```

Then validate in the UI:

- organizations
- repositories
- vulnerability analysis
- scan history/imports
- scanners/zones
- users/roles
- licensing
- feed/plugin status
- scheduled jobs

Watch for new errors:

```bash
tail -n 0 -f /opt/sc/admin/logs/sc-error.log
```

### 15. Java diagnostic

If diagnostics report Java missing, install an approved OpenJDK package available from the RHEL repositories and verify:

```bash
dnf list available 'java-*-openjdk*'
dnf install java-21-openjdk.x86_64
java -version
```

Use the Java major version approved for the environment and supported by the installed Tenable.sc release.

## Diagnostic Decision Tree

### SecurityCenter will not start after full `/opt/sc` restore
Likely cause: container/Kubernetes runtime files replaced the standalone RHEL runtime.

Action: rebuild clean RHEL RPM runtime, validate RDS, then selectively overlay persistent data instead of replacing all of `/opt/sc`.

### RDS connection works but installer reports PostgreSQL setup failure
Check `.pgvars`, use Tenable's bundled `psql`, and verify the target database directly before assuming the database is bad.

### Jobs/feed updates hang or fail with `Operation not permitted`
Test the same file as `tns`. If it becomes readable when fapolicyd is stopped, troubleshoot fapolicyd trust/rules. Do not broadly chmod/chown application trees to solve this symptom.

### Feed update fails with PHP memory exhaustion
Check `sc-error.log`, increase the documented PHP `memory_limit` / `post_max_size` values, restart, and retry.

### Feed directories accumulate and `/opt/sc` grows rapidly
Investigate failed feed updates first. Historical feed directories may be leftovers from failed jobs; prune only after confirming what is active and following vendor guidance.

## Safety Rules

- Never paste database passwords into chat or tickets unnecessarily.
- Never modify the SecurityCenter database schema manually unless explicitly directed by Tenable Support.
- Do not delete the original migration backup until the new host is fully validated.
- Do not restore old `.pgvars` or `enc.key` over a fresh RHEL host without understanding the encryption/connection implications.
- Do not run simultaneous restores or backups against the same destination.
- Do not start SecurityCenter while a filesystem restore/overlay is still in progress.
