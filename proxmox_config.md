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
# "/etc/network/interface
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
# /etc/network/interfaces

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
ip link | grep wl
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

Zerotier es  un daemon que se instala en cada pc que pertenece a la red virtual, si lo utilizamos en un contenedor LXC, no tendremos control total sobre la placa de red virtual que este software crea, ya que queda el kernel  y sus controladores por fuera y el demon dentro del controlador separados, es por esta razón que usaré una VM completa con Alpine Linux para instalarlo y asi si hacer uso de esa placa virtual para configurarla dentro de la VM y con en nuestra VM router que tenemos que crear.

Entonces, con nuestro server dns proxy y router presentes en nuestro diseño, serían las próximas 3 piezas claves a seguir construyendo.

Por una cuestión de no tener dependencias, tenemos que construir nuestro router firewall de Iptables primero dentro de un contenedor LXC, luego nuestro server DNS, luego zerotier y continuaremos con el resto del diagrama.



