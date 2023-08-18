# Cloud Images

Cloud images are pre-installed disk images that have been customized to run on
cloud platforms. They are shipped with `cloud-init` that simplifies the
installation and provisioning of virtual machines.

Unlike ISOs and LXC container images, Proxmox's API lacks support for uploading
cloud images directly from a given URL (see
[here](https://bugzilla.proxmox.com/show_bug.cgi?id=4141) and
[here](https://forum.proxmox.com/threads/new-vm-from-cloud-init-image-via-api.111091/)).
Instead, they must be manually downloaded and converted into a VM
template to be available to Proxmox.

>**Warning**: When cloning the cloud image template with Terraform,
>`qemu-guest-agent` must be installed and `agent=1` must be set. Otherwise,
>Terraform will timeout. As such, it is recommended to create a further
>bootstrapped template with [Packer and Ansible](./packer.md).


## Manual Upload

1. Download any cloud image:

```bash
$ wget https://cloud.debian.org/images/cloud/bullseye/20230124-1270/debian11-generic-amd64-20230124-1270.qcow2
$ wget https://cloud.debian.org/images/cloud/bookworm/20230802-1460/debian-12-generic-amd64-20230802-1460.qcow2
```

2. Add qemu-guest-agent to the image. ssh into the proxmox node (same dir as above) and run the following

```bash
$ sudo apt install -y libguestfs-tools
$ sudo virt-customize --install qemu-guest-agent -a debian-12-generic-amd64-20230802-1460.qcow2
```

3. Create a Proxmox VM from the downloaded image:

```bash
$ sudo qm create 9001 \
    --name "debian-12-amd64" \
    --cpu "cputype=host" \
    --net0 "virtio,bridge=vmbr0" \
    --serial0 socket \
    --vga serial0 \
    --scsihw virtio-scsi-pci \
    --scsi0 "local:0,import-from=/home/brian/debian-12-generic-amd64-20230802-1460.qcow2" \
    --bootdisk scsi0 \
    --boot "order=scsi0" \
    --ide1 "local:cloudinit" \
    --ostype l26 \
    --cores 1 \
    --sockets 1 \
    --memory 512 \
    --agent 1
```

4. Resize the new VM (if necessary):

```bash
$ sudo qm resize 9001 scsi0 5G
```

5. Convert the VM into a template:

```bash
$ sudo qm template 9001
```

## Script

A full script of the steps above can be found at
[bin/import-cloud-image](https://github.com/kencx/homelab/blob/master/bin/import-cloud-image).

```bash
$ import-cloud-image --help

Usage: import-cloud-image [--debug|--force] [URL] [FILENAME]
```

## References
- [Proxmox Wiki - cloud-init Support](https://pve.proxmox.com/wiki/Cloud-Init_Support)
