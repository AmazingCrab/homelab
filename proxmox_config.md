# Hardware
- Lenovo Thinkpad x230, CPU 3ra Gen, i5-320M TPD 35W, Idle 5-12W ,16GB de RAM, 256GB SATA SSD, 1 puerto Ethernet 100/1000Mbps, 1 puerto wifi 802.11a/b/g/n
- Se eligió por su bajo consumo y robustez.

## Proxmox installer
(Listamos opciones)
- zfs -> ( xq tengo un solo disco, permite pre-cachear antes de mover info y es mas fiable)
- country,
- timezone,
- keymap,
- email,
- managment interface,  -> nic0(ethernet) -> 192.168.1.66
- hostname -> pve.hostage

## Firewall pve
Nota: no asignamos source porque usamos la lan ethernet,
luego la wlan.

En pve, Firewall:

Proxmox GUI
```
- ping, accept rule for admin IP
- tcp 8006,22 acept rule for admin IP
```
## Firewall en Datacenter
En Datacenter, Firewall, Options.

Proxmox GUI
```
- firewall policy input en Drop (se aplican al final)
```
## Limitamos la RAM de ZfS
Modificamos máximo y mínimo porque contamos solo con 16gb.

edit
```
# "/etc/modprobe.d/zfs.conf"

# Min 1GB (1073741824 bytes)
options zfs zfs_arc_min=1073741824

# Max 4GB (4294967296 bytes)
options zfs zfs_arc_max=4294967296
```

## Tweak personalizado para apagar los monitores

Agregamos apagado automatico luego de 1min a  tty1

bash
```
mkdir -p /etc/systemd/system/getty@tty1.service.d

cat <<EOF > /etc/systemd/system/getty@tty1.service.d/override.conf
[Service]
ExecStartPre=/usr/bin/setterm --blank 1 --powerdown 1
EOF
```
### Aplicar cambios sin reiniciar:

bash
```
systemctl daemon-reload
systemctl restart getty@tty1
```
### Revertir desde consola
bash
```
setterm --blank 0 --powerdown 
```

## 1. Deshabilitar los repositorios Enterprise:
bash
```
# "/etc/apt/sources.list.d/"

mv ceph.sources ceph.sources.bak
mc pve-enterprise.sources pve-enterprise.sources.bak
```

## 2. Agregar el repositorio no-subscription para PVE:
bash
```
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

## Update && Upgrade
bash
```
uname -r

apt update && apt dist-upgrade

proxmox-boot-tool kernel list # Kernels instalados en proxmox

reboot # si el anterior es mas viejo q el ultimo instalado reinicio 

```
## Configuración WIFI con IP estática y PSK
bash
```
apt install wpasupplicant

wpa_passphrase "SSID" "Password" # genero HASH
```

edit
```
# "/etc/wpa_supplicant/wpa_supplicant.conf"

ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=ES
network={
    ssid="SSID"
    psk=HASH_GENERADO
    key_mgmt=WPA-PSK
}
```
bash
```
chmod 600 /etc/wpa_supplicant/wpa_supplicant.conf # Permisos solo para root
```
## Configuramos todas las placas de red

bash
```
ip a | grep "wlp"# buscamos el nombre de la placa de red wifi (wlp3s0)
```
edit
```
# "/etc/network/interface"
#auto inicia la interfaz
auto lo
# define la ip para los servicios internos 127.0.0.1
iface lo inet loopback

# --- INTERFAZ ETHERNET FÍSICA (LAN / Administración) ---
# nic0 se conecta al router físico (192.168.1.1)
iface nic0 inet manual 
# Manual porque vamos a usarla con el bridge

# --- BRIDGE PRINCIPAL (Salida a Internet física) ---
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.66/24
# conecta el bridge a la interfaz nic0
    bridge-ports nic0
# stp off porque nuestra red no tiene redundancia de cables
    bridge-stp off
# fd 0 para que arranque sin demoras
    bridge-fd 0
# metric mientras el numero crece baja la prioridad de uso de la ruta
    metric 100

# --- WIFI (Internet y Administración Remota) ---
auto wlan0              
iface wlan0 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    dns-nameservers 9.9.9.9 1.1.1.1
    metric 10
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

# --- BRIDGE VIRTUAL INTERNO (Red de laboratorio 10.0.10.0/24) ---
# Este bridge no tiene puertos físicos, es un switch virtual dentro del host
auto vmbr1
iface vmbr1 inet static
    address 10.0.10.99/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    metric 200
