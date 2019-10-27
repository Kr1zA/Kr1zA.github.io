---
layout: post
title:  DHCP server
date:   2019-10-27
image:  dhcp.jpg
tags:   [AOS, DHCP]
---

Po nastavení [NATU](https://kr1za.github.io/linux-NAT-router/), máme nastavené pripojenie klienta k serveru nastavením pevnej IP adresy na klientovi. Funguje to ale je to nepraktické, ak sa často pripája a odpája zo siete veľa klientov. Poďme teda nastaviť DHCP server, ktorý priradí pripojeným klientom IP adresy automaticky.

Najprv nainštalujeme dhcp príkazom:

> apt-get install isc-dhcp-server

Zapnutie/vypnutie/reštartovanie DHCP servera vieme urobiť príkazom:

> service isc-dhcp-server start/stop/restart

Po inštalácia sa DHCP server automaticky spustí, spustenie dhcp ale bude neúspešné. Dǒvodom je zlé konfigurácia. Potrebujeme nastaviť na ktorom rozhraní bude fungovať DHCP server. Urobíme to pridaním riadka `INTERFACES="eth0s8"` v súbore `/etc/default/isc-dhcp-server`. Ak sa predtým nachádzajú iné nastavenia `INTERFACESv4` alebo `INTERFACESv6` zakomentujeme ich. Ak chceme aby DHCP fungovalao na viacerých rozhraniach odvelíme ich medzerou, napr.: `INTERFACES="eth0s8 eth0s9"`.

DHCP stále nepôjde spustiť. Potrebujeme ešte deklarovať podsieť. Urobíme to v konfiguračnom súbore `/etc/dhcp/dhcpd.conf` pridaním nasledovných riadkov na koniec súboru:

```
subnet 192.168.1.0 netmask 255.255.255.0 {
	range 192.168.1.2 192.168.1.100; #nastavienie rozsahu pridelovanych, zaciname od 192.168.1.2 lebo 192.168.1.1 je brana
	option routers 192.168.1.1; #adresa routra
	option subnet-mask 255.255.255.0; #maska podsiete
	option broadcast-address 192.168.1.255; #broadcastova adresa
}
```
Takýmto nastavením budú pripojeným zariadeniam priraďované adresy z rozsahu 192.168.1.2 až 192.168.1.100. Ak chceme niektorému zariadeniu priradiť pevnú ip adresu (napríklad tlačiarni) pridáme do časti subnet na pridáme:

```
host laser-printer-lex1 {
	hardware ethernet 08:00:2b:4c:a3:82; #konkretna MAC adresa tlaciarne
        fixed-address 192.168.1.120; #priradena pevna adresa
}
```

V danom súbore môžeme ešte nastaviť:

* `option domain-name "kriza.org"`, nastavenie doménoveho mena v mojom prípade na kriza.org
* `option domain-name-servers 8.8.8.8;` nastavenie DNS serverna v mojom prípade googlovský 
* `max-lease-time 600`, nastavenie ako dlho budeme držat IP pre klientskú stanicu
* `authoritative`, nastavenie, že by sme mali byť oficialny/jediný DHCP server v tejto sieti

Ak chceme zobraziť priradené IP adresy použijeme príkaz:

> cat /var/lib/dhcp/dhcpd.leases

Chybové hlášky DHCP servera vieme vypísať príkazom:

> tail /var/log/syslog

Máme nastavený DHCP server. Ale na klientovi máme nastavenú statickú IP adresu. Ak chceme aby klient dostával IP adresu od DHCP servera musím vrátiť nastavenie v súbore /etc/network/interfaces na pôvodné:

```
auto enp0s3
iface enp0s3 inet dhcp
```
