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

Este par de reglas en la cadena de salida y entrada respecticamente, permitirán el tráfico a los paquetes que salgan o entren  por la interfaz `ens5` que sean del tipo `icmp`, y con el contador activado.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens5" icmp type echo-reply counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens5" icmp type echo-request counter accept
```

Realizamos una prueba desde el exterior hacia el cortafuegos, y podemos ver que funciona.

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

Como podemos ver que el contador ha subido y por lo tanto está funcionando.

```bash
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



### Permitir hacer ping desde la LAN

Este par de reglas van a permitir el paso de solicitudes ping que vengan de la interfaz `ens4` dentro de la red indicada, y las respuestas ping que vengan por la interfaz externa `ens5` y con destino cualquier dirección IP en la red 192.168.100.0

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 icmp type echo-request counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 icmp type echo-reply counter accept
```


Comprobamos desde la máquina dentro de la LAN que puede salir haciendo ping.

```bash
debian@pc-lan-alex:~$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=62 time=5.12 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=62 time=4.93 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=62 time=4.92 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=62 time=6.06 ms
^C
--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 4.915/5.256/6.059/0.470 ms

debian@pc-lan-alex:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=18.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=15.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=15.7 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=116 time=16.0 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 15.054/16.188/18.092/1.145 ms
```

Como vemos las reglas han contabilizado los paquetes, entonces estos han sido aceptados.

```bash
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
		counter packets 17 bytes 1428
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 icmp type echo-request counter packets 8 bytes 672 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 icmp type echo-reply counter packets 8 bytes 672 accept
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 666 bytes 123808
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
		oifname "ens5" ip saddr 192.168.100.0/24 counter packets 2 bytes 168 masquerade
	}
}
```

### Consultas y respuestas DNS desde la LAN

Con el siguiente par de reglas permitiremos el funcionamiento del servicio DNS entre la LAN y el exterior mediante los paquetes UDP con puerto 53.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 udp dport 53 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 udp sport 53 ct state established counter accept
```

Comprobaremos haciendo ping a un dominio.

```bash
debian@pc-lan-alex:~$ ping arianagrande.com
PING arianagrande.com (199.83.132.192) 56(84) bytes of data.
64 bytes from 199.83.132.192.ip.incapdns.net (199.83.132.192): icmp_seq=1 ttl=54 time=116 ms
64 bytes from 199.83.132.192.ip.incapdns.net (199.83.132.192): icmp_seq=2 ttl=54 time=117 ms
64 bytes from 199.83.132.192.ip.incapdns.net (199.83.132.192): icmp_seq=3 ttl=54 time=117 ms
64 bytes from 199.83.132.192.ip.incapdns.net (199.83.132.192): icmp_seq=4 ttl=54 time=117 ms
^C
--- arianagrande.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 116.172/116.593/116.914/0.268 ms
```

O con el comando `dig` para que nos resuelva un nombre de dominio y nos dé la IP.

```bash
debian@pc-lan-alex:~$ dig arianagrande.com

; <<>> DiG 9.18.24-1-Debian <<>> arianagrande.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39888
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;arianagrande.com.		IN	A

;; ANSWER SECTION:
arianagrande.com.	1604	IN	A	199.83.128.184
arianagrande.com.	1604	IN	A	199.83.132.192

;; Query time: 36 msec
;; SERVER: 8.8.8.8#53(8.8.8.8) (UDP)
;; WHEN: Fri Mar 01 18:07:32 UTC 2024
;; MSG SIZE  rcvd: 77
```

Como vemos el contador ha aumentado pues ha aceptado los paquetes.

```bash
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
		counter packets 82 bytes 7083
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 icmp type echo-request counter packets 16 bytes 1344 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 icmp type echo-reply counter packets 16 bytes 1344 accept
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 udp dport 53 ct state established,new counter packets 16 bytes 1122 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 udp sport 53 ct state established counter packets 16 bytes 1897 accept
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 1352 bytes 196840
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
		oifname "ens5" ip saddr 192.168.100.0/24 counter packets 20 bytes 1458 masquerade
	}
}
```

### Permitir la navegación web desde la LAN

