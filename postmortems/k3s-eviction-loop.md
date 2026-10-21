# 422 restarts from a single character

**Date:** 13 October 2026 · **Duration:** ~6 hours · **Impact:** control plane down, workloads unaffected

## Summary

A disk-pressure threshold I had added to the kubelet configuration used `=` where the
kubelet expects `<`. The kubelet refused to parse the flag and exited, k3s requested
shutdown, systemd restarted it, and the cycle repeated 422 times over six hours before
I noticed. The applications on the node kept serving traffic the whole time, which is
precisely why it took so long to notice.

## Background

The node is a single-server k3s cluster on a 457 GB disk that had drifted to 95% full.
The kubelet's default eviction thresholds are percentages, so 5% of that disk meant
reserving roughly 23 GB for no good reason. I replaced them with absolute values in
`/etc/rancher/k3s/config.yaml`:

```yaml
kubelet-arg:
  - "eviction-hard=nodefs.available=5Gi,imagefs.available=5Gi"   # wrong
```

## What actually happened

`eviction-hard` is the one kubelet flag that separates each pair with a comparison
operator, not an equals sign. The correct form is:

```yaml
kubelet-arg:
  - "eviction-hard=nodefs.available<5Gi,imagefs.available<5Gi"   # right
```

With `=`, the kubelet fails at flag parsing — `malformed pair, expect string<string` —
and never reaches the point of starting. k3s treats that as a fatal condition and asks
to shut down; systemd's restart policy brings it straight back up into the same failure.

The trap is that neighbouring settings in the same file use the opposite convention:
`eviction-minimum-reclaim` does take `=`, and the `image-gc-*-threshold` values are
plain scalars. The file reads as internally consistent, so reviewing it visually does
not catch the error.

## Why it was hard to see

The data plane survives the control plane. Pods already scheduled on the node keep
running, the ingress keeps routing, and the web interface keeps answering. Every check
I would normally reach for — load the site, is it up? — came back green. The cluster
was dead and the product was fine.

The second problem was `systemctl status k3s`: in a restart loop the unit alternates
between `activating` and `active`, and a glance at it at the wrong moment looks healthy.
Only `is-active` returning `activating`, or the restart counter moving, tells the truth.

## Resolution

Fix the operator, restart the unit, confirm the kubelet registers. Checks that actually
distinguish the failure:

```bash
systemctl is-active k3s                              # must be 'active', not 'activating'
journalctl -u k3s | grep "malformed pair"            # the parse error, if it is still there
kubectl get nodes                                    # the control plane is genuinely back
```

## What I changed afterwards

- **Never edit kubelet args without checking `is-active` afterwards.** The edit looks
  successful because the file saves; the failure is one layer down.
- **Do not diagnose a cluster through the application URL.** A green page proves the data
  plane is alive and says nothing about the control plane. I now check the node and the
  apiserver explicitly.
- **Write down the syntax, not just the reasoning.** The config file had a long comment
  explaining *why* the thresholds were absolute, which is what made the wrong line look
  deliberate and correct. It now records the operator rule too.
