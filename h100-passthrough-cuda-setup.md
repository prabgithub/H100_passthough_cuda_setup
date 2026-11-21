# NVIDIA H100 PCI Passthrough, Driver, and CUDA Toolkit Setup

A repeatable setup guide for an NVIDIA H100 NVL passed through from
VMware ESXi to an Ubuntu 26.04 virtual machine.

This guide documents the working configuration established in the
original setup:

  Component         Configuration
  ----------------- -------------------------------------------------
  GPU               NVIDIA H100 NVL (94 GB)
  Hypervisor        VMware ESXi
  Guest OS          Ubuntu 26.04
  NVIDIA driver     620.70.06
  Kernel module     NVIDIA open kernel module
  CUDA Toolkit      13.5
  Profiling tools   Nsight Compute (`ncu`), Nsight Systems (`nsys`)
  MIG               Initially disabled

> **Important:** This guide records a configuration that worked in one
> environment. Check NVIDIA's current driver, CUDA, Ubuntu, and ESXi
> compatibility documentation before reproducing it on different
> versions or hardware. Do not assume that the exact driver package
> remains available indefinitely.

------------------------------------------------------------------------

## 1. Configure PCI passthrough in ESXi

Perform these steps on the ESXi host, not inside Ubuntu.

1.  Enable PCI passthrough for the physical NVIDIA H100.
2.  Assign the H100 PCI device to the Ubuntu VM.
3.  Enable **Reserve all guest memory (All locked)** in the VM settings.
4.  Ensure the VM has sufficient 64-bit PCI MMIO space for the GPU.
5.  Use a known-good VM hardware configuration. Avoid changing PCI reset
    settings unless diagnosing a specific passthrough problem.
6.  Boot Ubuntu.

Inside Ubuntu, check whether the PCI device is visible:

``` bash
lspci -nn | grep -i nvidia
```

Inspect the device and its driver binding:

``` bash
lspci -nnk -s 03:00.0
```

The PCI address `03:00.0` is an example from the original VM. Your
address may differ. Use the address reported by `lspci`.

Expected device identification is similar to:

``` text
03:00.0 3D controller: NVIDIA GH100 [H100L 94GB] [10de:2321]
```

At this stage, the GPU can be visible even though its NVIDIA driver is
not yet installed.

------------------------------------------------------------------------

## 2. Prepare Ubuntu

Update package metadata and install basic tools:

``` bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y wget ca-certificates build-essential dkms
```

Check the running kernel:

``` bash
uname -r
```

Install headers matching that kernel:

``` bash
sudo apt install -y linux-headers-$(uname -r)
```

The kernel headers are needed if DKMS has to build the NVIDIA kernel
module for the running kernel.

### 2.1 Disable Nouveau

Nouveau is the open-source NVIDIA driver included with Linux. For this
compute-driver setup, blacklist it so that it does not claim the H100.

Create the configuration:

``` bash
sudo tee /etc/modprobe.d/blacklist-nouveau.conf <<'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
```

Regenerate the initramfs:

``` bash
sudo update-initramfs -u
```

Reboot:

``` bash
sudo reboot
```

After reconnecting, check whether Nouveau is loaded:

``` bash
lsmod | grep nouveau
```

No output is expected. Note that seeing `nouveau` in the
`Kernel modules` list from `lspci` only means that a module is
available; it does not prove that it is loaded or bound to the device.

------------------------------------------------------------------------

## 3. Install the NVIDIA Data Center Driver

The original setup used NVIDIA's local repository package for driver
version `620.70.06`.

### 3.1 Obtain the local repository package

Download the matching package from NVIDIA's Data Center Driver download
page:

``` text
nvidia-driver-local-repo-ubuntu2604-620.70.06_1.0-1_amd64.deb
```

The original download was approximately 560 MB. The package name and
availability may change; use the package matching your Ubuntu release
and intended driver version.

### 3.2 Register the local repository

From the directory containing the downloaded `.deb` file:

``` bash
sudo dpkg -i nvidia-driver-local-repo-ubuntu2604-620.70.06_1.0-1_amd64.deb
```

