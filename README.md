# ALT Linux network boot and installation

## Prerequisites

* Machine running ALT Linux or Ubuntu (called `target` later on). This one will run DHCP, TFTP, and other necessary services
* Another machine running Linux (called `host` later on).

### Target machine requirements

1. Internet connection (to install packages, etc)
2. A compete installation of python is available
3. Enough disk space to hold ISOs in `/srv/export/dist`
4. TCP ports 80, 445 and udp ports 67, 69 are free (nothing listens there)
5. User account (called `remote user` later on)
6. `remote user` has passwordless sudo access
7. `remote user` can login via ssh without password (by ssh keys)

### Host machine requirements

1. Ansible is installed
2. Enough disk space to hold ISOs in `/srv/export/dist`
3. It's possible to login as `remote user` from the `host` to the `target` via ssh


## Preparations

* List every target machine in `hosts`
  ```
  cp hosts.sample hosts
  $EDITOR hosts
  ```
* If target machines have several network interfaces, set the
  `altinstall_dhcp_interface` variable (in the `settings.yml` file)
  to define where network boot services should be provided
* List necessary ISO images in `alt_images.yml`
* (optional) Define which ISO should be booted by default
* Download ISOs:
  ```
  ansible-playbook download_distro_images.yml
  ```

## Deployment

* Check if target machines are reachable via ssh:
  ```
  ansible -i hosts -m ping all
  ```

* Deploy DHCP proxy, TFTP, and HTTP services on `target`:
  ```
  ansible-playbook -i hosts site.yml
  ```


## Hacker notes

### Network boot overview

* UEFI gets IP(v4) address and boot settings (file name)
* dnsmasq (proxy DHCP) replies with bootfile=snponly-$ARCH.efi (iPXE binary)
* UEFI downloads boot file (iPXE binary, snponly-$ARCH.efi) and runs it
* iPXE gets IP(v4) address and boot settings with a different *user_class* `iPXE`
* dnsmasq (proxy DHCP) replies with bootfile=config-$ARCH.ipxe (iPXE *script*)
* iPXE runs script: downloads kernel, initramfs, and boots kernel (with specified command line)
* initramfs (`propagator`) parses kernel command line, configures network,
  downloads ISO image (to RAM), and runs a live system


### Simple iPXE script (for aarch64)

```
kernel tftp/netboot/aarch64/alt_p10_xfce_20220312_aarch64/boot/vmlinuz initrd=full.cz root=bootchain bootchain=fg,altboot ip=dhcp4 changedisk fastboot live automatic=method:http,network:dhcp,server:10.42.0.96,directory:/dist/alt-p10-xfce-20220312-aarch64.iso  stagename=live showopts lang=ru_RU
initrd tftp/netboot/aarch64/alt_p10_xfce_20220312_aarch64/boot/full.cz
boot
```

```
kernel path/to/vmlinuz initrd=full.cz ${propagator_arguments}
initrd path/to/full.cz
boot
```


### dnsmasq as proxy DHCP

`Proxy DHCP` means that this instance of dnsmasq does NOT allocate IP(v4)
addresses. Instead it requests IP(v4) addresses (from another DHCP server),
adds boot options, and replies to client (with address and boot options)

```
interface=eth0 # serve requests from this interface

port=0 # disable DNS service
no-resolve
no-hosts

user=_dnsmasq
group=_dnsmasq
# don't run as root

dhcp-range=10.1.0.1,proxy 
# 10.1.0.1 is IP of host running dnsmasq (assigned to eth0 interface
# specified above)
# proxy: be a proxy DHCP

enable-tftp
tftp-root=/path/to/tftpdir
# config-aarch64.ipxe, snponly-aarch64.efi must be there

# Distinguish between firmware downloading iPXE and iPXE downloading config
# (iPXE sets user-class option to `iPXE...something`, UEFI firmwares use
# a different user-class)
dhcp-userclass=set:ipxe,iPXE

# ARM64 UEFI firmware should boot iPXE (aarch64/snponly.efi)
pxe-service=tag:!ipxe,ARM64_EFI,"Network Boot",snponly-aarch64.efi
# iPXE booted by ARM64 UEFI should "run" this config
pxe-service=tag:ipxe,ARM64_EFI,"iPXE boot menu",config-aarch64.ipxe
```
