# Firewall I

## Realiza con NFTABLES el ejercicio de la página https://fp.josedomingo.org/seguridadgs/u03/perimetral_iptables.html documentando las pruebas de funcionamiento realizadas.

### Esquema de red
![](imagenes/Pasted%20image%2020240228210200.png)

**r-firewall**

```bash
debian@r-firewall-alex:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0c:75:2d:a8:00:00 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 192.168.100.2/24 brd 192.168.100.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::e75:2dff:fea8:0/64 scope link 
       valid_lft forever preferred_lft forever
3: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0c:75:2d:a8:00:01 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 192.168.122.243/24 brd 192.168.122.255 scope global dynamic ens5
       valid_lft 3496sec preferred_lft 3496sec
    inet6 fe80::e75:2dff:fea8:1/64 scope link 
       valid_lft forever preferred_lft forever
```

**pc-lan**

```bash
debian@pc-lan-alex:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0c:20:da:bd:00:00 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 192.168.100.10/24 brd 192.168.100.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::e20:daff:febd:0/64 scope link 
       valid_lft forever preferred_lft forever
```

### Tablas

Creamos la tabla `inet filter`, y le agregamos tres cadenas:
- **input**: para la entrada de paquetes, con prioridad 0, contará los paquetes que coincidan con esta cadena, y estará predeterminada en `drop`.

- **output**: para la salida de paquetes, con prioridad 0, también contará los paquetes y predeterminada en `drop`.

- **forward**: para el reenvío de paquetes igualmente con prioridad 0, contando paquetes y predeterminada en `drop`.

```bash
root@r-firewall-alex:~# nft add table inet filter
root@r-firewall-alex:~# nft add chain inet filter input { type filter hook input priority 0 \; counter \; policy drop \; }
root@r-firewall-alex:~# nft add chain inet filter output { type filter hook output priority 0 \; counter \; policy drop \; }
root@r-firewall-alex:~# nft add chain inet filter forward { type filter hook forward priority 0 \; counter \; policy drop \; }
```

La tabla `inet nat`, para la traducción de direcciones de red, con dos cadenas:
- **prerouting**: para antes del enrutamiento de los paquetes, con prioridad 0.

- **postrouting**: para después de que el sistema haya enrutado los paquetes pero antes de que salgan de la interfaz de red, con una prioridad de 100 pues se ejecutará después de las cadenas que tengan la prioridad más baja.

```bash
root@r-firewall-alex:~# nft add table inet nat
root@r-firewall-alex:~# nft add chain inet nat prerouting { type nat hook prerouting priority 0 \; }
root@r-firewall-alex:~# nft add chain inet nat postrouting { type nat hook postrouting priority 100 \; }
```

Así quedarian:

```bash
root@r-firewall-alex:~# nft list table inet filter
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 0 bytes 0
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 0 bytes 0
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 0 bytes 0
	}
}

root@r-firewall-alex:~# nft list table inet nat
table inet nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
	}
}

```

### Permitir ssh al cortafuegos

En mi caso estoy usando GNS3 y no necesitaría la conexión SSH pues me conecto directamente a la consola de la máquina, pero en un espacio real, los más seguro sería permitir solo las conexiones por SSH desde y hacia la red:

- La primera regla se aplica a la cadena saliente para permitir el tráfico SSH hacia cualquier dirección IP en la red 192.168.122.0 con puerto de destino 22 en las conexiones que estén en estado ya "establecido" es decir que solo permite la salida a conexiones que ya han sido antes establecidas.

- En la segunda regla se realiza lo mismo pero para permitir el tráfico entrante SSH desde cualquier IP en la red indicada con el puerto de origen 22, lo permitirá para conexiones nuevas o ya establecidas.

``` bash
root@r-firewall-alex:~# nft add rule inet filter output ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter accept

root@r-firewall-alex:~# nft add rule inet filter input ip saddr 192.168.122.0/24 tcp dport 22 ct state new,established counter accept
```

**Prueba**

Desde una maquina exterior conectamos con la máquina cortafuegos.

```bash
alex@alex-debian:~$ ssh debian@192.168.122.243
debian@192.168.122.243's password: 
Linux r-firewall-alex 6.1.0-15-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.66-1 (2023-12-09) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Feb 28 18:10:18 2024 from 192.168.122.1
debian@r-firewall-alex:~$ 
```

Como podemos ver se han aceptado los paquetes en ambas reglas pues ha permitido la conexión.

```bash
root@r-firewall-alex:~# nft list ruleset inet
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 105 bytes 12144
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 51 bytes 7132 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 0 bytes 0
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 74 bytes 11242
		ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter packets 48 bytes 8048 accept
	}
}
```


### Activar el bit de forward

La máquina que actúa como cortafuegos también es un router pues le activaremos el bit de forward.

```bash
root@r-firewall-alex:~# sysctl -p
net.ipv4.ip_forward = 1
```


### SNAT

Esta regla se encarga de realizar la traducción de direcciones de red para el tráfico saliente de la red 192.168.100.0/24 a través de la interfaz de red "ens5". 

```bash
root@r-firewall-alex:~# nft add rule inet nat postrouting oifname "ens5" ip saddr 192.168.100.0/24 counter masquerade
```


### Permitir el ssh desde el cortafuego a la LAN

- Primero a la salida aplicamos la regla que permite el tráfico SSH que sale por la interfaz indicada hacia la cualquier IP en la red 192.168.100.0 con puerto de destino 22, aplicado a conexiones nuevas y ya establecidas.