Copy the repository signing key:

``` bash
sudo cp /var/nvidia-driver-local-repo-ubuntu2604-620.70.06/nvidia-driver-*-keyring.gpg /usr/share/keyrings/
```

Refresh APT:

``` bash
sudo apt update
```

### 3.3 Check package versions

``` bash
apt-cache policy libnvidia-compute
apt-cache policy nvidia-dkms-open
```

In the original setup, the candidate version was:

``` text
620.70.06-1ubuntu1
```

Confirm that APT is selecting the intended NVIDIA repository and version
before installing.

### 3.4 Simulate the installation

``` bash
sudo apt -s install libnvidia-compute nvidia-dkms-open
```

Review the proposed package changes. Stop if the simulation proposes
unexpected removals or replacement of important packages.

### 3.5 Install the driver

``` bash
sudo apt install libnvidia-compute nvidia-dkms-open
```

This installs the NVIDIA compute driver components and the open kernel
module used in the original configuration.

> **Do not casually add `cuda-drivers` or the `cuda` meta-package.**
> Those packages can pull in or change driver packages. The procedure
> here deliberately installs the driver separately from the CUDA
> Toolkit.

Reboot:

``` bash
sudo reboot
```

### 3.6 Verify the driver and GPU

``` bash
nvidia-smi
```

Inspect PCI binding:

``` bash
lspci -nnk -s 03:00.0
```

The important line is:

``` text
Kernel driver in use: nvidia
```

Check loaded NVIDIA modules:

``` bash
lsmod | grep nvidia
```

Modules commonly include:

-   `nvidia`
-   `nvidia_uvm`
-   `nvidia_modeset`
-   `nvidia_drm`

The original working `nvidia-smi` output reported:

-   NVIDIA-SMI: `620.70.06`
-   Kernel module version: `620.70.06`
-   GPU: NVIDIA H100 NVL
-   GPU memory: approximately 95,830 MiB
-   MIG mode: Disabled

**Checkpoint:** Do not proceed until `nvidia-smi` works and the NVIDIA
driver is bound to the H100.

------------------------------------------------------------------------

## 4. Install CUDA Toolkit 13.5

The NVIDIA driver and CUDA Toolkit are separate components:

-   The **driver** allows the operating system and applications to
    communicate with the GPU.
-   The **CUDA Toolkit** provides the compiler (`nvcc`), development
    libraries, headers, debugging tools, and profiling tools.

The CUDA version displayed by `nvidia-smi` is not proof that the CUDA
compiler is installed.

### 4.1 Install NVIDIA's CUDA repository keyring

``` bash
sudo apt install -y wget ca-certificates
```

Download the Ubuntu 26.04 CUDA repository keyring:

``` bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2604/x86_64/cuda-keyring_1.1-1_all.deb
```

Install it:

``` bash
sudo dpkg -i cuda-keyring_1.1-1_all.deb
```

Refresh APT:

``` bash
sudo apt update
```

### 4.2 Simulate the Toolkit installation

``` bash
sudo apt -s install cuda-toolkit-13-5
```

Review the package plan before proceeding. In the original setup, the
simulation reported:

``` text
Upgrading: 0
Installing: 76
Removing: 0
```

The plan included `cuda-nvcc-13-5`, CUDA development libraries, Nsight
Compute, and Nsight Systems.

If the simulation proposes driver upgrades or unexpected removals, stop
and investigate before installing.

### 4.3 Install the Toolkit

``` bash
sudo apt install cuda-toolkit-13-5
```

Use the version-specific `cuda-toolkit-13-5` package rather than the
broader `cuda` or `cuda-drivers` meta-packages when the intention is to
preserve the separately installed driver.

### 4.4 Add CUDA to PATH

