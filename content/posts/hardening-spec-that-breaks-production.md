---
title: "The Linux Hardening Spec That Would Have Broken Every Server It Touched"
date: 2026-08-20
lastmod: 2026-08-20
draft: false
tags: ["linux", "hardening", "security", "cis", "sysadmin"]
summary: "A plausible, well-structured, CIS-aligned hardening specification. Thirty-five defects. Eleven of them lock you out of the box."
---

I audited a requirements document last week for a Linux server hardening toolkit. It was organised well: architectural invariants, a JSON state schema, module specs, a rollback engine, a CI test matrix. It cited CIS Level 1/2 and DISA-STIG. It read like the work of someone who knew the domain.

It also had eleven defects that would take a production server off the network, and twelve more that write configuration doing nothing at all.

The second group is the interesting one. A control that fails loudly gets fixed. A control that writes a file, exits zero, and changes nothing will pass your audit forever.

Here are the ones worth knowing regardless of what tooling you use.

## The blacklist that doesn't blacklist

Every hardening guide tells you to blacklist unused kernel modules:

```
blacklist cramfs
blacklist dccp
blacklist usb-storage
```

This does less than it looks like it does. `blacklist` suppresses alias-based autoloading, which is the kernel asking for a module by a generic name like `fs-cramfs` when something tries to mount that filesystem. It does not stop an explicit request:

```console
# modprobe cramfs && lsmod | grep cramfs
cramfs                 32768  0
```

The module loads. The blacklist is still sitting there. Nothing failed.

To actually block loading you need the `install` override, which replaces the load command:

```
install cramfs /bin/false
blacklist cramfs
```

Both lines. The `install` line does the work. The `blacklist` line keeps the autoload path quiet. A tool that emits only `blacklist` produces a compliance artifact, not a control.

Two more traps in module blacklists.

Don't blacklist `udf` on Azure. The Azure Linux Agent reads its provisioning data, `ovf-env.xml`, off a UDF-formatted ISO attached to the instance. Blacklist `udf` and provisioning breaks. CIS lists `udf` as a candidate. CIS does not know you are on Azure.

Don't blacklist `squashfs` on Ubuntu either. Every snap is a squashfs mount. You will disable `snapd` by accident, from a distance, without meaning to.

## The ciphers that were never applied

On RHEL 9 and its rebuilds, you write a modern cipher list into `/etc/ssh/sshd_config.d/99-hardening.conf`:

```
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
KexAlgorithms curve25519-sha256
```

Then you check what is actually running:

```console
# sshd -T | grep ^ciphers
ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
```

Not your list. That is the system crypto policy.

`/etc/sysconfig/sshd` contains `CRYPTO_POLICY=`, which the systemd unit expands into command line arguments: `-oCiphers=...`, `-oMACs=...`. Command line options beat the config file. Your drop-in loses every time, silently.

The fix is either a crypto policy submodule:

```console
# cat /etc/crypto-policies/policies/modules/HARDENING.pmod
cipher@SSH = CHACHA20-POLY1305 AES-256-GCM
# update-crypto-policies --set DEFAULT:HARDENING
```

or emptying `CRYPTO_POLICY=` in `/etc/sysconfig/sshd` so the config file gets a say. Either works. Doing neither, while believing you have set ciphers, is the usual outcome.

General lesson: after any sshd change, verify with `sshd -T`. Reading back the file you just wrote tells you nothing. `sshd -T` is the only thing that reports what the daemon actually resolved.

## The sysctl file that lost

Same failure, different subsystem. You write:

```
# /etc/sysctl.d/99-hardening.conf
kernel.kptr_restrict = 2
```

`99-` is the highest prefix, so it wins. Except `systemd-sysctl` applies the `.d` directories in lexical order and then, on most distributions, applies `/etc/sysctl.conf` last. If someone left a line in `sysctl.conf` three years ago, that line wins and your file is decoration.

```console
# sysctl -n kernel.kptr_restrict
0
```

The file is correct. The hash matches. The system is not hardened.

Any check comparing file contents instead of runtime values reports success here. Compare `sysctl -n <key>` against intent. The same goes for `systemctl show` on units and `nft list ruleset` on firewall state. The file is your intent. The runtime is your state. Hardening is about state.

## /etc/shadow with mode 0000

This one appears in a lot of checklists. On RHEL it is correct. On Debian it breaks authentication.

Debian ships `/sbin/unix_chkpwd` setgid to the `shadow` group. It exists so non-root processes can verify a password without reading the shadow file as root: `su`, screen lockers, some PAM stacks. It needs group read on `/etc/shadow` to do that.

```
# Debian family
-rw-r----- 1 root shadow /etc/shadow      # 0640 root:shadow

# RHEL family
---------- 1 root root   /etc/shadow      # 0000 root:root
```

Set `0000 root:root` on Debian and password verification starts failing for anything that is not already root. It will not fail immediately or obviously. It fails the next time someone runs `su`, or when the screensaver refuses to unlock.

Permissions are family specific. One table applied everywhere is a bug.

## The firewall with no way in

The spec's base ruleset: default drop inbound, default allow outbound, accept established and related, accept loopback.

