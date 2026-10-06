# Multi spark cluster setup script

> **Fork of [NVIDIA/dgx-spark-playbooks](https://github.com/NVIDIA/dgx-spark-playbooks).**
> Upstream put the CX7 fabric on `192.168.0-5.0/24`, which collides with the
> usual `192.168.0.0/24` / `192.168.1.0/24` office LAN. This fork starts it at
> **`192.168.10.0/24`** and refuses to run if the block would overlap your
> management network. Verified on a 3-node ring: **22.66 GB/s** NCCL bus bandwidth.

```json
// config/spark_config_ring.json -- default 10, ring consumes B..B+5
{ "cluster_base_octet": 10, "nodes_info": [ ... ] }
```

```bash
bash spark_cluster_setup.sh -c config/spark_config_ring.json --run-setup
```

Node-side equivalent: `--base-octet N`. Subnets consumed from base `B`:
ring `B..B+5`, 2-node b2b and switch `B..B+1`.

> [!WARNING]
> **Stale NetworkManager connections silently wipe the fabric IPs.**
> If a node's CX7 interfaces were ever touched by NetworkManager, you get
> `/etc/netplan/90-NM-<uuid>.yaml` files claiming them with `dhcp4: true`.
> They sort *after* `40-cx7.yaml` and override the static addresses, so the
> interfaces come up with **no IP** and NCCL fails with `unhandled system
> error`. Setup still reports success — the IPs were assigned, then removed.
> Tell-tale: one node has an empty IP column while its peers are fine.
>
> ```bash
> # detect
> ls /etc/netplan/ | grep 90-NM
> nmcli -t -f UUID,NAME con show | grep -E 'netplan-(enp1s0|enP2p1)'
>
> # fix -- CX7 connections ONLY, never enP7s7 (that is your SSH session)
> nmcli -t -f UUID,NAME con show | grep -E 'netplan-(enp1s0|enP2p1)' \
>   | cut -d: -f1 | xargs -rn1 sudo nmcli con delete
> sudo rm -f /etc/netplan/90-NM-*.yaml
> ```
>
> This also deletes `40-cx7.yaml` — expected. Re-run `--run-setup` to
> regenerate it; don't hand-write it, the generator derives addresses from
> live MAC ordering.

### Notes

- Run from a node **in** the cluster, via the `.sh` wrapper (it sets the
  env var the Python script checks for).
- Full `--run-setup` takes ~8–12 min (20 s L2 discovery per node, then an
  NCCL build). Use `nohup ... &` and poll the log; a foreground SSH call
  will time out.
- In a ring, some node pairs *cannot* ping each other — each node only has
  interfaces on its two links. That is correct. The real check is NCCL bus
  bandwidth (`10 GB/s` ring floor, `21.875 GB/s` b2b/switch).
- Keep `"password"` empty in committed configs; set it locally only.
- Needs Python ≥3.12 — upstream uses PEP 701 nested f-string quotes, which
  older parsers misreport as `SyntaxError: f-string: unmatched '['`.
- `IP_PREFIX` / `LAST_OCTET_START` / `SUBNET_SIZE` in `spark_cluster_setup.py`
  are dead constants; the real allocation lives in the node script.

The full writeup is captured as the Hermes skill
**`dgx-spark-cluster-fabric-networking`** (`mlops/` category).

## Upstream usage

### Step 1. Clone the repo

Clone the dgx-spark-playbooks repo from GitHub

### Step 2. Switch to the multi spark cluster setup scripts directory

```bash
cd dgx-spark-playbooks/nvidia/multi-sparks-through-switch/assets/spark_cluster_setup
```

### Step 3. Create or edit a JSON config file with your cluster information

```bash
# Create or edit JSON config file under the `config` directory with the ssh credentials for your nodes.
# Adjust the number of nodes in "nodes_info" list based on the number of nodes in your cluster

# Example: (config/spark_config_b2b.json):
# {
#     "nodes_info": [
#         {
#             "ip_address": "10.0.0.1",
#             "port": 22,
#             "user": "nvidia",
#             "password": "nvidia123"
#         },
#         {
#             "ip_address": "10.0.0.2",
#             "port": 22,
#             "user": "nvidia",
#             "password": "nvidia123"
#         }
#
```

### Step 4. Run the cluster setup script with your json config file

The script can be run with different options as mentioned below

```bash
# To run validation, cluster setup and NCCL bandwidth test (all steps)

bash spark_cluster_setup.sh -c <JSON config file> --run-setup

# To only run pre-setup validation steps

bash spark_cluster_setup.sh -c <JSON config file> --pre-validate-only

# To run NCCL test and skip cluster setup (use this after cluster is already set up)

bash spark_cluster_setup.sh -c <JSON config file> --run-nccl-test

```

> [!NOTE]
> The full cluster setup (first command above) will do the following
> 1. Create a python virtual env and install required packages
> 2. Validate the environment and cluster config
> 3. Detect the topology and configure the IP addresses
> 4. Configure password-less ssh between the cluster nodes
> 5. Run NCCL BW test