Estas reglas permitirán el tráfico de los puertos TCP 80 y 443 entre las interfaces de entrada y salida para las conexiones que salgan o entren hacia la red 192.168.100.0

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip protocol tcp ip saddr 192.168.100.0/24 tcp dport { 80,443} ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip protocol tcp ip daddr 192.168.100.0/24 tcp sport { 80,443} ct state established counter accept
```

Comprobaremos desde la máquina dentro de la LAN realizando un `curl` a una web del exterior y nos devuelve el código 200 que significa una conexión con éxito.

```bash
debian@pc-lan-alex:~$ curl -I https://biblioteca.alexnm.es/
HTTP/2 200 
server: nginx/1.22.1
date: Fri, 01 Mar 2024 18:38:08 GMT
content-type: text/html; charset=UTF-8
set-cookie: PHPSESSID=n5304547mrbr41tbvchbtcs81p; path=/
expires: Thu, 19 Nov 1981 08:52:00 GMT
cache-control: no-store, no-cache, must-revalidate
pragma: no-cache
```

### Permitir el acceso a nuestro servidor web de la LAN desde el exterior

Para permitir el acceso desde el exterior al servidor web dentro de la LAN, en primer lugar deberemos de configurar una regla en al tabla NAT para redirigir el tráfico del puerto 80 que entra por la interfaz exterior `ens5` hacia la dirección IP donde está el servidor, en este caso la 192.168.100.10, gracias a la traducción de dirección DNAT.

```bash
root@r-firewall-alex:~# nft add rule inet nat prerouting iifname "ens5" tcp dport 80 counter dnat ip to 192.168.100.10
```

Las siguientes dos permitirán el tráfico nuevo entrante por la interfaz `ens5` que sale por la `ens4` con destino cualquier IP en la red, con puerto de destino 80, y permitirá asimismo la respuesta que entre y salga por las interfaces desde cualquier IP en la red con puerto de origen 80. 

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 80 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 80 ct state established counter accept
```

Para probarlo, accedemos desde el exterior a la dirección IP de la interfaz exterior, y podemos ver que tenemos acceso a la web.

![](imagenes/Pasted%20image%2020240301211353.png)

Podemos también probarlo con el comando `curl` que nos devuelve el código 200 de una conexión con la web correcta.

```bash
alex@alex-debian:~$ curl -I 192.168.122.243
HTTP/1.1 200 OK
Date: Fri, 01 Mar 2024 20:14:03 GMT
Server: Apache/2.4.57 (Debian)
Last-Modified: Fri, 01 Mar 2024 20:13:38 GMT
ETag: "1a7-6129f02f2a35d"
Accept-Ranges: bytes
Content-Length: 423
Vary: Accept-Encoding
Content-Type: text/html
```

Vemos en la lista de reglas que los contadores de las correspondientes reglas ha subido.

```bash
root@r-firewall-alex:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 250 bytes 38436
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 219 bytes 34660 accept
		iifname "lo" counter packets 22 bytes 2008 accept
		iifname "ens5" icmp type echo-request counter packets 4 bytes 336 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 3306 bytes 2911813
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 icmp type echo-request counter packets 18 bytes 1512 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 icmp type echo-reply counter packets 18 bytes 1512 accept
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 udp dport 53 ct state established,new counter packets 103 bytes 6840 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 udp sport 53 ct state established counter packets 101 bytes 12309 accept
		iifname "ens4" oifname "ens5" ip protocol tcp ip saddr 192.168.100.0/24 tcp dport { 80, 443 } ct state established,new counter packets 950 bytes 65772 accept
		iifname "ens5" oifname "ens4" ip protocol tcp ip daddr 192.168.100.0/24 tcp sport { 80, 443 } ct state established counter packets 1995 bytes 2811624 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 80 ct state established,new counter packets 25 bytes 2639 accept
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 80 ct state established counter packets 18 bytes 3669 accept
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 2296 bytes 336640
		ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter packets 0 bytes 0 accept
		oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state established,new counter packets 261 bytes 30904 accept
		oifname "lo" counter packets 23 bytes 2092 accept
		oifname "ens5" icmp type echo-reply counter packets 4 bytes 336 accept
	}
}
table inet nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
		iifname "ens5" tcp dport 80 counter packets 4 bytes 240 dnat ip to 192.168.100.10
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens5" ip saddr 192.168.100.0/24 counter packets 127 bytes 8400 masquerade
	}
}
```


