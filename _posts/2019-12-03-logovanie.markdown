---
layout: post
title:  logovanie
date:   2019-12-03
image:  logovanie.jpg
tags:   [AOS, RSYSLOG]
---

Mať logovací server (posielať logy z viacerých strojon na centralizované miesto), nemusí byť na škodu. 
Nastavme si teda logovací server pomocou Rsyslogu.

### [Rsyslog](https://www.rsyslog.com/)

Na serveri upravíme konfigurák `/etc/rsyslog.conf` nasledovne:

* najprv nastavíme ako a na akom porte bude server počúvať (v našom prípade UDT aj TCP na štandardnom porte):

```
# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")
# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
```

* následne nastavíme v akom formáte sa budú logy ukladať:

```
$template IpTemplate,"/var/log/%FROMHOST-IP%/syslog.log"
*.* ?IpTemplate
& ~
```
Už len 1 raštart služby:

> sudo systemctl restart rsyslog


Na klientovi nastavíme:


### Odkazy:

[How to Setup Central Logging Server with Rsyslog in Linux](https://www.tecmint.com/install-rsyslog-centralized-logging-in-centos-ubuntu/)

[Dokumentácia 5](https://linux.die.net/man/5/rsyslog.conf)
[Dokumentácia 8](https://linux.die.net/man/8/rsyslogd)
