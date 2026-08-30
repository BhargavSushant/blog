# CLAUDE.md, Linux Hardening & Compliance Toolkit

This file is read automatically by Claude Code. It carries the context, decisions,
and constraints from the design sessions that produced `docs/SPEC.md`.

---

## What this project is

A transactional, idempotent, reversible Linux hardening toolkit for Debian/Ubuntu
and RHEL-family servers. Aligns with CIS Level 1/2 and DISA-STIG, but the point of
the project is **correctness of mechanism**, not checklist coverage.

The full specification is `docs/SPEC.md` (v2.0.0, ~117 tagged requirements).
**Read it before writing any module code.** Do not re-derive design decisions that
are already settled there.

This is a portfolio project for a DevOps role transition. It has to be genuinely
good, not demo-quality, the code will be read by interviewers.

---

## The one idea the whole project is built on

> **Verify the effect, not the artifact.**

A hardening control that writes a correct-looking file and has no runtime effect is
worse than no control, because it passes audit forever. Almost every defect found in
the v1.0.0 specification was an instance of this.

Concretely, this means:

| Never verify with | Always verify with |
|---|---|
| `cat /etc/sysctl.d/99-hardening.conf` | `sysctl -n <key>` |
| hash of `sshd_config.d/99-hardening.conf` | `sshd -T` |
| presence of `blacklist x` in modprobe.d | actual `modprobe x` attempt + `lsmod` |
| unit file contents | `systemctl show <unit>` |
| firewall rule file | `nft list ruleset` |

If a verification step reads back the same file the apply step wrote, it tests that
the disk works. Reject that pattern in review.

---

## Architectural invariants (from SPEC §1)

Six properties that must hold on every code path. A change that cannot satisfy all
six does not go in the default profile.

1. **Zero lockout**, never break the active admin session or the management path.
2. **Drop-in configuration**, never regex vendor base files when a `.d` dir exists
   and is honoured. Fall back to a delimited managed block only when it isn't, and
   record the fallback.
3. **Atomic transactions**, every mutation gets a manifest entry with pre-hashes,
   a backup artifact, and a concrete reversal operation.
4. **Environment awareness**, unsupported operations are `skipped` with a reason,
   never `failed`, never fatal.
5. **Idempotency**, second `apply` on unchanged state: zero writes, zero reloads,
   exit 0. Measured against effective runtime state.
6. **Irreversibility register**, controls that need a reboot to undo are enumerated
   in SPEC Appendix A, excluded from `baseline`, and require explicit profile opt-in.

Invariant 6 exists because v1.0.0 promised deterministic rollback while mandating
`kernel.unprivileged_bpf_disabled=1` and auditd `-e 2`. Those are mutually exclusive.
Don't reintroduce that contradiction.

---

## Open decision: implementation language

**Not yet decided.** Pick one before writing engine code, then delete this section
and record the choice in `docs/ADR-001-language.md`.

- **Python 3**, stdlib covers everything needed (json, hashlib, subprocess, tarfile
  with zstd via `python3-zstandard`), ships on every target OS, no build step for a
  root-run tool. Fastest path to working code.
- **Go**, single static binary, no runtime dependency on the target, better
  portfolio artifact. More upfront work; cross-compilation and testing matrix cost.

Bash is not an option. The manifest, schema validation, and drift engine are not
tractable in shell.

Whichever is chosen: **no dependency may be required on the target host at runtime**
beyond what the OS ships. A hardening tool that needs `pip install` on a locked-down
box is self-defeating.

---

## Build order

Do not build modules first. The engine is the hard part and the modules are thin
once it exists.

1. **Engine core**, lock (`flock` on `/run/lock/hardening-toolkit.lock`), execution
   phases (SPEC §4.1), manifest write/fsync, backup archive, crash recovery.
2. **Manifest + schema**, implement SPEC §4.5 exactly, including the per-`action_type`
   state variants. Validate the schema in CI.