## Debes añadir después las reglas necesarias para que se permitan las siguientes operaciones:

A partir de ahora no mostraré el contador de paquetes para verificar el funcionamiento de cada regla pues se haría muy extenso, al final de la practica colocaré todo el rulset donde se podrán ver los contadores de cada una de las reglas.

### a) Permite poder hacer conexiones ssh al exterior desde la máquina cortafuegos.

Permitirán el tráfico SSH por el puerto 22 con conexiones salientes por `ens5` y permitiendo las respuestas que entren por la misma interfaz.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens5" tcp dport 22 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens5" tcp sport 22 ct state established counter accept
```

Comprobaremos realizando ssh a la máquina de mi VPS que se encuentra obviamente en el exterior.

```bash
root@r-firewall-alex:~# ssh alex@217.160.88.28
The authenticity of host '217.160.88.28 (217.160.88.28)' can't be established.
ED25519 key fingerprint is SHA256:JkmOZrbyrTVgfAslvGpQHwOKCLLQuTKe108RwG0aUos.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '217.160.88.28' (ED25519) to the list of known hosts.
alex@217.160.88.28's password: 
Linux isrevol 6.1.0-18-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Fri Mar  1 15:59:07 2024 from 87.220.173.181
alex@isrevol:~$ hostname -f
isrevol.alexnm.es
```


### b) Permite hacer consultas DNS desde la máquina cortafuegos sólo al servidor 8.8.8.8. Comprueba que no puedes hacer un dig @1.1.1.1.

Permiten el tráfico DNS por la interfaz exterior con destino y origen la IP 8.8.8.8, aceptará paquetes salientes con puerto destino 53 hacia el DNS y  aceptará asimismo los que entren con origen 53 y 8.8.8.8.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens5" ip daddr 8.8.8.8 udp dport 53 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens5" ip saddr 8.8.8.8 udp sport 53 ct state established counter accept
```

Probaremos que nos permite usar como DNS la ip 8.8.8.8, para resolver el dominio de mi web.

```bash
root@r-firewall-alex:~# dig @8.8.8.8 biblioteca.alexnm.es

; <<>> DiG 9.18.24-1-Debian <<>> @8.8.8.8 biblioteca.alexnm.es
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61551
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;biblioteca.alexnm.es.		IN	A

;; ANSWER SECTION:
biblioteca.alexnm.es.	3600	IN	CNAME	isrevol.alexnm.es.
isrevol.alexnm.es.	3600	IN	A	217.160.88.28

;; Query time: 31 msec
;; SERVER: 8.8.8.8#53(8.8.8.8) (UDP)
;; WHEN: Fri Mar 01 20:38:21 UTC 2024
;; MSG SIZE  rcvd: 87
```

Y que con la IP 1.1.1.1 no nos dejará.

```bash
root@r-firewall-alex:~# dig @1.1.1.1 biblioteca.alexnm.es
;; communications error to 1.1.1.1#53: timed out
;; communications error to 1.1.1.1#53: timed out
;; communications error to 1.1.1.1#53: timed out

; <<>> DiG 9.18.24-1-Debian <<>> @1.1.1.1 biblioteca.alexnm.es
; (1 server found)
;; global options: +cmd
;; no servers could be reached
```
### c) Permite que la máquina cortafuegos pueda navegar por https.

