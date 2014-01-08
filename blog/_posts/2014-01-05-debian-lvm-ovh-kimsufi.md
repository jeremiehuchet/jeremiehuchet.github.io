---
layout: post
title: 'Installer Debian avec LVM2 sur un serveur dédié OVH Kimsufi'
meta:
 keywords: debian, lvm, ovh, kimsufi, howto
---

Le manager pour la gamme OVH Kimsufi ne permet pas d'effectuer une installation sur LVM. Voici comment j'ai procédé pour une Debian 7 Wheezy. Le principe doit être applicable à d'autres systèmes.

### installation du système de base

Il faut commencer par installer un système de base via le manager. 

### sauvegarde 

Une fois que le système fonctionne, il faut **redémarrer le serveur en mode _recovery_** pour s'y connecter et effectuer des sauvegardes des partitions. 

    mkdir /backup
    dd if=/dev/sda1 bs=1M | gzip > /backup/root.img.gz

Il faut bien les sauvegarder sur une autre machine pour ne pas les perdre. 

    scp /backup/root.img.gz backuphost.net:/home/user/backup

### mise en place de LVM

Toujours dans le mode _recovery_, il faut supprimer les paritions du disque et y créer une unique partition.

    fdisk /dev/sda

Commencer par `d` pour supprimer les partitions,
puis `c` pour en créer une nouvelle avec le _system id_ `8e`

On créé les volumes avec LVM. Voir le [LVM HOWTO](http://tldp.org/HOWTO/LVM-HOWTO/) pour plus d'informations si besoin, en particulier l'[anatomie de LVM](http://tldp.org/HOWTO/LVM-HOWTO/anatomy.html) et les [tâches courante](http://tldp.org/HOWTO/LVM-HOWTO/commontask.html).

    pvcreate /dev/sda1
    vgcreate vg0 /dev/sda1
    lvcreate -L 10G vg0/root
    lvcreate -L 50G vg0/home
    lvcreate -L 1G vg0/swap

On restaure le(s) dump(s).

    dd if=/backup/root.img.gz bs=1M | gunzip > /dev/vg0/root

On active le swap.

    mkswap -l swap /dev/vg0/swap`

### réinstallation de GRUB2

Toujours en mode _recovery_, il faut reconfigurer GRUB2 pour lui indiquer comment trouver la partition racine et le noyau. Pour cela il faut _chrooter_ dans l'environnement cible.

Préparation de l'environnement cible.

    # on monte la partition racine
    mount /dev/vg0/root /mnt
    # puis /sys, /dev et /proc
    mount -t proc none /mnt/tmp/proc
    mount -t sysfs none /mnt/tmp/sys
    mount -o bind /dev /mnt/tmp/dev

Entrer dans l'environnement cible.

    chroot /mnt

Mettre à jour Grub.

    # génère un nouveau fichier /boot/grub/grub.cfg en fonction de la configuration /etc/grub.d/*
    update-grub
    # réinstallation
    grub-install /dev/sda

On quitte l'environnement cible.

    exit
    umount /dev \
           /proc \
           /sys \
           /

### redémarrer le serveur

Penser à changer la configuration du manager pour booter de nouveau sur le disque.
Puis redémarrer.