``` bash
echo 'export PATH=/usr/local/cuda-13.5/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verify the compiler:

``` bash
which nvcc
nvcc --version
```

The expected compiler path is:

``` text
/usr/local/cuda-13.5/bin/nvcc
```

------------------------------------------------------------------------

## 5. Verify Nsight Compute and Nsight Systems

Check Nsight Compute:

``` bash
which ncu
ncu --version
```

Check Nsight Systems:

``` bash
which nsys
nsys --version
```

  -----------------------------------------------------------------------
  Tool                                Purpose
  ----------------------------------- -----------------------------------
  `nvcc`                              Compiles CUDA C/C++ source code

  `ncu`                               Profiles individual CUDA kernels,
                                      including available occupancy,
                                      memory, and hardware metrics

  `nsys`                              Profiles application timelines,
                                      CUDA API calls, CPU/GPU activity,
                                      kernel launches, and
                                      synchronisation

  `nvidia-smi`                        Monitors GPU utilisation, memory,
                                      power, temperature, and MIG state
  -----------------------------------------------------------------------

For custom CUDA kernels, use Nsight Compute to investigate kernel-level
bottlenecks. Use Nsight Systems to understand application-level
execution and identify launch, synchronisation, or CPU-side overhead.

### 5.1 Hardware counter permissions

Nsight Compute may report that access to GPU performance counters is
restricted.

PCI passthrough does not automatically guarantee unrestricted access to
every hardware counter. The relevant driver setting and permissions
depend on the installed driver version and system configuration.

If profiling reports a permission error:

1.  Read the exact Nsight Compute error.
2.  Inspect the installed NVIDIA driver documentation for the applicable
    profiling permission setting.
3.  Check the current driver configuration before changing it.
4.  Re-test with the required privileges.

Do not blindly change an NVIDIA module parameter based on instructions
for a different driver version.

------------------------------------------------------------------------

## 6. Run a CUDA smoke test

This verifies that the compiler can build a CUDA application and that
the application can execute a kernel on the H100.

Create a source file:

``` bash
nano test.cu
```

Paste:

``` cpp
#include <iostream>
#include <cuda_runtime.h>

__global__ void hello() {
    printf("Hello from GPU thread %d\n", threadIdx.x);
}

int main() {
    int count = 0;

    cudaError_t err = cudaGetDeviceCount(&count);

    if (err != cudaSuccess) {
        std::cerr << "CUDA error: "
                  << cudaGetErrorString(err) << std::endl;
        return 1;
    }

    std::cout << "Number of CUDA devices: "
              << count << std::endl;

    hello<<<1, 8>>>();

    err = cudaDeviceSynchronize();

    if (err != cudaSuccess) {
        std::cerr << "Kernel error: "
                  << cudaGetErrorString(err) << std::endl;
        return 1;
    }

    return 0;
}
```

Compile:

``` bash
nvcc test.cu -o test
```

Execute:

``` bash
./test
```

Expected output is similar to:

``` text
Number of CUDA devices: 1
Hello from GPU thread 0
Hello from GPU thread 1
...
Hello from GPU thread 7
```

If compilation or execution fails, resolve that issue before proceeding
to MIG configuration or performance benchmarking.

------------------------------------------------------------------------

## 7. Test Nsight profiling

Run Nsight Systems:

``` bash
nsys profile -o cuda_test ./test
```

This creates a profiling report with an application execution timeline.

Run Nsight Compute:

``` bash
ncu ./test
```

This profiles the CUDA kernel and reports the metrics available to the
current driver, GPU, and permissions.

------------------------------------------------------------------------

## 8. Enable MIG (optional)

The original working driver configuration showed MIG disabled. Keep it
disabled until ordinary CUDA execution and profiling have been verified.

Enable MIG:

``` bash
sudo nvidia-smi -mig 1
```

Check MIG status:

``` bash
nvidia-smi -q | grep -A 15 "MIG Mode"
```

List supported GPU instance profiles:

``` bash
nvidia-smi mig -lgip
```

List compute instance profiles:

``` bash
nvidia-smi mig -lcip
```

Use the profile list returned by the installed driver to plan GPU
instances (GI) and compute instances (CI). Do not assume profile IDs or
instance layouts from another H100 configuration.

Some profiling metrics may be physical-GPU-wide or restricted when
operating on MIG instances.

------------------------------------------------------------------------

## 9. Compact command checklist

This is a quick reference for the guest OS. It assumes:

-   The H100 is already assigned to the VM in ESXi.
-   The NVIDIA local repository `.deb` has been downloaded.
-   The package names and repository paths match the versions in this
    guide.

``` bash
# Prepare Ubuntu
sudo apt update
sudo apt upgrade -y
sudo apt install -y wget ca-certificates build-essential dkms
sudo apt install -y linux-headers-$(uname -r)