Con estas reglas permitimos el tráfico de los puertos 80 y 443, por la interfaz `ens5` para las conexiones entrantes y salientes.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens5" ip protocol tcp tcp dport { 80,443} ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens5" ip protocol tcp tcp sport { 80,443} ct state established counter accept
```

Probaremos la regla con el comando `curl` que nos devuelve una conexión con la web con éxito.

```bash
root@r-firewall-alex:~# curl -I https://biblioteca.alexnm.es/
HTTP/2 200 
server: nginx/1.22.1
date: Fri, 01 Mar 2024 20:47:26 GMT
content-type: text/html; charset=UTF-8
set-cookie: PHPSESSID=3acqp92q9mpgkq0vn0agej9qg4; path=/
expires: Thu, 19 Nov 1981 08:52:00 GMT
cache-control: no-store, no-cache, must-revalidate
pragma: no-cache
```


### d) Los equipos de la red local deben poder tener conexión al exterior.

Estas reglas permitirán el tráfico que pase por el cortafuegos hacia el exterior aceptando paquetes salientes y respuestas de entrada.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 icmp type echo-request counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 icmp type echo-reply counter accept
```

Comprobamos la conexión al exterior mediante un ping a 8.8.8.8, o a una web...

```bash
debian@pc-lan-alex:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=16.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=15.9 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=49.5 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=116 time=16.4 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 15.887/24.545/49.524/14.422 ms

debian@pc-lan-alex:~$ ping www.arianagrande.com
PING sy7b8.x.incapdns.net (192.230.82.184) 56(84) bytes of data.
64 bytes from 192.230.82.184.ip.incapdns.net (192.230.82.184): icmp_seq=1 ttl=57 time=15.6 ms
64 bytes from 192.230.82.184.ip.incapdns.net (192.230.82.184): icmp_seq=2 ttl=57 time=15.9 ms
64 bytes from 192.230.82.184.ip.incapdns.net (192.230.82.184): icmp_seq=3 ttl=57 time=16.2 ms
64 bytes from 192.230.82.184.ip.incapdns.net (192.230.82.184): icmp_seq=4 ttl=57 time=16.3 ms
^C
--- sy7b8.x.incapdns.net ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 15.558/15.983/16.283/0.277 ms
```
### e) Permitimos el ssh desde el cortafuegos a la LAN

Con estas reglas permitiremos el tráfico SSH (puerto 22) desde el cortafuegos hacia la LAN mediante la interfaz interna `ens4` permitiendo la conexiones salientes y entrantes para las respuestas.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens4" tcp dport 22 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens4" tcp sport 22 ct state established counter accept
```

Comprobaremos realizando ssh desde el cortafuegos hacia una máquina de la LAN.

```bash
root@r-firewall-alex:~# ssh debian@192.168.100.10
debian@192.168.100.10's password: 
Linux pc-lan-alex 6.1.0-18-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Mar  1 17:55:45 2024
debian@pc-lan-alex:~$ 
```


### f) Permitimos hacer ping desde la LAN a la máquina cortafuegos.

Este par de reglas permitirán el tráfico ICMP mediante la interfaz interna `ens4` permitiendo las respuestas ping salientes y las solicitudes de ping que entran.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens4" icmp type echo-reply counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens4" icmp type echo-request counter accept
```

Probaremos las reglas realizando ping desde una máquina de la LAN hacia el cortafuegos.

```bash
debian@pc-lan-alex:~$ ping 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=0.732 ms
64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=0.635 ms
64 bytes from 192.168.100.2: icmp_seq=3 ttl=64 time=0.724 ms
64 bytes from 192.168.100.2: icmp_seq=4 ttl=64 time=0.932 ms
^C
--- 192.168.100.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 0.635/0.755/0.932/0.108 ms
```

### g) Permite realizar conexiones ssh desde los equipos de la LAN

