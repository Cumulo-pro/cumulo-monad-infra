# Upgrade Log

Non-trivial or undocumented operations on the Monad testnet node.

---

## 2026-05-01 - CVE-2026-31431 Mitigation (algif_aead)

**Type:** Security mitigation  
**Node:** full_Cumulo-1 (192.155.100.132)  
**Announced by:** Monad Foundation

### Context

A local privilege escalation vulnerability was disclosed in the Linux kernel affecting the `algif_aead` module. The vulnerability allows an unprivileged user to gain root access on affected systems.

### Mitigation applied

Official interim mitigation recommended by the Monad Foundation. Executed as root:

```bash
echo "install algif_aead /bin/false" > /etc/modprobe.d/disable-algif.conf
rmmod algif_aead 2>/dev/null || true
modprobe algif_aead
lsmod | grep algif_aead | wc -l
rmmod algif_aead 2>/dev/null || true
```

### Verification

Output confirmed correct:

```
Verifying...
modprobe: ERROR: ../libkmod/libkmod-module.c:1084 command_do() Error running install command '/bin/false' for module algif_aead: retcode 1
modprobe: ERROR: could not insert 'algif_aead': Invalid argument
0
```

### Result

- Module blacklisted via `/etc/modprobe.d/disable-algif.conf` - persists across reboots
- Module unloaded immediately from memory
- No node restart required

### Pending

Long-term fix: update to a kernel containing the upstream patch for CVE-2026-31431.
