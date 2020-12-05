---
layout: single
comments: true
title:  "Scalelite BigblueButton karga banatzailea"
toc: true
toc_label: "Edukien aurkibidea"
toc_icon: "cog"
tags: scalelite bigbluebutton karga-banatzailea
classes: wide
---

Artikulu honetan ikusiko dugunez, BigBlueButton eskalatu ahal izango dugu ehundaka erabiltzaile konkurrenteri zerbitzua emateko.

# Sarrera

Aurrez ikusi izan degu nola instalatzen den BigBlueButton (aurrerantzean BBB) zerbitzari bat. 

BBB zerbitzari bat nahikoa da batera (konkurrentzian) 175 erabiltzaileri zerbitzua emateko, beti ere denek ez badute batera bideokamera aktibatzen.

Erakunde edo ikastetxe txiki baten kasuan, zerbitzari bakarra nahikoa izan daiteke, baina ikastetxe ertain edo handi batean kopuru horiek gainditu daietezke, batez ere konfinamendu eta pandemia garai hauetan.

Karga handia izango dugula aurreikusten dugunerako, oso interesgarria da karga banatzaile (load balancer) bat jartzea. Gure kasuan  [Scalelite](https://github.com/blindsidenetworks/scalelite) Scalelite karga banatzaile nola instalatzen den ikusiko degu.


![Scalelite karga banatzailea](https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/images/scalelite.png  "Scalelite karga banatzailea")

Ideia nagusia hau da:
- Scalelite izango da gure karga banatzailea
- Scaleliten atzetik hainbat (N) BBB zerbitzari jarriko ditugu. Post honetan azalduko dugun adibidean 3 BBB zerbitzari konfiguratuko ditugu, baina gehiago ere jarri daitezke.
- Moodlekin batera erabiltzen badugu, Moodle konfigurazioan Scalelite zerbitzaria konfiguratuko dugu BBB zerbitzari bezela.
- BBB ezberdinetan egiten diren saioen grabaketak, Scalelite zerbitzarian batuko dira.

Hauek izango dira zerbitzarien datuak:
- scalelite: scalelite.niredomeinua.eus , IP: 192.168.1.10
- BBB1: bbb1.niredomeinua.eus , IP: 192.168.1.11
- BBB2: bbb2.niredomeinua.eus , IP: 192.168.1.12
- BBB3: bbb3.niredomeinua.eus , IP: 192.168.1.13
- COTURN: coturn.niredomeinua.eus 


Sarrera honen ondoren, abisua: Post hau luze xamarra atera da, baina prozesu guztia pausoz pauso azaltzen da, beraz hemen azaltzen den guztia egiten badezu, zure Scalelite zerbitzaria prestatu ahal izango dezu.

# TURN zerbitzaria

Etxeetako Internet konexioetan ez da honelako arazorik egoten, baina NAT aplikatzen den konexioetan, suhesi edo Firewall baten babespean dauden konexioetan, arazoak sortu daitezke ahotsarekin. Hau konpontzeko TURN zerbitzariak erabiltzen dira.

Sarean aurkitu daitezke doako TURN zerbitzariak, baina hor zalantzak sortzen dira: seguruak al dira ? 

Zure TURN zerbitzaria instalatzea oso erreza da,  [hemen azaltzen da nola egiten den](https://docs.bigbluebutton.org/2.2/setup-turn-server.html) . Posible badezu, zure TURN zerbitzaria instalatu eta erabiltzea aholkatuko nizuke. Posible ez bada, saiatu zaitez konfidantza ematen dizun doako TURN zerbitzari bat aurkitzen.

TURN zerbitzariaren IP helbidea eta sekretua (secret) beharko dituzu. Ondo apuntatu, laister beharko dugu eta.

# NFS Zerbitzua SCALELITE zerbitzarian

Hasieran azaldu den moduan, gure instalazioan hainbat BBB zerbitzari izango ditugu. Erabilera arrunt batean, BBB zerbitzari bakoitzak bere grabazioak bere disko lokalean gordetzen ditu, baina guk Scalelite zerbitzarian izan nahi ditugu grabazio guztiak, gero Moodlek jarduera ezberdinen grabaketa guztiak SCALELITE zerbitzariari eskatuko dizkiolako, berak ez bait daki SCALELITE-n atzean beste BBB zerbitzari batzuk lanean dauzkagula.

Hau lortzeko, SCALELITE zerbitzarian NFS zerbitzua jarriko dugu martxan, eta BBB zerbitzari bakoitzean sortzen diren grabazioak, NFS karpeta honetan kopiatuko/sinkronizatuko dira.

[SCALELITE-ko Dokumentazioan dioena jarraituko dugu](https://github.com/blindsidenetworks/scalelite/blob/master/sharedvolume-README.md).

Grabazio asko egiten badira, azkenean diskoan lekua okupatuko dutenez, SCALELITE zerbitzariari bigarren disko bat jartzea erabaki dugu. **fdisk** komandoa erabiliz partizioa sortu dugu eta ext4 formatua eman ondoren, **/etc/fstab** editatu eta ondorengo lerroa gehitu diogu zerbitzariak arrankatu bakoitzean diskoa muntatu dadin.

>Oharra: **ls -l /dev/disk/by-uuid** komandoak esango digu zein den sortu dugun diskoaren uuid-a, ondoren jartzen dudanak ez dizu balioko.

	#### /dev/disk/by-uuid/d6205bd6-f03b-466e-bcf0-bc24731c43ae /mnt/ ext4 defaults 0 0

Orain NFS zerbitzaria instalatuko dugu eta **/mnt/scalelite-recordings** karpeta prestatuko degu grabazioak gordetzeko

	mkdir /mnt/scalelite-recordings
	apt install nfs-kernel-server
	
Orain, /etc/exports fitxategia editatu eta BBB1, BBB2 eta BBB3 zerbitzarien IP helbideekin konfiguratuko dugu. Honekin, BBB zerbitzariek **/mnt/scalelite-recordings** karpeta muntatzeko baimena izango dute

	nano /etc/exports
	
	/mnt/scalelite-recordings 192.168.1.11(rw,all_squash,sync,no_subtree_check)  192.168.1.12(rw,all_squash,sync,no_subtree_check)  192.168.1.13(rw,all_squash,sync,no_subtree_check)
	

Ulertu beharra dago NFS fitxategien baimenak UID/GID balioetan oinarritzen direla, ez erabiltzaile izen eta taledeetan. Ondorioz, garrantzitsua da erabiltzaile eta taldeein UID/GID zenbaki egokiak ezartzea. 

BBB instalazio berri batean, "*bigbluebutton*" erabiltzaileak UID 997 izan beharko luke, baina ezin da ziurtzat hartu. Hau dela eta, eta arazoak ekiditeko, (geroago) BBB zerbitzari bakoitzean talde berri bat sortuko dugu **2000 GID zenbakia** izango duena, eta zenbaki hau erabiliko dugu orain NFS zerbitzariko baimenak ezartzeko.


	# Create the spool directory for recording transfer from BigBlueButton
	mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/spool
	chown 1000:2000 /mnt/scalelite-recordings/var/bigbluebutton/spool
	chmod 0775 /mnt/scalelite-recordings/var/bigbluebutton/spool
	
	# Create the temporary (working) directory for recording import
	mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/recording/scalelite
	chown 1000:1000 /mnt/scalelite-recordings/var/bigbluebutton/recording/scalelite
	chmod 0775 /mnt/scalelite-recordings/var/bigbluebutton/recording/scalelite
	
	# Create the directory for published recordings
	mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/published
	chown 1000:1000 /mnt/scalelite-recordings/var/bigbluebutton/published
	chmod 0775 /mnt/scalelite-recordings/var/bigbluebutton/published
	
	# Create the directory for unpublished recordings
	mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/unpublished
	chown 1000:1000 /mnt/scalelite-recordings/var/bigbluebutton/unpublished
	chmod 0775 /mnt/scalelite-recordings/var/bigbluebutton/unpublished


>OHARRA: Teorian hau ez da beharrezkoa, baina baimen arazoak izan ditut spool karpetan, BBB zerbizariek ezin zuten bertan idatzi,  eta arazoa konpontzeko hau egin det:
>
>	chmod -Rf 0777 /mnt/scalelite-recordings/var/bigbluebutton/spool
>

Honekin SCALELITE zerbitzarian NFS karpeta sortu eta baimen egokiak jarri ditugu. Ondoren BBB zerbitzarietan karpeta hau muntatuko dugu

# BBB1, BBB2 eta BBB3 zerbitzarietan NFS karpeta muntatu eta BBB instalazioa egin

Ondorengo komandoak hiru zerbitzarietan egin behar dira. Hau da zerbitzari bakoitzean egingo duguna:
- nfs-common paketea instalatu
- /mnt/scalelite-recordings karpeta sortu eta bertan muntatu NFS zerbitzariko  /mnt/scalelite-recordings karpeta
- fstab zerbitzaria konfiguratu karpeta hau zerbitzaria piztu bakoitzean muntatu dadin
- BBB instalazioa egin
- Lehenago aipatu moduan, 2000 GID duen taldea sortu, eta bertan sartu "bigbluebutton" erabiltzaile lokala

Instalazioa egiterakoan, COTURN zerbitzaria zein den esango diogu, eta bere secret zein den ere bai. Zure COTURN zerbitzaria instalatu badezu, ***secret*** zein den jakiteko, COTURN zerbitzariko ***/etc/turnserver.conf*** fitxategian ***static-auth-secret*** aldagaian begiratu.

Ordu erdi inguru beharko da zerbitzari bakoitzean instalazioa burutzeko, aprobetxatu tarte hau kafe bat hartzeko :-)

BBB1 zerbitzarian: 

	apt install nfs-common
	mkdir /mnt/scalelite-recordings
	
	nano /etc/fstab
	
	scalelite.niredomeinua.eus:/mnt/scalelite-recordings        /mnt/scalelite-recordings   nfs defaults          0       0
	

	wget -qO- https://ubuntu.bigbluebutton.org/bbb-install.sh | bash -s -- -v xenial-22 -s bbb2.niredomeinua.eus -e nirehelbidea@niredomeinua.eus -c turn.niredomeinua.eus:NIRETURNSEKRETUA

NFS baimenentzako 2000 GID taldea sortu eta bigbluebutton erabiltzaile lokala hor sartu

	groupadd -g 2000 scalelite-spool
	usermod -a -G scalelite-spool bigbluebutton
	

BBB2 zerbitzarian: 


	apt install nfs-common
	mkdir /mnt/scalelite-recordings
	
	nano /etc/fstab
	
	scalelite.niredomeinua.eus:/mnt/scalelite-recordings        /mnt/scalelite-recordings   nfs defaults          0       0
	
	wget -qO- https://ubuntu.bigbluebutton.org/bbb-install.sh | bash -s -- -v xenial-22 -s bbb2.niredomeinua.eus -e nirehelbidea@niredomeinua.eus -c turn.niredomeinua.eus:NIRETURNSEKRETUA

NFS baimenentzako 2000 GID taldea sortu eta bigbluebutton erabiltzaile lokala hor sartu

	groupadd -g 2000 scalelite-spool
	usermod -a -G scalelite-spool bigbluebutton
	

BBB3 zerbitzarian: 

	apt install nfs-common
	mkdir /mnt/scalelite-recordings
	
	nano /etc/fstab
	
	scalelite.niredomeinua.eus:/mnt/scalelite-recordings        /mnt/scalelite-recordings   nfs defaults          0       0
	
	wget -qO- https://ubuntu.bigbluebutton.org/bbb-install.sh | bash -s -- -v xenial-22 -s bbb3.niredomeinua.eus -e nirehelbidea@niredomeinua.eus -c turn.niredomeinua.eus:NIRETURNSEKRETUA

NFS baimenentzako 2000 GID taldea sortu eta bigbluebutton erabiltzaile lokala hor sartu

	groupadd -g 2000 scalelite-spool
	usermod -a -G scalelite-spool bigbluebutton
	

***

Instalazioa bukatu ondoren, komenigarria da BBB zerbitzari bakoitzaren secret-a atera eta nonbaiten apuntatzea, geroago beharko dugu. Honetarako **bbb-conf --secret** komandoa erabiliko dugu


BBB1 zerbitzarian: 

	bbb-conf --secret
	
	    URL: https://bbb1.niredomeinua.eus/bigbluebutton/
	    Secret: BBB1-SEKRETUA
	    
	    
BBB2 zerbitzarian: 

	bbb-conf --secret
	
	    URL: https://bbb2.niredomeinua.eus/bigbluebutton/
	    Secret: BBB2-SEKRETUA
	    
	    
BBB3 zerbitzarian: 

	bbb-conf --secret
	
	    URL: https://bbb3.niredomeinua.eus/bigbluebutton/
	    Secret: BBB3-SEKRETUA
	

***

# BBB Zerbitzariak konfiguratu grabazioak automatikoki SCALELITE zerbitzarira kopiatu daitezen

Berez, BBB bakoitzean egingo diren grabazioak, zerbitzari bakoitzean geratzen dira, baina guk SCALELITE zerbitzarian izan nahi ditugu. Honetarako, [dokumentazioan azaltzen diren pausoak](https://github.com/blindsidenetworks/scalelite/blob/master/bigbluebutton/README.md)  jarraituko ditugu. BBB zerbitzari bakoitzean hau egin

	cd /usr/local/bigbluebutton/core/scripts/

scalelite.yml fitxategia sortu

	nano scalelite.yml 

eta barruan ondorengo lerroak kopiatu

	 # Local directory for temporary storage of working files
	work_dir: /var/bigbluebutton/recording/scalelite
	# Directory to place recording files for scalelite to import
	spool_dir: /mnt/scalelite-recordings/var/bigbluebutton/spool

Azkenik **post_publish** karpetara mugitu eta hor **scalelite_post_publish.rb** scripta deskargatu. Script hau arduratuko da BBB zerbitzari bakoitzean grabaketa bat egiten denean NFS karpetara kopiatzea. Honek ez ditu BBB zerbitzarietatik grabaketak kenduko, nahi izanez gero hori ere automatizatu daiteke.

	cd /usr/local/bigbluebutton/core/scripts/post_publish
	wget -c https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/bigbluebutton/scalelite_post_publish.rb

# SCALELITE karga banatzailearen instalazioa

Orain arte egin dugunarekin, BBB zerbitzariak instalatu ditugu eta NFS karpeta prestatu dugu, baina SCALELITE zerbitzua instalatzea falta zaigu.

Instalazio hau egiteko era ezberdinak daude, [SCALELITEko gunean azaltzen dutena](https://github.com/blindsidenetworks/scalelite/) , BigBlueButton-eko gunean ere beste hainbat aukera azaltzen dira [Ansible moduzko tresnekin](https://docs.bigbluebutton.org/2.2/install.html#ansible)  automatizatzen direnak, baina [nik beste bat azalduko det](https://medium.com/@jesusfederico_39370/scalelite-lazy-deployment-745a7be849f6) , niretzat errezena dena.

Instalazioan docker kontainerrak erabiliko direnez, lehenik eta behin docker eta docker-compose instalatuko ditugu SCALELITE zerbitzarian.

	apt-get update
	
	sudo apt-get install \
	    apt-transport-https \
	    ca-certificates \
	    curl \
	    gnupg-agent \
	    software-properties-common
	
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	
	sudo add-apt-repository \
	   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
	   $(lsb_release -cs) \
	   stable"
	
	apt-get update
	
	apt-get install docker-ce docker-ce-cli containerd.io
	
	docker -v 
	
	sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
	
	chmod +x /usr/local/bin/docker-compose
	
	docker-compose --version

Behin docker instalatu dugunean, SCALELITE instalazioarekin hasiko gara. Artikulu honetan egingo dugun instalazioa, hemen azaltzen dena da: [https://jffederico.medium.com/scalelite-lazy-deployment-745a7be849f6](https://jffederico.medium.com/scalelite-lazy-deployment-745a7be849f6) 

Esan beharra dago [egilea](https://github.com/jfederico) BBBko [garapenean inplikatuta dagoen enpresa bateko](https://github.com/blindsidenetworks) langilea dela, beraz tresna honen ezagutza handia du.

Berak azaltzen dituen pausoak jarraituko ditugu:

Instalatzailearen errepositorioa gure SCALELITE zerbitzarian klonatu:

	cd /root
	git clone https://github.com/jfederico/scalelite-run
	
Ondoren scalelite-run karpetan sartu eta **dotenv** fitxategiaren **.env** izeneko fitxategian kopiatu

	cd scalelite-run
	cp dotenv .env

**SECRET_KEY_BASE** eta **LOADBALANCER_SECRET** aldagaietarako pasahitz edo sekretuak sortu behar ditugu, gero .env barruan kopiatzeko.

**SECRET_KEY_BASE**:  hau lortzeko komando hau exekutatu eta balioa apuntatu.

	openssl rand -hex 64

**LOADBALANCER_SECRET**:  hau lortzeko komando hau exekutatu eta balioa apuntatu.

	openssl rand -hex 24

Orain .env fitxategia editatu eta honela utzi



	SECRET_KEY_BASE=  GOIANLORTUDUZUNSECRET_KEY_BASE
	LOADBALANCER_SECRET= GOIANLORTUDUZUNSLOADBALANCER_SECRET
	URL_HOST= scalelite.niredomeinua.eus
	NGINX_SSL=true
	SCALELITE_RECORDING_DIR=/mnt/scalelite-recordings/var/bigbluebutton
	LETSENCRYPT_EMAIL=nirehelbidea@niredomeinua.eus
	
Instalazioaren [ohar honek](https://medium.com/@falahatnejad.work/please-note-that-ce7773103373) dioena jarraituz **docker-compose.yml** editatu eta 127.0.0.0 IPa 127.0.0.1 IParekin ordezkatu det.

	services:
	  postgres:
	    image: postgres:11.5-alpine
	    container_name: postgres
	    restart: unless-stopped
	    ports:
	#      - "127.0.0:5432:5432"
	# https://medium.com/@falahatnejad.work/please-note-that-ce7773103373
	      - "127.0.0.1:5432:5432"
	    volumes:
	      - postgres-data:/var/lib/postgresql/data
	    environment:
	      - POSTGRES_USER=${POSTGRES_USER:-postgres}
	      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-password}

Beste aldaketa bat ere egingo degu. redis zerbitzuaren konfigurazioan ***restart: on-failure*** dago konfiguratuta, eta honen arabera akats bat dagoenean bakarrik berrabiaraziko du. Guk zerbitzari osoa nahita berrabiarazten badugu, akats bezala ez da ikusten, beraz zerbitzaria piztu ondoren redis ez litzateke automatikoki abiatuko. Hau konpontzeko, restart: ***unless-stopped*** jarriko dugu konfigurazioan.

	  redis:
	    image: redis:5.0-alpine
	    container_name: redis
	#    restart: on-failure
	# Aldekata: bestela ekipoa berrabiarazi ondoren redis ez da martxan jartzen
	    restart: unless-stopped
	    ports:
	      - 127.0.0.1:6379:6379
	    volumes:
	      - redis-data:/data

Hau egin ondoren docker-compose erabiliz dena geratu eta berriro martxan jartzen dugu

	docker-compose down
	docker-compose up -d
	

	      	
SCALELITE zerbitzarian HTTPS konfiguratu dugu letsencrypt erabiliz. 

	./init-letsencrypt.sh

Orain zerbitzuak martxan jarriko ditugu

	docker-compose up -d

Eta datubasea hasieratuko dugu

	docker exec -i scalelite-api bundle exec rake db:setup

Ikus dezagun nola dagoen SCALELITE zerbitzaria

	docker exec -i scalelite-api bundle exec rake status

Karga banatzaileari oraindik ez diogu zerbitzaririk gehitu. Bere menpean dituen zerbitzariak komando honekin ikusiko genituzke

	docker exec -i scalelite-api bundle exec rake servers

# SCALELITEn menpe egongo diren BBB zerbitzariak konfiguratu

Orain arte egin dugunarekin, BBB zerbitzariak eta SCALELITE karga banatzailea konfiguratu ditugu, baina oraindik ez dute elkarren berririk, ez daude elkarrekin lanean. Oraintxe konponduko dugu.

SCALELITEri bere menpean dauden BBB zerbitzariak zeintzuk diren esan behar zaio. Zerbitzari bakoitza konfiguratzen dugunean, bere SECRET-a erabili beharko dugu. Hau BBB zerbitzari bakoitza instalatu dugunean apuntatu dugun datua da.

SCALELITEko konfigurazioan gure BBB zerbitzariak gehituko ditugu:

	docker exec -i scalelite-api bundle exec rake servers:add[https://bbb1.niredomeinua.eusbigbluebutton/api/,BBB1-SEKRETUA]
	OK
        id: 5646943a-e70c-4b5d-8010-89bcf436ed05


	docker exec -i scalelite-api bundle exec rake servers:add[https://bbb2.niredomeinua.eusbigbluebutton/api/,BBB2-SEKRETUA]
	OK
        id: f363c7f4-abf6-4bd5-kc22-9ec9033c6e73

	docker exec -i scalelite-api bundle exec rake servers:add[https://bbb3.niredomeinua.eusbigbluebutton/api/,BBB3-SEKRETUA]
	OK
        id: fa01c243-1h74-81b5-9su9-h41614d7h794
	
Zerbitzari bakoitza gehitu ondoren ID bat bueltatu digu. ID hori hurrengo komandoan erabiliko dugu zerbitzaria gaitzeko, datu hauek ere nonbaiten apuntatu.

	docker exec -i scalelite-api bundle exec rake servers:enable[5646943a-e70c-4b5d-8010-89bcf436ed05]
	docker exec -i scalelite-api bundle exec rake servers:enable[f363c7f4-abf6-4bd5-kc22-9ec9033c6e73]
	docker exec -i scalelite-api bundle exec rake servers:enable[fa01c243-1h74-81b5-9su9-h41614d7h794]

SCALELITE komandoz kudeatzen da, hemen adibide batzuk:


Honekin komandoen laguntza ateratzen da: 

	docker exec -i scalelite-api bundle exec rake --tasks

Zerbitzarien egoera ikuseko:

	docker exec -i scalelite-api bundle exec rake status

Zerbitzariak zerrendatzeko:

	docker exec -i scalelite-api bundle exec rake servers

Zerbitzari bat gehitzeko:

	docker exec -i scalelite-api bundle exec rake servers:add[https://bbb1.example.com/bigbluebutton/api/,bbb-secret]

Zerbitzari bat gaitzeko (bere IDa erabiliz)

	docker exec -i scalelite-api bundle exec rake servers:enable[27243e91–35a3–42ee-80a7-bd5980b0728f]

Zerbitzari bat desgaitzeko

	docker exec -i scalelite-api bundle exec rake servers:disable[27243e91–35a3–42ee-80a7-bd5980b0728f]

Erabilgarri dauden zerbitzarietatik atzerako

	docker exec -i scalelite-api bundle exec rake servers:panic[27243e91–35a3–42ee-80a7-bd5980b0728f]

Zerbitzaria SCALELITEtik kentzeko

	docker exec -i scalelite-api bundle exec rake servers:remove[27243e91–35a3–42ee-80a7-bd5980b0728f]

Komando guztiak hemen aurkitu ditzakezu: [https://github.com/blindsidenetworks/scalelite#administration](https://github.com/blindsidenetworks/scalelite#administration) 

# Moodle

MOODLE plugina instalatuta badugu, beste edozein jarduera moduan erabili ahal izango ditugu BBB konferentziak, beraz oso interesgarri eta gomendagarria da gure MOODLE zerbitzarian BBB plugina instalatu eta konfiguratzea.

MOODLEk SCALELITEren datuak beharko ditu

- URL: https://scalelite.niredomeinua.eus/bigbluebutton/api/
- SECRET: GOIANLORTUDUZUNSLOADBALANCER_SECRET

Gogoan izan, SECRET hau **.env** fitxategiko LOADBALANCER_SECRET aldagaian aurkituko dezu.
	
# Greenlight zerbitzaria

Demagun BBB erabili nahi duzula baina Moodlen beharrik izan gabe, hau da Jitsi erabiltzen den moduan erabili nahi duzula: Gela bat sortu, esteka partekatu eta listo. Hau posible al da ? Bai posible da.

Horretarako Greenligth interfazea erabili ahal izango dugu. Ikastetxean ikasleekin Moodle bidez erabili dezakezu, eta bestelako online bilerak egiteko Greenlight bidez.

Hemen azalduko degu nola instalatu eta konfiguratuko dugun Greenlight gure SCALELITE zerbitzua erabiltzeko. Zuk BBB zerbitzari bakar batekin lanean jarri nahi badezu, irakurri beste artikulu hau.

Bi aukera ezberdin: 

- Gure instalazioan 3 BBB zerbitzari dugunez, bat aukeratu eta bertan instalatu Greenlight zerbitzaria, adibidez bbb1.niredomeinua.eus zerbitzarian. Alde ona, zerbitzari bat gutxiago instalatu behar dela. Alde txarra, bbb1.niredomeinua.eus ezin izango duzula itzali mantenu lan bat egiteko, zure erabiltzaileek hori erabiltzen dutelako beraien **gelak** sortu eta erabiltzeko.
- Beste zerbitzari bat instalatzea zerbitzu hau emateko. Zerbitzari txiki batekin nahikoa litzateke, ez baitu lan handirik egingo.

## Docker instalazioa

Docker bidez instalatzen denez, aukeratu dugun zerbitzarian docker instalatu behar dugu:


	apt-get update
	
	sudo apt-get install \
	    apt-transport-https \
	    ca-certificates \
	    curl \
	    gnupg-agent \
	    software-properties-common
	
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	
	sudo add-apt-repository \
	   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
	   $(lsb_release -cs) \
	   stable"
	
	apt-get update
	
	apt-get install docker-ce docker-ce-cli containerd.io
	
	docker -v 
	
	sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
	
	chmod +x /usr/local/bin/docker-compose
	
	docker-compose --version

Orain greenlight karpeta sortuko dugu eta karpeta horretan **.env** fitxategia sortuko dugu docker komando hau erabiliz

	mkdir ~/greenlight && cd ~/greenlight
	docker run --rm bigbluebutton/greenlight:v2 cat ./sample.env > .env
	
## Greenlight instalazioa

Teorian badirudi ez duela BBBkin dependentziarik eta [bera bakarrik instalatu daitekeela](https://docs.bigbluebutton.org/greenlight/gl-customize.html#customizing-greenlight) , baina horrela instalatzen saiatzen bazara, NGINX konfigurazioa jartzerakoan ikusten da BBB instalazioarekin dependentzia duela. Hau dela eta, nik erabaki det BBB instalazioan erabiltzen den scritp berbera erabiltzea Greenlight komando horrekin instalatzeko. 

BBB hori gero ez det ezertarako erabiliko, beraz tontakeria dirudi BBB bat instalatzea justu Greenlight instalatzeko, baina era azkarrena iruditu zait. Azken finean, BBB instalatzeko erabiltzen dugun aginduari **-g** parametroa gehitzea da egin behar den gauza bakarra. Beste hiru BBBak instalatzeko erabili dugun komando berbera erabiliko dugu, gaina -g parametroa gehituz. Zerbitzari honen izena **bbb.niredomeinua.eus** izango da.

	wget -qO- https://ubuntu.bigbluebutton.org/bbb-install.sh | bash -s -- -v xenial-22 -s bbb.niredomeinua.eus **-g** -e nirehelbidea@niredomeinua.eus -c turn.niredomeinua.eus:NIRETURNSEKRETUA

**.env** fitxategian erabiliko ditugun *sekretua* sortuko dugu docker komando honekin

	docker run --rm bigbluebutton/greenlight:v2 bundle exec rake secret

eta emaitza **.env** fitxategiko **SECRET_KEY_BASE** parametroan idatziko dugu.

BIGBLUEBUTTON_ENDPOINT: Gure SCALELITEko URLa, bukaeran **/bigbluebutton/api/ ** jarrita

	BIGBLUEBUTTON_ENDPOINT=https://scalelite.niredomeinua.eus/bigbluebutton/api/

BIGBLUEBUTTON_SECRET: SCALELITEko LOADBALANCER_SECRET parametroan dugun balioa

	BIGBLUEBUTTON_SECRET=GOIANLORTUDUZUNSLOADBALANCER_SECRET

SAFE_HOSTS: bere domeinu izena

	SAFE_HOSTS=bbb.niredomeinua.eus
	
Zure ikastetxe edo erakundeko erabilitzaileek sistema erabili dezaten, bi aukera dituzu. Banaka bere datu basean sartu, edo zure erakundeko LDAP/AD-ko erabilitzaileak erabili. AD-arekin lotura egiteko, parametro hauek bete. Datu hauek adibiderako dira, zuk zure datu propioekin egokitu.

	LDAP_SERVER=ZURE-AD-IP-HELBIDEA
	LDAP_PORT=389
	LDAP_METHOD=plain
	LDAP_UID=sAMAccountName
	LDAP_BASE=ou=Users,dc=niredomeinua,dc=eus
	LDAP_BIND_DN=cn=bind-erabiltzailea,cn=Users,dc=niredomeinua,dc=eus
	LDAP_PASSWORD=bind-pasahitza
	LDAP_ROLE_FIELD=ou

Gerenlight-ek oharrak mail bidez bidaltzea nahi baduzu, parametro hauek egokitu


	ALLOW_MAIL_NOTIFICATIONS=true

	SMTP_SERVER=posta.niredomeinua.eus
	SMTP_PORT=25
	SMTP_DOMAIN=niredomeinua.eus
	SMTP_USERNAME=izena@niredomeinua.eus
	SMTP_PASSWORD=izena-pasahitza

	SMTP_SENDER=izena@niredomeinua.eus
	SMTP_TEST_RECIPIENT=izena@niredomeinua.eus

Edozeinek (zure erakundekoa izan ala ez) sortu dezake erabiltzaile bat zure Greenlight zerbitzarian, zure esku egongo da hau libre uztea edo ez. Hurrengo lerroarekin lortzen duguna zera da, Greenlight administrariak onartu arte, erabiltzaile berriak blokeatuta geratzea. Hau da, inork ezin izango du zure Greenlight zerbitzaria erabili zuk eskaera onartzen ez badiozu.

	DEFAULT_REGISTRATION=approval

Aldaketa guzti hauek egin ondoren, **.env** fitxategian akatsik ez dagoela ziurtatzeko, ondorengo komando hau exekutatuko degu

	docker run --rm --env-file .env bigbluebutton/greenlight:v2 bundle exec rake conf:check

Akatsik ez badago, nginx-eko konfigurazioa sortuko dugu komando honen bitartez

	docker run --rm bigbluebutton/greenlight:v2 cat ./greenlight.nginx | sudo tee /etc/bigbluebutton/nginx/greenlight.nginx
	
Konfigurazioa bere lekuan jarri denez, nginx berrabiaraziko dugu

	systemctl restart nginx
	
**docker-compose.yml** fitxategia komando baten bidez sortuko dugu. Horretarako greenlight karpetan sartu

	cd ~/greenlight

eta komando hau exekutatuko dugu

	docker run --rm bigbluebutton/greenlight:v2 cat ./docker-compose.yml > docker-compose.yml

Greenlight-ek erabiltzen duen PostgreSQL pasahitza sortu eta **.env** fitxategian jartzeko komando hau erabiliko dugu

	export pass=$(openssl rand -hex 8); sed -i 's/POSTGRES_PASSWORD=password/POSTGRES_PASSWORD='$pass'/g' docker-compose.yml;sed -i 's/DB_PASSWORD=password/DB_PASSWORD='$pass'/g' .env
	
Zerbitzuak arrankatuko ditugu

	docker-compose up -d
	

Administratzaile rola izango duen erabiltzailea sortuko dugu. Horretarako komando hau erabiliko dugu, baina *docker-compose up* egin eta gero, itxaron segundu batzuk, docker kontainerra sortzeko denbora izan dezan.

	docker exec greenlight-v2 bundle exec rake user:create["admin-erabiltzailea","nirehelbidea@niredomeinua.eus","ADMIN-ERABILTZAILEAREN-PASAHITZA","admin"]

Honekin Greenlight martxan izango genuke. Bere interfazean sartzeko, https://bbb.niredomeinua.eus**/b** helbidean sartu behar gara. Fijatu ondo, bukaeran dagoen **/b** horrekin, ez da akats bat.

## BBB bakar batetik SCALELITEra migratzen ari bagara, Greenlight datu basea eta grabazioak nola inportatu.

Aurrez ez badezu BBB instalazio isolatu bat produkzioan izan, salto egin hurrengo atalera. Aldiz, SCALELITEra migratu nahi baduzu lehendik erabili izan duzun zerbitzari bakar bateko BBB instalazio bat, eta instalazio horretan Greenlight bidez egindako grabazioak instalazio berrian izan nahi badituzu, hau interesatuko zaizu.

Aurreko pausoan Greenlight huts bat sortu degu, eta hau nahikoa izan daiteke instalazio berri bat denean, baina gure kasuan aurrez ere BBB erabili izan dugu eta Greenlight bidez erabiltzaileek sistema erabili dute. Datu base horretan berreskuratu nahi dugun informazio asko dago, beraz guk orain Greenlight instalazio berriarekin sortu dugun datubase hutsa lehendik dugunarekin ordezkatuko degu.

[Hemen diotenez](https://github.com/bigbluebutton/greenlight/issues/1567) , datu basea kopiatzea ez da nahikoa, greenlight karpeta osoa kopiatu behar da. Hortaz, jatorrizko Greenlight-era joan eta greenlight karpeta osoa konprimatuko dugu tar komandoa erabiliz. Posible bada, hau egin aurretik hobe greenligh docker kontainerra geratzea.

**BBB zaharrean (SCALELITEkin ordeztu nahi dugula)**

	root@bigbluebutton:~/greenlight# docker-compose down
	cd /root
	tar -cvpf greenlight.tar greenlight

**Greenlight berria instalatu dugun zerbitzarian**

Greenlight zerbitzua geratu

	cd greenlight/
	docker-compose down

Badaezpada orain arte genuen instalazioari izena aldatu

	cd /root
	mv greenlight greenlight-jatorrizkoa

BBB zaharrean konprimitu dugun karpeta ekarri eta deskonprimatu

	cd /root
	scp root@bbb-zaharra:root/greenlight.tar .
	tar -xvf greenlight.tar 
	
Karpetan sartu eta **.env** fitxategian SCALELITEko balio egokiak jarri (SECRET_KEY_BASE, BIGBLUEBUTTON_ENDPOINT, BIGBLUEBUTTON_SECRET). Gogoan izan balio hauek **/root/greenlight-jatorrizkoa/.env** barruan ditugula.

Arrankatu greenlight

	cd /root/greenlight
	docker-compose up -d


eta segundu batzuk zai egon ondoren, saiatu web bidez sartzen: **https://bbb.niredomeinua.eus/b**

Ikusiko dugu Greenlight zaharrean zeuden gelak, erabilitzaileak eta beste informazioa, Greenlight berrian ere badugula. Zer falta zaigu ekartzea ? Grabazioak.

Grabazio zaharrak ez dauzkagu SCALELITE zerbitzarian, beraz [dokumentazioan dioten moduan](https://github.com/blindsidenetworks/scalelite/tree/master/bigbluebutton) , BBB zerbitzari zaharreko grabazioak SCALELITE zerbitzarira ekarri behar dira. Hau da BBB zaharrean egin beharko duguna:

### NFS zerbitzua instalatu eta konfiguratu

Lehenik eta behin SCALELITE zerbitzarian instalatu dugun NFS zerbitzuko **/etc/exportfs** editatu eta lerroaren bukaeran gehitu BBB zaharren IP helbidea eta konfigurazioa. Demagun BBB zaharraren IP helbidea hau dela 192.168.1.5

	/mnt/scalelite-recordings 192.168.1.11(rw,all_squash,sync,no_subtree_check)  192.168.1.12(rw,all_squash,sync,no_subtree_check)  192.168.1.13(rw,all_squash,sync,no_subtree_check) **192.168.1.5(rw,all_squash,sync,no_subtree_check)**
	
Eta zerbitzua berrabiarazi

	/etc/init.d/nfs-kernel-server restart
	
Orain BBB zaharrean nfs konfiguratu

	apt install nfs-common
	mkdir /mnt/scalelite-recordings
	
	nano /etc/fstab
	
	scalelite.niredomeinua.eus:/mnt/scalelite-recordings        /mnt/scalelite-recordings   nfs defaults          0       0
	
	
BBB zaharretik grabazio zaharrak transferitzeko egingo duguna zera da, BBB zaharrean /mnt/scalelite-recordings karpeta sortu, besteetan egin degun bezela, eta scalelite.yml eta scalelite_post_publish.rb bere lekuan jarri, besteetan egin degun bezela. 

	cd /usr/local/bigbluebutton/core/scripts/
	nano scalelite.yml 

Eta barruan hau kopiatu 

	work_dir: /var/bigbluebutton/recording/scalelite
	spool_dir: /mnt/scalelite-recordings/var/bigbluebutton/spool

Ondoren

	cd /usr/local/bigbluebutton/core/scripts/post_publish
	wget -c https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/bigbluebutton/scalelite_post_publish.rb
	chmod +x /usr/local/bigbluebutton/core/scripts/post_publish/scalelite_post_publish.rb

Ziurtatu */var/bigbluebutton/recording/scalelite* karpeta existitzen dela, eta bestela sortu eta baimen egokiak jarri *bigbluebutton.bigbluebutton* erabiltzaileari *chmod* batekin.

	ls -l /var/bigbluebutton/recording
	total 88
	drwxr-xr-x   3 bigbluebutton bigbluebutton  4096 mar 15  2020 process
	drwxr-xr-x   3 bigbluebutton bigbluebutton  4096 mar 14  2020 publish
	drwxr-xr-x 190 bigbluebutton bigbluebutton 69632 nov  7 06:25 raw
	drwxr-xr-x   2 bigbluebutton bigbluebutton  4096 nov  6 21:03 scalelite
	drwxr-xr-x   8 bigbluebutton bigbluebutton  4096 mar 15  2020 status

Beste BBB zerbitzarietan honekin lortzen deguna zera da, grabazio berriak sortzen direnean, automatikoki spool karpetara mugitu eta transferitzea, eta gero scalelite zerbitzaria bera arduratzen da DBa eguneratzen. Gure kasuan ez dira grabazio berriak, lehendik daudenak baizik, beraz nolabait ***agindu*** behar zaio lehenik daudenak transferitu behar dituela.

Hau egin ahal izateko ***scalelite_batch_import.sh*** scripta exekutatuko dugu, baina lehenengo, deskargatu egin behar dugu

	cd /usr/local/bigbluebutton/core/scripts
	wget -c https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/bigbluebutton/scalelite_batch_import.sh
	chmod +x scalelite_batch_import.sh  
  chmod +x ./post_publish/scalelite_post_publish.rb

Orain inportazioa exekutatu
	./scalelite_batch_import.sh
	
Honekin orain arteko grabazioak transferitu dira, eta Scaleliteko datubasean behar ziren erregistroak sortu dira, beraz Moodle eta Greenligh berritik ikusgai egongo dira.


# Monitorizazioa Graphana erabiliz

Orain arte ikusi dugunarekin, gure SCALELITE instalazioa martxan dugu. Posible da beraz, Moodle edo Greenlight interfazeetik online bilerak sortu eta erabiltzea. SCALELITEk bere atzean dauden BBB zerbitzarien karga ikusi, eta bilera berriak karga gutxien duen BBB zerbitzarian sortuko ditu.

Honelako instalazioak monitorizatzea oso interesgarria da, eta horrearako Grafana nola erabili dezakegun ikusiko dugu. Hau egin ondoren, web panel batetik begirada batekin gure zerbitzarien karga nolakoa den ikusi ahal izango dugu.

![Grafana bidez gure BBB zerbitzarien karga monitorizatu](https://bigbluebutton-exporter.greenstatic.dev/assets/img_grafana_dashboard_all_servers.png  "Grafana bidez gure BBB zerbitzarien karga monitorizatu")



Instalazioa  [hemen diotena jarraituz](https://bigbluebutton-exporter.greenstatic.dev/)  egingo degu.

- BBB nodo bakoitzean  [BigBlueButton Exporter](https://bigbluebutton-exporter.greenstatic.dev/installation/bigbluebutton_exporter/)  eta [Node_exporter](https://bigbluebutton-exporter.greenstatic.dev/installation/node_exporter/) instalatuko ditugu
- Grafikak ikusiko ditugun zerbitzarian (gure adibidean https://bbb.niredomeinua.eus), [All-In-One Monitoring Stack](All-In-One Monitoring Stack) instalatuko dugu.


## BBB-EXPORTER (BBB nodo bakoitzean)


	mkdir ~/bbb-exporter
	cd bbb-exporter/
	wget -c https://raw.githubusercontent.com/greenstatic/bigbluebutton-exporter/master/extras/docker-compose.exporter.yaml
	
	mv docker-compose.exporter.yaml docker-compose.yaml
	
Atera bbb nodoaren secret-a
  
	bbb-conf --secret

sortu **secrets.env** fitxategia eta hor sartu balioa **API_SECRET** aldagaian

	nano ~/bbb-exporter/secrets.env


	API_BASE_URL=https://bbb1.niredomeinua.eus/bigbluebutton/api/
	API_SECRET=BBB1-SEKRETUA

Eta orain igo zerbitzua

	docker-compose up -d

*htpasswd* komandoa erabili ahal izateko *apache2-utils* instalatu

	apt install apache2-utils

Erabiltzaile izen bat asmatuko dugu eta pasahitza ezarriko diogu. Pasahitz hau ondo apuntatu, laister beharko dugu.

Oharra: BBB nodo guztietan pasahitz berbera jarri. Adibiderako, pasahitz hau jarri diot: **Metrikak-Pasahitza**

	htpasswd -c /etc/nginx/.htpasswd metrikak


NGINX-en gehituko dugun konfigurazioa **/etc/bigbluebutton/nginx/monitoring.nginx** fitxategian sortuko dugu eta ondoren dauden lerroak bertan kopiatu

	nano /etc/bigbluebutton/nginx/monitoring.nginx


	# BigBlueButton Exporter (metrics)
	location /metrics/ {
	    auth_basic "BigBlueButton Exporter";
	    auth_basic_user_file /etc/nginx/.htpasswd;
	    proxy_pass http://127.0.0.1:9688/;
	    include proxy_params;
	}

NGINX berrabiarazi aurretik testatu konfigurazioa ondo dagoela

	nginx -t

eta dena ondo joan bada, berreabiarazi: 

	sudo systemctl reload nginx

Metrikak hemen publikatzen ari da Prometeus formatuan: https://bbb1.niredomeinua.eus/metrics/

## NODE_EXPORTER (BBB nodo bakoitzean)

Orain BBB nodo bakoitzean [hau instalatuko dugu](https://bigbluebutton-exporter.greenstatic.dev/installation/node_exporter/#step-by-step-guide)

	git clone https://github.com/greenstatic/bigbluebutton-exporter.git
	cp -r bigbluebutton-exporter/extras/node_exporter ~/
	cd node_exporter/

	docker-compose up -d

Orain monitoring erabiltzailea sortu eta berarentzat pasahitza sortuko dugu

	htpasswd -c /etc/nginx/.htpasswd monitoring

Oharra: BBB nodo guztietan pasahitz berbera jarri. Adibiderako, pasahitz hau jarri diot: **Metrikak-Pasahitza**

Orain */etc/nginx/sites-available/bigbluebutton* editatu eta azken **location {...}** ondoren, hau kopiatu

	# node_exporter metrics
	location /node_exporter/ {
	    auth_basic "node_exporter";
	    auth_basic_user_file /etc/nginx/.htpasswd;
	    proxy_pass http://127.0.0.1:9100/;
	    include proxy_params;
	}

## Grafikak publikatuko dituen zerbitzarian (adibidean bbb.niredomeinua.eus)

Zerbitzari honetan [All-In-One Monitoring Stack](https://bigbluebutton-exporter.greenstatic.dev/installation/all_in_one_monitoring_stack/)  instalazioa egingo degu. 



	mkdir ~/bbb-monitoring
	cd bbb-monitoring/
	wget -c https://raw.githubusercontent.com/greenstatic/bigbluebutton-exporter/master/extras/all_in_one_monitoring/bbb_exporter_secrets.env
	wget -c https://raw.githubusercontent.com/greenstatic/bigbluebutton-exporter/master/extras/all_in_one_monitoring/docker-compose.yaml
	wget -c https://raw.githubusercontent.com/greenstatic/bigbluebutton-exporter/master/extras/all_in_one_monitoring/prometheus.yaml


example.com erreferentziak aldatu bi fitxategi hauetan:

- bbb_exporter_secrets.env: API_BASE_URL=https://bbb1.niredomeinua.eus/bigbluebutton/api/
- docker-compose.yaml: GF_SERVER_ROOT_URL: "https://bbb.niredomeinua.eus/monitoring"

prometheus.yaml fitxategia editatu, eta **job_name:'bbb'** eta **job_name: 'bbb_node_exporter'** ataletako targets parametroan gure BBB zerbitzari guztiak gehitu

	global:
	  scrape_interval: 15s
	  evaluation_interval: 15s
	  scrape_timeout: 15s

	scrape_configs:
	  - job_name: 'prometheus'
	    static_configs:
	      - targets: ['localhost:9090']

	  - job_name: 'bbb'
	    scrape_interval: 5s
	    scheme: https
	    basic_auth:
	      username: "metrikak"
	      password: "Metrikak-Pasahitza"
	    static_configs:
	    - targets: ['bbb1.niredomeinua.eus','bbb2.niredomeinua.eus','bbb3.niredomeinua.eus']


	  - job_name: 'bbb_node_exporter'
	    metrics_path: '/node_exporter/metrics'
	    params:
	      format: [prometheus]
	    honor_labels: true
	    scheme: https
	    basic_auth:
	      username: "metrikak"
	      password: "Metrikak-Pasahitza"
	    static_configs:
	    - targets: ['bbb1.niredomeinua.eus','bbb2.niredomeinua.eus','bbb3.niredomeinua.eus']



Zerbitzuak arrankatu

	cd ~/bbb-monitoring
	sudo docker-compose up -d

NGINX-en gehituko dugun konfigurazioa sortu eta blokea kopiatu

	nano /etc/bigbluebutton/nginx/monitoring.nginx

	# BigBlueButton monitoring
	location /monitoring/ {
	  proxy_pass http://127.0.0.1:3001/;
	  include proxy_params;
	}

Orain Nginx berrabiaraziko dugu

	/etc/init.d/nginx restart

Eta grafanako interfazean sartuko gara. Horretarako https://bbb.niredomeinua.eus/monitoring/login helbidean sartu. Eskatuko digun erabiltzailea eta pasahitza, biak **admin** dira (esan beharrik ez dago, aldatu eta jarri beste bat seguruagoa)

Orain Grafana konfiguratuko dugu gure zerbitzarien datuak erakutsi ditzan. Honetarako

- Gehitu Prometeus "data source" bat:  **Add Data Source --> Prometeus --> http://localhost:9090**
- Ezkerreko menuan dagoen "+" ikonoaren gainean klikatu, eta **import** aukeratu.
- "Import via panel JSON" testu kutxan kopiatu [helbide honetan](https://raw.githubusercontent.com/greenstatic/bigbluebutton-exporter/master/extras/dashboards/server_instance_node_exporter.json)  aurkituko dugun JSON kodea.
- Berriro ere ezkerreko menuan dagoen "+" ikonoaren gainean klikatu, eta **import** aukeratu.
- "Import via panel JSON" testu kutxan kopiatu [beste helbide honetan](https://raw.githubusercontent.com/greenstatic/bigbluebutton-exporter/master/extras/dashboards/all_servers.json)  aurkituko dugun JSON kodea.

Listo ! Egin dugunarekin gure BBB zerbitzarien karga monitorizatu dezakegu panel txukun eta itxuroso batetik. 

  
# BBB nodoetan konfigurazio pertsonalizatuak

## Diskoa sarriago garbitu

Hau ez da beharrezkoa, baina bai gomendagarria.

Behar ez diren tarteko fitxategien garbiketa sarriago egin dezan,  editatu **/etc/cron.daily/bigbluebutton** eta parametro hauek aldatu

	#history=5
	history=2
	#unrecorded_days=14
	unrecorded_days=3 
	#published_days=14
	published_days=3 
	#log_history=28
	log_history=3 

## Bideoaren kalitatea egokitu

Bideo kalitatearen arabera banda zabalera gehiago edo gutxiago behar da. Ez bakarrik gure zerbitzarian, baita ere etxetik konferentzia jarraitzen ari diren ikasleen konexioan ere. 

Etxean konexio arazoak dituen ikasleak, edo banda zabaleko konexiorik ez eta 3G/4G konexioa erabiltzen ari denak, eskertuko du irakaslearen aurpegia kalitate gutxiagorekin heltzea.

Hau nola egiten den [hemen dago](https://docs.bigbluebutton.org/2.2/customize.html#reduce-bandwidth-from-webcams) dokumentatuta.

/usr/share/meteor/bundle/programs/server/assets/app/config/settings.yml fitxategia editatu eta **cameraProfiles** atalean **low** profila 50eko bitrate-arekin utziko dugu, eta **medium** profila 100eko bitrate-arekin


	    - id: low
	      name: Low
	      default: false
	      bitrate: 50
	    - id: medium
	      name: Medium
	      default: true
	      bitrate: 100


## Konferentzian sartzerakoan, parte hartzaileen mikroa ixilarazi (mute)

Hau praktikoa da, horrela zarata asko kentzen da. Edonola ere, parte-hartzaileek edozein momentutan aktibatu ahal izango dute beraien mikrofonoa (irakasle/moderatzaileak ez baditu mikroak debekatu, noski)

editatu **/usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties** fitxategia eta **muteOnStart** parametroa aldatu

	muteOnStart=true 


## Defektuz, ikasleek irakaslearen kamara **BAKARRIK** ikus dezaten

Ezagutzen ditugun beste konferentzia sistemetan, parte-hartzaile guztiek kamara aktibatzen dutenean, parte-hartzaile bakoitzak beste guztien bideo seinalea jasotzen du. Ondorioz:

- Zerbitzariko lana asko handitzen da. 20 pertsona baleude, sistemak 20*19 bideo transmisio egin beharko lituzke, eta honek zerbitzariari baliabide asko kenduko lizkioke, agian alferrik.
- Parte-hartzaile bakoitzak, beste guztien bideo seinaleak (19) jaso beharko lituzke, eta etxean konexio makala dutenentzako, honek arazo bat suposatzen du, agian alferrik.

BBBn konfigurazio lehenetsiak portaera berbera du, baina konferentzia baten moderatzaile edo irakaslek hau aldatu dezake. Parametro hau aldatzen badu, hau da gertatuko dena:
- Irakasleak ikasle guztien kameren seinalea ikusiko du
- Ikasle bakoitzak, irakaslearen kameraren seinalea bakarrik ikusiko du, nahiz eta entzun, denen ahotsak entzungo dituen.
- Banda zabaleraren erabilera asko jeitsiko da, bai zerbitzarian, eta baita ere parte-hartzaileen etxeetan.

![beste ikusleen kamerak]({{site.url}}/assets/images/bbb-ikusi-beste-ikusleen-kamerak.png  "beste ikusleen kamerak")


Hau oso praktikoa da irakaskuntza ingurunean, baina irakaslea parametro hau aldatzeaz gororatu behar da, eta hau ez da beti honela gertatzen. Posible al da balio hau aktibatuta geratzea balio lehenetsi moduan saio guztietarako ? Bai, ikus dezagun nola konfiguratzen den.

[Hau hemen](https://groups.google.com/g/bigbluebutton-setup/c/2k6Likm1Mms?pli=1) dago dokumentatuta

Editatu **/usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties** fitxategia eta **webcamsOnlyForModerator** parametroa aldatu

	webcamsOnlyForModerator=true

Kontutan izan hau aldatuz gero, denentzako aldatzen dela, eta zure erabiltzaileek hau jakin behar dutela, bestela parekoen arteko bilerak egiten dituztenean (adibidez irakasleen arteko bilerak) ez direlako denak alkar ikusiko. Aldatzea oso erreza da eta beraiek egin dezakete, bileran berriro ere parametro hori aktibatuz.


# Konfigurazio aukera gehiago

[Hemen ikus dezakegu](https://docs.bigbluebutton.org/2.2/customize.html)  konfigurazio beste hainbat aukera nola egokitu daitezkeen

# Artikulu honi buruzko zalantzak, galderak eta iruzkinak

[Hemen idatzi zure iruzkina](https://lemmy.eus/post/23)


