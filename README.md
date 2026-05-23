# 🔐 L'Énigme — Coffre-Fort à Combinaison

> **Projet informatique** | Thomas Monin & Anna Caraulean | 2025–26  
> Prototype robotique à base de **Raspberry Pi Pico H** — Trouvez la bonne combinaison pour ouvrir le coffre !

---

## 📋 Table des matières

1. [Description du projet](#description-du-projet)
2. [Documentation technique](#documentation-technique)
   - [Schéma de câblage](#schéma-de-câblage)
   - [Liste des composants](#liste-des-composants)
   - [Explication du code](#explication-du-code)
   - [Trame du projet](#trame-du-projet)
3. [Nouveaux composants](#nouveaux-composants)
   - [Buzzer passif SE044](#1-buzzer-passif-se044)
   - [Écran LCD COM-LCD16X2](#2-écran-lcd-com-lcd16x2)
   - [Potentiomètres Piher PT15LV](#3-potentiomètres-piher-pt15lv)
4. [Sources](#sources)

---

## Description du projet

**L'Énigme** est un coffre-fort électronique interactif. Le joueur doit régler trois potentiomètres pour reproduire une combinaison secrète à trois chiffres (valeurs possibles : 1, 2 ou 3). Le système donne un retour visuel (LEDs + écran LCD) et sonore (buzzer) à chaque tentative.

**Combinaison par défaut : `1 – 2 – 2`**

### Fonctionnalités principales

- 🎛️ Saisie de la combinaison via trois potentiomètres
- 💡 Retour visuel par LEDs vertes (correct) et rouges (incorrect)
- 🔊 Retour sonore par buzzer passif (son aigu = correct, grave = erreur)
- 🖥️ Affichage en temps réel sur écran LCD 16×2
- ⚠️ Avertissement après 5 erreurs consécutives
- 🔄 Réinitialisation du code par appui long sur le bouton

---

## Documentation technique

### Schéma de câblage

> *Schéma non contractuel pour raisons de clarté : certains câbles ne sont pas représentés et la position de certains composants peut différer du montage réel.*

![Schéma de câblage](./schéma-câblage.png)

#### Correspondance des broches (GPIO → Composant)

| GPIO | Composant | Rôle |
|------|-----------|------|
| GP26 (ADC0) | Potentiomètre 1 | Lecture chiffre 1 |
| GP27 (ADC1) | Potentiomètre 2 | Lecture chiffre 2 |
| GP28 (ADC2) | Potentiomètre 3 | Lecture chiffre 3 |
| GP9 | Buzzer passif (S) | Signal sonore |
| GP0 | LED verte 1 | Validation chiffre 1 |
| GP1 | LED rouge 1 | Erreur chiffre 1 |
| GP2 | LED verte 2 | Validation chiffre 2 |
| GP3 | LED rouge 2 | Erreur chiffre 2 |
| GP4 | LED verte 3 | Validation chiffre 3 |
| GP5 | LED rouge 3 | Erreur chiffre 3 |
| GP15 | Bouton poussoir | Validation / Réinitialisation |
| GP10–GP14 | Écran LCD (D4–D8, RS, E) | Affichage |
| 3V3 | Buzzer (+), LCD (VDD) | Alimentation 3.3V |
| GND | Tous composants | Masse |

---

### Liste des composants

| Composant | Référence | Quantité |
|-----------|-----------|----------|
| Carte microcontrôleur | Raspberry Pi Pico H | 1× |
| Écran LCD 16×2 | COM-LCD16X2 | 1× |
| Buzzer passif ⭐ *(nouveau)* | SE044 | 1× |
| Potentiomètre ⭐ *(nouveau)* | Piher PT15LV | 4× (3 utilisés + 1 pour contraste LCD) |
| LED verte | Kingbright L-53G3C | 3× |
| LED rouge | Kingbright L-53SRD | 3× |
| Résistance 220 Ω ¼W | Résistance carbone | 6× |
| Bouton poussoir | 6×6×6mm Tact Push Button | 1× |
| Câbles de connexion | — | 43× |

---

### Explication du code

Le programme est écrit en **MicroPython** pour la Raspberry Pi Pico H.

#### Structure générale

```python
# Initialisation des composants
# - ADC pour les 3 potentiomètres (GP26, GP27, GP28)
# - PWM pour le buzzer passif (GP9)
# - GPIO pour les 6 LEDs (GP0 à GP5)
# - GPIO pour le bouton (GP15)
# - Bibliothèque LCD pour l'écran (via GPIO)

CODE_SECRET = [1, 2, 2]  # Combinaison par défaut
erreurs = 0              # Compteur d'erreurs consécutives
last_vals = [0, 0, 0]    # Dernières valeurs lues
```

#### Fonction `lire_pot(adc)` — Lecture d'un potentiomètre

```python
def lire_pot(adc):
    """
    Lit la valeur analogique d'un potentiomètre (0–65535)
    et la convertit en chiffre discret entre 1 et 3.
    """
    val = adc.read_u16()   # Lecture 16 bits (0 à 65535)
    if val < 21845:
        return 1
    elif val < 43690:
        return 2
    else:
        return 3
```

#### Fonction `jouer_son(freq, duree)` — Buzzer passif

```python
def jouer_son(freq, duree):
    """
    Génère un signal PWM à la fréquence donnée (en Hz)
    pendant 'duree' millisecondes.
    - Son aigu : freq élevée (ex. 1500 Hz) → validation
    - Son grave : freq basse (ex. 300 Hz)  → erreur
    """
    buzzer.freq(freq)
    buzzer.duty_u16(32768)  # 50% duty cycle = signal carré
    sleep_ms(duree)
    buzzer.duty_u16(0)      # Silence
```

#### Logique principale — Boucle principale

```python
while True:
    # 1. Lecture en continu des 3 potentiomètres
    vals = [lire_pot(adc1), lire_pot(adc2), lire_pot(adc3)]
    
    # 2. Mise à jour de l'écran si changement détecté
    if vals != last_vals:
        lcd.clear()
        lcd.putstr(f"{vals[0]} - {vals[1]} - {vals[2]}")
        last_vals = vals.copy()
    
    # 3. Détection d'appui bouton
    if not bouton.value():
        t_appui = mesurer_duree_appui()
        
        if t_appui > 1000:      # Appui long → réinitialisation
            reinitialiser_code(vals)
        else:                   # Appui court → vérification
            verifier_combinaison(vals)
```

#### Fonction `verifier_combinaison(vals)` — Vérification

```python
def verifier_combinaison(vals):
    """
    Compare les 3 valeurs lues avec CODE_SECRET.
    Pour chaque chiffre :
      - Correct  → LED verte + son aigu (1500 Hz)
      - Incorrect → LED rouge + son grave (300 Hz)
    Si tout correct → message victoire + mélodie de succès
    Sinon → incrémenter erreurs ; avertir après 5 échecs
    """
    global erreurs
    succes = True
    
    for i in range(3):
        if vals[i] == CODE_SECRET[i]:
            allumer_led(verte[i])
            jouer_son(1500, 200)
        else:
            allumer_led(rouge[i])
            jouer_son(300, 400)
            succes = False
    
    if succes:
        lcd.putstr("BRAVO ! Ouvert !")
        melodie_victoire()
        erreurs = 0
    else:
        erreurs += 1
        lcd.putstr("Reessayez !")
        if erreurs >= 5:
            lcd.putstr("ATTENTION : 5 erreurs !")
```

#### Fonction `reinitialiser_code(vals)` — Appui long

```python
def reinitialiser_code(vals):
    """
    Enregistre les positions actuelles des potentiomètres
    comme nouveau code secret.
    Feedback : toutes les LEDs clignotent + son de confirmation.
    """
    global CODE_SECRET
    CODE_SECRET = vals.copy()
    
    for _ in range(3):          # 3 clignotements
        allumer_toutes_leds()
        jouer_son(800, 150)
        eteindre_toutes_leds()
        sleep_ms(100)
    
    lcd.putstr(f"Code: {vals[0]}-{vals[1]}-{vals[2]}")
```

---

### Trame du projet

```
Démarrage
    │
    ▼
Initialisation (GPIO, ADC, LCD, Buzzer)
    │
    ▼
┌─────────────────────────────────┐
│  Boucle principale              │
│                                 │
│  1. Lire les 3 potentiomètres   │
│  2. Afficher sur LCD si changé  │
│  3. Attendre appui bouton       │
│         │                       │
│    ┌────┴────┐                  │
│  Court     Long                 │
│  (< 1s)   (> 1s)                │
│    │         │                  │
│  Vérifier  Réinitialiser        │
│  code      le code              │
│    │                            │
│  Succès ?                       │
│  Oui → Victoire 🎉              │
│  Non → Erreur (max 5)           │
└─────────────────────────────────┘
```

---

## Nouveaux composants

### 1. Buzzer passif SE044

**Référence :** SE044 — IDUINO / OpenPlatform

#### Fonctionnement technique

Contrairement à un buzzer *actif* (qui émet un son fixe dès qu'il est alimenté), le **buzzer passif SE044** nécessite un signal carré externe pour vibrer et produire du son. C'est le microcontrôleur qui génère ce signal via une sortie PWM.

- **3 broches :** `S` (signal PWM), `+` (alimentation 3.3V), `-` (GND)
- **Broche signal :** reliée au **GPIO9** de la Pico
- **Principe :** plus le signal change d'état (haut/bas) rapidement → plus le son est **aigu** ; plus il est lent → plus le son est **grave**
- **Fréquence utilisée :** 300 Hz (grave / erreur) à 1500 Hz (aigu / validation)

```
Pico GP9 ──[ Signal PWM ]──► S  [SE044]  + ──► 3V3
                                          - ──► GND
```

#### Rôle dans le projet

Le buzzer est le **retour sonore** du coffre-fort, permettant au joueur de comprendre le résultat sans regarder l'écran :

| Événement | Son | Fréquence |
|-----------|-----|-----------|
| Chiffre correct | Aigu bref | ~1500 Hz |
| Chiffre incorrect | Grave | ~300 Hz |
| Code trouvé (victoire) | Mélodie montante | 800–2000 Hz |
| Réinitialisation code | 3 bips moyens | ~800 Hz |

---

### 2. Écran LCD COM-LCD16X2

**Référence :** COM-LCD16X2 — Joy-IT / SIMAC Electronics

#### Fonctionnement technique

L'écran est un **afficheur alphanumérique 16 caractères × 2 lignes** basé sur le contrôleur standard **HD44780**, compatible avec de nombreuses bibliothèques MicroPython.

- **Interface :** 4 bits de données (D4–D7) + RS + E → économise les GPIO
- **Rétroéclairage :** LED bleue intégrée (contrôlable via broche `A/K`)
- **Contraste :** réglé par le 4e potentiomètre (diviseur de tension sur `V0`)
- **Bibliothèque utilisée :** `lcd_api` + driver HD44780 pour MicroPython

#### Rôle dans le projet

L'écran est l'**interface visuelle principale** du coffre :

| Moment | Affichage |
|--------|-----------|
| En attente | `1 - 2 - 2` (valeurs des potentiomètres) |
| Vérification réussie | `BRAVO ! Ouvert !` |
| Vérification échouée | `Reessayez !` |
| 5 erreurs | `ATTENTION : 5 erreurs !` |
| Réinitialisation | `Code: X-X-X` |

---

### 3. Potentiomètres Piher PT15LV

**Référence :** Piher PT15LV — Piher Sensors & Controls / Meggitt

#### Fonctionnement technique

Un potentiomètre est une **résistance variable** fonctionnant comme un diviseur de tension. Le curseur tourne sur un angle mécanique d'environ **235°**.

- **Élément résistif :** piste en carbone
- **Lecture :** entrées analogiques ADC de la Pico (GP26, GP27, GP28)
- **Résolution :** 16 bits (0 à 65 535) → converti en 3 zones discrètes (1, 2, 3)
- **Détection de mouvement :** comparaison avec `last_vals` à chaque cycle

```
           ┌──── Curseur (GP26/27/28) → lecture tension variable
3V3 ───────┤
           └──── GND
```

#### Conversion analogique → chiffre discret

```
Valeur ADC :   0 ────── 21845 ────── 43690 ────── 65535
Chiffre    :      1           2            3
```

#### Rôle dans le projet

Les potentiomètres sont l'**interface de saisie** du coffre-fort :

- **Chiffre 1, 2, 3 :** chaque potentiomètre contrôle un chiffre du code
- **Mise à jour temps réel :** tourner un potentiomètre met instantanément à jour l'écran LCD
- **Programmation :** lors d'un appui long, les positions actuelles deviennent le nouveau code secret

---

## Sources

- IDUINO. (s. d.). *Passive Buzzer SE044* [Fiche technique]. OpenPlatform. http://www.openplatform.cc
- SIMAC Electronics GmbH. (2022, 7 mars). *16x2 LCD Module COM-LCD16X2* [Fiche technique]. Joy-IT. https://joy-it.net
- SIMAC Electronics GmbH. (2022, 7 mars). *16x2 LCD Modul* [Manuel d'utilisation]. Joy-IT. https://joy-it.net
- Piher Sensors & Controls. (s. d.). *PT15LV Carbon Potentiometer* [Fiche technique]. Meggitt. https://www.piher.net
- Raspberry Pi Ltd. (2023). *Raspberry Pi Pico Datasheet*. https://datasheets.raspberrypi.com/pico/pico-datasheet.pdf

---

*Projet réalisé dans le cadre du cours d'informatique — Gymnase de Plan-les-Ouates, 2025–26*