Permitiremos con estas reglas el paso del tráfico entre las interfaces interna y externa, aceptará las conexiones SSH entrantes por `ens4` y que salgan por `ens5`, y aceptará las respuestas que vengan desde la interfaz del exterior y salgan por la de adentro.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" tcp dport 22 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" tcp sport 22 ct state established counter accept
```

Probaremos realizando una conexión SSH desde una máquina de la LAN a una máquina del exterior como por ejemplo mi VPS.

```bash
debian@pc-lan-alex:~$ ssh alex@217.160.88.28
The authenticity of host '217.160.88.28 (217.160.88.28)' can't be established.
ED25519 key fingerprint is SHA256:JkmOZrbyrTVgfAslvGpQHwOKCLLQuTKe108RwG0aUos.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '217.160.88.28' (ED25519) to the list of known hosts.
alex@217.160.88.28's password: 
Linux isrevol 6.1.0-18-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Fri Mar  1 23:14:40 2024 from 87.220.173.181
alex@isrevol:~$ 
```

Las siguientes permitirán el tráfico SSH desde la red interna hacia el cortafuegos, para ello indicamos la interfaz interna `ens4` que permitirá las solicitudes entrantes con puerto de destino 22 y asimismo las salidas de las respuestas.


```bash
root@r-firewall-alex:~# nft add rule inet filter input iifname "ens4" tcp dport 22 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter output oifname "ens4" tcp sport 22 ct state established counter accept
```

Probaremos desde una máquina en la red interna a realizar una conexión SSH hacia el cortafuegos.

```bash
debian@pc-lan-alex:~$ ssh debian@192.168.100.2
The authenticity of host '192.168.100.2 (192.168.100.2)' can't be established.
ED25519 key fingerprint is SHA256:zn2i5rAyilMi1i+Kqb6ys8GhldKuHKYZCDKbD1aXqjQ.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:1: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.2' (ED25519) to the list of known hosts.
debian@192.168.100.2's password: 
Linux r-firewall-alex 6.1.0-15-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.66-1 (2023-12-09) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Mar  1 23:09:39 2024 from 192.168.122.1
debian@r-firewall-alex:~$ 
```


### h) Instala un servidor de correos en la máquina de la LAN. Permite el acceso desde el exterior y desde el cortafuegos al servidor de correos. Para probarlo puedes ejecutar un telnet al puerto 25 tcp.

En primer lugar crearemos una regla DNAT para que rediriga el tráfico con puerto de destino 25, que entre por la interfaz `ens5` y vaya hacia la dirección IP 192.168.100.10.

```bash
root@r-firewall-alex:~# nft add rule inet nat prerouting iifname "ens5" tcp dport 25 counter dnat ip to 192.168.100.10
```

Las siguientes reglas permitirán el paso por el cortafuegos del tráfico de correo por el puerto 25, que permitiría las conexiones entrantes por la `ens5` hacia la red interna, y permitiá la salida de las respuestas desde la red pasando por la `ens4`.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 25 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 25 ct state established counter accept
```

Para comprobarlo realizaremos una conexión con `telnet` a la dirección IP del cortafuegos al puerto 25, que redirigirá y permitirá la conexión al servidor de correos postfix.

```bash
alex@alex-debian:~$ telnet 192.168.122.243 25
Trying 192.168.122.243...
Connected to 192.168.122.243.
Escape character is '^]'.
220 pc-lan-alex ESMTP Postfix (Debian/GNU)
```

Ahora para permitir desde el cortafuegos, creamos un par de reglas que aceptarán los paquetes que salgan con puerto de destino 25 por `ens4` y permitirán las respuestas que entren por la misma con puerto de origen 25.

```bash
root@r-firewall-alex:~# nft add rule inet filter output oifname "ens4" tcp dport 25 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter input iifname "ens4" tcp sport 25 ct state established counter accept
```

Lo probaremos desde la máquina del cortafuegos realizando asimismo una conexión con `telnet` al puerto 25.

```bash
root@r-firewall-alex:~# telnet 192.168.100.10 25
Trying 192.168.100.10...
Connected to 192.168.100.10.
Escape character is '^]'.
220 pc-lan-alex ESMTP Postfix (Debian/GNU)
```

### i) Permite hacer conexiones ssh desde exterior a la LAN

Primero creamos una regla DNAT para redirigir el tráfico que llegue a la interfaz `ens5` con puerto de destino 22 a la máquina 192.168.100.10-

```bash
root@r-firewall-alex:~# nft add rule inet nat prerouting iifname "ens5" tcp dport 22 counter dnat ip to 192.168.100.10
```

