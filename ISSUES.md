# K8s Cluster Issues & Fixes — 2026-05-21

## Cluster Info
- **Master:** 192.168.29.5 (sirpiserver2)
- **Workers:** node1 (192.168.29.6), node2 (192.168.29.7), node3 (192.168.29.10), node4 (192.168.29.8)
- **Kubernetes version at time of investigation:** v1.35.3
- **Target upgrade version:** v1.36.1

---

## Issue 1 — Ansible: Missing `ansible.posix` collection

**Symptom:**
```
[ERROR]: couldn't resolve module/action 'authorized_key'
No module named 'ansible_collections.ansible.posix'
```

**Cause:** The `ansible.posix` collection was not installed.

**Fix:**
```bash
ansible-galaxy collection install ansible.posix
```

---

## Issue 2 — Ansible: server5 SSH key copy failed (missing sudo)

**Symptom:**
```
Task failed: Missing sudo password
fatal: [server5]: FAILED
```

**Cause:** The playbook used `become: true` to write to `~/.ssh/authorized_keys`, but `sirpi-server5` user doesn't have passwordless sudo. Sudo is not needed to write to your own home directory.

**Fix:** Removed `become: true` and the `owner:` field from the `add-ssh-key.yaml` playbook.

---

## Issue 3 — Ansible: Python interpreter warning on all nodes

**Symptom:**
```
[WARNING]: Host 'serverX' is using the discovered Python interpreter at '/usr/bin/python3.12'
```

**Cause:** Ansible auto-discovers Python but warns when multiple versions exist.

**Fix:** Added to `[all:vars]` in inventory:
```ini
ansible_python_interpreter=auto_silent
```

---

## Issue 4 — K8s upgrade playbook: version hardcoded in 5 places

**Symptom:** Playbook had `k8s_version: "1.33.5"` hardcoded across 4 phases and `expected_version: "v1.33.5"` in the verify phase. Needed manual edits for every upgrade.

**Fix:**
- Updated all versions to `1.36.1`
- Changed `expected_version` to `"v{{ k8s_version }}"`
- Changed summary message to use `{{ k8s_version }}`
- Now pass version at runtime: `-e "k8s_version=1.36.1"`

---

## Issue 5 — K8s upgrade playbook: apt keyring `creates:` guard

**Symptom:** Keyring task would skip if `/etc/apt/keyrings/kubernetes-apt-keyring.gpg` already existed (from previous k8s install), meaning the new repo key for v1.36 would never be written.

**Fix:** Removed `creates:` from all 3 keyring tasks (master, additional masters, workers) and added `--yes` to gpg to force overwrite:
```yaml
ansible.builtin.shell: "curl -fsSL https://pkgs.k8s.io/core:/stable:/{{ k8s_short_version }}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg --yes"
```

---

## Issue 6 — K8s upgrade: Phase 1 (plan) not running during `--tags apply`

**Symptom:**
```
Specified version to upgrade to "v1.36.1" is at least one minor release higher than the kubeadm minor release (36 > 35)
```

**Cause:** Phase 1 installs the new kubeadm binary, but it only had `tags: [plan]`. When running `--tags apply`, Phase 1 was skipped and Phase 2 ran with the old kubeadm (v1.35.3).

**Fix:** Added `apply` to Phase 1's tags:
```yaml
tags: [plan, apply]
```

---

## Issue 7 — K8s upgrade: master node unnecessarily drained

**Symptom:** Playbook drained the master before upgrading it. With a single master, this evicts workloads for no benefit.

**Cause:** Drain/uncordon was designed for HA multi-master setups.

**Fix:** Removed the drain and uncordon tasks from Phase 2 (single master upgrade). `kubeadm upgrade apply` restarts static pods in-place without needing a drain.

---

## Issue 8 — etcdctl `version` subcommand not found

**Symptom:**
```
No help topic for 'version'
non-zero return code
```

**Cause:** The system `etcd-client` package is an older version that uses `--version` flag instead of `version` subcommand.

**Fix:**
```yaml
command: "etcdctl --version"
```

---

## Issue 9 — node4 (sirpi-server5) missing passwordless sudo

**Symptom:**
```
Task failed: Missing sudo password
fatal: [node4]: FAILED
```

**Cause:** `sirpi-server5` user not in sudoers with NOPASSWD.

