# Run RHCOS live ISO with libvirt


```
virt-install \
--name=telerhcos \
--ram=8384 \
--vcpus=4 \
--cpu host-model-only \
--os-type linux \
--os-variant rhel8.0 \
--events on_reboot=restart \
--noautoconsole \
--boot hd,cdrom \
--import \
--disk /var/lib/libvirt/images/rhcos-49.84.202111051605-0-live.x86_64.iso,device=cdrom \
--disk path=/var/lib/libvirt/images/telerhcos.img,size=120,pool=default

Remember that by default SSH access is disabled (we could enabled it at build time, just edit manifest.yaml and find the related commands) or we can pass and ignition config with our ssh-ke
