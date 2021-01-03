---
layout: single
comments: true
title:  "Nextcloud eta segurtasuna: Geolokalizazioa"
toc: true
toc_label: "Edukien aurkibidea"
toc_icon: "cog"
tags: Nextcloud segurtasuna firewall geoip geolokalizazioa
classes: wide
---


Aurrez azaldu izan dudan moduan, nire etxeko router-aren alboan bizi den raspberrypi-an jarrita dut gure etxeko Nextcloud zerbitzari txikia.  Zerbitzari honetan gure hodei zerbitzu pribatua dugu, eta gure datuak guk kudeatzen ditugunez, gure datu pertsonalak ez daude denok ezagutzen ditugun atzerriko monopolioen eskuetan.

Gaur egun Internet sarean konektatuta dauden beste zerbitzu guztiak bezala, gure raspberriak erasoak jasatzen ditu. Mundu osotik, bot automatizatuak gure zerbitzari txikian sartzen saiatzen dira, beraz oso garrantzitsua da gure zerbitzaria babestea. Gure zerbitzua denez, gure ardura izango da segurtasuna bermatzea.

Hau egiteko ez dago soluzio bakarra. Hoberena kipularen estrategia jarraitzea da: teknika ezberdinak erabiliz hainbat segurtasun-geruza eraikiko ditugu, denen artean soluzio eraginkor bat osatzeko.

Artikulu honetan Geolokalizazioa erabiliz, Apache web zerbitzaria konfiguratuko dugu gu bizi garen herrialdeko IP publikoetatik datozen eskaerak bakarrik onartu ditzan. Gure Nextcloud zerbitzaria etxekoek bakarrik erabiltzen dugu, eta gu ez gera bizi ez Txinan, ez Errusian, ez Vietnamen, eta ezta ere AEBtan, beraz trafiko guzti hori ez dugu onartuko, eta onartuko ez dugunez, ez diegu aukerarik emango gure zerbitzarian atzitzeko.

Hego Euskal Herrian bizi garenez, Espainiako estatuko IP publikoen trafikoa bakarrik onartuko dugu. Munduko beste edozein lekutatik gure Nextcloud zerbitzarira atzitzeko saiakera guztiak debekatuko ditugu. 

# Maxmind.com-en erabiltzailea sortu eta datubasea deskargatu