Estas reglas van a permitir el tráfico SSH que pase por el cortafuegos aceptando paquetes entrantes hacia la red interna por la interfaz `ens5` y asimismo las respuestas de las conexiones que vienen de la red y entran por la `ens4`.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter accept
```

Probaremos realizando una conexión SSH desde una máquina del exterior, a la dirección IP de la máquina del cortafuegos que redirigirá y aceptará el tráfico para la conexión.

```bash
alex@alex-debian:~$ ssh debian@192.168.122.243
debian@192.168.122.243's password: 
Linux pc-lan-alex 6.1.0-18-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Mar  1 23:51:16 2024
debian@pc-lan-alex:~$ 
```

### j) Modifica la regla anterior, para que al acceder desde el exterior por ssh tengamos que conectar al puerto 2222, aunque el servidor ssh este configurado para acceder por el puerto 22.

Creamos una nueva regla DNAT que redirigirá todo el tráfico que llegue a `ens5` con puerto de destino 2222 hacia la máquina con dirección IP 192.168.100.10 al puerto 22.

```bash
root@r-firewall-alex:~# nft add rule inet nat prerouting iifname "ens5" tcp dport 2222 counter dnat ip to 192.168.100.10:22
```

Probamos realizando una conexión por SSH indicando el puerto 2222.

```bash
alex@alex-debian:~$ ssh debian@192.168.122.243 -p 2222
debian@192.168.122.243's password: 
Linux pc-lan-alex 6.1.0-18-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Mar  2 00:14:27 2024 from 192.168.122.1
debian@pc-lan-alex:~$ 
```

Podemos realizar también una conexión por `telnet` al puerto 2222.

```bash
alex@alex-debian:~$ telnet 192.168.122.243 2222
Trying 192.168.122.243...
Connected to 192.168.122.243.
Escape character is '^]'.
SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u2
```

### k) Permite hacer consultas DNS desde la LAN sólo al servidor 8.8.8.8. Comprueba que no puedes hacer un dig @1.1.1.1.

Anteriormente hemos establecido un par de reglas que permitían a las máquinas en la LAN consultar al DNS, pero a cualquiera entonces debemos de eliminarlas.

(salida recortada)
```bash
root@r-firewall-alex:~# nft -a list ruleset
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 udp dport 53 ct state established,new counter packets 16 bytes 1122 accept #handle 33
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 udp sport 53 ct state established counter packets 16 bytes 1897 accept #handle 34
```

```bash
root@r-firewall-alex:~# nft delete rule inet filter forward handle 33

root@r-firewall-alex:~# nft delete rule inet filter forward handle 34
```


Este par de reglas permitirán el tráfico de paquetes DNS que pasen por el cortafuegos con puerto de destino 53 que entren por la `ens5` y con paquetes de puerto de origen 53.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip daddr 8.8.8.8 udp dport 53 ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip saddr 8.8.8.8 udp sport 53 ct state established counter accept
```

Probamos que resuelve con el servidor 8.8.8.8

```bash
debian@pc-lan-alex:~$ dig @8.8.8.8 alexnm.es

; <<>> DiG 9.18.24-1-Debian <<>> @8.8.8.8 alexnm.es
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64065
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;alexnm.es.			IN	A

;; ANSWER SECTION:
alexnm.es.		3600	IN	A	217.160.88.28

;; Query time: 52 msec
;; SERVER: 8.8.8.8#53(8.8.8.8) (UDP)
;; WHEN: Sat Mar 02 01:02:55 UTC 2024
;; MSG SIZE  rcvd: 54
```

Vemos que con 1.1.1.1 no lo permite.

```bash
debian@pc-lan-alex:~$ dig @1.1.1.1 alexnm.es
;; communications error to 1.1.1.1#53: timed out
;; communications error to 1.1.1.1#53: timed out
;; communications error to 1.1.1.1#53: timed out

; <<>> DiG 9.18.24-1-Debian <<>> @1.1.1.1 alexnm.es
; (1 server found)
;; global options: +cmd
;; no servers could be reached
```


### l) Permite que los equipos de la LAN puedan navegar por internet, excepto a la página www.realbetisbalompie.es

Anteriormente agregamos un par de reglas que permitian navegar a las máquinas en la LAN, para bloquear que naveguen por la pagina indicada, eliminaremos las reglas anteriores.

