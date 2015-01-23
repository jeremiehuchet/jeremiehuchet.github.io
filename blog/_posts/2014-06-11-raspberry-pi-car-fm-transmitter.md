---
layout: post
title: 'Boite à musique avec un Raspberry PI'
meta:
 keywords: debian, raspbian, raspberry pi, howto, mpd, mpc, music player daemon, fm transmitter
---

Voici une petite solution pour utiliser le Raspberry PI comme boite à musique. Personnellement je compte l'installer dans ma voiture :)

### l'idée

Le Raspberry PI héberge la musique et un démon [mpd](http://fr.wikipedia.org/wiki/Music_Player_Daemon) permet de la diffuser vers l'autoradio grâce à [pifm](http://www.icrobotics.co.uk/wiki/index.php/Turning_the_Raspberry_Pi_Into_an_FM_Transmitter). Pour contrôler la lecture de la musique, il existe de nombreux clients pour _mpd_ : Android, web, ligne de commande, ... Le client doit communiquer avec le démon donc on met en place un hotspot WiFi sur le Raspberry PI sur lequel n'importe quel smarphone pourra se connecter.

### le matériel

- un Raspberry PI
- un fil de 10cm pour l'antenne
- une clé USB WiFi qui supporte le mode <span title="Access Point">_AP_</span>

#### installation de raspbian

<span class="pull-right badge badge-info">[source](http://www.raspberrypi.org/documentation/installation/installing-images/README.md)</span>

Téléchargement et création d'une carte SD avec [Raspbian](http://www.raspbian.org/)

    wget -0 raspbian.img http://downloads.raspberrypi.org/raspbian_latest
    dd bs=4M if=raspbian.img of=/dev/mmcblk0

#### configuration de l'adaptateur WiFi

J'utilise une clé USB WiFi à base de Intersil ISL3887 et du coup j'ai suivi le [wiki Debian](https://wiki.debian.org/fr/prism54#p54usb).

    wget -O /lib/firmware/isl3887usb https://daemonizer.de/prism54/prism54-fw/fw-usb/2.13.25.0.lm87.arm --no-check-certificate
    modprobe -r p54usb ; modprobe p54usb

#### configuration d'un hotspot WiFi

<span class="pull-right badge badge-info">[source](http://elinux.org/RPI-Wireless-Hotspot)</span>

Installation des logiciels nécessaires :

    sudo apt-get install hostapd udhcpd

##### udhcpd

On édite le fichier de configuration de `udhcpd`, `sudo vim /etc/udhcpd.conf` :

    start 192.168.1.2
    end 192.168.1.20
    interface wlan0
    remaining yes
    opt dns 192.168.1.1
    opt subnet 255.255.255.0
    opt router 192.168.1.1
    opt lease 86400 # 1 day DHCP lease

Il faut ausi l'activer explicitement dans `/etc/default/udhcpd` en commantant la ligne `DHCPD_ENABLED="no"` :

    #DHCPD_ENABLED="no"

Le Raspberry PI doit avoir une adresse IP statique, onpeut ajouter dans `/etc/network/interfaces` :

    iface wlan0 inet static
      address 192.168.1.1
      netmask 255.255.255.0

On démarre le service et on l'active au démarrage :

    sudo service udhcpd start
    sudo update-rc.d udhcpd enable

##### hostapd

On créé un fichier de configuration pour le réseau WiFi à créer, `sudo vim /etc/hostapd/music.conf` :

    interface=wlan0
    driver=nl80211
    ssid=music
    hw_mode=g
    channel=6
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=secret_music_box
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP

Dans `/etc/default/hostapd` on précise la configuration à charger : 

    DAEMON_CONF="/etc/hostapd/music.conf"

On démarre le service et on l'active au démarrage :

    sudo service hostapd start
    sudo update-rc.d hostapd enable


#### installation de pifm

    sudo mkdir /opt/pifm
    sudo chown pi:pi /opt/pifm
    cd /opt/pifm
    wget http://omattos.com/pifm.tar.gz
    tar zxvf pifm.tar.gz

pifm [nécessite les droits root](https://github.com/rm-hull/pifm#accessing-hardware) pour être exécuté. On configure sudo pour autoriser l'utilisateur MPD à exécuter pifm en root sans avoir besoin de mot passe.
Il faut ajouter par exemple la lighe suivante dans `/etc/sudoers` :

    mpd ALL=(ALL) NOPASSWD: /opt/pifm/pifm

#### installation de MPD

Installation du démon : `sudo apt-get install mpd`.

Il faut configurer une sortie autio pour transmettre la musique vers [pifm] :

    audio_output {
        type            "pipe"
        name            "FM 103.3"
        command         "sudo /opt/pifm/pifm - 103.3 44100 stereo"
        format          "44100:16:2"
    }

On démarre le service et on l'active au démarrage :

    sudo service mpd start
    sudo update-rc.d mpd enable

Il est possible d'en définir plusieurs afin de pouvoir facilement changer la fréquence sur laquelle [pifm] transmet le son.

