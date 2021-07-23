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
