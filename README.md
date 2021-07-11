***OPENWRT Pİ KURULUMU***

Bu kurulum dosyası raspberry pi üzerinde openwrt ile subnet modem oluşturulması ve sanal gizli ağın tanımlanmasını içermektedir. 

- Raspberry pi modelliniz ile uyumlu OpenWrt görüntüsünü [bu sayfayı](https://openwrt.org/toh/hwdata/raspberry_pi_foundation/start) ziyaret ederek seçebilirsiniz. 
Eğer Raspberry pi 4 B kullanıyorsanız direkt [buradan](http://downloads.openwrt.org/snapshots/targets/brcm2708/bcm2711/openwrt-brcm2708-bcm2711-rpi-4-squashfs-factory.img.gz) indirebilirsiniz.

- Sd kart üzerine indirdiğiniz image görüntüsünü yazın.

- Sd kartı Raspberry pi'a takın ve bir başka bilgisayar ile ethernet bağlantısı kurun ve Pi'yi çalıştırın.

- openwrt varsayılan olarak '192.168.1.1' ip adresini kullanır.

- root useri ile 192.168.1.1 adresine ssh bağlantısı ile bağlanın.
```
ssh root@192.168.1.1
```
- openwrt root kullanıcısına parola ataması yapmak için 'passwd' komutu ile yeni parolanızı belirleyin.
- *NOTE* : Parola atamasında türkçe karakter kullanmamaya dikkat edin. openwrt görüntüsünde varsayılan dil seçimi ingilizcedir.

```         
passwd
```

- Openwrt kurulumlarımızın devam edebilmemiz için internet bağlantısı için ayarlarımızı yapmalıyız.

- Openwrt görüntüsünde static ip ataması yapma noktasında problem yaşadığım için modem üzerinde static ip atamsını gerçekleştirdim.

    - Modem üzerinden ip ataması için openwrt cihazımızın ip adresini almamız gerekiyor.
            
            ifconfig eth0 | grep HWaddr
        
        ---
        
            output: eth0      Link encap:Ethernet  HWaddr - DC:A6:32:CE:3B:13

- Modem üzerinden HWaddr mac adresini kullanarak dilediğiniz ip adresinin atayabiliirsiniz.

- OpenWRT üzerinde geçici ağ yapılandırmasının yapılması
    - yapılandırma için ```vi``` editörünü kullanıcağız. ```vi``` ile dosyayı açtıktan sonra düzenleme işlemi ```i``` tuşuna bastığınız anda akrif olur. Dosya ile işiniz bittikten sonta ```esc``` tuşuyla düzenleme işleminden çıkılır. ```:wq``` operatörü ile kaydedip çıkabilirsiniz. eğer yanlış bir düzenlemeden çıkmak isterseniz ```:q!``` operatörünü kullanın.

    ```
    vi /etc/config/network
    ```
----

- Örnek geçici ağ yapılandırması
``` 
config interface 'lan'
    option proto 'dhcp'
    option device 'eth0'
```
- Cihazı kapatın ve yerel ağınıza bağlayın. Tekrar çalıştırın.

```
 ssh root@yerel_ağ_adresi

 opkg update
 opkg install nano
 opkg install luci
 opkg install luci-ssl
 /etc/init.d/uhttpd restart

 #ikincil ethernet bağlantısı için usb arayüzüne ethernet bağlantısı için driver
 opkg install kmod-usb-net-rtl8152

```

* OpenWrt cihazımıza luci web servisi ile web üzerinden erişebilirsiniz.
```
http://yerel_ağ_adresi
```

* Son olarak kalıcı ağ yapılandırma ayarları için:
```
 nano /etc/config/network
```
* Ağ yapılandırmasını aşağıdaki gibi değiştirin.
```
config interface 'wan'
        option proto 'dhcp'
        option device 'eth0'

config interface 'lan'
        option proto 'static'
        option ipaddr '10.0.0.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
        option device 'eth1'

config device
        option name 'br-lan'
        option type 'bridge'
        list ports 'eth1'

```
* ağ ayarlarını tekrar başlatın
```
service network restart
```
---
---
---
---

***OPENVPN SERVER KURULUMU***

**1.2) Gereksinimler**
```
opkg update
opkg install openvpn-openssl openvpn-easy-rsa

#Konfigurasyon parametreleri
OVPN_DIR="/etc/openvpn"
OVPN_PKI="/etc/easy-rsa/pki"
OVPN_PORT="1194"
OVPN_PROTO="udp"
OVPN_POOL="192.168.8.0 255.255.255.0"
OVPN_DNS="${OVPN_POOL%.* *}.1"
OVPN_DOMAIN="$(uci get dhcp.@dnsmasq[0].domain)"

# WAN IP adresini al
. /lib/functions/network.sh
network_flush_cache
network_find_wan NET_IF
network_get_ipaddr NET_ADDR "${NET_IF}"
OVPN_SERV="${NET_ADDR}"

#DDNS istemcisinden FQDN
NET_FQDN="$(uci -q get "$(uci -q show ddns \
| sed -n -e "/\.enabled='1'$/s//.lookup_host/p" \
| sed -n -e "1p")")"
if [ -n "${NET_FQDN}" ]
then OVPN_SERV="${NET_FQDN}"
fi
```


**2.)Anahtar Yöneticisi**

