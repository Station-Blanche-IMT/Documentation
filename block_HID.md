    sudo vim /etc/modprobe.d/usbhid.conf
		blacklist usbhid
    update-initramfs -u -k all

  commande pour débloquer (à exécuter en ssh)
  
    insmod /lib/modules/$(uname -r)/kernel/drivers/hid/usbhid/usbhid.ko
