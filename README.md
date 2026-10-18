# openshift_qnap_csi

QNAP CSI (Trident) v1.6.2 over **iSCSI** on a single-node OpenShift cluster, backed by a
QNAP TS-h886 running **QuTS hero**.

This repo records the configuration that is verified working on this cluster, plus the
operational traps found while bringing it up.

## Status

| Item | Value |
|------|-------|
| Cluster | OpenShift 4.22.18 / Kubernetes v1.35.8 |
| Topology | Single node (`sno.syangsao.net`), control-plane + worker |
| Node OS | RHCOS 9.8 |
| NAS | `nas01` — QNAP TS-h886, **QuTS hero h5.2.9** |
| iSCSI portal | `192.168.40.20:3260` (VLAN 40, routable) |
| Driver | `qnapsystem/qnap-csi:v1.6.2` via `TridentOrchestrator` |
| Backend | `TridentBackendConfig/backend-nas01` (namespace `trident`) |
| StorageClass | `qnap-iscsi` |
| Verified | 2 Gi PVC bound, pod mounted `/dev/mapper/mpathb` (ext4), read/write OK, delete OK |

Driver compatibility: upstream v1.6.2 supports QTS 5.1+ **and** QuTS hero h5.1+ — this NAS
(h5.2.9) is in scope. QuTS hero-specific volume options (`Compression`, `Deduplication`,
`FastClone`) exist in the driver but are not used here.

## Architecture

```
OpenShift SNO (sno.syangsao.net, 192.168.40.45)
└── namespace trident
    ├── trident-operator        (deployment)
    ├── trident-controller      (deployment, 6 containers)
    └── trident-node-linux      (daemonset, runs privileged on the node)
        │
        │  management API  http://192.168.40.20:8080  (authLogin.cgi)
        │  iSCSI data     192.168.40.20:3260
        ▼
nas01 (QuTS hero)
    └── storage pools zpool1 / zpool2 / zpool3
        └── driver-created zvol + iSCSI target/LUN
```

## Prerequisites (verified state)

Node-side, no extra MachineConfigs were required on this cluster:

* `iscsid.socket` is **enabled** and `iscsid` is active (socket activation).
* `multipathd` is enabled and active.
* `/etc/multipath.conf` contains `find_multipaths no` (required so single-path iSCSI LUNs are
  claimed deterministically).
* `iscsi-initiator-utils` and `device-mapper-multipath` are present in RHCOS 9.8.
* No active iSCSI sessions before the test.

NAS-side:

