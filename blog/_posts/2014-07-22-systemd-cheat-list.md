---
layout: post
title: 'Systemd reminder'
meta:
 keywords: linux, systemd, systemctl, howto, reminder
---

Petite liste de commandes pour lesquelles je fais un recherche à chaque fois que j'ai besoin de les utiliser...

Lister les services actifs :

    systemctl list-units -t service

Lister tous les services disponibles :

    systemctl list-units -t service -all

Obtenir le status d'un service :

    systemctl status mdmonitor.service

Consulter le journal d'un service :

    journalctl -u mdmonitor.service
