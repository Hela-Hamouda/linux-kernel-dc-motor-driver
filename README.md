# Contrôle de vitesse d'un moteur DC sous Linux embarqué (BeagleBone Black)

Mesure et régulation de la vitesse d'un moteur à courant continu depuis un système Linux généraliste, qui n'offre par construction aucune garantie temps réel strict. Le projet répond à cette contrainte en répartissant le travail entre **l'espace noyau**, qui traite les interruptions de l'encodeur sans en perdre une seule, et **l'espace utilisateur**, qui exécute la boucle de commande dans une architecture POSIX à trois threads.

Résultat mesuré au banc : **4 031 interruptions traitées sans aucune perte**, pour une latence d'IRQ **inférieure à 10 µs**, validée à l'oscilloscope.

> Projet de systèmes embarqués réalisé à l'École Nationale d'Ingénieurs de Tunis (ENIT), cycle ingénieur Génie Électrique.

---

## 1. Démonstration

<!-- Glisser le fichier .mp4 ici depuis l'éditeur GitHub pour l'insérer -->

---

## 2. Principe de fonctionnement

```text
                        ESPACE NOYAU                    ESPACE UTILISATEUR
                 ┌──────────────────────┐        ┌───────────────────────────┐
  Encodeur ──IRQ─>│ Module noyau         │        │ Thread 1 : lecture /proc  │
                 │ comptage d'impulsions│<─/proc─>│ Thread 2 : régulation     │
                 │ horodatage           │        │ Thread 3 : commande PWM   │
                 └──────────────────────┘        └───────────────────────────┘
                            ^                                  │
                            │                                  v
  Potentiomètre ──> ADC ────┘                    PWM ──> Étage de puissance ──> Moteur DC
```

1. **Configuration matérielle** — un Device Tree Overlay déclare les broches utilisées (GPIO d'encodeur, sortie PWM, entrée ADC) et les rend disponibles au noyau.
2. **Acquisition** — chaque front de l'encodeur déclenche une interruption traitée par le module noyau, qui incrémente un compteur et horodate l'événement.
3. **Communication** — une entrée `/proc` expose ces données à l'espace utilisateur, sans appel système coûteux ni copie inutile.
4. **Consigne** — la position du potentiomètre est lue par l'ADC et convertie en vitesse de consigne.
5. **Commande** — l'application utilisateur calcule le rapport cyclique et l'applique via le PWM, la gestion des lignes GPIO passant par libgpiod.

---

## 3. Matériel

| Élément | Rôle |
| --- | --- |
| BeagleBone Black (AM335x, Cortex-A8) | Carte Linux embarqué, unité de traitement |
| Moteur DC avec encodeur | Actionneur et retour de vitesse |
| Étage de puissance | Commande du moteur depuis la sortie PWM |
| Potentiomètre | Consigne de vitesse, lue par l'ADC |
| Oscilloscope | Validation des latences et du signal PWM |

---

## 4. Architecture logicielle

### Espace noyau

Le module noyau prend en charge ce qui ne tolère aucun retard :

- enregistrement du gestionnaire d'interruption sur la ligne GPIO de l'encodeur
- comptage et horodatage des impulsions dans le contexte d'interruption, maintenu aussi court que possible
- exposition des compteurs via une entrée `/proc`

Le choix de l'espace noyau est ici déterminant : traiter les fronts de l'encodeur depuis l'espace utilisateur exposerait le comptage aux aléas d'ordonnancement de Linux, et donc à des pertes d'impulsions sous charge.

### Device Tree Overlay

L'overlay décrit la configuration matérielle appliquée au démarrage : multiplexage des broches, déclaration du contrôleur PWM, activation des canaux ADC. Cette description déclarative remplace toute configuration de registres en dur dans le code.

### Espace utilisateur — architecture à trois threads POSIX

| Thread | Responsabilité |
| --- | --- |
| Acquisition | Lecture périodique des compteurs via `/proc` et calcul de la vitesse |
| Régulation | Comparaison à la consigne et calcul du rapport cyclique |
| Commande | Application du PWM et gestion des lignes GPIO via libgpiod |

La séparation en threads indépendants évite qu'un traitement lent ne retarde l'acquisition, et permet de cadencer chaque tâche selon sa propre contrainte temporelle.

---

## 5. Résultats

| Indicateur | Valeur |
| --- | --- |
| Interruptions traitées | **4 031** |
| Interruptions perdues | **0** |
| Latence d'IRQ | **< 10 µs** |
| Méthode de validation | Mesure à l'oscilloscope |

L'absence totale de perte sur plus de quatre mille interruptions confirme que le traitement en espace noyau tient la contrainte, là où une implémentation entièrement en espace utilisateur aurait été soumise à la latence d'ordonnancement du système.

---

## 6. Compilation et déploiement

**Prérequis :** les en-têtes du noyau correspondant à la version installée sur la carte, un compilateur croisé `arm-linux-gnueabihf` si la compilation se fait depuis un PC, et `libgpiod-dev`.

```bash
# Compilation du module noyau
make

# Chargement du Device Tree Overlay
sudo cp overlay.dtbo /lib/firmware/
sudo sh -c "echo overlay > /sys/devices/platform/bone_capemgr/slots"

# Chargement du module
sudo insmod motor_driver.ko
dmesg | tail          # vérifier l'enregistrement de l'IRQ

# Vérification de l'interface /proc
cat /proc/motor_speed

# Lancement de l'application utilisateur
sudo ./motor_control
```

Pour décharger le module : `sudo rmmod motor_driver`.

---

## 7. Compétences mises en œuvre

- Développement de modules noyau Linux et gestion d'interruptions matérielles
- Configuration matérielle par Device Tree Overlay
- Communication noyau / espace utilisateur via l'interface `/proc`
- Programmation concurrente POSIX (threads, synchronisation)
- Conception de systèmes temps réel sur plateforme non déterministe
- Pilotage PWM, acquisition ADC, manipulation GPIO via libgpiod
- Programmation C en espace noyau et en espace utilisateur
- Validation expérimentale à l'oscilloscope

---

## 8. Auteurs

Projet réalisé à l'ENIT, cycle ingénieur Génie Électrique.
