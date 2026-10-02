# Tenable.sc Migration / Recovery Skill

This package contains a reusable `SKILL.md` distilled from the successful migration and troubleshooting process used for:

- standalone RHEL 8 Tenable.sc
- external AWS RDS PostgreSQL
- migration from a Kubernetes/container `/opt/sc` tree
- selective filesystem restoration
- plugin/feed update troubleshooting
- `fapolicyd` access failures
- PHP feed memory troubleshooting
- upgrade preparation and validation

The core pattern is:

**Protect RDS -> clean RHEL runtime -> validate external DB -> selectively restore persistent data -> validate jobs/feed -> upgrade -> re-enable host security controls deliberately.**
