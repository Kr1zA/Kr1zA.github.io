---
layout: post
title:  Linux NAT Router
date:   2019-10-27
tags:   AOS, NAT
---
## 1. Linux NAT Router

Konfigurovať NAT server budeme na Debiane 9 nainstalovanom vo Virtualboxe. Nainštalujeme si 2 virtuálne stroje, kde jeden bude slúžiť ako brána/NAT server a druhý ako klient na ktorom budeme testovať, či nastavená konfigurácia na NAT serveri funguje. 

Na NAT serveri nastavime v nastaveniach virtuálky 2 sieťové adaptéry (jeden potrebujeme na dotiahnutie internetu zvonku - WAN a jeden pre lokálnu siet LAN). Jeden bude nastavený na NAT aby sme dostali IP addresu z voknu z DHCP. Druhý bude nastaveny na Internal Network teda pre nás LAN. Na druhom klientskom stroji nastavíme jednu sieťovku na Internal Network.

Po zapnutí oboch strojov si možeme príkazom `ip a` poziriet dostupné sieťové adaptéry. Na serveri uvidíme 3 adaptéry, v mojom prípade: lo - loopback, enp0s3 ktorý má pridelenú adresu od DHCP, teda je to WAN a enp0s8, ktorý má status DOWN, ktorý bude náš adaptér do lokálnej siete. Na klientovi uvidíme 2 adaptéry, v mojom prípade: lo - loopback a enp0s3, ktorý je tiež DOWN.

Jednotlivé sieťové rozhrania vieme zapínať/vypínať príkazom `ip link set dev ethX up/down` je sieťové rozhranie, napríklad eth0s8.

Teraz potrebujeme nastaviť sietové rozhrania na serveri. Konfiguračný súbor je `/etc/network/interfaces`. Upravíme konfigurák tak, aby vyzeral nasledovne (v komentároch vysvetlenia):

```
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
auto enp0s3  #pridame, aby sa interface zaplo po reštartovaní sieťových rozhraní
iface enp0s3 inet dhcp

# LAN interface
auto enp0s8 #pridame, aby sa interface zaplo po reštartovaní sieťových rozhraní
iface enp0s8 inet static #nastavime aby bola staticka ip a v dalších riadkoch nastavime parametre
	address 192.168.1.1
	netmask 255.255.255.0
	network 192.168.1.0
	broadcast 192.168.1.255
```

Teraz potrebujeme reštartovať sieťové rozhrania príkazom:

> systemctl restart networking

Týmto máme nastavené rozhranie pre LAN na bráne.

Ak chceme vedieť s klientským počítačom komunikovať s bránou potrebujeme tiež upraviť konfiguračný súbor `/etc/network/interfaces` aby vyzeral nasledovne:

```
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
auto enp0s3  #pridame, aby sa interface zaplo po reštartovaní sieťových rozhraní
iface enp0s3 inet static #nastavime aby bola staticka ip a v dalších riadkoch nastavime parametre
address 192.168.1.2
netmask 255.255.255.0
gateway 192.168.1.1
broadcast 192.168.1.255
```

Teraz už vieme komunikovať medzi týmito dvoma strojmi, čo vieme skontrolovať napriklad ningnutím. Problém ale nastane, ak chceme pristupovať na internet. Preto potrebujeme nastaviť na bráne smerovanie. To vieme dočastne (do najbližsieho reštartu) urobiť príkazom `sysctl -w net.ipv4.ip_forward=1`. Pre trvalé zapnutie smerovania je potrebné pridať na koniec konfiguračného súboru `/etc/systcl.conf` riadok `net.ipv4.ip_forward=1`. Následne aplikujeme nastavenie príkazom `sysctl -p /etc/sysctl.conf`. 

Smerovaciu tabuľku si pre kontorlu vieme zobraziť príkazom `ip route show`.

Síce už máme nastavené smerovanie, na klientovi stále nemáme prístup na internet. Potrebujeme ešte povoliť natovanie. Urobíme to nastavením iptables a to príkazmi:

```
iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE
iptables --append FORWARD --in-interface eth1 -j ACCEPT
```
Ak sme náhodou ničo urobili zle vieme nastavenie zahodit príkazom `iptables -F`

Ale toto nastavenie ostáva len do reštartovania. Automatické nastavenie po reštarte vieme nastavit nasledovne:

* Aktuálnu konfiguráciu iptables si uložíme do súboru `/etc/iptables.rules` príkazom: 

> iptables-save > /etc/iptables.rules

* Aby sa táto uložená konfigurácia načítala po reštarte pridáme do `/etc/network/interfaces` hneď pod riadok `iface lo inet loopback` nasledovný riadok:

> pre-up iptables-restore < /etc/iptables.rules
