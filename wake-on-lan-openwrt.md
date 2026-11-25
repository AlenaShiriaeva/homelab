# Wake on LAN desde OpenWrt

El AP con OpenWrt tiene `luci-app-wol` y `etherwake`. La idea era prender el NAS desde afuera, entrando a LuCI por la VPN. Mandaba el paquete, LuCI decia que todo bien y el NAS no prendia. Habia tres problemas juntos.

## 1. La MAC guardada no era la del NAS

LuCI arma la lista de equipos con los hosts estaticos de DHCP y con la tabla ARP. Yo tenia un host estatico con el nombre del NAS, pero la MAC era de otro equipo. Elegia el nombre de la lista y el paquete iba a otro lado.

La MAC real se saca en el propio equipo.

```
ip -br link show enp3s0
```

Y se corrige en el router.

```
uci set dhcp.@host[0].mac='aa:bb:cc:dd:ee:50'
uci commit dhcp
```

En este AP el DHCP de la LAN esta apagado, asi que ese host estatico no reparte ninguna IP. Solo sirve para que aparezca el nombre en la lista de WOL.

## 2. El paquete salia por el puerto equivocado

En la pagina de WOL hay un campo de interfaz y por defecto dice "Broadcast on all interfaces". No hace eso. Deja el campo vacio, y etherwake sin `-i` usa `eth0`. En este AP `eth0` es el puerto de rescate, que no tiene nada enchufado. etherwake contesta "Sendto worked" y el paquete no llega a ningun lado.

A mano tiene que ser asi.

```
etherwake -i br-lan aa:bb:cc:dd:ee:50
```

Para no tener que acordarme de elegir `br-lan` cada vez, le agregue una linea a la pagina de LuCI. En la version vieja de `luci-app-wol` (la de Lua) el archivo es `/usr/lib/lua/luci/model/cbi/wol.lua`. Justo despues de la opcion "Broadcast on all interfaces" va esto.

```lua
iface.default = "br-lan"
```

Despues hay que borrar la cache de LuCI para que lo tome.

```
rm -rf /tmp/luci-indexcache* /tmp/luci-modulecache
```

Si se actualiza el paquete, el archivo se pisa y hay que volver a hacerlo.

## 3. El NAS tenia el WOL apagado en la placa

```
sudo ethtool enp3s0 | grep Wake-on
	Supports Wake-on: pumbg
	Wake-on: d
```

`d` es desactivado. `g` es despertar con magic packet. Se puede prender en el momento con ethtool, pero eso se pierde al reiniciar. Como el NAS usa NetworkManager, lo deje fijo en la conexion.

```
sudo nmcli con modify "Wired connection 1" 802-3-ethernet.wake-on-lan magic
sudo ethtool -s enp3s0 wol g
```

El primer comando lo deja guardado y el segundo lo aplica ya, sin tener que reconectar.

## Como probar sin apagar nada

Con el NAS prendido se puede ver si el paquete llega. En el NAS se escucha el ethertype de WOL.

```
sudo tcpdump -i enp3s0 -n -c1 'ether proto 0x0842'
```

Y desde el router se manda el paquete. Si en tcpdump aparece un paquete con la MAC del NAS repetida 16 veces, la red esta bien. Lo que queda despues depende de la placa.

## Si igual no prende

Eso ya es de la BIOS. Hay que buscar una opcion tipo "Wake on LAN" o "Power On by PCI-E" y apagar ErP, que corta la energia de la placa de red cuando el equipo esta apagado.
