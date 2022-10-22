# Raspberry Pi Hotspot EcoFlow Workaround
## Setup Hotspot on Raspberry Pi OS with second usb wifi connected to EcoFlow_XXXX<br>
### Workaround, thanks to https://github.com/vwt12eh8/hassio-ecoflow/issues/60#issuecomment-1279856354
### Tested on Raspberry 400 with latest Raspberry Pi OS (64-bit) bullseye<br>
### This is a list of all needed commands<br>
### You should fairly know what you do<br>
<br>
# Predictable names in raspi-config does not work on internal devices <br>
# We rename our network devices ourselves<br>
sudo su <br>
# Show network devices and their MAC adresses<br>
ip l <br>
# Internal wifi has the same MAC in the beginning like eth0 <br>
export MAC_WLAN_INT=xx:xx:xx:xx:xx:xx <br>
export MAC_WLAN_USB=xx:xx:xx:xx:xx:xx <br> 
<br>
cat << EOF > /etc/systemd/network/19-onboard_wifi_hotspot.link<br>
[Match]<br>
MACAddress=$MAC_WLAN_INT<br>
[Link]<br>
Name=hotspot<br>
EOF<br>
<br>
cat << EOF > /etc/systemd/network/20-freifunk.link<br>
[Match]<br>
MACAddress=$MAC_WLAN_USB <br>
[Link]<br>
Name=ecoflow<br>
EOF<br>
<br>
# Reboot<br>
<br>
<br>
sudo su<br>
# check, if your changes are working  <br>
ip l <br>
# Set SSID of EcoFlow_XXXX Wifi<br>
export EcoFlow_Wifi=EcoFlow_XXXX <br>
# Set Hotspot Password <br>
export PW="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" 	<br>
# Set  Hotspot SSID <br>
export SSID="XXXXXXXX" <br>
# Set country code in the Format DE (Germany) or US or .... <br>
export COUNTRY-CODE="XX"  <br>
<br>
apt update<br>
apt install dnsmasq hostapd haproxy<br>
<br>
cat << EOF >> /etc/network/interfaces<br>
# ecoflow<br>
auto ecoflow<br>
iface ecoflow inet dhcp<br>
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf<br>
##### Removing unnecessary default route #########<br>
post-up route del default dev $IFACE
EOF<br>
<br>
cat << EOF >> /etc/wpa_supplicant/wpa_supplicant.conf<br>
network={<br>
        ssid="$EcoFlow_Wifi"<br>
        scan_ssid=1<br>
        key_mgmt=NONE<br>
        }<br>
EOF<br>
<br>
systemctl restart networking<br>
<br>
cat << EOF >> /etc/haproxy/haproxy.cfg<br>
# Setup haproxy for DeltaMax<br>
frontend ecoflow<br>
	bind *:8055<br>
	mode tcp<br>
	use_backend ecoflowdeltamax<br>
<br>
backend ecoflowdeltamax<br>
	mode tcp<br>
	server deltamax 192.168.4.1:8055 check<br>
EOF<br>
<br>
systemctl restart haproxy<br>
<br>
cat << EOF >> /etc/dhcpcd.conf<br>
# Hotspot_Settings<br>
interface hotspot<br>
static ip_address=192.168.1.1/24<br>
nohook wpa_supplicant<br>
EOF<br>
<br>
systemctl restart dhcpcd<br>
<br>
mv /etc/dnsmasq.conf /etc/dnsmasq.conf_$(date +%Y%m%e%H%M%S)<br>
cat << EOF > /etc/dnsmasq.conf<br>
# DHCP-Server active for Hotspot<br>
interface=hotspot<br>
# DHCP-Server not activ for<br>
no-dhcp-interface=eth0 ecoflow<br>
# IPv4-Adressrange and Lease-Time, infinite is perhaps better than 24h lease time<br>
dhcp-range=192.168.1.100,192.168.1.200,255.255.255.0,24h<br>
# DNS<br>
dhcp-option=option:dns-server,192.168.1.1<br>
EOF<br>
<br>
#dnsmasq --test -C /etc/dnsmasq.conf<br>
systemctl restart dnsmasq<br>
systemctl enable dnsmasq<br>
systemctl status dnsmasq<br>
<br>
cat << EOF > /etc/hostapd/hostapd.conf<br>
# Hotspot Settings<br>
# Interface<br>
interface=hotspot<br>
# Wifi-Configuration<br>
ssid=$SSID<br>
channel=1<br>
hw_mode=g<br>
ieee80211n=1<br>
ieee80211d=1<br>
country_code=$COUNTRY-CODE<br>
wmm_enabled=1<br>
# Wifi-Encryption<br>
auth_algs=1<br>
wpa=2<br>
wpa_key_mgmt=WPA-PSK<br>
rsn_pairwise=CCMP<br>
wpa_passphrase=$PW<br>
EOF<br>
<br>
chmod 600 /etc/hostapd/hostapd.conf<br>
<br>
cat << EOF >> /etc/default/hostapd<br>
# Hotspot_Setting<br>
RUN_DAEMON=yes<br>
DAEMON_CONF="/etc/hostapd/hostapd.conf"<br>
EOF<br>
<br>
systemctl unmask hostapd<br>
systemctl start hostapd<br>
systemctl enable hostapd<br>
systemctl status hostapd<br>
<br>
cat << EOF >> /etc/sysctl.conf<br>
# Hotspot_Setting<br>
net.ipv4.ip_forward=1<br>
EOF<br>
<br>
# Minimal Firewallsettings, will be extended in the future<br>
cat << EOF >> /etc/nftables.conf<br>
table ip nat {<br>
	chain postrouting {<br>
		type nat hook postrouting priority srcnat; policy accept;<br>
		oifname "eth0" counter masquerade<br>
	}<br>
}<br>
EOF<br>
<br>
systemctl start nftables<br>
systemctl enable nftables.service<br>
<br>
# Deactivate sudo without password, very important for security!!!<br>
# Be carefull!!! Danger of locking out yourself from sudo!!!!<br>
# As a backupsolution, start a second terminal, gain root: <br>
sudo su<br>
cp /etc/sudoers.d/010_pi-nopasswd .<br>
# If your username is pi change the line to: <br>
visudo /etc/sudoers.d/010_pi-nopasswd 
pi ALL=(ALL) ALL<br>
# else<br>
yourusername ALL=(ALL) ALL<br>
