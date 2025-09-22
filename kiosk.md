sudo useradd -m -s /bin/bash kiosk

sudo apt install --no-install-recommends xorg openbox lightdm chromium pulseaudio vim
(chromium-browser au lieu de chomium sur raspberry)

sudo vim /etc/lightdm/lightdm.conf
	autologin-user=kiosk
	xserver-command=X -bs -core -nocursor
(désactivation du curseur à voir)

sudo vim /etc/xdg/openbox/autostart
	xset -dpms
	xset s off
	chromium --kiosk {URL VOULUE}
(encore une fois chromium-browser au lieu de chomium sur raspberry)

sudo reboot
