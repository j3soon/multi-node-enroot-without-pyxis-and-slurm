# Running Multi-Node Tasks with Enroot (without Pyxis and Slurm)

## Introduction

Setting up environments for user applications on an HPC cluster is often tedious and divert attention from the application itself. To tackle this, [containerization support](https://developer.nvidia.com/blog/deploying-rich-cluster-api-on-dgx-for-multi-user-sharing/) is a great way to simplify the process. HPC clusters often use [Slurm](https://slurm.schedmd.com/documentation.html) workload manager along with containerization tools such as [Singularity](https://sylabs.io/singularity/)/[Apptainer](https://apptainer.org/), [Rootless Docker](https://docs.docker.com/engine/security/rootless/) (environment module), or [Enroot](https://github.com/NVIDIA/enroot)+[Pyxis](https://github.com/NVIDIA/pyxis), for easier environment management.

Based on my experience working with Slurm and all these containerization options, I personally prefer Slurm with Enroot+Pyxis as it offers the simplest workflow for users familiar with Docker, while also ensuring [minimal performance overhead](https://github.com/NVIDIA/pyxis/wiki#presentations).

The setup instructions are already documented in the official Pyxis repository. Enroot documentation also contains detailed usage guide on single-node tasks. However, there is no documentation for running multi-node tasks directly with Enroot without Pyxis. Using Enroot directly without Pyxis may be needed when you have direct (bare metal) access to multiple Ubuntu nodes and do not want to set up a scheduler or use a workload manager like Slurm. In such cases, Enroot itself alone can serve as a lightweight and effective containerization solution for HPC environments.

This (unofficial) document describes the minimal setup required for running multi-node tasks directly with Enroot without Pyxis and Slurm. Please note that running multi-node tasks with Enroot is more of a hack than a fool-proof solution, the recommended method for multi-node tasks remains using Enroot+Pyxis.

## Sample Environment

- GPU Hardware: Two nodes, each with eight H200 NVL GPUs
  > The commands below can be easily adapted to arbitrary number of nodes (or other NVIDIA GPUs).
- Network Hardware: Each node is equipped with four ConnectX-7 NICs, providing eight InfiniBand connections per node through InfiniBand switches
  > Ethernet (RoCE) should also work in theory.
- OS: Ubuntu 22.04.5 LTS
  > Other Linux distributions should also work.
- Pre-installed Software: [NVIDIA Driver](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/), [Docker](https://docs.docker.com/engine/install/ubuntu/), [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html), InfiniBand Driver ([DOCA-Host](https://developer.nvidia.com/doca-downloads?deployment_platform=Host-Server) or [MLNX_OFED (legacy)](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)).
  > Not sure if Docker and Container Toolkit are required, but we have them installed by default on all nodes.
- A running [OpenSM](https://docs.nvidia.com/networking/display/mlnxofedv461000/opensm) on the InfiniBand switch or manually launched.
  > Usually it's already running on the InfiniBand switch.
- All nodes have IP addresses assigned within a private network
  > Assume require VPN connection to access the nodes.

## Prerequisites

Most multi-node cluster will have basic user account, NFS, and SSH configured. If not, you'll need to set up these first.

### User Account

Create a user account (with same username/UID/GID) with sudo privileges on all nodes, with the home directory set to `/mnt/home/<username>`. If the user already exists, skip this step.

You'll want to use tools like [LDAP](https://documentation.ubuntu.com/server/how-to/openldap/install-openldap/) to manage the user account. Alternatively, you can manually create the user account on all nodes:

```sh
# Create user account
USERNAME=<username>
sudo useradd -m -d /mnt/home/${USERNAME} -s /bin/bash -u 10001 -g 10001 -G sudo ${USERNAME}
# Enable password-less sudo
echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
```

### Network File System (NFS)

[Set up](https://documentation.ubuntu.com/server/how-to/networking/install-nfs/) an NFS server on the head node and mount the shared home directory on all other nodes. If NFS is already configured, ensure the necessary paths are exported and mounted correctly.

On head node:

```sh
# Install
sudo apt update
sudo apt install -y nfs-kernel-server
sudo systemctl start nfs-kernel-server.service
# Export
sudo mkdir -p /mnt/home/${USER}
sudo chown -R $(id -u):$(id -g) /mnt/home/${USER}
echo "/mnt/home     *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -a
```

On all other nodes:

```sh
# Install
sudo apt install -y nfs-common
# Mount
NFS_SERVER=<HEAD-NODE-IP>
sudo mkdir -p /mnt/home
echo "$NFS_SERVER:/mnt/home /mnt/home nfs defaults 0 0" | sudo tee -a /etc/fstab
sudo mount -a
mount | grep nfs
```

### SSH Configuration

Skip this step if password-less SSH is already configured.

On head node:

```sh
# Generate SSH key
ssh-keygen -t ed25519 # and press Enter multiple times to accept the default values
# Copy to shared home directory (will automatically work on all nodes due to shared home directory)
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```

## Install Enroot

[Download Enroot](https://github.com/NVIDIA/enroot/blob/main/doc/installation.md) to the shared directory.

On head node, download Enroot deb files:

```sh
cd /mnt/home/${USER}
mkdir -p enroot/deb && cd ~/enroot/deb
arch=$(dpkg --print-architecture)
curl -fSsL -O https://github.com/NVIDIA/enroot/releases/download/v3.5.0/enroot_3.5.0-1_${arch}.deb
curl -fSsL -O https://github.com/NVIDIA/enroot/releases/download/v3.5.0/enroot+caps_3.5.0-1_${arch}.deb # optional
```

On all nodes, [install Enroot](https://github.com/NVIDIA/enroot/blob/main/doc/installation.md):

```sh
# run once for all nodes (including head node)
IP=<IP>
ssh $IP 'cd ~/enroot/deb && sudo apt install -y ./*.deb'
```

You may run the [ubuntu example](https://github.com/NVIDIA/enroot) or [cuda example](https://github.com/NVIDIA/enroot/blob/main/doc/usage.md) to test the installation. But a simple `enroot version` is usually sufficient.

## Multi-node Setup

On all nodes, edit Enroot config for shared container file system (assumes Bash shell):

```sh
# run once for all nodes (including head node)
IP=<IP>
# You may edit /etc/enroot/enroot.conf directly, but the following idempotent commands are recommended for consistency
# Set ENROOT_DATA_PATH to /mnt/home/${USER}/enroot/data
ssh $IP "sudo grep -q '^ENROOT_DATA_PATH[[:space:]]\+/mnt/home/${USER}/enroot/data\$' /etc/enroot/enroot.conf || sudo sed -i '/^#ENROOT_DATA_PATH[[:space:]]\+\\\${XDG_DATA_HOME}\/enroot\$/a ENROOT_DATA_PATH           /mnt/home/${USER}/enroot/data' /etc/enroot/enroot.conf"
# Set ENROOT_MOUNT_HOME to yes
ssh $IP "sudo grep -q '^ENROOT_MOUNT_HOME[[:space:]]\+yes\$' /etc/enroot/enroot.conf || sudo sed -i '/^#ENROOT_MOUNT_HOME[[:space:]]\+no\$/a ENROOT_MOUNT_HOME          yes' /etc/enroot/enroot.conf"
```

On head node, add Enroot hook for OpenMPI:

```sh
sudo tee /etc/enroot/hooks.d/ompi.sh > /dev/null << 'EOF'
#!/bin/bash

echo "OMPI_MCA_orte_launch_agent=enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app ${ENROOT_ROOTFS##*/} orted" >> "${ENROOT_ENVIRON}"
EOF
sudo chmod +x /etc/enroot/hooks.d/ompi.sh
```

## Download and Create Container Image

On head node, download NGC [HPC-Benchmarks](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks) container image to the shared directory:

```sh
cd /mnt/home/${USER}/enroot
mkdir -p sqsh && cd sqsh
enroot import docker://nvcr.io#nvidia/hpc-benchmarks:25.04
ls ./nvidia+hpc-benchmarks+25.04.sqsh
```

On head node, create container (the created container will be visible on all nodes due to the `ENROOT_DATA_PATH` setting we set earlier):

```sh
cd /mnt/home/${USER}/enroot
mkdir -p data && cd sqsh
enroot create --name hpc-benchmarks-25-04 nvidia+hpc-benchmarks+25.04.sqsh
ls ../data/hpc-benchmarks-25-04
enroot list
# Single node MPI quick test
enroot start hpc-benchmarks-25-04 mpirun hostname
```

Create a workspace directory, and store the hostfile there (for multi-node tasks, assuming all nodes have 8 GPUs):

```sh
cd /mnt/home/${USER}/enroot
mkdir -p workspace && cd workspace
IP_LIST=("<IP1>" "<IP2>")
for IP in "${IP_LIST[@]}"; do
    echo "$IP slots=8" >> hosts.txt
done
cat hosts.txt
```

Run multi-node quick test (assuming 2 nodes with 8 GPUs each):

```sh
enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app hpc-benchmarks-25-04 mpirun -np 16 --hostfile /app/hosts.txt hostname
# should see 8 hostnames for each node
```

> **Note**: The command prefix before the container name (i.e., `enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app`) must match exactly what is set in the `/etc/enroot/hooks.d/ompi.sh` hook. **Do not modify** this part of the command, or the multi-node launch will not work correctly. You can change everything after the container name (e.g., `mpirun ...`) though. In addition, it is highly recommended to use absolute paths in the command.

## Running HPL

Prepare a suitable `HPL.dat` file for your machine.

```sh
enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app hpc-benchmarks-25-04
# in the container
cp hpl-linux-x86_64/sample-dat/HPL-H200-8GPUs.dat /app/
cp hpl-linux-x86_64/sample-dat/HPL-H200-16GPUs.dat /app/
# Ctrl+D to exit the container
```

Test single node HPL:

```sh
enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app hpc-benchmarks-25-04 mpirun -np 8 ./hpl.sh \
  --dat /app/HPL-H200-8GPUs.dat
```

> The result may not be optimal. You may tune the dat file, mpirun flags, and environment variables according to your machine for better HPL performance.

Test multi-node HPL:

```sh
enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app hpc-benchmarks-25-04 mpirun -np 16 \
  --hostfile /app/hosts.txt \
  --mca mca_base_env_list "HPL_USE_NVSHMEM=0" \
  ./hpl.sh \
  --dat /app/HPL-H200-16GPUs.dat
```

> The result may not be optimal. You may tune the dat file, mpirun flags, and environment variables according to your machine for better HPL performance.

> NVSHMEM is disabled by default to provide less assumptions about the network hardware. See: [How to run HPL script over Ethernet](https://forums.developer.nvidia.com/t/how-to-run-hpl-script-over-ethernet/294684/4).

> (To be verified) Theoretically, you can enable NVSHMEM (remove `HPL_USE_NVSHMEM=0`) if you are using InfiniBand and have correctly set up GPUDirect RDMA ([DMA-BUD](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/24.6.2/gpu-operator-rdma.html#about-gpudirect-rdma-and-gpudirect-storage), or [nvidia-peermem (legacy)](https://docs.nvidia.com/cuda/gpudirect-rdma/#using-nvidia-peermem) installed with [`.run` driver install](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/23.6.1/gpu-operator-rdma.html#about-gpudirect-rdma-and-gpudirect-storage), or [nv_peer_memory (legacy) on GitHub](https://github.com/Mellanox/nv_peer_memory)) on all nodes. See more at: [NVSHMEM requirements](https://docs.nvidia.com/nvshmem/release-notes-install-guide/install-guide/abstract.html#software-requirements). I'm not sure if NVSHMEM is supported over Ethernet though...

All done! Now you can use Enroot to run your multi-node tasks with ease!

## FAQ

### What is Inside the HPC-Benchmarks Container?

All software other than system-level drivers and kernel modules are included in the container.

NVIDIA HPC-Benchmarks 25.04 includes:
- [Sample files](https://docs.nvidia.com/nvidia-hpc-benchmarks/HPL_benchmark.html) such as HPL dat files.
- [HPL, HPL-MxP, HPCG, STREAM](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks)
- [NCCL, NVSHMEM, GDR Copy](https://docs.nvidia.com/nvidia-hpc-benchmarks/release_notes.html)
- [NVIDIA Optimized Frameworks 25.01](https://docs.nvidia.com/deeplearning/frameworks/support-matrix/index.html#framework-matrix-2025)
  - including: CUDA, cuBLAS, cuDNN, cuTENSOR, DALI, NCCL, TensorRT, rdma-core, NVIDIA HPC-X (OpenMPI, UCX), Nsight Compute, Nsight Systems, [and more (by searching `25.01`)](https://docs.nvidia.com/deeplearning/frameworks/support-matrix/index.html#framework-matrix-2025).

So basically all software other than those listed in the [Sample Environment](#sample-environment) is included in the container.

For sanity check:

```sh
enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app hpc-benchmarks-25-04
# in the container
ucx_info -v
ompi_info | grep "MPI extensions"
# ...
# Ctrl+D to exit the container
```

You can see that both [UCX and OpenMPI](https://docs.open-mpi.org/en/main/tuning-apps/networking/cuda.html) are built with CUDA support, even though you may not have installed UCX, OpenMPI, or even CUDA on the host OS.

### How does the Multi-node Setup Work?

To the best of my knowledge, this Enroot multi-node setup (or hack) is first introduced by [@3XX0](https://github.com/3XX0) in [this issue](https://github.com/NVIDIA/enroot/issues/49).

Aside from normal single-node Enroot setup, there are four major points in [Multi-node Setup](#multi-node-setup):

1. Setting `ENROOT_DATA_PATH` to a NFS shared directory in `/etc/enroot/enroot.conf`.  
   This path is used to store the container file system (unpacked by `enroot create`). Setting it to a NFS shared directory ensures that the container file system is visible (by `enroot list`) on all nodes once being created. Without this option, user need to manually run `enroot create` on each node, which is tedious and error-prone. Executing `enroot remove` will delete the container file system from this path. ([Reference](https://github.com/NVIDIA/enroot/issues/49#issuecomment-755178843))

2. Setting `ENROOT_MOUNT_HOME` to `yes` in `/etc/enroot/enroot.conf`.  
   Mounting the home directory allows the container to access the `~/.ssh` folder. This is necessary for MPI (`mpirun`) to automatically use password-less SSH authentication to launch `orted` processes on all nodes. ([Reference](https://github.com/NVIDIA/enroot/issues/49#issuecomment-755945810))

3. Setting `OMPI_MCA_orte_launch_agent` to `enroot start ... orted`.  
   Setting the `OMPI_MCA_orte_launch_agent` environment variable is a [common trick](https://catalog.ngc.nvidia.com/orgs/hpc/containers/bigdft) to make `mpirun` launch the `orted` process within a (Enroot/Singularity) container. Basically it tells `mpirun` to run `enroot start ... orted` instead of running `orted` directly.
   
3. Adding a (executable) hook for OpenMPI in `/etc/enroot/hooks.d/ompi.sh`.  
   This hook removes the need of manually setting `OMPI_MCA_orte_launch_agent` environment variable every time you run a task via `enroot start`. In our case, without this hook will require running the following everytime:
   ```sh
   enroot start -e OMPI_MCA_orte_launch_agent='enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app ${CONTAINER_NAME} orted' --rw --mount /mnt/home/${USER}/enroot/workspace:/app ${CONTAINER_NAME} mpirun -np 16 --hostfile hosts.txt ...
   ```
   Adding the following [pre-start hook script](https://github.com/NVIDIA/enroot/blob/main/doc/configuration.md):
   ```
   #!/bin/bash

   echo "OMPI_MCA_orte_launch_agent=enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app ${ENROOT_ROOTFS##*/} orted" >> "${ENROOT_ENVIRON}"
   ```
   simplifies the command to:
   ```sh
   enroot start --rw --mount /mnt/home/${USER}/enroot/workspace:/app ${CONTAINER_NAME} mpirun -np 16 --hostfile hosts.txt ...
   ```
   which makes life easier. ([Reference](https://github.com/NVIDIA/enroot/issues/49#issuecomment-755945810))

### Why not Use the OpenMPI on Host OS?

Running `mpirun ... enroot start ...` may prevent intra-node optimizations, resulting in worse performance. In addition, using OpenMPI in the Enroot container makes life easier, as we don't even need to install OpenMPI on any node.

## Limitations

- This approach is less robust compared to using Pyxis and Slurm.
- Requires specifying fixed Enroot flags in the pre-start hook.
- The examples provided are tailored for a single-user cluster. Some adjustments will be required to support multiple users running Enroot containers concurrently.
  > Hopefully I'll have time to get back to fix this later.

## Acknowledgments

This note has been made possible through the support of [ElsaLab][elsalab] and [NVIDIA AI Technology Center (NVAITC)][nvaitc].

Thanks to ElsaLab HPC Study Group and especially **Kuan-Hsun Tu** for environment setup.

And of course, thanks to [@3XX0](https://github.com/3XX0) for sharing this workaround.

[elsalab]: https://github.com/elsa-lab
[nvaitc]: https://github.com/NVAITC