```

### Aplicamos cambios en vivo

bash
```
ifreload -a
```

### Error, ifupdown2, el gestor de red que usa Proxmox para sus bridges,no puede iniciar wpa_supplicant correctamente

bash
```
root@pve:~# ifreload -a
  File "/usr/share/ifupdown2/ifupdown/scheduler.py", line 325, in run_iface_list
    cls.run_iface_graph(ifupdownobj, ifacename, ops, parent,
    ~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            order, followdependents)
            ^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/share/ifupdown2/ifupdown/scheduler.py", line 315, in run_iface_graph
    cls.run_iface_list_ops(ifupdownobj, ifaceobjs, ops)
    ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/share/ifupdown2/ifupdown/scheduler.py", line 188, in run_iface_list_ops
    cls.run_iface_op(ifupdownobj, ifaceobj, op,
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^
        cenv=ifupdownobj.generate_running_env(ifaceobj, op)
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            if ifupdownobj.config.get('addon_scripts_support',
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                '0') == '1' else None)
                ^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/share/ifupdown2/ifupdown/scheduler.py", line 150, in run_iface_op
    ifupdownobj.log_error('%s: %s %s' % (ifacename, op, str(e)))
    ~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/share/ifupdown2/ifupdown/ifupdownmain.py", line 226, in log_error
    raise Exception(str)
error: wlp3s0 : wlp3s0: pre-up cmd '/etc/network/if-pre-up.d/wpasupplicant' failed: returned 1
bash
```

### Utilizaremos network-manager y limpiamos la configuración

bash
```
apt install -y network-manager
```
edit
```
# "/etc/network/interfaces"

auto lo
iface lo inet loopback

# --- INTERFAZ ETHERNET FÍSICA (LAN / Administración) ---
iface nic0 inet manual

# --- BRIDGE PRINCIPAL (Salida a Internet física) ---
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.66/24
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

# --- BRIDGE VIRTUAL INTERNO (Red de laboratorio 10.0.10.0/24) ---
auto vmbr1
iface vmbr1 inet static
    address 10.0.10.99/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

### Quitamos wlp3s0 del gestor de redes ifupdown2

bash
```
ifdown wlp3s0 2>/dev/null
```

### Usamos network-manager para conectar
bash
```
ip link set wlp3s0 up

nmcli device wifi connect "SSID" password "CONTRASEÑA"
```
### Error, no reconoce la interfaz

edit
```
# /etc/NetworkManager/NetworkManager.conf

[ifupdown]
managed=true
```

bash
```
systemctl restart NetworkManager
# Verificamos que esté "UP"
ip a | grep wl
```

NOTA: Usamos network-manager para conectar la placa wifi y ifupdown2 para los bridges de las VM's con su config en "/etc/network/interfaces"

### Reconfiguramos IP/gatewayt/dns
- Cómo los perdimos al borrarlos de /etc/network/interfaces y utilizar network-manager los seteamos denuevo

bash
```
# Cambiar el método a manual (estático)
nmcli connection modify "Gazzo-5GHz" ipv4.method manual

# Asignar la IP fija, puerta de enlace y DNS
nmcli connection modify "Gazzo-5GHz" ipv4.addresses 192.168.1.50/24
nmcli connection modify "Gazzo-5GHz" ipv4.gateway 192.168.1.1
nmcli connection modify "Gazzo-5GHz" ipv4.dns "8.8.8.8 1.1.1.1"

# Asegurar que la conexión se active automáticamente
nmcli connection modify "Gazzo-5GHz" connection.autoconnect yes
```
### Aplicamos cambios

bash
```
nmcli connection down "Gazzo-5GHz"
nmcli connection up "Gazzo-5GHz"

```
### Corroboramos cambios

bash
```
nmcli connection show
ip route 
# por mas que la metrica sea mayor en WAN, como vbr0 (nuestro puente ethernet) no tiene gateway, el tráfico a internet saldrá por la WAN sin competir con la metrica
```
## Nota Importante
El próximo paso planeado era utilizar zerotier, es un proyecto opensource que se puede instalar en un server vps y permite crear una red privada cifrda extremo a extremo entre varios equipos con conexión a internet. Además permite otorgar permiso de conexión entre equipos de la red.
La idea es utilizarlo para conectar nuestro servidor proxmox y la pc que lo administra y agregar una capa de seguridad adicional.
Para no contratar un server vps, utilizaré la versión web gratuita, disponible en https://www.zerotier.com/ que me da los recursos suficientes para nuestro homelab.

