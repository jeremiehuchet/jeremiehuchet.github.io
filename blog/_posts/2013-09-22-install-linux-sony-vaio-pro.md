---
layout: post
title: Linux sur Sony Vaio Pro 13
meta:
 keywords: sony, vaio, pro, 13, efi, uefi, installation, linux, arch, boot, grub
---

Petit retour d'expérience sur l'installation de Arch Linux sur le Sony Vaio Pro 13. De manière générale, tout - ou presque - fonctionne bien du premier coup. LE truc qui m'a fait galérer c'est l'UEFI.

Je pense avoir essayé de faire les choses dans les règles mais ça n'a pas abouti. À titre d'information voici ce que j'ai tenté.

### partitions

À l'origine le SSD est partitionné en mode GPT avec 6 paritions : 
<table class="table">
  <tr>
    <th>device</th>
    <th>taille</th>
    <th>type</th>
    <th>description</th>
  </tr>
  <tr>
    <td>/dev/sda1</td>
    <td>260Mo</td>
    <td>EFI</td>
    <td>SONYSYS ???</td>
  </tr>
  <tr>
    <td>/dev/sda2</td>
    <td>1.4Go</td>
    <td>Basic</td>
    <td>Windows tools</td>
  </tr>
  <tr>
    <td>/dev/sda3</td>
    <td>260Mo</td>
    <td>EFI</td>
    <td>Semble héberger le programme de démarrage windows</td>
  </tr>
  <tr>
    <td>/dev/sda4</td>
    <td>128Mo</td>
    <td>Microsoft reserved</td>
    <td>???</td>
  </tr>
  <tr>
    <td>/dev/sda5</td>
    <td>212Go</td>
    <td>Basic</td>
    <td>Partition windows</td>
  </tr>
  <tr>
    <td>/dev/sda6</td>
    <td>23.8Go</td>
    <td>Basic</td>
    <td>Partition de restauration</td>
  </tr>
</table>

Avant l'installation J'ai redimensionné la partition `/dev/sda5` pour faire un peu de place à une nouvelle partition.

<table class="table">
  <tr>
    <th>device</th>
    <th>taille</th>
    <th>type</th>
    <th>description</th>
  </tr>
  <tr>
    <td>/dev/sda7</td>
    <td>153Go</td>
    <td>LVM</td>
    <td>Linux</td>
  </tr>
</table>

### installation arch linux

J'ai suivi le guide d'installation Arch Linux sans aucune complication durant l'installation.
J'ai installé Grub2 après avoir monté la partition `/dev/sda3` dans `/boot/efi`.

### boot et uefi

#### fail

J'ai tenté de jouer avec `efibootmgr` pour ajouter une entrée. 

    $ efibootmgr -v
    BootCurrent: 0007
    Timeout: 0 seconds
    BootOrder: 0005,0007,0008,0009,0001,000A
    Boot0001* Windows Boot Manager  HD(3,363800,82000,9d44d229-3120-440e-aaab-feafe27db15e)File(\EFI\Microsoft\Boot\bootmgfw.efi)WINDOWS.........x...B.C.D.O.B.J.E.C.T.=.{.9.d.e.a.8.6.2.c.-.5.c.d.d.-.4.e.7.0.-.a.c.c.1.-.f.3.2.b.3.4.4.d.4.7.9.5.}...\................
    Boot0005* Sony Original HD(1,800,82000,43ed05b6-502f-4166-bb93-4f4b01fa154f)File(\EFI\Microsoft\Boot\bootmgfw.efi)
    Boot0007* Windows Boot Manager  HD(3,363800,82000,9d44d229-3120-440e-aaab-feafe27db15e)File(\EFI\Boot\bootx64.efi)
    Boot0008* Windows Boot Manager  HD(5,425800,1a939000,e982cb93-f4b3-477d-b3cc-936586a030a7)File(\EFI\Boot\bootx64.efi)
    Boot0009* Windows Boot Manager  HD(5,425800,6ec1000,e982cb93-f4b3-477d-b3cc-936586a030a7)File(\EFI\Boot\bootx64.efi)
    Boot000A* Windows Boot Manager  HD(5,425800,7555000,e982cb93-f4b3-477d-b3cc-936586a030a7)File(\EFI\Boot\bootx64.efi)

C'est là que les choses se sont compliquées :

1. Je n'ai jamais vu aucun menu me proposant les options précédentes. J'ai donc commencé par modifier le timeout du menu avec `efibootmgr -t 5`. Aucun menu ne s'est affiché au démarrage.
2. J'ai quand même essayé d'ajouter une entrée dans le menu et de modifier le _boot order_. Non seulement mon entrée n'était pas utilisée au démarrage suivant, mais en plus elle disparaissait.

#### solution

Au final j'ai remplacé le fichier `/EFI/Boot/bootx64.efi` de la partition `/dev/sda3` par le fichier `grubx64.efi` généré lors de l'installation de grub.

    $ mount /dev/sda3 /boot/efi
    $ cp /boot/efi/EFI/arch_grub/grubx64.efi /boot/efi/EFI/Boot/bootx64.efi

Dorénavant c'est grub qui apparait au lieu de windows 8. 
Il suffit ensuite d'ajouter la configuration suivante dans `/etc/grub.d/40_custom` pour ajouter une entrée pour booter sous windows 8 :

    #!/bin/sh
    exec tail -n +3 $0
    # This file provides an easy way to add custom menu entries.  Simply type the
    # menu entries you want to add after this comment.  Be careful not to change
    # the 'exec tail' line above.

    menuentry "Microsoft Windows 8" {
            insmod part_gpt
            insmod fat
            insmod search_fs_uuid
            insmod chain
            search --fs-uuid --set=root --hint-bios=hd0,gpt3 --hint-baremetal=ahci0,gpt3 5044-08C1
            chainloader /EFI/Microsoft/Boot/bootmgfw.efi
    }

Le dernier paramètre de la ligne search est obtenu avec la commande `blkid /dev/sda3`.

### liens

* [Page Sony Vaio Pro sur le wiki ArchLinux.org](https://wiki.archlinux.org/index.php/Sony_Vaio_Pro_SVP-1x21)
* [Post de blog sur le sujet](http://elouisyoung.blogspot.fr/2013/07/configuring-2013-sony-vaio-pro-13-with.html)