* iSCSI service enabled; target port 3260 listening.
* A valid administrator account. **The password must be exactly correct** — see
  [QNAP Access Protection](#qnap-access-protection-the-biggest-trap).

Network:

* Pods egress with SNAT to the node's `br-ex` address: **192.168.40.45**.
* The NAS management/iSCSI address must be routable from that source. Here `192.168.40.20`
  is; `192.168.80.20` (10GbE) is **not routable** and only works from the node's host
  namespace, not from pods. Use `192.168.40.20` in the backend.

## Install

1. Install the upstream driver (v1.6.2):

   ```bash
   git clone https://github.com/qnap-dev/QNAP-CSI-PlugIn.git
   cd QNAP-CSI-PlugIn
   oc apply -k Deploy/crds
   oc apply -k Deploy/Trident
   oc get pods -n trident
   ```

   The orchestrator used here is `qnapsystem/qnap-csi:v1.6.2` (see upstream
   `Deploy/Trident/tridentorchestrator.yaml`).

2. Create the secret (edit the password first — it is **not** committed):

   ```bash
   $EDITOR manifests/secret.yaml
   oc apply -f manifests/secret.yaml
   ```

3. Create the backend and StorageClass:

   ```bash
   oc apply -f manifests/backend.yaml
   oc apply -f manifests/storageclass.yaml
   oc get tridentbackendconfig -n trident
   ```

   Expected:

   ```
   NAME            BACKEND NAME   BACKEND UUID                             PHASE   STATUS
   backend-nas01   nas01          13bd18b0-a266-47c5-9cdc-bc64b27742cf             Success
   ```

4. Provision a test volume:

   ```bash
   oc apply -f manifests/example-pvc.yaml
   oc get pvc qnap-iscsi-example
   ```

## Configuration

### Secret (`trident/backend-nas01-secret`)

```yaml
stringData:
  username: syangsao
  password: "<NAS admin password>"   # must be correct, see warning below
  storageAddress: "192.168.40.20"
  https: "false"
```

`storageAddress` is both the management API address (port 8080) and the iSCSI portal.

### Backend (`trident/backend-nas01`)

```yaml
spec:
  version: 1
  storageDriverName: qnap-nas   # supports QTS and QuTS hero
  backendName: nas01
  credentials:
    name: backend-nas01-secret
  storage:
    - serviceLevel: k8s-pool    # virtual pool name (arbitrary)
      labels:
        usage: kubernetes       # matched by the StorageClass selector
```

A `TridentBackendConfig` change to the **secret** does not trigger reconciliation. Bump the
backend config to force it:

```bash
oc annotate tridentbackendconfig backend-nas01 -n trident \
  force-reconcile="$(date +%s)" --overwrite
```

### StorageClass (`qnap-iscsi`)

```yaml
provisioner: csi.trident.qnap.io
parameters:
  selector: "usage=kubernetes"
  fsType: "ext4"
  replacementTimeout: "120"
allowVolumeExpansion: true
```

PVCs are `ReadWriteOnce` block volumes formatted ext4. Default binding is `Immediate`.

## Verification performed

```text
$ oc get pvc qnap-iscsi-test
NAME              STATUS   VOLUME                                     CAPACITY   STORAGECLASS
qnap-iscsi-test   Bound    pvc-bd3873f3-dd35-4b83-a8fc-9c13d529b20f   2Gi        qnap-iscsi

$ oc logs qnap-iscsi-test
mounted: /dev/mapper/mpathb ext4  2.0G   24K  1.8G   1% /data
qnap-csi-ok 1792326366

$ oc exec qnap-iscsi-test -- df -hT /data
Filesystem         Type  Size  Used Avail Use% Mounted on
/dev/mapper/mpathb ext4  2.0G   28K  1.8G   1% /data

$ # on the node
$ iscsiadm -m session
tcp: [1] 192.168.40.20:3260,1 iqn.2004-04.com.qnap:ts-h886:iscsi.iscsi-trident-pvc-<id>.<suffix>

$ multipath -ll
mpathb (36e843b631...) dm-0 QNAP,iSCSI Storage
size=2.0G features='0' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=50 status=active
  `- 33:0:0:1 sdb 8:16 active ready running
```

The PV is a block volume with `protocol: block` in its CSI attributes. Deleting the PVC
removes the LUN, target, iSCSI session and multipath device.

## QNAP Access Protection (the biggest trap)

QNAP's **Security → Access Protection** bans a source IP for **5 minutes** after repeated
failed logins. With an incorrect password in the secret, Trident's reconcile retry (~every
5 minutes) re-triggers the ban forever:

```
[Security] Added IP address "192.168.40.45" to IP block list. Duration: for 5 minutes.
[Security] Removed IP "192.168.40.45" from IP block list.
[Security] Added IP address "192.168.40.45" to IP block list. Duration: for 5 minutes.
```

Symptoms while banned:

* `TridentBackendConfig` status: `Failed` with
  `dial tcp 192.168.40.20:8080: i/o timeout`.
* `ping 192.168.40.20` still works from the node; **all TCP** is dropped.
* NAS `iptables` is empty; the block is applied outside netfilter.

Recovery:

```bash
# 1. Fix the password in the secret.
oc apply -f manifests/secret.yaml

# 2. Stop the retry loop so the ban can expire.
oc scale deploy/trident-controller -n trident --replicas=0
sleep 300   # wait out the 5-minute block

# 3. Confirm management + iSCSI ports are reachable, then resume.
oc scale deploy/trident-controller -n trident --replicas=1
oc annotate tridentbackendconfig backend-nas01 -n trident \
  force-reconcile="$(date +%s)" --overwrite
```

A persistent wrong password can black out storage for 5 minutes per attempt, and because the
driver retries on the same cadence, the ban can renew indefinitely.

## Physical pool selection

The driver chose **zpool1** (SATA SSD RAID-1) for the test volume; it was created as
`zpool1/zfs279` and deleted on PVC cleanup.

* `serviceLevel` names a **virtual** pool; it does **not** select a physical pool.
* Physical pools are filtered by the backend `features` map against pool attributes.
* On this NAS, zpool1 and zpool3 both report `raidlevel:1, ssdcache:false, tiering:Disable`,
  i.e. identical matchable attributes. There is no documented feature that pins zpool3.
* `Index`/`index` features were tried to target zpool3 and both failed with
  `backend has no physical pools that can satisfy request`.

So with driver v1.6.2, dynamic provisioning cannot be pinned to zpool3 on this pool layout.
Placing a volume specifically on zpool3 would require the driver's import path against a
pre-created LUN (not tested here), or a change on the QNAP side. (`raidLevel`, `ssdCache`
and `tiering` are the documented feature filters.)

## Operations

Create:

```bash
oc apply -f manifests/example-pvc.yaml
```

Expand (supported by the StorageClass):

```bash
oc patch pvc qnap-iscsi-example -p \
  '{"spec":{"resources":{"requests":{"storage":"4Gi"}}}}'
```

Delete:

```bash
oc delete pvc qnap-iscsi-example
```

## Uninstall

```bash
oc delete -f manifests/example-pvc.yaml
oc delete tridentbackendconfig backend-nas01 -n trident
oc delete storageclass qnap-iscsi
oc delete tridentorchestrator trident
```

The upstream driver also documents `tridentctl uninstall -n trident` for a full removal.
