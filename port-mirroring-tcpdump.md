# Port mirroring en OpenWrt y captura desde la PC

Queria ver todo el trafico de un aparato enchufado por cable sin instalarle nada. El switch interno del C60 (un AR934X) tiene port mirroring. Se le dice que copie lo que entra y sale de un puerto hacia otro, y si el destino es el CPU, el trafico se puede capturar con tcpdump en el mismo router.

## Instalar tcpdump

En un router chico conviene la version mini.

```
opkg update
opkg install tcpdump-mini
```

## Prender el mirror

El aparato a mirar esta en el puerto 2 de `swconfig`. El destino es el puerto 0, que es el CPU.

```
swconfig dev switch0 set mirror_monitor_port 0
swconfig dev switch0 port 2 set enable_mirror_rx 1
swconfig dev switch0 port 2 set enable_mirror_tx 1
```

El `set` aplica directo. No existe un `apply` para esto y si lo corres da error.

Para apagarlo es lo mismo con 0.

```
swconfig dev switch0 port 2 set enable_mirror_rx 0
swconfig dev switch0 port 2 set enable_mirror_tx 0
```

Esto no queda guardado. Si el router se reinicia el mirror se apaga solo.

## Capturar desde la PC

La captura se manda por SSH directo a un archivo en la PC. El router casi no tiene espacio y asi no se llena.

```
ssh ap "tcpdump -i eth1.1 -U -s0 -w - 'host 192.168.1.105 and not port 22'" > captura.pcap
```

* `-w -` escribe el pcap por la salida estandar.
* `-U` vacia el buffer en cada paquete. Sin esto el archivo llega a los tirones.
* `not port 22` saca la propia sesion de SSH, que si no se mete en la captura.
* Se corta con Ctrl+C y lo que llego queda en el archivo.

Si se quiere ver en vivo, se le puede pasar a Wireshark.

```
ssh ap "tcpdump -i eth1.1 -U -s0 -w - 'not port 22'" | wireshark -k -i -
```

## Lo que no anduvo

Probe primero dejar la captura en el router con un tiempo limite.

```
timeout 60 tcpdump -i eth1.1 -w /tmp/captura.pcap
```

El pcap quedaba vacio. No me puse a investigar por que. Con la captura por SSH y `-U` anduvo bien y no le di mas vueltas.

## Limites

El trafico de los dos sentidos del puerto termina en el CPU del router. En un router de este tamano esta bien para un aparato que habla poco. Para algo que mueve mucho trafico se pueden perder paquetes.
