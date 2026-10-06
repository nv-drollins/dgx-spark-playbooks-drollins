# Multi spark cluster setup script

> [!IMPORTANT]
> **Fork note — cluster fabric subnet range.**
> Upstream NVIDIA allocated the high-speed cluster fabric starting at
> `192.168.0.0/24`, so a 3-node ring consumed `192.168.0.0/24` through
> `192.168.5.0/24`. That collides with the most common home/office management
> LANs (`192.168.0.0/24` and `192.168.1.0/24`) — the fabric steals the subnet
> your copper network is on and breaks routing to the nodes.
>
> This fork starts the fabric at **`192.168.10.0/24`** instead, so a 3-node
> ring uses `192.168.10.0/24` .. `192.168.15.0/24`.
>
> Control it with `"cluster_base_octet"` in your JSON config (default `10`):
>
> ```json
> { "cluster_base_octet": 10, "nodes_info": [ ... ] }
> ```
>
> The setup script now **refuses to run** if the fabric block would overlap
> the management IPs of any node in the config, instead of silently taking
> down your network. The node-side script also accepts `--base-octet N`.
>
> Subnets consumed per topology (base `B`, default 10):
>
> | Topology     | Subnets used        | Default (B=10)            |
> |--------------|---------------------|---------------------------|
> | 2-node b2b   | `B` .. `B+1`        | `192.168.10-11.0/24`      |
> | 3-node ring  | `B` .. `B+5`        | `192.168.10-15.0/24`      |
> | switch       | `B` .. `B+1`        | `192.168.10-11.0/24`      |

## Usage

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