# Disable Nouveau
sudo tee /etc/modprobe.d/blacklist-nouveau.conf <<'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
sudo update-initramfs -u
sudo reboot

# Install NVIDIA local repository
sudo dpkg -i nvidia-driver-local-repo-ubuntu2604-620.70.06_1.0-1_amd64.deb
sudo cp /var/nvidia-driver-local-repo-ubuntu2604-620.70.06/nvidia-driver-*-keyring.gpg /usr/share/keyrings/
sudo apt update

# Verify driver package candidates
apt-cache policy libnvidia-compute
apt-cache policy nvidia-dkms-open

# Simulate, then install the driver
sudo apt -s install libnvidia-compute nvidia-dkms-open
sudo apt install libnvidia-compute nvidia-dkms-open
sudo reboot

# Verify NVIDIA driver
nvidia-smi
lspci -nnk -s 03:00.0
lsmod | grep nvidia

# Add CUDA repository
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2604/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Simulate and install CUDA Toolkit 13.5
sudo apt -s install cuda-toolkit-13-5
sudo apt install cuda-toolkit-13-5

# Configure PATH
echo 'export PATH=/usr/local/cuda-13.5/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify CUDA and profiling tools
nvcc --version
ncu --version
nsys --version

# Optional: enable MIG after CUDA testing
sudo nvidia-smi -mig 1
nvidia-smi mig -lgip
nvidia-smi mig -lcip
```

------------------------------------------------------------------------

## 10. Precautions and recovery discipline

1.  **Preserve the working driver.** Avoid unnecessary driver upgrades
    while setting up CUDA.
2.  **Keep driver and Toolkit installation separate.** Use
    `cuda-toolkit-13-5` for this version-specific setup, not the broader
    `cuda` or `cuda-drivers` packages.
3.  **Do not install Ubuntu's `nvidia-cuda-toolkit` for this
    procedure.** The goal is to use NVIDIA's CUDA 13.5 repository and
    packages.
4.  **Do not immediately run `sudo apt autoremove`.** Retain older
    kernel packages until the H100 has passed reboot and CUDA tests.
5.  **Do not enable MIG prematurely.** First verify ordinary CUDA
    compilation, execution, and profiling.
6.  **Back up the VM.** Once the driver and Toolkit work, take an ESXi
    snapshot or another suitable backup before experimenting with
    kernel, driver, or MIG changes.
7.  **Record versions.** Save the output of `uname -r`, `nvidia-smi`,
    `nvcc --version`, `ncu --version`, and `nsys --version` with your
    setup notes.

Useful diagnostic commands:

``` bash
uname -r
lspci -nnk -s 03:00.0
nvidia-smi
lsmod | grep -E 'nvidia|nouveau'
dpkg -l | grep -E 'nvidia|cuda|nsight'
```

------------------------------------------------------------------------

## 11. Suggested next steps

Once the base environment is stable, use it as a development platform
for:

-   Writing and benchmarking custom CUDA kernels.
-   Studying H100 memory hierarchy, occupancy, and tensor core
    utilisation.
-   Profiling kernels with Nsight Compute.
-   Analysing end-to-end GPU execution with Nsight Systems.
-   Experimenting with MIG isolation and GPU resource partitioning.
-   Benchmarking inference workloads and serving engines such as vLLM
    and SGLang.

The first milestone is a successful CUDA smoke test and successful
`ncu`/`nsys` execution. Establish that baseline before moving to more
complex workloads.