Laister konfiguratu dugun apache moduluak **GeoLite2-Country.mmdb** datubasea erabiltzen du. Datubase hau lortzeko, [Maxmind gunean](https://www.maxmind.com/en/geolite2/signup) kontu bat zabaldu beharko dugu, eta horrela Geolite2 datubasea doan deskartzeko aukera izango dugu.

Behin kontua sortuta, gure erabiltzaile izenarekin Maxmid gunean sartu eta **My License Key** atalean **key** edo gako berri bat sortuko dugu. Demagun sortu dugun gakoa *NireSuperGakoa* dela.

Raspberrypi-an datubasea deskargatu eta eguneratzeko, [zuzeneko deskarga bidez](https://dev.maxmind.com/geoip/geoip-direct-downloads/) egin beharko dugu. Deskargarako helbidea hau izango da:

https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=YOUR_LICENSE_KEY&suffix=tar.gz

Kontutan izan URLan YOUR_LICENSE_KEY dagoen lekuan, adibide honetan *NireSuperGakoa* jarri beharko dugula, beraz honela geratuko da:

https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=NireSuperGakoa&suffix=tar.gz

tar.gz fitxategia deskargatu ondoren, deskonprimatu egin beharko degu
```
tar -zxvf GeoLite2-Country_20201208.tar.gz
```

Orain datubasea utziko dugun karpeta sortu eta deskonprimatu dugun datubasea bere lekuan kopiatuko dugu

```
mkdir -p /usr/local/share/GeoIP

cp GeoLite2-Country_20201208/GeoLite2-Country.mmdb /usr/local/share/GeoIP/
```


# Apache modulua deskargatu eta instalatu

[MaxMind DB Apache Module](https://github.com/maxmind/mod_maxminddb) erabiliko dugu lan honetarako. [Azken bertsioa](https://github.com/maxmind/mod_maxminddb/releases/tag/1.2.0) deskargatu (artikulu hau idazterakoan, azken bertsioa 1.2.0 da) eta deskonprimatuko degu. Instalazioa nola egin behar den [hemen azaltzen da](https://github.com/maxmind/mod_maxminddb).

```
root@raspberrypi4:~# wget -c https://github.com/maxmind/mod_maxminddb/releases/download/1.2.0/mod_maxminddb-1.2.0.tar.gz

root@raspberrypi4:~# tar -zxvf mod_maxminddb-1.2.0.tar.gz 
```


Orain sortu den karpetara mugitu eta instalatuko degu

```
root@raspberrypi4:~# cd mod_maxminddb-1.2.0/
```

Aurrez **apache2-dev** eta **libmaxminddb0** paketeak instalatu behar dira

```
root@raspberrypi4:~/mod_maxminddb-1.2.0# apt-get install apache2-dev
root@raspberrypi4:~/mod_maxminddb-1.2.0# apt-get install libmaxminddb0 libmaxminddb-dev
```

Orain konpilatu eta instalatu
```
root@raspberrypi4:~/mod_maxminddb-1.2.0# ./configure
root@raspberrypi4:~/mod_maxminddb-1.2.0# make install
```


Honekin apache modulua instalatu eta gaitu du, apacheko **mods-enabled** karpetan ikus dezakegun moduan

```
root@raspberrypi4:~/mod_maxminddb-1.2.0# ls -l /etc/apache2/mods-enabled/ | grep maxminddb.load
lrwxrwxrwx 1 root root 32 abe 11 21:17 maxminddb.load -> ../mods-available/maxminddb.load
```

Apache berrabiaraziko dugu modulua kargatu dadin

```
root@raspberrypi4:~/mod_maxminddb-1.2.0# systemctl restart apache2
```


Orain gure nextcloud zerbitzariaren konfigurazioa editutatuko dugu: `nano /etc/apache2/sites-enabled/nextcloud-le-ssl.conf` 

```
<Directory /var/www/html/nextcloud/>

MaxMindDBEnable On
MaxMindDBFile COUNTRY_DB /usr/local/share/GeoIP/GeoLite2-Country.mmdb
MaxMindDBEnv MM_COUNTRY_CODE COUNTRY_DB/country/iso_code

SetEnvIf MM_COUNTRY_CODE ^(ES) AllowCountry # Adibidean, Espainiako IP publikoak baimentzen ditugu
Deny from all # Lehenik eta behin, dena debekatu
Allow from env=AllowCountry # AllowCountry aldagaian definitutako estatuen IPak daude, hauek baimentzen ditugu

Allow from 192.168.1.0/24 # Etxeko sare lokala ere baimentzen dugu

Options +FollowSymlinks
AllowOverride All

<IfModule mod_dav.c>
Dav off
</IfModule>

SetEnv HOME /var/www/html/nextcloud
SetEnv HTTP_HOME /var/www/html/nextcloud

</Directory>
```



Azkenik Apache berrabiaraziko dugu
```
/etc/init.d/apache2 restart
```


Apacheko log fitxategian (`/var/log/apache2/error.log`), laster hasiko dira honelako lerroak agertzen

```
[Sat Jan 02 11:22:23.652220 2021] [access_compat:error] [pid 6215] [client 107.181.180.165:8903] AH01797: client denied by server configuration: /var/www/html/nextcloud/
```


Adibidean, ikus dezakegu 107.181.180.165 IPa zuen bezeroaren eskaera atzera bota duela. IP hau nongoa den ikusteko, https://geoip.com/ zerbitzua erabili ondoren, esaten du Amsterdam-ekoa dela, beraz funtzionatzen du !!

# Maxmind datubasearen eguneraketak

Maxmind datubasea noizbehinka eguneratzea gomendagarria da. Hau eskuz egin daiteke, instalazioan ikusi ditugun komandoak erabiliz, edo noizbehinka (astean behin nahikoa litzateke) exekutatuko den script baten bitartez automatizatu daiteke. 

Nik  **/root/scripts** karpetan sortu det **deskargatu-geolite2.sh**  izeneko scripta, eta hau da bere edukia:


```
#!/bin/bash

cd /root/scripts

#Aurreko bertsioa ezabatu

rm -Rf ./GeoLite2_Azkena
rm -Rf ./GeoLite2.tar.gz

#Bertsio eguneratua deskargatu. Gogoan izan NireSuperGakoa testua zure gakoarekin ordeztu behar dezula

curl --output GeoLite2.tar.gz "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=NireSuperGakoa&suffix=tar.gz" 

#Datubasea deskonprimatu

tar -zxvf GeoLite2.tar.gz

#Karpetaren izen zehatza ez dakigunez, izena normalizatu

mv GeoLite2-Country_* GeoLite2_Azkena

#Orain arte lanean izan dugun DBa beste izen batekin gordeku dugu badezpada ere

GAUR=$(date +%Y%m%d)
mv /usr/local/share/GeoIP/GeoLite2-Country.mmdb /usr/local/share/GeoIP/GeoLite2-Country.mmdb.$GAUR

#Orain datubasearen azken bertsioa produkzioan jartzen dugu

cp GeoLite2_Azkena/GeoLite2-Country.mmdb /usr/local/share/GeoIP/

#Aurreko azken 4 bertsioak bakarrik mantenduko ditugu, zaharragoak ezabatuz.

#Deskargatu dugun azken bertsioarekin arazoren bat nabaritzen badugu, beti izango dugu eskura zaharragoren bat.

cd /usr/local/share/GeoIP/
ls -t | tail -n +4 | xargs rm --
```

scriptari exekuzio baimenak emango dizkiogu

```
chmod +x /root/scripts/deskargatu-geolite2.sh
```


eta crontab-ean gehituko dugu: `crontab -e`

```
0 6 * * 1 /root/scripts/deskargatu-geolite2.sh
```


Eta listo, honekin bukatu da. Konfigurazio honi esker, zure zerbitzariaren segurtasuna apurtu eta bere kontrola hartzeko hainbat saiakera (ez denak) ekidin dituzu.

# Letsencryp ziurtagirien eguneraketak

Nextcloud zerbitzaria Letsencrypt ziurtagiriarekin konfiguratuta daukat. Ziurtagiria hiru hilean behin berritu beharra dago, baina hortaz letsencrypt instalatzerakoan sortzen det cron scripta arduratzen da.

Kontua da, hau egin ahal izateko, Internetetik HTTP eta HTTPS baimenduta egon behar direla, eta letsencrypt zerbitzariak atzerrian daudenez, hemen egin dugunarekin gure ziurtagiriak ez lirateke berrituko.

Hau konpontzeko trikimailua hau litzateke: ziurtagiria berritzeko saiakeraren aurretik geolokalizazioa desgaitu, eta berritu ondoren berriro ere gaitzea. Ikus dezagun nola egin daitekeen.

Lehenik eta behin, Apache konfigurazio karpetara joan eta Nextcloud konfigurazioaren bi kopia egingo ditugu:

```
cd /etc/apache2/sites-available/
cp nextcloud-le-ssl.conf nextcloud-le-ssl.conf.geoip-gaituta 
cp nextcloud-le-ssl.conf nextcloud-le-ssl.conf.geoip-gaitugabe 
```

Orain `nextcloud-le-ssl.conf.geoip-gaitugabe` izenekoa editatu eta bloke hau ezabatuko dugu (geolokalizazioa desgaituz)

```
  MaxMindDBEnable On
  MaxMindDBFile COUNTRY_DB  /usr/local/share/GeoIP/GeoLite2-Country.mmdb
  MaxMindDBEnv MM_COUNTRY_CODE COUNTRY_DB/country/iso_code

  SetEnvIf MM_COUNTRY_CODE ^(ES) AllowCountry
  Deny from all
  Allow from env=AllowCountry
  Allow from 192.168.1.0/24
```

Orain /root/scripts karpetan bi script berri hauek sortuko ditugu:

nano geolokalizazioa-desgaitu.sh
```
#!/bin/bash

cd /etc/apache2/sites-available/
cp nextcloud-le-ssl.conf.geoip-gaitugabe nextcloud-le-ssl.conf
/etc/init.d/apache2 reload
```
nano geolokalizazioa-gaitu.sh

```
#!/bin/bash

cd /etc/apache2/sites-available/
cp nextcloud-le-ssl.conf.geoip-gaituta nextcloud-le-ssl.conf
/etc/init.d/apache2 reload
```
Biei exekuzio baimena eman
```
chmod +x geolokalizazioa-gaitu.sh geolokalizazioa-desgaitu.sh
```

Ondoren, letsencrypt ziurtagiria berritzeko programazioa aldatuko dugu `/etc/cron.d/certbot` fitxategia editatuz. --pre-hook eta --post-hook parametroak erabiliz, esango diogu ziurtagiria berritu beharra dagoenean, hasi aurretik geolokalizazioa desgaitzeko, eta bukatu ondoren berriro gaitzeko.

Lerroaren kopia bat egingo dugu, eta jatorrizkoa komentatuta utziko dugu. 

```
#0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot -q renew
0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot -q renew --pre-hook /root/scripts/geolokalizazioa-desgaitu.sh --post-hook /root/scripts/geolokalizazioa-gaitu.sh
```


# Artikulu honi buruzko zalantzak, galderak eta iruzkinak

[Hemen idatzi zure iruzkina](https://lemmy.eus/post/122)