- Asimismo en la cadena de entrada permitiremos el tráfico SSH que proviene de la red indicada y desde el puerto de origen 22, solo aplicada para conexiones ya establecidas.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter accept
```

**Prueba**

Realizamos conexión SSH desde el cortafuegos a la máquina dentro de la LAN.

```bash
root@r-firewall-alex:~# ssh debian@192.168.100.10
debian@192.168.100.10's password: 
Linux pc-lan-alex 6.1.0-15-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.66-1 (2023-12-09) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Feb 28 20:28:44 2024 from 192.168.100.2
debian@pc-lan-alex:~$ 
```


Podemos que para las reglas que hemos agregado se han aceptado paquetes.

```bash
root@r-firewall-alex:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 219 bytes 34660
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 219 bytes 34660 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 1 bytes 84
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 352 bytes 46314
		ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter packets 0 bytes 0 accept
		oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state established,new counter packets 261 bytes 30904 accept
	}
}
```

### Permitir tráfico para la interfaz loopback

Aceptar tráfico saliente y entrante por la interfaz loopback, es decir a la misma máquina.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "lo" counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "lo" counter accept 
```

**Prueba**

Ya es posible realizar ping a la misma máquina.

```bash
root@r-firewall-alex:~# ping localhost
PING localhost(localhost (::1)) 56 data bytes
64 bytes from localhost (::1): icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from localhost (::1): icmp_seq=2 ttl=64 time=0.030 ms
64 bytes from localhost (::1): icmp_seq=3 ttl=64 time=0.030 ms
64 bytes from localhost (::1): icmp_seq=4 ttl=64 time=0.030 ms
^C
--- localhost ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 0.030/0.031/0.036/0.002 ms
```

Podemos comprobar que las reglas han aceptado los paquetes.

```bash
root@r-firewall-alex:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 242 bytes 36752
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 219 bytes 34660 accept
		iifname "lo" counter packets 22 bytes 2008 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 1 bytes 84
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 439 bytes 64222
		ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter packets 0 bytes 0 accept
		oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state established,new counter packets 261 bytes 30904 accept
		oifname "lo" counter packets 23 bytes 2092 accept
	}
}
table inet nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens5" ip saddr 192.168.100.0/24 counter packets 0 bytes 0 masquerade
	}
}
```


### Peticiones y respuestas protocolo ICMP


```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens5" icmp type echo-reply counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens5" icmp type echo-request counter accept
```

```bash
alex@alex-debian:~$ ping 192.168.122.243
PING 192.168.122.243 (192.168.122.243) 56(84) bytes of data.
64 bytes from 192.168.122.243: icmp_seq=1 ttl=64 time=0.522 ms
64 bytes from 192.168.122.243: icmp_seq=2 ttl=64 time=0.480 ms
64 bytes from 192.168.122.243: icmp_seq=3 ttl=64 time=0.496 ms
64 bytes from 192.168.122.243: icmp_seq=4 ttl=64 time=0.480 ms
^C
--- 192.168.122.243 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3067ms
rtt min/avg/max/mdev = 0.480/0.494/0.522/0.017 ms
```



```bash
root@r-firewall-alex:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 242 bytes 36752
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 219 bytes 34660 accept
		iifname "lo" counter packets 22 bytes 2008 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 1 bytes 84
root@r-firewall-alex:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 248 bytes 37762
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 219 bytes 34660 accept
		iifname "lo" counter packets 22 bytes 2008 accept
		iifname "ens5" icmp type echo-request counter packets 4 bytes 336 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 1 bytes 84
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 631 bytes 121502
		ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter packets 0 bytes 0 accept
		oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state established,new counter packets 261 bytes 30904 accept
		oifname "lo" counter packets 23 bytes 2092 accept
		oifname "ens5" icmp type echo-reply counter packets 4 bytes 336 accept
	}
}
table inet nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens5" ip saddr 192.168.100.0/24 counter packets 0 bytes 0 masquerade
	}
}
```















## Debes añadir después las reglas necesarias para que se permitan las siguientes operaciones:

### a) Permite poder hacer conexiones ssh al exterior desde la máquina cortafuegos.
### b) Permite hacer consultas DNS desde la máquina cortafuegos sólo al servidor 8.8.8.8. Comprueba que no puedes hacer un dig @1.1.1.1.
### c) Permite que la máquina cortafuegos pueda navegar por https.
### d) Los equipos de la red local deben poder tener conexión al exterior.
### e) Permitimos el ssh desde el cortafuegos a la LAN
### f) Permitimos hacer ping desde la LAN a la máquina cortafuegos.
### g) Permite realizar conexiones ssh desde los equipos de la LAN
### h) Instala un servidor de correos en la máquina de la LAN. Permite el acceso desde el exterior y desde el cortafuegos al servidor de correos. Para probarlo puedes ejecutar un telnet al puerto 25 tcp.
### i) Permite hacer conexiones ssh desde exterior a la LAN
### j) Modifica la regla anterior, para que al acceder desde el exterior por ssh tengamos que conectar al puerto 2222, aunque el servidor ssh este configurado para acceder por el puerto 22.
### k) Permite hacer consultas DNS desde la LAN sólo al servidor 8.8.8.8. Comprueba que no puedes hacer un dig @1.1.1.1.
### l) Permite que los equipos de la LAN puedan navegar por internet, excepto a la página www.realbetisbalompie.es