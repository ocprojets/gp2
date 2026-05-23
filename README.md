# 🔐 L'Énigme — Coffre-Fort à Combinaison

> **Projet informatique** | Thomas Monin & Anna Caraulean | 2025–26  
> Prototype robotique à base de Raspberry Pi Pico H — Trouvez la bonne combinaison du coffre !

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
4. [Sources](#sources)

---

# Description du projet

**L'Énigme** est un coffre-fort électronique interactif. Le joueur doit régler trois potentiomètres afin de reproduire une combinaison secrète composée de trois chiffres.

Le système donne un retour visuel (LEDs + écran LCD) et sonore (buzzer) à chaque tentative.

**Combinaison par défaut : `1 – 2 – 2`**

## Fonctionnalités principales

- 🎛️ Saisie de la combinaison via trois potentiomètres
- 💡 Retour visuel par LEDs vertes (correct) et rouges (incorrect)
- 🔊 Retour sonore par buzzer passif
- 🖥️ Affichage en temps réel sur écran LCD 16×2
- ⚠️ Avertissement après 5 erreurs consécutives
- 🔄 Possibilité de définir une nouvelle combinaison par appui long sur le bouton

---

# Documentation technique

## Schéma de câblage

> *Schéma non contractuel pour raisons de clarté : certains câbles ne sont pas représentés et la position de certains composants peut différer du montage réel.*

![Schéma de câblage](./Schéma câblage.png)

### Correspondance des broches (GPIO → Composant)

| GPIO | Composant | Rôle |
|------|-----------|------|
| GP26 (ADC0) | Potentiomètre 1 | Lecture chiffre 1 |
| GP27 (ADC1) | Potentiomètre 2 | Lecture chiffre 2 |
| GP28 (ADC2) | Potentiomètre 3 | Lecture chiffre 3 |
| GP9 | Buzzer passif (S) | Signal sonore |
| GP7 | LED verte 1 | Validation chiffre 1 |
| GP6 | LED rouge 1 | Erreur chiffre 1 |
| GP18 | LED verte 2 | Validation chiffre 2 |
| GP17 | LED rouge 2 | Erreur chiffre 2 |
| GP20 | LED verte 3 | Validation chiffre 3 |
| GP19 | LED rouge 3 | Erreur chiffre 3 |
| GP8 | Bouton poussoir | Validation / Nouveau code |
| GP10–GP15 | Écran LCD | Affichage |
| 3V3 | Composants | Alimentation |
| GND | Tous composants | Masse |

---

## Liste des composants

| Composant | Référence | Quantité |
|-----------|-----------|----------|
| Carte microcontrôleur | Raspberry Pi Pico H | 1× |
| Écran LCD 16×2 ⭐ *(nouveau)* | COM-LCD16X2 | 1× |
| Buzzer passif ⭐ *(nouveau)* | SE044 | 1× |
| Potentiomètre | Piher PT15LV | 4× |
| LED verte | Kingbright L-53G3C | 3× |
| LED rouge | Kingbright L-53SRD | 3× |
| Résistance 220 Ω ¼W | Résistance carbone | 6× |
| Bouton poussoir | 6×6×6mm Tact Push Button | 1× |
| Câbles de connexion | — | 43× |

---

## Explication du code

Le programme est écrit en **MicroPython** pour la Raspberry Pi Pico H.

```python
from machine import Pin, ADC, PWM
import utime
import time

# =========================
# INITIALISATION DE L'ECRAN LCD
# =========================

# Définition des broches utilisées par l'écran LCD
rs = Pin(10, Pin.OUT)
e = Pin(11, Pin.OUT)
d4 = Pin(12, Pin.OUT)
d5 = Pin(13, Pin.OUT)
d6 = Pin(14, Pin.OUT)
d7 = Pin(15, Pin.OUT)

# Fonction qui active l'écran LCD
def pulse_enable():
    e.value(1)
    utime.sleep_us(1)
    e.value(0)
    utime.sleep_us(100)

# Envoie 4 bits de données à l'écran
def send_nibble(data):
    d4.value((data>>0)&1)
    d5.value((data>>1)&1)
    d6.value((data>>2)&1)
    d7.value((data>>3)&1)
    pulse_enable()

# Envoie un octet complet à l'écran
def send_byte(data, mode):
    rs.value(mode)
    send_nibble(data>>4)
    send_nibble(data & 0x0F)

# Commande LCD
def lcd_command(cmd):
    send_byte(cmd,0)
    utime.sleep_ms(2)

# Affichage LCD
def lcd_data(data):
    send_byte(data,1)
    utime.sleep_ms(2)

# Initialisation de l'écran LCD
def lcd_init():
    utime.sleep_ms(20)
    send_nibble(0x03)
    utime.sleep_ms(5)

    send_nibble(0x03)
    utime.sleep_us(100)

    send_nibble(0x03)
    send_nibble(0x02)

    lcd_command(0x28)
    lcd_command(0x0C)
    lcd_command(0x06)
    lcd_command(0x01)

# Affiche un texte sur l'écran
def lcd_print(txt):
    for c in txt:
        lcd_data(ord(c))

# Efface l'écran
def lcd_clear():
    lcd_command(0x01)

# =========================
# INITIALISATION DES COMPOSANTS
# =========================

# LEDs
led_v1 = Pin(7, Pin.OUT)
led_r1 = Pin(6, Pin.OUT)

led_v2 = Pin(18, Pin.OUT)
led_r2 = Pin(17, Pin.OUT)

led_v3 = Pin(20, Pin.OUT)
led_r3 = Pin(19, Pin.OUT)

# Potentiomètres
pot1 = ADC(26)
pot2 = ADC(27)
pot3 = ADC(28)

# Bouton poussoir
bouton = Pin(8, Pin.IN, Pin.PULL_UP)

# Buzzer passif
buzzer = PWM(Pin(9))

# =========================
# VARIABLES PRINCIPALES
# =========================

# Combinaison secrète par défaut
combinaison = [1,2,2]

# Nombre d'erreurs
tentatives = 0

# Dernières valeurs affichées
last_vals = [0,0,0]

# =========================
# FONCTIONS PRINCIPALES
# =========================

# Joue un son avec le buzzer
def beep(f,d):
    buzzer.freq(f)
    buzzer.duty_u16(30000)

    time.sleep(d)

    buzzer.duty_u16(0)

# Lit la valeur d'un potentiomètre
# et la convertit en chiffre 1, 2 ou 3
def lire_pot(p):

    v = p.read_u16()

    if v < 21845:
        return 1

    elif v < 43690:
        return 2

    else:
        return 3

# Éteint toutes les LEDs
def reset_leds():

    for led in [led_v1,led_r1,led_v2,led_r2,led_v3,led_r3]:
        led.value(0)

# =========================
# INITIALISATION
# =========================

lcd_init()
lcd_clear()

# Affichage de la consigne dans la console
print("But du jeu : trouver la bonne combinaison.")

# =========================
# BOUCLE PRINCIPALE
# =========================

while True:

    # Lecture des potentiomètres
    vals = [lire_pot(pot1), lire_pot(pot2), lire_pot(pot3)]

    # Mise à jour de l'écran si changement
    for i in range(3):

        if vals[i] != last_vals[i]:

            lcd_clear()

            lcd_print("Chiffre "+str(i+1)+"="+str(vals[i]))

            last_vals = vals[:]

    # Détection d'appui bouton
    if bouton.value() == 0:

        t0 = time.ticks_ms()

        while bouton.value() == 0:
            time.sleep(0.05)

        duree = time.ticks_diff(time.ticks_ms(), t0)

        # =========================
        # NOUVEAU CODE
        # =========================

        if duree >= 1000:

            combinaison = vals[:]

            tentatives = 0

            # Clignotement de confirmation
            for i in range(5):

                led_v1.value(1)
                led_r1.value(1)

                led_v2.value(1)
                led_r2.value(1)

                led_v3.value(1)
                led_r3.value(1)

                beep(1479, 0.1)

                led_v1.value(0)
                led_r1.value(0)

                led_v2.value(0)
                led_r2.value(0)

                led_v3.value(0)
                led_r3.value(0)

                time.sleep(0.1)

            lcd_clear()

            lcd_print("Nouveau code:")

        # =========================
        # VERIFICATION CODE
        # =========================

        else:

            reset_leds()

            correct = True

            leds_v = [led_v1,led_v2,led_v3]
            leds_r = [led_r1,led_r2,led_r3]

            lcd_clear()

            lcd_print("Code:")

            lcd_command(0xC0)

            affichage = ""

            # Vérification des 3 chiffres
            for i in range(3):

                affichage += str(vals[i])

                if i < 2:
                    affichage += "-"

                lcd_command(0xC0)

                lcd_print(affichage)

                # Bonne valeur
                if vals[i] == combinaison[i]:

                    leds_v[i].value(1)

                    beep(2959, 0.2)

                # Mauvaise valeur
                else:

                    leds_r[i].value(1)

                    beep(369, 0.2)

                    correct = False

                time.sleep(0.5)

            lcd_clear()

            # Combinaison correcte
            if correct:

                lcd_print("Correct !")

                tentatives = 0

                for i in range(5):

                    beep(2959, 0.1)

                    time.sleep(0.1)

            # Combinaison incorrecte
            else:

                tentatives += 1

                if tentatives >= 5:

                    lcd_print("T'abuse non")

                else:

                    lcd_print("Essaye encore")

    time.sleep(0.1)
