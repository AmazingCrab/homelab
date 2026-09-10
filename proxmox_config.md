# Hardware
Leonovo Thinkpad x230, CPU 3ra Gen, i5-320M TPD 35W, Idle 5-12W ,16GB de RAM, 256GB SATA SSD, 1 pto Ethernet 100/1000Mbps, 1 pto wifi               802.11a/b/g/n
Se eligio por su bajo consumo y robustez.

## Proxmox installer
(Listamos opciones)
- zfs -> ( xq tengo un solo disco, permite precachear antes de mover info y es mas fiable)
- country,
- timezone,
- keymap,
- email,
- managment interface,  -> nic0(ethernet) -> 192.168.1.66
- hostname -> pve.hostage

## Firewall pve
Nota: no asignamos source porque usamos la lan ethernet,
luego la wlan y quiza un vlan de zerotier

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


Bash
```
# "en  /etc/modprobe.d/zfs.conf"

# Min 1GB (1073741824 bytes)
options zfs zfs_arc_min=1073741824

# Max 4GB (4294967296 bytes)
options zfs zfs_arc_max=4294967296
```

## Tweak personalizado para apagar los monitores

Agregamos pagado automatico luego de 1min a  tty1

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

## 1. Deshabilitar los repositorios Enterprise:
bash
```
# Dentro de   /etc/apt/sources.list.d/

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
