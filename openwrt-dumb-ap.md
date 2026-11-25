# OpenWrt como dumb AP

Puse un TP-Link Archer C60 v2 con OpenWrt 18.06.1 como AP y switch dentro de la misma red del router del proveedor. El router del proveedor sigue dando DHCP y salida a internet. El C60 solo hace de puente entre sus puertos LAN, el wifi y la red.

## La idea

* Los cuatro puertos LAN y los dos radios quedan en el bridge `br-lan`.
* El bridge tiene una IP fija dentro de la red, para entrar por SSH y LuCI.
* El DHCP de la LAN queda apagado. Lo sigue dando el router del proveedor.
* El firewall y odhcpd quedan apagados. No hacen falta en un puente.
* El puerto WAN (el azul) queda aislado, con su propia IP y su propio DHCP. Es el puerto de rescate. Si rompo la config de la LAN, enchufo la notebook ahi y entro igual.

## Red

```
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.1.2'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='192.168.1.1'

# el puerto WAN pasa a ser el de rescate
uci delete network.wan
uci delete network.wan6
uci set network.mgmt='interface'
uci set network.mgmt.ifname='eth0'
uci set network.mgmt.proto='static'
uci set network.mgmt.ipaddr='192.168.54.1'
uci set network.mgmt.netmask='255.255.255.0'
uci commit network
```

En el C60 la LAN es `eth1.1` y el WAN es `eth0`. Las cinco bocas del switch interno van en la VLAN 1 con el CPU tagueado, o sea `ports '1 2 3 4 0t'`.

## DHCP y servicios

```
uci set dhcp.lan.ignore='1'          # nada de DHCP en la LAN
uci set dhcp.mgmt='dhcp'
uci set dhcp.mgmt.interface='mgmt'
uci set dhcp.mgmt.start='100'
uci set dhcp.mgmt.limit='50'
uci set dhcp.mgmt.leasetime='12h'
uci commit dhcp

/etc/init.d/firewall disable && /etc/init.d/firewall stop
/etc/init.d/odhcpd disable && /etc/init.d/odhcpd stop
```

dnsmasq queda prendido porque es el que da DHCP en el puerto de rescate.

## Wifi

Los dos radios en modo `ap` y con `network='lan'`. Con eso los clientes del wifi caen en la misma red que todo lo demas y reciben IP del router del proveedor.

## Cosas que me hicieron perder tiempo

**Los numeros de puerto estan al reves.** Los puertos de `swconfig` no coinciden con lo que dice la carcasa, van invertidos. El CPU es el puerto 0. Para saber cual es cual lo mas facil es enchufar un cable por vez y mirar `swconfig dev switch0 show | grep link`.

**Dropbear viejo no acepta ed25519.** El dropbear de 18.06 es de 2017 y solo acepta claves RSA. La clave publica va en `/etc/dropbear/authorized_keys`. Desde un cliente OpenSSH nuevo, como el que trae Windows, hay que habilitar RSA a mano.

```
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.1.2
```

O dejarlo fijo en `~/.ssh/config`.

```
Host ap
    HostName 192.168.1.2
    User root
    IdentityFile ~/.ssh/ap_rsa
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```
