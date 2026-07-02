# 20260702_rhel_raid10_wsl
An exercise implementing a minimal RAID 10 array consisting of four identical USB sticks on RHEL 10 with WSL2. This activity was done on a Windows 11 laptop.

## Prepare WSL2
Install WSL2. Additional features like Hyper-V must be enabled. Other virtualization technologies may not work while Hyper-V is active.

## Install RHEL 10 on WSL2
+ Create a free Red Hat Developer Account and download the [official RHEL 10 WSL2 image](https://access.redhat.com/downloads/content/479/ver=/rhel---10/10.2/x86_64/product-software).
+ On Powershell, create a directory to store your RHEL 10 installation, and import the WSL2 image.
  ```Powershell
  mkdir C:\Linux\RHEL10
  wsl --import RHEL10 C:\Linux\RHEL10 C:\Users\"Your Username"\Downloads\rhel-10.2-x86_64-wsl2.wsl
  ```
+ Start RHEL 10.
  ```Powershell
  wsl -d RHEL10
  ```
## Attach USB devices to WSL
+ WSL2 require additional software to be able to directly access USB devices. We use ```usbipd-win```. Download the [correct MSI file for your CPU](https://github.com/dorssel/usbipd-win/releases/latest).
+ Plug in the four USB drives.
+ On Powershell, find the Bus IDs of the four USB drives.
  ```Powershell
  usbipd list
  
  ```