Zerotier trabaja también con un daemon que se instala en cada pc que queremos que pertenezca a la red virtual, si lo utilizamos en un contenedor LXC dentro de proxmox, no tendremos control total sobre la placa de red virtual que este software crea, ya que queda el kernel y sus controladores por fuera del contenedor y el daemon dentro del contenedor no puedo modificar los controladores que necesitamos para poder manipularlos a gusto y configurar. Es por esta razón que usaré una VM completa con Alpine Linux para instalarlo y asi si hacer uso de esa placa virtual que el daemon crea para configurarla dentro de la VM y posteriormente con nuestra VM router de iptables que tenemos que crear.

Entonces, con nuestro server dns proxy y router presentes en nuestro diseño, serían las próximas 3 piezas claves a seguir construyendo.

Por una cuestión de no tener dependencias, tenemos que construir nuestro router firewall de Iptables primero dentro de un contenedor LXC, luego nuestro server DNS, luego zerotier y continuaremos con el resto del diagrama.
Además, para router, utilizaremos lo mas básico a modo de práctica para el homelab, netfilter administrado por iptables.

## Diagrama: pasos a seguir

1. Router LXC          --> con sus placas de subredes y wan
        
2. DNS LXC             --> lo usaremos luego con los server de aplicación, NTP y NGNIX
        
3. VM ZeroTier         --> Acceso remoto a Proxmox, y los servicios de aplicación        

4. NTP Server          --> Time server. Necesita el DNS
        
5. NFS/SMB Server      --> Almacenamiento y backups. Necesita el DNS        

6. Prometheus/Grafana  --> Monitoreo. Necesita el DNS y los servicios        

7. Forgejo             --> Git autoalojado. Necesita el DNS (usaré el almacenamiento en el disco de la VM, no externo)
        
8. NGINX               --> Reverse proxy. Necesita el DNS y los servicios

NOTA: la idea de tener el DNS Server y ngnix configurado usando los nombres de dominios, esto me va a permitir migrar servicios desacoplando del hardware cada uno.
Por ejemplo, en el nuevo server:
- instalo proxmox, conecto el nuevo server con zerotier a la red del server viejo.
- Configuro el nuevo server ara que usa el NDS LXC del server viejo.
- Clono el contenedor del servicio que quiero tener en el nuevo server y le asigno una nueva IP.
- Desactivo el viejo contenedor del servicio que cloné. 
Ahora al tener Ngnix instalado configurado para buscar un dominio y no una IP voy a poder acceder al nuevo contenedor sin contratiempos de forma inmediata.

