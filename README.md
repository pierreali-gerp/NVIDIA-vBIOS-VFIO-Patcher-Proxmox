# NVIDIA vBIOS VFIO Patcher

All credit goes to [Matoking](https://github.com/Matoking) for providing their super-convenient Python script.

I just added a brief Preparation section below for users who need to dump their NVIDIA (Pascal series) ROM from Proxmox. This is actually very simple, since Proxmox doesn't run any processes on discrete GPUs by default, such as desktop managers. On the contrary, if you need to dump your GPU's ROM from a Linux distro with a desktop manager running (i.e., any Linux distro with a Graphical User Interface in use), follow the instructions at [this link](https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/6)-Preparation-and-placing-of-the-ROM-file#dumping-your-gpu-rom).

## Preparatory steps on Proxmox

1. **Disable Secure Boot on your system, if it's enabled.** You can use `mokutil --sb-state` to check this out. You can disable Secure Boot by entering your motherboard's UEFI settings.

2. **Unload all kernel modules your GPU is using.** Check which kernel modules can interfere with the dump process by running: 1) `lspci | grep -i vga` to get the PCI address of your discrete GPU, then 2) `lspci -ks 01:00` (replace _01:00_ with the actual address of your GPU). Your output should look like this:
   ```
   01:00.0 VGA compatible controller: NVIDIA Corporation GP104 [GeForce GTX 1070] (rev a1)
        Subsystem: Dell GP104 [GeForce GTX 1070]
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau
   01:00.1 Audio device: NVIDIA Corporation GP104 High Definition Audio Controller (rev a1)
        Subsystem: Dell GP104 High Definition Audio Controller
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
   ```
   We are interested in the part related to the VGA controller, so, in my case, I have to unload _vfio-pci_, _nvidiafb_, and _nouveau_. To do so, run (replace `rmmod` with `modprobe -r`, shouldn't it work):
   ```
   rmmod nouveau
   rmmod nvidiafb
   rmmod vfio-pci
   ```
   You can check if other potentially problematic kernel modules are loaded by inspecting the output from `lsmod | grep -i nvidia` (which shouldn't return any, if you didn't install Nvidia drivers on Proxmox...) and `lsmod | grep -i vfio`. From the latter output, I also decided to unload _vfio_pci_core_, for precaution, though it might not be necessary.
   
3. **Download the NVFlash utility for Linux.** Go to [this link](https://www.techpowerup.com/download/nvidia-nvflash/) and download the version for Linux. You need to use an interactive browser to download it; then, transfer the executable compatible with your CPU architecture from the downloaded ZIP to the Proxmox host.

4. **Dump your GPU's ROM**. Move to the path where you transferred the NVFlash utility file, make it executable with `chmod +x nvflash`, and save the firmware to a file with `./nvflash --save vbios.rom`. It should produce an output like this:
   ```
   NVIDIA Firmware Update Utility (Version 5.867.0)
   Copyright (C) 1993-2024, NVIDIA Corporation. All rights reserved.
   
   Reading EEPROM (this operation may take up to 30 seconds)
   
   Build GUID            : 00000000000000000000000000000000
   Build Number          : 21238717
   IFR Subsystem ID      : 1028-3301
   Subsystem Vendor ID   : 0x1028
   Subsystem ID          : 0x3301
   Version               : 86.04.50.40.25
   Image Hash            : 413A090E9CFB34220A9E2581645EDA3C
   Hierarchy ID          : Normal Board
   Build Date            : 10/07/16
   Modification Date     : 03/20/17
   UEFI Version          : 0x30007 ( x64 )
   UEFI Variant ID       : 0x0000000000000007 ( GP1xx )
   UEFI Signer(s)        : Microsoft Corporation UEFI CA 2011
   XUSB-FW Version ID    : N/A
   XUSB-FW Build Time    : N/A
   InfoROM Version       : G001.0000.01.04
   InfoROM Backup        : Present
   License Placeholder   : Present
   GPU Mode              : N/A
   CEC OTA-signed Blob   : Not Present
   ```
5. **Re-enable Secure Boot on your system if you previously disabled it.** From now on, you can turn Secure Boot back on if you wish. To do so, reboot your system, enter the UEFI settings, and revert the changes you made in step 1.

6. Follow the instructions in the next section to patch the saved ROM through Matoking's Python script.

## NVIDIA vBIOS VFIO Patcher instructions
**This tool is known to be compatible only with the Pascal series (1xxx) of NVIDIA GPUs.**

nvidia_vbios_vfio_patcher.py is a script that creates a patched/spliced copy of a NVIDIA vBIOS that allows PCI passthrough when using libvirt. This copy of the vBIOS can then be passed to libvirt, allowing the NVIDIA GPU to be used in the guest VM. This can be done by adding the following line to the VM domain XML file.

```
   <hostdev>
     ...
     <rom file='/path/to/your/patched/gpu/bios.bin'/>
     ...
   </hostdev>
```

This script may be useful if you are using one of the Pascal (1xxx series) series of NVIDIA GPUs and you are having passing the GPU to the guest VM. In this case, the vBIOS of the system's primary GPU is tainted when booting the host OS, making GPU passthrough impossible unless a clean copy of the vBIOS is used.

The patching process requires a full copy of the clean vBIOS. You can either extract it from the graphics card using [nvflash](https://www.techpowerup.com/download/nvidia-nvflash/) or [GPU-Z](https://www.techpowerup.com/gpuz/) under Windows (recommended), or download one for your specific GPU model from [TechPowerUp](https://www.techpowerup.com/vgabios/).

# DISCLAIMER

**Use this script at your own discretion. This script has NOT been tested extensively, and has only been tested with a few GPUs belonging to he Pascal series of NVIDIA GPUs.**

**The script performs only a few rudimentary sanity checks, but no guarantees are made of the validity of the patched ROM!**

# Usage

The script should work with both Python 2 and 3.

To create a patched version of the BIOS, run the script with the following parameters.

```
python nvidia_vbios_vfio_patcher.py -i <ORIGINAL_ROM> -o <PATCHED_ROM>
```

A patched version of <ORIGINAL_ROM> will be written to <PATCHED_ROM>.