3. **`mod_kernel_sysctl`**, the pilot module. Simplest complete round trip:
   apply → verify effective value → detect drift → roll back. Exercises the
   precedence trap (REQ-SYSCTL-1) which is the whole thesis of the project.
4. **Rollback engine**, SPEC §6, including drift branching and `partially_rolled_back`.
5. **`mod_ssh`**, only once transactions are proven. Highest lockout risk, most
   capability-probing, hardest ordering constraints (REQ-SSH-9 through 13).
6. **`mod_network`**, last. Largest blast radius. The pre-flight assertion in
   REQ-NET-4 (SSH port must be in the accept list) is non-negotiable.

`mod_auth_pam`, `mod_mounts`, `mod_systemd`, `mod_audit`, `mod_mac` after that,
in any order.

---

## Hard rules for module code

- **Capability probes, never version comparisons.** SPEC §2.2 has the probe table.
  RHEL 8 has no `sshd_config.d` include; Ubuntu 24.04 is socket-activated; RHEL
  crypto-policies override sshd ciphers from the command line. All of these are
  probes, not version checks.
- **Family-specific correctness.** `/etc/shadow` is `0640 root:shadow` on Debian and
  `0000 root:root` on RHEL. A single table applied everywhere is a bug.
- **Applicability checks before every control.** `rp_filter=1` breaks asymmetric
  routing. `ip_forward=0` severs container networking. Masking `rpcbind` breaks NFS
  *clients*. `udf` blacklist breaks Azure provisioning. `squashfs` breaks snapd.
  Every one of these is in the spec with a REQ id.
- **Absolute paths, sanitised env** for all external commands
  (`PATH=/usr/sbin:/usr/bin:/sbin:/bin`, `LC_ALL=C`, cleared `IFS`). No shell
  interpolation of anything read from disk or profile.
- **Never log secrets.** Diffs of shadow-family files are suppressed and replaced
  with a mode/owner summary. Backup archives are `0600`, they contain `/etc/shadow`.
- **Reference the REQ id in code comments and test names** so the traceability matrix
  can be generated mechanically. `# REQ-SSH-9: validate after write, not before`.

---

## Testing

Test IDs T-1 through T-14 are in SPEC §8.2. The ones that catch real bugs:

- **T-1 idempotency**, assert via manifest transaction count, not log scraping.
- **T-4 reboot persistence**, mounts, modprobe, sysctl, faillock and journald only
  prove themselves across a reboot. This is not optional.
- **T-7 lockout resistance**, no keys on host + profile disabling password auth
  must exit 2 with zero changes.
- **T-12 precedence traps**, pre-seed a conflicting key in `/etc/sysctl.conf` and a
  directive before the `Include` in `sshd_config`. The tool must detect and resolve.

**Containers cannot test half of this.** SPEC §2.2 aborts in Docker and disables
sysctl/mounts/modprobe/audit in LXC. Anything touching kernel, mount, audit, or
firewall state needs a real VM in CI. Don't write container tests that appear to
cover VM-only behaviour.

---

## Explicitly out of scope

Locked during career-strategy sessions; do not expand without being asked.

- Deep Kubernetes (Docker only)
- Go and Rust, unless chosen as the implementation language above
- Java, Python/ML frameworks
- Windows targets
- Switching a host between firewall backends
- Auto-removing SUID bits or world-writable files (report only, REQ-PERM-4)

---

## Related repos

- `blog/`, Hugo site at sushantbhargav.com. The first post is an audit of the
  v1.0.0 spec. Further posts document this implementation as it goes.
- Terraform provisioning for the CI VM images lives in `terraform/` in this repo.

---

## Working style

- Read `docs/SPEC.md` before proposing design changes; most questions are answered
  there with a REQ id.
- If the spec is wrong, say so and propose a numbered amendment, don't silently
  deviate. Amendments go in `docs/SPEC-AMENDMENTS.md`.
- Prefer being told a control is unsafe over having it implemented politely.
- Blog claims and resume claims must be defensible under interview questioning.
  Describe what the code actually does, not what it aspires to.