**Fix:**
```bash
ansible server5 -i ansible/inventory-ssh-setup.ini \
  -m shell \
  -a "echo 'sirpi-server5 ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/sirpi-server5" \
  --become --ask-become-pass
```

---

## Issue 10 — Control plane instability: apiserver restarting 489 times over 62 days

**Symptom:**
```
kube-apiserver-master       1/1   Running   489 restarts   62d
kube-controller-manager     0/1   CrashLoopBackOff   514 restarts
kube-scheduler              0/1   CrashLoopBackOff   501 restarts
```

**Root cause:** Master node running on a slow disk. During disk I/O spikes, etcd's fsync takes too long. The apiserver liveness probe had a **1 second timeout** (default) — any etcd response over 1s caused the probe to fail, kubelet killed the apiserver, and the controller-manager/scheduler cascaded into CrashLoopBackOff.

**Cascade:**
```
Disk I/O spike
→ etcd slow to respond
→ apiserver liveness probe times out (1s default too tight)
→ kubelet kills apiserver
→ controller-manager & scheduler lose leader election lease
→ they exit (designed behavior)
→ kubelet restarts them → 5 minute backoff builds up
→ stale leases block new instances from acquiring leadership
```

**Fix 1 — apiserver liveness probe** (`/etc/kubernetes/manifests/kube-apiserver.yaml`):
```yaml
livenessProbe:
  failureThreshold: 20   # was 8
  timeoutSeconds: 15     # was missing (defaulted to 1s)
```

**Fix 2 — controller-manager leader election** (`/etc/kubernetes/manifests/kube-controller-manager.yaml`):
```
--leader-elect-lease-duration=60s   # was 15s
--leader-elect-renew-deadline=40s   # was 10s
--leader-elect-retry-period=5s      # was 2s
```

**Fix 3 — scheduler leader election** (`/etc/kubernetes/manifests/kube-scheduler.yaml`):
```
--leader-elect-lease-duration=60s
--leader-elect-renew-deadline=40s
--leader-elect-retry-period=5s
```

**Fix 4 — clear stale leases and reset backoff:**
```bash
kubectl delete lease kube-controller-manager kube-scheduler -n kube-system
kubectl delete pod kube-controller-manager-master kube-scheduler-master -n kube-system
```

---

## Issue 11 — Worker drain forbidden after master upgrade

**Symptom:**
```
Error from server (Forbidden): nodes "node1" is forbidden: User "kubernetes-admin" cannot get resource "nodes"
```

**Cause:** `kubeadm upgrade apply` renews all certificates on the master. The existing `/root/.kube/config` still had the old certificate embedded, so kubectl calls were rejected by the apiserver.

**Fix (manual):**
```bash
cp /etc/kubernetes/admin.conf /root/.kube/config
```

**Fix (in playbook):** Added a task in Phase 2 to automatically refresh the kubeconfig right after the upgrade applies, before any kubectl commands run.

---

## Issue 12 — dpkg interrupted by broken nvidia-dkms-535 on master

**Symptom:**
```
E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'
Error! Bad return status for module build on kernel: 6.8.0-111-generic
```

**Cause:** `nvidia-dkms-535` was trying to build a kernel module but failed. This blocked dpkg from configuring any other packages including kubelet and kubectl.

**Fix:**
```bash
apt-get remove --purge -y --allow-change-held-packages nvidia-dkms-535 nvidia-driver-535
dpkg --configure -a
```

The master node has no GPU — NVIDIA drivers should never have been installed on it.

---

## Permanent Fix Still Pending

**Move etcd to a dedicated SSD.** All fixes above are workarounds that give the cluster more tolerance for disk slowness. The real fix is giving etcd its own fast disk so I/O from other processes doesn't affect it.

Steps (to be done after upgrade):
1. Attach a dedicated SSD to the master node
2. Take a fresh etcd backup
3. Stop the apiserver (move manifest out of `/etc/kubernetes/manifests/`)
4. Move etcd data: `mv /var/lib/etcd /mnt/ssd/etcd`
5. Update etcd manifest `--data-dir` to point to the new location
6. Restore apiserver manifest
7. Verify cluster health

---

## Issue 13 — longhorn-csi-plugin on node2: exec format error (corrupt containerd snapshot)

**Symptom:**
```
exec /csi-node-driver-registrar: exec format error
exit code 255
```

