# notas-homelab

Apuntes de cosas que fui armando en la red de mi casa. Son notas para mi, por si tengo que volver a hacerlo. Si a alguien le sirven, mejor.

Las IPs y MACs son de ejemplo. La red de ejemplo es `192.168.1.0/24`, el router del proveedor es `192.168.1.1`, el AP con OpenWrt es `192.168.1.2` y el NAS es `192.168.1.50`.

| Nota | De que va |
|---|---|
| [openwrt-dumb-ap.md](openwrt-dumb-ap.md) | Un Archer C60 con OpenWrt 18.06 usado como AP y switch, sin DHCP ni firewall, con un puerto de rescate aparte |
| [port-mirroring-tcpdump.md](port-mirroring-tcpdump.md) | Copiar el trafico de un puerto del switch al CPU del router y capturarlo desde la PC |
| [wake-on-lan-openwrt.md](wake-on-lan-openwrt.md) | Por que el WOL de LuCI no prendia el NAS y como quedo andando |
| [migrar-so-a-ssd.md](migrar-so-a-ssd.md) | Pasar el sistema del NAS de un disco de 3 TB a un SSD y dejar el disco viejo para datos |

Nada de esto es una guia oficial. Antes de copiar un comando, leelo.