```
# Configuration parameters
export EASYRSA_PKI="${OVPN_PKI}"
export EASYRSA_REQ_CN="ovpnca"
export EASYRSA_BATCH="1"
 
# Remove and re-initialize the PKI directory
easyrsa init-pki
 
# Generate DH parameters
easyrsa gen-dh
 
# Create a new CA
easyrsa build-ca nopass
 
# Generate a key pair and sign locally for a server
easyrsa build-server-full server nopass
 
# Generate a key pair and sign locally for a client
easyrsa build-client-full client nopass
 
# Generate TLS PSK
openvpn --genkey --secret ${OVPN_PKI}/tc.pem


```

**3.) Güvenlik duvarı**

Lan bölgesine vpn ağ ataması yapılır ve Wan bölgesinden vpn sunusucuna erişime izin verir.

```
uci rename firewall.@zone[0]="lan"
uci rename firewall.@zone[1]="wan"
uci del_list firewall.lan.device="tun+"
uci add_list firewall.lan.device="tun+"
uci -q delete firewall.ovpn
uci set firewall.ovpn="rule"
uci set firewall.ovpn.name="Allow-OpenVPN"
uci set firewall.ovpn.src="wan"
uci set firewall.ovpn.dest_port="${OVPN_PORT}"
uci set firewall.ovpn.proto="${OVPN_PROTO}"
uci set firewall.ovpn.target="ACCEPT"
uci commit firewall
/etc/init.d/firewall restart
```
**4.) VPN  Servisi ve İstemci ataması**
```
# Configuration parameters
OVPN_DH="$(cat ${OVPN_PKI}/dh.pem)"
OVPN_TC="$(sed -e "/^#/d;/^\w/N;s/\n//" ${OVPN_PKI}/tc.pem)"
OVPN_CA="$(openssl x509 -in ${OVPN_PKI}/ca.crt)"
NL=$'\n'
 
# Configure VPN service and generate client profiles
umask go=
ls ${OVPN_PKI}/issued \
| sed -e "s/\.\w*$//" \
| while read -r OVPN_ID
do
OVPN_KEY="$(cat ${OVPN_PKI}/private/${OVPN_ID}.key)"
OVPN_CERT="$(openssl x509 -in ${OVPN_PKI}/issued/${OVPN_ID}.crt)"
OVPN_EKU="$(openssl x509 -in ${OVPN_PKI}/issued/${OVPN_ID}.crt -purpose)"
OVPN_CONF_SERVER="\
user nobody
group nogroup
dev tun
port ${OVPN_PORT}
proto ${OVPN_PROTO}
server ${OVPN_POOL}
topology subnet
client-to-client
keepalive 10 60
persist-tun
persist-key
push \"dhcp-option DNS ${OVPN_DNS}\"
push \"dhcp-option DOMAIN ${OVPN_DOMAIN}\"
push \"redirect-gateway def1\"
push \"persist-tun\"
push \"persist-key\"
<dh>${NL}${OVPN_DH}${NL}</dh>"
OVPN_CONF_CLIENT="\
dev tun
nobind
client
remote ${OVPN_SERV} ${OVPN_PORT} ${OVPN_PROTO}
auth-nocache
remote-cert-tls server"
OVPN_CONF_COMMON="\
<tls-crypt>${NL}${OVPN_TC}${NL}</tls-crypt>
<key>${NL}${OVPN_KEY}${NL}</key>
<cert>${NL}${OVPN_CERT}${NL}</cert>
<ca>${NL}${OVPN_CA}${NL}</ca>"
case ${OVPN_EKU} in
(*"SSL server : Yes"*) cat << EOF > ${OVPN_DIR}/${OVPN_ID}.conf ;;
${OVPN_CONF_SERVER}
${OVPN_CONF_COMMON}
EOF
(*"SSL client : Yes"*) cat << EOF > ${OVPN_DIR}/${OVPN_ID}.ovpn ;;
${OVPN_CONF_CLIENT}
${OVPN_CONF_COMMON}
EOF
esac
done
/etc/init.d/openvpn restart
ls ${OVPN_DIR}/*.ovpn
```

Modem kurulumu ve openvpn server sunucusu hazır.


/etc/openvpn/client.ovpn dosyasını istemci olarak kullanmak istediğiniz cihazda vpn bağlantısı için kullanabilirsiniz.

Yeni bir Client istemcisi oluşturmak isterseniz repositoride yer alan [ovpnclient_creator.sh](wget) dosyasını indirip çalıştırabilirsiniz.