(salida recortada)
```bash
root@r-firewall-alex:~# nft -a list ruleset
		iifname "ens4" oifname "ens5" ip protocol tcp ip saddr 192.168.100.0/24 tcp dport { 80, 443 } ct state established,new counter packets 805 bytes 48654 accept #handle 21
		iifname "ens5" oifname "ens4" ip protocol tcp ip daddr 192.168.100.0/24 tcp sport { 80, 443 } ct state established counter packets 1532 bytes 2271927 accept #handle 23
```

```bash
root@r-firewall-alex:~# nft delete rule inet filter forward handle 21

root@r-firewall-alex:~# nft delete rule inet filter forward handle 23
```

Crearemos primero una regla que descartará el tráfico con destino 51.255.76.196 que es la dirección IP a la que corresponde la página web, en ambis puertos y que pasa por la `ens4` y sale por `ens5`.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip protocol tcp ip daddr 51.255.76.196 tcp dport { 80,443} ct state new,established counter drop
```

Ahora permitiremos las conexiones salientes desde la red interna hacia los puertos HTTP y HTTPS que pasen desde la `ens4` y salgan por la `ens5`, también la repsuestas de las conexiones que entran por la interfaz externa que van hacia la red interna.

```bash
root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens4" oifname "ens5" ip protocol tcp ip saddr 192.168.100.0/24 tcp dport { 80,443} ct state new,established counter accept

root@r-firewall-alex:~# nft add rule inet filter forward iifname "ens5" oifname "ens4" ip protocol tcp ip daddr 192.168.100.0/24 tcp sport { 80,443} ct state established counter accept
```

Probaremos a realizar una conexión con `curl` a cualquier pagina web.

```bash
debian@pc-lan-alex:~$ curl -I https://biblioteca.alexnm.es/
HTTP/2 200 
server: nginx/1.22.1
date: Sat, 02 Mar 2024 01:36:44 GMT
content-type: text/html; charset=UTF-8
set-cookie: PHPSESSID=lpg7gbkliiej4tltl3i23ci1fu; path=/
expires: Thu, 19 Nov 1981 08:52:00 GMT
cache-control: no-store, no-cache, must-revalidate
pragma: no-cache
```

Para la web realizaremos una conexión con `curl` indicándole un tiempo para que no se quede indefinidamente intentandolo, como vemos no llega.

```bash
debian@pc-lan-alex:~$ curl -I -v --max-time 10 --connect-timeout 5 https://www.realbetisbalompie.es/
*   Trying 51.255.76.196:443...
* ipv4 connect timeout after 4968ms, move on!
* Failed to connect to www.realbetisbalompie.es port 443 after 5001 ms: Timeout was reached
* Closing connection 0
curl: (28) Failed to connect to www.realbetisbalompie.es port 443 after 5001 ms: Timeout was reached
```


## Reglas

Estas son todas las reglas, algunas no han contabilizado paquetes pues son redundantes, ya que por ejemplo estas dos, es directamente aceptada por la primera pues la red interna es 192.168.100.0 y es la red a la que está conectada la interfaz `ens4` es por eso que ya se permiten los paquetes con la primera.

```bash
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 184 bytes 28436 accept
---
		iifname "ens4" tcp sport 22 ct state established counter packets 0 bytes 0 accept
