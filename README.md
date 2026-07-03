# 20260702_rhel_raid10_wsl
An exercise implementing a minimal RAID 10 array consisting of four identical USB sticks on RHEL 10 with WSL2. This activity was done on a Windows 11 laptop.

## Prepare WSL2
Install WSL2. Additional features like Hyper-V must be enabled. Other virtualization technologies may not work while Hyper-V is active.

## Install RHEL 10 on WSL2
+ Create a free Red Hat Developer Account and download the [official RHEL 10 WSL2 image](https://access.redhat.com/downloads/content/479/ver=/rhel---10/10.2/x86_64/product-software).
+ On Powershell, create a directory to store your RHEL 10 installation, and import the WSL2 image.
  ```Powershell
  PS> mkdir C:\Linux\RHEL10
  PS> wsl --import RHEL10 C:\Linux\RHEL10 C:\Users\"Your Username"\Downloads\rhel-10.2-x86_64-wsl2.wsl
  ```
+ Start RHEL 10.
  ```Powershell
  PS> wsl -d RHEL10
  ```

## Attach USB devices to WSL
+ WSL2 require additional software to be able to directly access USB devices. We use ```usbipd-win```. Download the [correct MSI file for your CPU](https://github.com/dorssel/usbipd-win/releases/latest).
+ WSL2 images are minimal so many packages that come preinstalled in a full installation are missing. We add our subscription to the installation, then install necessary packages.
  ```console
  $ sudo subscription-manager register
  $ sudo dnf clean all
  $ sudo dnf -y install kmod usbutils
  ``` 
+ Plug in the four USB drives.
+ On Powershell, find the Bus IDs of the four USB drives.
  ```Powershell
  PS> usbipd list
  Connected:
  BUSID  VID:PID    DEVICE                                                        STATE
  (...)
  2-1    AAAA:BBBB  USB Mass Storage Device                                       Not shared
  2-2    AAAA:BBBB  USB Mass Storage Device                                       Not shared
  3-1    AAAA:BBBB  USB Mass Storage Device                                       Not shared
  3-2    AAAA:BBBB  USB Mass Storage Device                                       Not shared
  (...)
  ```
+ Bind the devices, then attach them to RHEL using the acquired BUS IDs.
  ```Powershell
  PS> usbipd bind --busid 2-1
  PS> usbipd bind --busid 2-2
  PS> usbipd bind --busid 3-1
  PS> usbipd bind --busid 3-2
  PS> usbipd attach --wsl --busid 2-1
  usbipd: info: Using WSL distribution 'RHEL10' to attach; the device will be available in all WSL 2 distributions.
  usbipd: info: Loading vhci_hcd module.
  usbipd: info: Detected networking mode 'nat'.
  usbipd: info: Using IP address 172.18.160.1 to reach the host.
  PS> usbipd attach --wsl --busid 2-2
  PS> usbipd attach --wsl --busid 3-1
  PS> usbipd attach --wsl --busid 3-2
  ```
+ Confirm that the USB devices are now accessible from WSL2.
  ```Powershell
  PS> usbipd list
  BUSID  VID:PID    DEVICE                                                        STATE
  (...)
  2-1    AAAA:BBBB  USB Mass Storage Device                                       Attached
  2-2    AAAA:BBBB  USB Mass Storage Device                                       Attached
  3-1    AAAA:BBBB  USB Mass Storage Device                                       Attached
  3-2    AAAA:BBBB  USB Mass Storage Device                                       Attached  
  ```
  ```console
  $ lsusb
  Bus 002 Device 001: ID AAAA:BBBB USB Mass Storage Device
  Bus 002 Device 002: ID AAAA:BBBB USB Mass Storage Device
  Bus 003 Device 001: ID AAAA:BBBB USB Mass Storage Device
  Bus 003 Device 002: ID AAAA:BBBB USB Mass Storage Device
  ```
+ Load kernel modules so that the kernel can map the USBs to block devices.
  ```console
  $ sudo modprobe usb-storage
  $ sudo modprobe uas
  ```

## Create Raid
+ Identify the block device names of the four USBs.
  ```console
  $ lsblk
  NAME    MAJ:MIN RM   SIZE RO TYPE   MOUNTPOINTS
  (...)
  sdc       8:XX   1  58.8G  0 disk
  sdd       8:XX   1  58.8G  0 disk
  sde       8:XX   1  58.8G  0 disk
  sdf       8:XX   1  58.8G  0 disk
  (...)
  ```
+ Format each USB with GPT, and a single Linux RAID partition.
  ```console
  $ fdisk /dev/sdc
  # g (GPT), d (Delete if partitions already exist), n (New), p (Primary), 1 (Partition No.), t (Partition Type), raid (Linux Raid Partition), p (Check Changes), w (Write Changes)
  $ fdisk /dev/sdd
  $ fdisk /dev/sde
  $ fdisk /dev/sdf
  ```
+ Create the RAID 10 Array, format it using ext4 and mount it.
  ```console
  $ sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sdc1 /dev/sdd1 /dev/sde1 /dev/sdf1
  $ sudo mkfs.ext4 /dev/md0
  $ sudo mkdir -p /mnt/raid10
  $ sudo mount /dev/md0 /mnt/raid10
  ```
+ Edit ```fstab``` and ```mdadm``` config file so that the RAID array would automatically mount in the next boot.
  ```console
  $ sudo nano /etc/fstab
  /dev/md0 /mnt/raid10 ext4 defaults,nofail 0 0
  $ sudo mdadm --detail --scan > /etc/madam.conf
  ```
+ Close the RHEL 10 Terminal, and fully shutdown WSL2 to simulate a hard startup.
  ```Powershell
  PS> wsl --shutdown
  ```

## Confirm RAID Configuration Survives Reboot
+ Restart RHEL 10.
+ Because connections were severed, the USBs must be attached to WSL2 again.
  ```Powershell
  PS> usbipd attach --wsl --busid 2-1
  usbipd: info: Using WSL distribution 'RHEL10' to attach; the device will be available in all WSL 2 distributions.
  usbipd: info: Loading vhci_hcd module.
  usbipd: info: Detected networking mode 'nat'.
  usbipd: info: Using IP address 172.18.160.1 to reach the host.
  WSL wsl: Processing /etc/fstab with mount -a failed.
  PS> usbipd attach --wsl --busid 2-2
  PS> usbipd attach --wsl --busid 3-1
  PS> usbipd attach --wsl --busid 3-2
  ```
+ Interestingly, WSL2 automatically attempts to process ```/etc/fstab```. I forgot to reload the kernel modules, hence the failures. In future labs I should create a custom module file in ```/etc/modules-load.d``` to automate this.
  ```console
  $ sudo modprobe usb-storage
  $ sudo modprobe uas
  $ modprobe md_mod
  $ mdadm --assemble --scan
  ```

## Perform Hot Swap and Confirm Data Integrity
+ Because cold unplugging is not possible due to how the abstraction layers interact, we will only test hot swapping. While data is being written to the RAID array, we randomly unplug one of the four USBs and swap it with a fifth USB.
  ```console
  $ cat /test.sh
  #!/bin/bash
  
  # Create temporary folder in RAID and move to that directory
  mkdir -p /mnt/raid10/tempdata
  cd /mnt/raid10/tempdata
  
  # Initialize counter
  i=1
  
  # Print explanation
  printf "Beginning RAID10 Integrity Test...\n"
  printf "Writing 30 random files in RAID...\n"
  
  # Write a total of 30 files
  while [ ${i} -le 30 ]
  do
  	# Instruct user to pull out one USB drive before writing 4th
  	if [ ${i} -eq 4 ]
  	then
  	printf "\n\e[1;31m PULL OUT A RANDOM DRIVE NOW!\e[0m\n\n"
  	fi
  	
  	# Instruct user to plug in replacement USB drive before writing 9th
  	if [ ${i} -eq 9 ]
  	then
  	printf "\n\e[1;32m PLUG IN REPLACEMENT DRIVE NOW!\e[0m\n\n"
  	fi
  
  # Set filename
  filename=$(printf "file%03d.dat" $i)
  printf "Writing %s at 1MB/s...\n" "${filename}"
  
  # Write random data at a steady 1MB/s, preventing unnecessary output to screen
  head -c 10M /dev/urandom | pv -L 1M > ${filename} 2>/dev/null
  
  # Record the hashes of the 30 files in a separate file
  md5sum ${filename} >> checksums.md5
  
  # Increment counter by 1
  ((i++))
  
  # End of loop
  done
  
  # Confirm data is intact
  printf "Finished writing random data..."
  printf "Verifying data integrity..."
  md5sum -c checksums.md5
  ```
  + Enable EPEL and download the ```pv``` package
    ```console
    $ sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
    $ sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
    $ sudo dnf clean all
    $ sudo dnf install -y pv
    ```
  + Execute the test
    ```console
    $ chmod +x /test.sh
    $ /test.sh
    ```

  ## Troubleshooting
  + I pulled out a USB as planned, but was unable to add the 5th USB to the RAID before the script concluded. Because of RAID 10's resilience, the integrity checks passed with the 3 drives anyway.
  + The new USB did not appear, only the controller.
    ```Powershell
    PS> usbipd list
    (...)
    Bus 002 Device 001: ID CCCC:DDDD Linux 6.18.33.2-microsoft-standard-WSL2 vhci_hcd USB/IP Virtual Host Controller
    (...)
    PS> usipd attach --wsl --busid 2-1
    ```
  + While I could still attach the controller to WSL2, it did not recognize it as a USB. I ended up removing then inserting the module, reattaching all four USBs again.
    ```console
    $ modprobe -r vhci_hcd
    $ modprobe vhci_hcd
    ```
    ```Powershell
    PS> usipd attach --wsl --busid 2-1
    PS> usipd attach --wsl --busid 2-2
    PS> usipd attach --wsl --busid 3-1
    PS> usipd attach --wsl --busid 3-2
    ```
  + Because I didn't mark which drive is which, I did not know which ```/dev/sdX``` was the new USB drive. So I needed to read the superblocks to determine it. The lone partition which lacked the RAID indicator was the new drive, which was then ```fdisk```'d.
    ```console
    $ sudo mdadm --examine /dev/sd[a-z]1
    ```
  + Finally, we rebuild the RAID and see the recovery live.
    ```console
    $ sudo mdadm --stop /dev/md0
    $ sudo mdadm --assemble --scan
    $ sudo mdadm --manage /dev/md0 --add /dev/sdi1 
    $ watch -n 1 cat /proc/mdstat
    ```
  + At this point, the new USB and the old USB on the same stripe begin blinking rapidly as the RAID gets rebuilt.
  + The rebuilding took over 30 minutes, and Wireshark captured thousands of TCP traffic per second between the virtualized hosts.
  + 
