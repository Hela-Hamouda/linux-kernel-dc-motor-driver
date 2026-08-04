# Contrôle de Vitesse d'un Moteur DC — Driver Noyau Linux (BeagleBone Black)

Module noyau Linux pour le pilotage et la mesure de vitesse d'un moteur DC
sur BeagleBone Black.

## Description

Ce projet implémente un **driver noyau Linux** permettant :
- la mesure de la vitesse d'un moteur DC par interruptions
- le pilotage PWM via un potentiomètre (ADC)
- la configuration du **Device Tree**
- l'interfaçage en espace utilisateur via **libgpiod**

## Technologies utilisées

- BeagleBone Black
- C (kernel + userspace)
- Linux Kernel Module
- Device Tree, PWM, ADC, libgpiod

## Auteur

Hela Hamouda — Élève ingénieure en Génie Électrique, ENIT