## Cambio Inesperado de ISP!
- Desconectamos la red Wifi
- Desactivamos el firewall  del datacenter y del server pve antes
- No tenemos acceso al router, y tenemos ip variable
- La red nueva del router tiene formato 192.168.18.0/24
- Cambiamos en proxmox vmbr0 (vinculada a la red (ethernet nic1) de 192.168.0.66 a 192.168.18.66

edit
```
# "/etc/network/interfaces"

# en vmbr0 cambiamos address
address 192.168.18.66/24
```
edit
```
# "/etc/hosts"

# cambiamos la dire de pve
#192.168.1.66
192.168.18.66
# no haremo más con esto, luego utilizaremos el server DNS en una VM
```
bash
```
# Borramos la antigua conexión
rm /etc/NetworkManager/system-connections/Gazzo-5GHz.nmconnection
# Creamos la nueva conexión
nmcli device wifi connect  "NewWIFI" password "NewPass"
systemctl restart NetworkManager
# chequeo que este UP y funcionando
ip a | grep wl
dig google

# Asignamos la IP fija, puerta de enlace y DNS
nmcli connection modify "NewWIFI" ipv4.addresses 192.168.18.50/24
nmcli connection modify "NewWIFI" ipv4.gateway 192.168.18.1
nmcli connection modify "NewWIFI" ipv4.dns "9.9.9.9 1.1.1.1"
# Aseguramos que la conexión se active automáticamente
nmcli connection modify "NewWIFI" connection.autoconnect yes
# Necesita primero una ip para poder cambiar el métodoo, las pusimos anteriormente
nmcli connection modify "NewWIFI" ipv4.method manual

systemctl restart NetworkManager
# chequeo que este UP y funcionando
ip a | grep wl
dig google
```
### Desde la GUI de proxmox:
- Configuramos las reglas del firewall para nmpc, ssh y admin de proxmox (pto:8006) con la IP nueva de la PC admin y cambiamos la del host.
- Utilizamos la IP de la red local 192.168.18.16

## Retomaremos con el próximo paso, la creación del DNS Server

### Creamos un contenedor LXC:

NOTA: Para este contenedor, utilizaremos el template de Debian 12, ya que permite usar iptables sin una capa de abstracción encima, es super estable, y bien documentado.
ASignamos 4Gb de disco porque Debian utiliza 1,5 dejando espacio para actualizaciones y logs, probaremos con 512Mb de ram inicialmente y 1 core.

Proxmox GUI
```
- Hostname: router-iptables
- ID : 100
- Template Debian 12 (La bajamos desde la GUI, en nuestro nodo local (pve))
- Disco 4Gb
- CPU : 1 core
- Memoria: 512Mb
- Sin privilegios
- Con Anidamientos (necesita simular /sys y /proc, para administrar sus int virtuales y propios procesos)
```

- Arquitectura de red para el router es:

    eth0 (WAN)  --> vmbr0 --> 192.168.18.2/24, gateway 192.168.18.1

    eth1 (LAN1) --> vmbr1 --> 10.0.10.1/24

    eth2 (LAN2) --> vmbr2 --> 10.0.20.1/24

    eth3 (LAN3) --> vmbr3 --> 10.0.30.1/24

    eth4 (LAN4) --> vmbr4 --> 10.0.40.1/24

- Creamos vbmr2, vbmr3 y vbmr4
  
Proxmox GUI
```
- Desde el nodo pve, System -> Network - Create Linux Bridge

vbmr2 --> Comment --> red de Servicios 10.0.20.0/24
vbmr3 --> Comment --> red de monitoreo 10.0.30.0/24
vbmr4 --> Comment --> red de ngnix 10.0.40.0/24

- Agregamos eth0 cuando creamos el contenedor.
- Agregamos eth1,eth2,eth3,eth4 desde --> CT100 --> network Device
- ASociamos cada uno con el bridge que le coreesponde, 2,3 y 4 respectivamente
- La GUI muestra error: bridge"vmbr2" dont exist (500)
Eror 500, error de comunicaciópn en el servidor, es porque no se aplicaron los cambios en la creación de bridges en el nodo pve, aplicamos para que se aqliquen y la api se pueda comunicar con el host.
- Completo la tarea con: WARN: missing 'source /etc/network/interfaces.d/sdn' directive for SDN support!
```

### En el nodo pve agregamos

edit
```
# "/etc/network/interfaces"
# Debajo del todo agregamos:
source /etc/network/interfaces.d/*
```
### Investigando un poco sobre el error
Note que con el comando anterior podemos dividir la configuración de red, pero Proxmox no lee configuraciones por fuera de /etc/network/interfaces como lo hace Debian con source/network/interfaces.d/* especificado en interfaces.
Con lo cual como mejora, por ejemplo, podríamos utilizar alguna configuración de respaldo si tuviéramos otra placa de red conectada al server proxmox, aislandola de la GUI de proxmox

bash
```
# Aplicamos
ifreload -a
# Aprovechamos a chequear los bridge
ip a | grep "vmbr"
```
## Actualización de /etc/network/interfaces luego de los cambios

edit
```
# network interface settings; autogenerated
# Please do NOT modify this file directly, unless you know what
# you're doing.
#
# If you want to manage parts of the network configuration manually,
# please utilize the 'source' or 'source-directory' directives to do
# so.
# PVE will preserve these directives, but will NOT read its network
# configuration from sourced files, so do not attempt to move any of
# the PVE managed interfaces into external files!

auto lo
iface lo inet loopback

iface nic0 inet manual

iface wlp3s0 inet manual
#Internet wifi --> Gestionada con Network-Manager
#Perfil --> /etc/NetworkManager/system-connections/Red123-5.8G

auto vmbr0
iface vmbr0 inet static
	address 192.168.18.66/24
	bridge-ports nic0
	bridge-stp off
	bridge-fd 0
#Red local-Ethernet admin backups

auto vmbr1
iface vmbr1 inet static
	address 10.0.10.99/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0
#Red Laboratorio Ppal. 10.0.10.0/24

auto vmbr2
iface vmbr2 inet manual
	bridge-ports none
	bridge-stp off
	bridge-fd 0
#Red de Servicios (DNS, NTP, NFS/SMB)10.0.20.0/24

auto vmbr3
iface vmbr3 inet manual
	bridge-ports none
	bridge-stp off
	bridge-fd 0
#Red de Monitoreo y Git 10.0.30.0/24

auto vmbr4
iface vmbr4 inet manual
	bridge-ports none
	bridge-stp off
	bridge-fd 0
#Red de Ngnix 10.0.40.0/24

source /etc/network/interfaces.d/*
```
## Probamos la red en el router
- Iniciamos el contenedor
  
bash
```
# -br brief --> resultados unificados
ip a -br
# Vemos que solo la Wan tiene Gateway
ip route
# chequemos la conectividad con Internet
dig google
```