**Root cause:** The image `longhornio/csi-node-driver-registrar:v2.15.0-20251226` had an incomplete download on node2 3 months prior. The blob `sha256:d95a15a...` (14MB layer containing the binary) was tracked in containerd's metadata but never written to disk. When the layers were unpacked into snapshot 5418, the missing blob produced a 0-byte binary. Every restart reused the broken snapshot.

**Fix:**
```bash
# On node2
crictl rmi docker.io/longhornio/csi-node-driver-registrar:v2.15.0-20251226
ctr -n k8s.io content rm sha256:d95a15a563123c2c93d15539ae4e9340e0ae3f78a7543003bba12c3411550beb
ctr -n k8s.io content rm sha256:6f4882febaf7a1c8ee2babe05ccbe88d335b4db8aec5019c1b318a7d5dd5844b
ctr -n k8s.io content rm sha256:b3c66459cb2f7676d64acfd574ad957030b0eec056aa5d1ba86e0f7c6c572745
systemctl stop containerd
rm -rf /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/5418
systemctl start containerd
```
Then delete the failing pod so Kubernetes re-pulls the image cleanly.

---

## Issue 14 — nvidia-device-plugin running on non-GPU nodes (CrashLoopBackOff)

**Symptom:**
```
failed to inject CDI devices: unresolvable CDI devices runtime.nvidia.com/gpu=all
exit code 128 (2331 restarts over 9 days)
```

**Root cause:** The nvidia-device-plugin DaemonSet had no `nodeSelector`, so it ran on all nodes including node3 (which had no working NVIDIA driver at the time). It would also run on node1 which has no CDI config.

**Fix 1 — restrict to GPU nodes only:**
```bash
kubectl patch daemonset nvidia-device-plugin-daemonset -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/nodeSelector","value":{"gpu":"true"}}]'
```

**Fix 2 — node3 NVIDIA driver not loaded (kernel updated, DKMS modules missing):**
```
Running kernel:  6.17.0-29-generic
Modules built for: 6.17.0-22-generic  ← mismatch
```
```bash
# On node3
apt-get install -y linux-modules-nvidia-580-6.17.0-29-generic
modprobe nvidia
mkdir -p /etc/cdi
nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
systemctl restart containerd
```

---

## Issue 15 — node1 NotReady after k8s 1.36 upgrade (cgroup v1 not supported)

**Symptom:** node1 stuck at v1.35.3/NotReady after upgrade. kubelet fails to start.

**Root cause:** k8s 1.36 dropped cgroup v1 support. node1 had `systemd.unified_cgroup_hierarchy=0` explicitly set in `/etc/default/grub`, forcing cgroup v1.

**Fix:**
```bash
# On node1
sed -i 's/systemd.unified_cgroup_hierarchy=0/systemd.unified_cgroup_hierarchy=1/' /etc/default/grub
update-grub
reboot
```

Note: the grub file had two `GRUB_CMDLINE_LINUX` entries — the second one with `=0` was the active one. Generic sed replacing `=""` would not have matched.

---

## Issue 16 — Longhorn volumes faulted after node1 went NotReady

**Symptom:** 7 Longhorn volumes in `detached/faulted` state. Grafana, n8n, rustfs, ferro-metrics pods stuck in `Init:0/1` or Pending.

**Root cause:** All affected volumes had their **only replica on node1**. When node1 went NotReady (Issue 15), replicas stopped. Single-replica volumes with no redundancy = total loss of access when that node goes down.

**Cascade:**
```
node1 NotReady
→ Longhorn instance manager on node1 stopped
→ all single-replica volumes on node1 → detached/faulted
→ grafana PVC unattachable → Init:0/1
→ n8n/rustfs/ferro-metrics PVCs same
```

**Fix:** Reboot node1 with cgroup v2 (Issue 15 fix). After node1 rejoined, Longhorn detected orphaned replica directories. For volumes where data is acceptable to lose, delete PVCs and let workloads recreate them fresh.

**Lesson: always configure Longhorn volumes with ≥2 replicas so a single node failure doesn't cause data unavailability.**

**Affected PVCs (data deleted/recreated):**
- `monitoring/kube-prometheus-stack-grafana`
- `n8n/data-n8n-postgresql-0`
- `n8n/node-modules-n8n-0`
- `n8n/redis-data-n8n-redis-master-0`
- `rustfs/rustfs-data`
- `rustfs/rustfs-logs`
- `ferro-metrics/ferrometrics-dataset-pvc`