Read that again. Nothing accepts new connections on port 22.

Your current session survives, because it is `established`. Every connection after it gets dropped. You find out when you close the terminal.

This is the most common way people lock themselves out of a remote host, and it gets through review precisely because whoever applied it stayed connected the whole time. If you build tooling that touches firewalls, the pre-flight check is not optional. Read the port `sshd` is actually listening on with `sshd -T`, confirm it is in the accept list, refuse to commit if it is not.

Two more rules that belong in any default-drop ruleset and get forgotten:

```nft
# ICMPv6 / NDP. Without this, IPv6 stops working entirely.
# Neighbour discovery is not optional the way ARP is not optional.
ip6 nexthdr icmpv6 icmpv6 type {
    nd-neighbor-solicit, nd-neighbor-advert,
    nd-router-solicit, nd-router-advert,
    destination-unreachable, packet-too-big,
    time-exceeded, parameter-problem
} accept

# ICMPv4 type 3 code 4, fragmentation-needed.
# Drop it and you get a PMTU blackhole: small requests work,
# large responses hang forever. Miserable to debug from the
# application side.
ip protocol icmp icmp type destination-unreachable accept
```

"Block all ICMP" is folklore from before PMTU discovery and IPv6. It produces intermittent, size-dependent hangs that look like application bugs.

## ProtectHome=yes on sshd

systemd sandboxing is good, and the spec recommended a solid list of directives for "common daemons":

```ini
ProtectSystem=strict
ProtectHome=yes
MemoryDenyWriteExecute=yes
```

Applied to sshd, `ProtectHome=yes` hides `/home` from the service. `~/.ssh/authorized_keys` becomes unreadable. Key authentication stops working for every non-root user on the box.

`ProtectSystem=strict` makes the whole filesystem read-only except `/dev`, `/proc` and `/sys`, so any daemon writing to its own state directory fails to start unless you enumerate `ReadWritePaths`. `MemoryDenyWriteExecute=yes` kills every JIT: the JVM, Node, nginx with Lua, PHP with JIT on.

No set of sandbox directives is safe to apply to an arbitrary service. Sandboxing is per-unit work. Pick the service, read what it does, write the drop-in, check it with `systemd-analyze verify`, restart it, confirm it is still running. A blanket drop-in across a service list is an outage with extra steps.

## rpcbind is not just for NFS servers

Standard advice says mask obsolete network services, and `rpcbind` is always on the list.

`rpcbind` is required by NFS clients, not only servers. Mask it on a host with NFS mounts and those mounts break. Often not right away, but at the next remount or reboot, which is the worst time to find out.

Anything you are about to mask needs an in-use check first:

```console
# findmnt -t nfs,nfs4          # before touching rpcbind
# lpstat -p 2>/dev/null        # before touching cups
# exportfs -s                  # before touching nfs-server
```

If it is in use, skip it and say so. "Not applicable on this host" is a legitimate audit result. Masking it anyway to turn a checklist green is how hardening earned its reputation.

## -e 2 and a full disk

auditd's `-e 2` makes the ruleset immutable until reboot. That is the point of it, and it is reasonable on a high-assurance host.

The spec mandated it. The spec did not specify `disk_full_action`, `admin_space_left_action`, or the audit failure mode `-f`.

Default failure behaviour on some configurations is panic. Combine an immutable ruleset, an unset failure policy, and a `/var` partition filling up, and the kernel panics. Your hardening control took the server down, and because the rules are immutable you cannot back it out without the reboot you were about to get anyway.

If you use `-e 2`, set the disk policy first:

```
space_left_action = SYSLOG
admin_space_left_action = SINGLE
disk_full_action = SYSLOG
```

and use `-f 1` (printk) rather than `-f 2` (panic) unless you have a specific reason and a monitored disk.

## The pattern

Six of these come from the same place: the blacklist, the ciphers, the sysctl, the shadow permissions, the sandbox directives, the masked `rpcbind`. Someone took a control from CIS, which describes an outcome, and built the mechanism they assumed would produce it. Nobody checked the outcome on a real system.

CIS says "ensure cramfs kernel module is not available." That is a statement about system state. `blacklist cramfs` is a mechanism, and on its own it does not produce that state.

So the working rule, boring and durable:

Verify the effect, not the artifact. `sysctl -n`, not `cat`. `sshd -T`, not the drop-in file. `lsmod` plus a real `modprobe` attempt, not a line in `modprobe.d`. `systemctl show`, not the unit file. If your verification step reads back the same file your apply step wrote, you have tested that your disk works.

The second rule follows from the first. Every control needs an applicability check. `rp_filter=1` breaks asymmetric routing. `ip_forward=0` severs container networking. `accept_ra=0` strands a SLAAC-addressed host at the next lease expiry. None of these are wrong controls. They are controls with preconditions, and a hardening tool that skips the preconditions is a well-organised way to cause an incident.

---

I am rewriting that specification properly: capability probes instead of version assumptions, typed transaction states, an explicit register of controls that cannot be reversed without a reboot. The toolkit implementation is next. I will write that up as it goes.
