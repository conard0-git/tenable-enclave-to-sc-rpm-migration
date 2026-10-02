# Tenable Enclave to Tenable.sc RPM Migration

> Migrate or recover Tenable Enclave Security / Tenable Security Center data onto a standalone RHEL 8 Tenable.sc host backed by external PostgreSQL (e.g. AWS RDS). Preserves the authoritative database and overlays persistent data onto the fresh RPM runtime instead of blindly replacing it.

A Claude Code / Agent Skills skill distilled from a real migration off a
Kubernetes/container Tenable Enclave deployment onto a standalone RHEL 8
Tenable.sc host (AWS Marketplace AMI) with an external AWS RDS PostgreSQL
database. It encodes the "protect the DB, rebuild the runtime, overlay only
persistent data" pattern so the next person doesn't have to rediscover it
by breaking a host first.

## Why

The obvious migration — stop services, `rsync` the old `/opt/sc` onto a new
host, start services — fails on standalone RHEL. The container `/opt/sc`
tree carries a different runtime (PHP, libraries, binaries) than the RPM
expects, and silently stomping the RPM runtime with container files leaves
the service unable to start, with errors that look like permission or
database problems but aren't. Add `fapolicyd` on top and `Operation not
permitted` appears on files that are mode-correct and owned correctly.

This skill captures the working recovery pattern: treat RDS as the source
of truth, rebuild a clean RPM runtime against it, then **selectively**
overlay persistent data (`orgs`, `repositories`, `data` minus runtime
subdirs) while preserving the fresh host's `.pgvars` and `data/enc.key`.

## What it does

Guides an operator through, in order:

1. Establish rollback points — RDS snapshot, preserved failed `/opt/sc`,
   stopped-service tar backup.
2. Build a clean RPM runtime (remove failed install, install signed RPM,
   verify `.pgvars`).
3. Prove the clean runtime starts against the external DB **before**
   touching data.
4. Selectively `rsync` persistent data from the old tree with an exclude
   list that keeps the RPM runtime intact and preserves `.pgvars` and
   `enc.key`.
5. Troubleshoot the usual follow-on failures: `fapolicyd` blocking PHP,
   PHP `memory_limit` exhaustion during feed updates, `/opt/sc` growing
   from leftover `data/feed.*` directories.
6. Prepare for and validate the next Tenable.sc upgrade.

See `SKILL.md` for the full workflow, decision tree, and safety rules.

## Prerequisites

- Standalone **RHEL 8** host (tested on AWS Marketplace AMI for Tenable.sc).
- Root shell access on the target host.
- External **PostgreSQL** reachable from the host (tested with AWS RDS).
- Ability to take an **RDS snapshot** before any destructive step.
- The failed/source `/opt/sc` tree available (from the container or prior
  host) for the overlay step.
- The signed Tenable.sc RPM for the target version.
- Optional but recommended: ability to toggle `fapolicyd` and SELinux for
  diagnostic isolation.

## How to run

This is a Claude Code / Agent Skills skill — Claude follows `SKILL.md`
when the skill is loaded in your Claude Code session.

**Install into Claude Code:**

```bash
mkdir -p ~/.claude/skills/tenable-enclave-to-sc-rpm-migration
cp SKILL.md ~/.claude/skills/tenable-enclave-to-sc-rpm-migration/
```

Then in Claude Code, invoke it when you're on (or ssh'd to) the RHEL host:

```
/tenable-enclave-to-sc-rpm-migration
```

Claude will walk through the workflow, prompt for the specific state of
your host, and run the diagnostic/validation commands step by step. Treat
it as a guided runbook, not an autonomous script — every step that
changes host state should be reviewed before execution.

## Outputs

This skill produces no files or reports of its own. The outputs are
side-effects on the target host:

- A rebuilt, running `/opt/sc` on the RHEL host, backed by the preserved
  external PostgreSQL database.
- Preserved artifacts for rollback: the original failed `/opt/sc` moved
  aside, a stopped-service tar backup, an RDS snapshot.
- A readable `tail -f /opt/sc/admin/logs/sc-error.log` that the skill
  uses to diagnose feed/job issues.

## Known limitations

- Tested specifically against **RHEL 8** + Tenable.sc on the AWS
  Marketplace AMI with **AWS RDS PostgreSQL**. Other RHEL versions,
  non-AWS Postgres, or non-standalone deployments may need adjustment.
- Encryption-key handling (`data/enc.key`) assumes the fresh RHEL host's
  key is authoritative; cross-key migration scenarios are not covered.
- Not a replacement for Tenable Support. Where vendor guidance exists
  (e.g. for `rpm --force` or schema changes), defer to it.
- Does not automate the actual RPM install, RDS snapshot, or `rsync` —
  the skill guides the operator through them but leaves destructive
  commands under human control.

## Safety

- Never pastes database passwords into chat or logs.
- Never does a blanket recursive `chmod`/`chown` on `/opt/sc`.
- Does not use `rpm --force` by default.
- Does not delete the original migration backup until the new host is
  validated.
