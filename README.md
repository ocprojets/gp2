# Projet Informatique — L'Énigme
**Auteurs :** Thomas Monin & Anna Caraulean  
**Année :** 2025-26

---

## Table des matières

- [Documentation technique](#documentation-technique)
  - [Schéma de câblage](#schéma-de-câblage)
  - [Liste des composants](#liste-des-composants)
  - [Explication du code](#explication-du-code)
  - [Trame du projet](#trame-du-projet)
- [Explication des nouveaux composants](#explication-des-nouveaux-composants)
  - [Buzzer Passif (SE044)](#nouvelle-composante-1--buzzer-passif-réf-se044)
  - [Écran LCD (COM-LCD16X2)](#nouvelle-composante-2--lécran-lcd-réf-com-lcd16x2)
- [Sources citées](#sources-citées)

---

## Documentation technique

### Schéma de câblage

![Schéma de câblage](image/Schema_cables.png)

> ⚠️ *Schéma non contractuel pour raison de clarté : certains câbles ne sont pas représentés et la position de certains composants peut différer du montage réel.*

> 💡 Il n'y a pas de buzzer similaire au nôtre dans le logiciel. Le nôtre possède 3 pins et le pin d'input est relié au GPIO9.

---

### Liste des composants

| Composant | Référence | Quantité |
|---|---|---|
| Câbles | — | 43x |
| Écran LCD | COM-LCD16X2 | 1x |
| Bouton | 6x6x6mm Tact Push Button Switch | 1x |
| Carte Raspberry Pi Pico H | PICO MIT HEADER | 1x |
| Résistance carbone 220 Ω 1/4W | — | 6x |
| LED verte | Kingbright L-53G3C | 3x |
| LED rouge | Kingbright L-53SRD | 3x |
| Buzzer passif | SE044 | 1x |
| Potentiomètre | Piher PT15LV | 4x |

---

### Explication du code

#### Trame du projet

Le but du programme est de trouver une combinaison composée de trois chiffres (par défaut : **1 - 2 - 2**). Chaque chiffre est contrôlé par un potentiomètre, qui peut prendre trois valeurs possibles selon l'angle : `1`, `2` ou `3`. Le joueur doit donc régler les trois potentiomètres afin de reproduire le bon code.

**Vérification de la combinaison (appui court sur le bouton) :**

Lorsque le bouton est pressé brièvement, le système vérifie la combinaison entrée. Chaque chiffre est analysé séparément :

- ✅ Si le chiffre est **correct** → une LED verte s'allume et un son aigu est joué par le buzzer.
- ❌ Si le chiffre est **faux** → une LED rouge s'allume et un son grave est émis.

L'écran LCD affiche les chiffres sélectionnés ainsi que le résultat de la tentative.

- Si la combinaison est **correcte** : un message de validation apparaît accompagné de plusieurs bips aigus.
- Sinon, le joueur doit réessayer. Après **cinq erreurs consécutives**, un message d'avertissement s'affiche.

**Réinitialisation du code (appui long sur le bouton) :**

Le programme possède aussi une fonction de réinitialisation du code, qui peut donc être choisi par le joueur. Lorsque le bouton est **maintenu appuyé pendant plus d'une seconde**, la valeur actuelle de chaque potentiomètre devient les nouvelles composantes du code. Toutes les LEDs clignotent alors ensemble et un son de confirmation est joué plusieurs fois. L'écran affiche ensuite le nouveau code enregistré.

---

## Explication des nouveaux composants

> Pour réaliser ce projet, nous avons utilisé les notices techniques des composants disponibles sur le site Reichelt. Ces documents nous ont permis de trouver les définitions à intégrer dans notre code, ainsi que les schémas de branchement nécessaires au montage du circuit.

---

### Nouvelle composante 1 — Buzzer Passif (Réf. SE044)

#### Fonctionnement technique

Contrairement à un buzzer *actif* qui émet un son continu dès qu'il est alimenté, le **buzzer passif (SE044)** a besoin d'un signal carré (onde sonore) pour vibrer et produire du son.

- Il possède trois broches : **S** (signal), **+** (alimentation 3.3V/5V) et **-** (GND).
- Dans notre code, on fait varier la fréquence du signal envoyé sur la broche de signal (reliée au **GPIO9** de la Pico) pour créer différentes notes.
  - Plus le signal change d'état (haut/bas) **rapidement** → le son est **aigu**.
  - Plus il est **lent** → le son est **grave**.

#### Rôle dans le projet

Le buzzer sert de retour sonore pour informer le joueur de l'état de sa tentative sans qu'il ait besoin de regarder l'écran :

| Événement | Son |
|---|---|
| Validation d'un chiffre correct | Son aigu |
| Erreur sur un chiffre | Son grave |
| Succès final (code complet) | Plusieurs bips aigus |
| Réinitialisation du code | Son de confirmation spécifique (joué plusieurs fois) |

---

### Nouvelle composante 2 — L'écran LCD (Réf. COM-LCD16X2)

#### Fonctionnement technique

L'écran est un afficheur capable de présenter **16 caractères sur 2 lignes** (d'où son nom 16×2).

- Il utilise le contrôleur standard **HD44780**, ce qui le rend compatible avec de nombreuses bibliothèques de programmation.
- Il possède un **rétroéclairage bleu** pour une meilleure lisibilité.

#### Rôle dans le projet

C'est l'interface visuelle principale qui guide l'utilisateur :

- **Affichage en temps réel :** montre la valeur actuelle (`1`, `2` ou `3`) de chaque potentiomètre pour que le joueur sache où il en est.
- **Résultats des tests :** affiche le message de validation ou demande de réessayer en cas d'échec.
- **Mode de programmation :** lors d'un appui long sur le bouton, confirme l'enregistrement et affiche visuellement le nouveau code choisi par le joueur.

---

## Sources citées

- IDUINO. (s. d.). *Passive Buzzer (SE044)* [Fiche technique]. OpenPlatform. http://www.openplatform.cc
- SIMAC Electronics GmbH. (2022, 7 mars). *16x2 LCD Module (COM-LCD16x2)* [Fiche technique]. Joy-IT. https://joy-it.net
- SIMAC Electronics GmbH. (2022, 7 mars). *16x2 LCD Modul* [Manuel d'utilisation]. Joy-IT. https://joy-it.net