```

```bash
root@r-firewall-alex:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		counter packets 1816 bytes 1842146
		ip saddr 192.168.122.0/24 tcp dport 22 ct state established,new counter packets 49 bytes 7104 accept
		iifname "ens4" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 184 bytes 28436 accept
		iifname "lo" counter packets 22 bytes 2288 accept
		iifname "ens5" icmp type echo-request counter packets 9 bytes 756 accept
		iifname "ens5" tcp sport 22 ct state established counter packets 60 bytes 9308 accept
		iifname "ens5" ip saddr 8.8.8.8 udp sport 53 ct state established counter packets 42 bytes 5097 accept
		iifname "ens5" ip protocol tcp tcp sport { 80, 443 } ct state established counter packets 1166 bytes 1768025 accept
		iifname "ens4" tcp sport 22 ct state established counter packets 0 bytes 0 accept
		iifname "ens4" icmp type echo-request counter packets 6 bytes 504 accept
		iifname "ens4" tcp dport 22 ct state established,new counter packets 266 bytes 19548 accept
		iifname "ens4" tcp sport 25 ct state established counter packets 5 bytes 359 accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 3450 bytes 2453934
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 icmp type echo-request counter packets 36 bytes 3024 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 icmp type echo-reply counter packets 35 bytes 2940 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 80 ct state established,new counter packets 11 bytes 671 accept
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 80 ct state established counter packets 9 bytes 1219 accept
		iifname "ens4" oifname "ens5" tcp dport 22 ct state established,new counter packets 38 bytes 6440 accept
		iifname "ens5" oifname "ens4" tcp sport 22 ct state established counter packets 36 bytes 7612 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 25 ct state established,new counter packets 14 bytes 768 accept
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 25 ct state established counter packets 14 bytes 910 accept
		iifname "ens5" oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state established,new counter packets 101 bytes 17596 accept
		iifname "ens4" oifname "ens5" ip saddr 192.168.100.0/24 tcp sport 22 ct state established counter packets 85 bytes 17884 accept
		iifname "ens4" oifname "ens5" ip daddr 8.8.8.8 udp dport 53 ct state established,new counter packets 77 bytes 5276 accept
		iifname "ens5" oifname "ens4" ip saddr 8.8.8.8 udp sport 53 ct state established counter packets 75 bytes 9429 accept
		iifname "ens4" oifname "ens5" ip protocol tcp ip daddr 51.255.76.196 tcp dport { 80, 443 } ct state established,new counter packets 46 bytes 2760 drop
		iifname "ens4" oifname "ens5" ip protocol tcp ip saddr 192.168.100.0/24 tcp dport { 80, 443 } ct state established,new counter packets 36 bytes 4248 accept
		iifname "ens5" oifname "ens4" ip protocol tcp ip daddr 192.168.100.0/24 tcp sport { 80, 443 } ct state established counter packets 29 bytes 12489 accept
	}

	chain output {
		type filter hook output priority filter; policy drop;
		counter packets 1528 bytes 172599
		ip daddr 192.168.122.0/24 tcp sport 22 ct state established counter packets 47 bytes 7784 accept
		oifname "ens4" ip daddr 192.168.100.0/24 tcp dport 22 ct state established,new counter packets 191 bytes 22512 accept
		oifname "lo" counter packets 22 bytes 2288 accept
		oifname "ens5" icmp type echo-reply counter packets 9 bytes 756 accept
		oifname "ens5" tcp dport 22 ct state established,new counter packets 74 bytes 8204 accept
		oifname "ens5" ip daddr 8.8.8.8 udp dport 53 ct state established,new counter packets 43 bytes 2911 accept
		oifname "ens5" ip protocol tcp tcp dport { 80, 443 } ct state established,new counter packets 598 bytes 33438 accept
		oifname "ens4" tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		oifname "ens4" icmp type echo-reply counter packets 6 bytes 504 accept
		oifname "ens4" tcp sport 22 ct state established counter packets 283 bytes 37128 accept
		oifname "ens4" tcp dport 25 ct state established,new counter packets 5 bytes 268 accept
	}
}
table inet nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
		iifname "ens5" tcp dport 80 counter packets 2 bytes 120 dnat ip to 192.168.100.10
		iifname "ens5" tcp dport 25 counter packets 4 bytes 240 dnat ip to 192.168.100.10
		iifname "ens5" tcp dport 22 counter packets 2 bytes 120 dnat ip to 192.168.100.10
		iifname "ens5" tcp dport 2222 counter packets 1 bytes 60 dnat ip to 192.168.100.10:22
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		oifname "ens5" ip saddr 192.168.100.0/24 counter packets 219 bytes 14925 masquerade
	}
}
```

Para que las reglas se guarden y estén siempre activas aunque se reinicie el cortafuegos activaremos el servicio y las guardaremos con el último comando en la ruta `/etc/nftables.conf`.

```bash
root@r-firewall-alex:~# systemctl start nftables
root@r-firewall-alex:~# systemctl enable nftables
root@r-firewall-alex:~# nft list ruleset > /etc/nftables.conf 
```