# WARDEN — Schéma Électronique Complet
**Version 1.1 | Juin 2026 | Guide de saisie KiCad**

> Document de saisie schéma. Chaque bloc détaille le câblage pin par pin.  
> Reproduire dans l'éditeur de schéma KiCad (Eeschema).

---

## Vue d'ensemble — Nets principaux

```
Alimentation :
  VBAT    → sortie batterie LiPo (3.0-4.2V)
  VBUS    → entrée USB-C (5V)
  +3V3    → sortie LDO (rail principal 3.3V)
  GND     → masse commune

NFC :
  NFC1    → module pin 52 (P0.09/NFC1) → antenne boucle PCB
  NFC2    → module pin 54 (P0.10/NFC2) → antenne boucle PCB
```

---

## BLOC 1 — Protection batterie LiPo

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| BT1 | Connecteur LiPo | JST-PH 2mm 2 pins |
| U1 | Contrôleur protection | DW01A (SOT-23-6) |
| Q1 | Double MOSFET | FS8205A (SOT-23-6) |
| C1 | Condensateur | 100nF 0402 |

### Câblage
```
BT1 pin 1 (+) ──┬── DW01A pin 1 (VCC/+)
                └── C1 ── GND
BT1 pin 2 (−) ──── FS8205A → masse protégée

DW01A pin 1 (VCC)  → BAT+
DW01A pin 2 (OD)   → FS8205A gate 2 (décharge)
DW01A pin 3 (OC)   → FS8205A gate 1 (charge)
DW01A pin 5 (CS)   → point milieu protection (B-)
DW01A pin 6 (GND)  → via R 100Ω → BAT−

FS8205A : entre BAT− et GND système (protection série)

Sortie protégée → net VBAT
```

> ℹ️ Si vous achetez une batterie LiPo **avec PCM intégré** (recommandé Adafruit), ce bloc DW01A+FS8205A est déjà dans la batterie — vous pouvez le retirer du schéma et connecter VBAT directement. À confirmer selon votre batterie finale.

---

## BLOC 2 — Charge USB-C + Détection USB

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| J1 | Connecteur USB-C | GCT USB4135-GF-A |
| U2 | Chargeur LiPo | MCP73831T-2ACI/OT (SOT-23-5) |
| C2 | Condensateur entrée | 4.7µF 0805 |
| C3 | Condensateur sortie | 4.7µF 0805 |
| R1 | Résistance Rprog | 10kΩ 0402 (charge 100mA) |
| R2 | Résistance LED | 1kΩ 0402 |
| D1 | LED charge | LED verte 0603 |
| R3, R4 | CC pull-down USB-C | 5.1kΩ 0402 (×2) |
| R16 | Détection USB — diviseur haut | 100kΩ 0402 |
| R17 | Détection USB — diviseur bas | 100kΩ 0402 |

### Câblage charge
```
J1 VBUS ──┬── C2 ── GND
          └── U2 pin 4 (VDD/VIN)

J1 CC1 ── R3 5.1kΩ ── GND
J1 CC2 ── R4 5.1kΩ ── GND    (configuration device USB-C)

U2 pin 1 (STAT) ── R2 1kΩ ── D1 (anode) ── +3V3 (cathode via LED)
U2 pin 2 (GND)  ── GND
U2 pin 3 (VBAT) ──┬── C3 ── GND
                  └── net VBAT (vers batterie)
U2 pin 4 (VDD)  ── VBUS
U2 pin 5 (PROG) ── R1 10kΩ ── GND    (Ich = 1000/10k = 100mA)
```

### Câblage détection USB (présence VBUS → P1.13)
```
J1 VBUS ── R16 100kΩ ──┬── U4 P1.13 (pin 6)  (lecture GPIO)
                        └── R17 100kΩ ── GND

V_detect = 5 V × 100k / (100k + 100k) = 2,5 V  (< 3,3 V ✓ marge 0,8 V)

Module pin VBUS (pin 32) → NC  (pas de communication USB du nRF)
Module D+ / D−           → NC  (configuration « USB Disabled »)
```

> ⚠️ Les résistances CC 5.1kΩ sont obligatoires pour qu'un chargeur USB-C délivre le courant. Sans elles, pas de charge.

> ℹ️ Pour réduire la consommation du pont diviseur à ~5 µA (au lieu de ~25 µA), utiliser 470 kΩ × 2 au lieu de 100 kΩ × 2 — à arbitrer selon la contrainte de courant.

---

## BLOC 3 — Régulateur LDO 3.3V

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| U3 | LDO 3.3V | XC6210B332MR-G (SOT-25) |
| C4 | Condensateur entrée | 1µF 0402 |
| C5 | Condensateur sortie | 1µF 0402 |

### Câblage
```
VBAT ──┬── C4 ── GND
       └── U3 pin 1 (VIN)

U3 pin 2 (VSS/GND) ── GND
U3 pin 3 (VOUT) ──┬── C5 ── GND
                  └── net +3V3 (rail principal)
U3 pin 4 (CE/Enable) ── VIN (toujours actif)
```

> ℹ️ Connecter CE à VIN pour activation permanente. Si vous voulez un mode ultra-low-power coupant le LDO, reliez CE à un GPIO — mais ça coupe aussi le module, donc déconseillé pour Warden qui doit rester en veille BLE.

> ℹ️ **Configuration alimentation Raytac (section 8.3 du datasheet) :** en alimentant le module via +3V3 sur les pins VDD, on se place en mode « External regulated power supply » (Reg0 — DC/DC et LDO internes désactivés). Ne pas connecter de résistance R32 ni d'inductance L4 sur la pin DCCH. Suivre les préconisations du chapitre 8.3 pour les pins DEC1–DEC4.

---

## BLOC 4 — Module Raytac MDBT50Q-1MV2 (cœur)

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| U4 | Module nRF52840 | Raytac MDBT50Q-1MV2 |
| C6-C8 | Découplage VDD | 100nF 0402 (×3) |
| C9 | Réservoir | 10µF 0805 |
| R5 | Pull-up RESET | 10kΩ 0402 |

### Attribution des GPIO (figée)

| Fonction | Pin nRF | Pin module | Détail |
|---|---|---|---|
| Coupe-circuit (jack) | P1.11 | 4 | GPIO interrupt, front RISING |
| Factory reset SW1 | P1.12 | 5 | GPIO interrupt, pull-up ext. R15, anti-rebond C12 |
| Détection USB | P1.13 | 6 | Lecture niveau via diviseur R16+R17 |
| LED Rouge | P1.14 | 7 | GPIO ON/OFF |
| LED Vert | P1.15 | 8 | GPIO ON/OFF |
| ADC batterie | P0.03 / AIN1 | 9 | Entrée analogique |
| Buzzer piézo | P0.29 | 10 | PWM ~2700 Hz |
| LED Bleu | P1.01 | 61 | GPIO ON/OFF |
| Quartz XL1 | P0.00 / XL1 | 17 | 32.768 kHz |
| Quartz XL2 | P0.01 / XL2 | 18 | 32.768 kHz |
| NFC1 | P0.09 / NFC1 | 52 | Antenne boucle PCB |
| NFC2 | P0.10 / NFC2 | 54 | Antenne boucle PCB |
| SWDIO | — | 51 | Debug / Tag-Connect |
| SWDCLK | — | 53 | Debug / Tag-Connect |

> ℹ️ **Répartition LF I/O vs standard I/O (chapitre 2.6 du datasheet Raytac) :** les signaux lents (< 10 kHz) — boutons, détections, LED ON/OFF — sont assignés aux pins P1.xx (Low Frequency I/O only). Les périphériques haute fréquence — PWM buzzer (P0.29) et ADC (P0.03/AIN1) — utilisent des pins P0.xx standard. Cette répartition respecte les contraintes du module.

### Câblage alimentation
```
+3V3 ──┬── C6 100nF ── GND   (près VDD pin 1)
       ├── C7 100nF ── GND   (près VDD pin 2)
       ├── C8 100nF ── GND   (près VDD pin 3)
       ├── C9 10µF  ── GND   (réservoir)
       └── U4 toutes pins VDD

U4 toutes pins GND ── GND (plan de masse)
```

### Câblage debug SWD
```
U4 SWDIO  (pin 51) ── Test Point TP1 (+ connecteur Tag-Connect recommandé)
U4 SWDCLK (pin 53) ── Test Point TP2
U4 RESET           ── R5 10kΩ ── +3V3   (pull-up uniquement — pas de bouton sur RESET)
```

> ⚠️ Télécharger le footprint officiel Raytac pour le MDBT50Q-1MV2 (dépôt GitHub Raytac : `raytac/mdbt50q-series`). Vérifier le pinout sur le datasheet officiel avant saisie — ne pas confondre avec le MDBT50Q-RX (dongle USB, footprint incompatible).

---

## BLOC 5 — Coupe-circuit (Jack 3.5mm)

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| J2 | Jack 3.5mm femelle | SJ1-3535 (vertical) |
| R6 | Pull-up | 100kΩ 0402 |

### Câblage
```
+3V3 ── R6 100kΩ ──┬── U4 P1.11 (pin 4)  (GPIO interrupt)
                   └── J2 TIP (pin 2)

J2 SLEEVE (pin 1) ── GND
J2 RING (pin 3)   ── non connecté (ou relié à TIP pour redondance)

LOGIQUE :
  Jack inséré (goupille en place) : TIP relié à SLEEVE → P1.11 = LOW
  Jack retiré (goupille tirée)     : TIP flottant → pull-up → P1.11 = HIGH

  Firmware : interrupt sur front MONTANT (RISING) = déclenchement
```

> ℹ️ Décision actée : RISING edge (jack normalement fermé à la masse). Firmware aligné en conséquence.

---

## BLOC 6 — Driver buzzer piézo

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| Q2 | Transistor NPN | BC847 (SOT-23) |
| R7 | Résistance base | 1kΩ 0402 |
| LS1 | Buzzer piézo | Transducteur 22mm |

### Câblage
```
U4 P0.29 (pin 10, PWM) ── R7 1kΩ ── Q2 Base

Q2 Collecteur ── LS1 (borne 1)
Q2 Émetteur   ── GND
+3V3 ────────── LS1 (borne 2)

Fréquence PWM firmware : 2700 Hz (résonance piézo)
Courant collecteur : limité par le piézo (~10-20mA)
```

> ℹ️ Pas de diode de roue libre nécessaire pour un piézo (charge capacitive, pas inductive). Si vous passez à un buzzer magnétique plus tard, ajouter une diode 1N4148 en parallèle.

---

## BLOC 7 — LED RGB indicateur

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| D2 | LED RGB | Cathode commune 0606 |
| R8 | Résistance R | 330Ω 0402 |
| R9 | Résistance G | 220Ω 0402 |
| R10 | Résistance B | 220Ω 0402 |

### Câblage
```
U4 P1.14 (pin 7)  ── R8 330Ω  ── D2 Rouge (anode)
U4 P1.15 (pin 8)  ── R9 220Ω  ── D2 Verte (anode)
U4 P1.01 (pin 61) ── R10 220Ω ── D2 Bleue (anode)

D2 Cathode commune ── GND

Calcul courant :
  Rouge : (3.3 - 2.0) / 330 ≈ 4mA
  Vert  : (3.3 - 3.2) / 220 ≈ proche limite — ajuster si trop faible
  Bleu  : (3.3 - 3.2) / 220 ≈ proche limite

NOTE : Vf vert/bleu élevé. Si luminosité insuffisante,
réduire R9/R10 à 150Ω, ou utiliser LED RGB basse-Vf.
```

---

## BLOC 8 — Bouton factory reset (SW1)

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| SW1 | Bouton factory reset | Tactile SMD (ex. B3U-1000P) |
| R15 | Pull-up P1.12 | 100kΩ 0402 |
| C12 | Anti-rebond | 100nF 0402 |

### Câblage
```
+3V3 ── R15 100kΩ ──┬── U4 P1.12 (pin 5)  (GPIO interrupt)
                    ├── C12 100nF ── GND   (anti-rebond hardware)
                    └── SW1 (borne 1)

SW1 (borne 2) ── GND

LOGIQUE :
  Repos : P1.12 = HIGH (pull-up R15)
  Appui : P1.12 = LOW  (court-circuité à GND via SW1)

  Firmware distingue :
    - appui court           → affichage statut (LED)
    - appui long ~5 s       + VBUS détecté sur P1.13 → factory reset complet
```

> ⚠️ **SW1 est OBLIGATOIRE** — c'est le seul accès physique au factory reset (désappairage + régénération du secret NFC). Ne PAS câbler sur la ligne RESET matérielle du module. La pin RESET reste uniquement reliée au pull-up R5 → +3V3. SW1 est lu par le firmware comme un GPIO ordinaire sur P1.12 (LF I/O).

---

## BLOC 9 — ADC niveau batterie

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| R13 | Diviseur haut | 100kΩ 0402 |
| R14 | Diviseur bas | 100kΩ 0402 |
| C11 | Filtre | 100nF 0402 |

### Câblage
```
VBAT ── R13 100kΩ ──┬── U4 P0.03 / AIN1 (pin 9)
                    ├── R14 100kΩ ── GND
                    └── C11 100nF ── GND  (filtre anti-bruit)

V_adc = VBAT × R14/(R13+R14) = VBAT / 2
VBAT max 4.2V → V_adc max 2.1V (< 2.4V réf interne nRF ✓)

Firmware : BAT_DIVIDER_FACTOR = 2
```

> ℹ️ Le pont diviseur consomme en permanence : I = 4.2V / 200kΩ = 21µA. C'est non négligeable face à l'objectif de consommation système. **Optimisation** : ajouter un MOSFET pour couper le pont diviseur entre les mesures (piloté par GPIO), ou augmenter les résistances à 1MΩ chacune (I = 2µA, mais impédance ADC plus élevée → augmenter le temps d'acquisition).

---

## BLOC 10 — Quartz 32.768 kHz

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| X2 | Quartz 32.768 kHz | SMD 2012, CL 6-12 pF (ex. Abracon AB38T-32.768KHZ-T, Epson FC-135R) |
| C13 | Condo charge XL1 | 12 pF 0402 NP0/C0G |
| C14 | Condo charge XL2 | 12 pF 0402 NP0/C0G |

### Câblage
```
U4 P0.00 / XL1 (pin 17) ──┬── X2 (borne 1)
                            └── C13 12pF ── GND

U4 P0.01 / XL2 (pin 18) ──┬── X2 (borne 2)
                            └── C14 12pF ── GND
```

> ℹ️ Le quartz 32.768 kHz est acté pour l'autonomie en connexion BLE permanente : il resserre les fenêtres d'écoute BLE (LFCLK précis → moins de guard time), réduisant la consommation en veille entre les connection events. Vérifier la capacité de charge CL du quartz retenu et ajuster C13/C14 si nécessaire (formule : C_ext ≈ 2 × CL − C_stray, avec C_stray ≈ 2-3 pF sur layout propre). Utiliser des condensateurs NP0/C0G pour la stabilité thermique.

---

## Antenne NFC — NFC natif nRF52840 (antenne boucle PCB)

### Composants
| Réf | Composant | Valeur/Modèle |
|---|---|---|
| C15 | Condensateur accord NFC 1 | 68-100 pF 0402 NP0 (à ajuster après mesure) |
| C16 | Condensateur accord NFC 2 | 68-100 pF 0402 NP0 (à ajuster après mesure) |

### Câblage
```
U4 P0.09 / NFC1 (pin 52) ──┬── Boucle PCB (4-6 spires, layer dédié)
                             └── C15 68-100 pF ── GND

U4 P0.10 / NFC2 (pin 54) ──┬── Boucle PCB (connexion opposée)
                             └── C16 68-100 pF ── GND

Dimensionnement condensateurs selon Nordic NWP_026 pour f = 13,56 MHz
(L_antenne ~2 µH → C_tune ≈ 68-100 pF, à affiner sur réseau vectoriel)

Fonctions assurées :
  • Wake-on-field   : réveil du nRF52840 par approche d'un lecteur NFC
  • Touch-to-Pair   : transmission NDEF (deviceId 4 octets + secret 16 octets)
                      → appairage BLE chiffré sans intervention manuelle
```

> ℹ️ Solution unique retenue. L'antenne PCB boucle est la seule implémentation NFC du projet — pas de puce externe (NT3H2111 supprimé). Voir Note de routage pour le placement et la séparation antenne NFC / antenne BLE.

---

## ERRATA & décisions à acter

| Point | Décision |
|---|---|
| Protection LiPo | Si batterie avec PCM intégré → retirer DW01A+FS8205A |
| NFC | **NFC natif nRF52840 retenu** (P0.09/NFC1 pin 52, P0.10/NFC2 pin 54, antenne boucle PCB). NT3H2111 supprimé. Bus I2C (SDA/SCL sur P0.26/P0.27) supprimé — aucun autre périphérique I2C. Raison : wake-on-field + Touch-to-Pair suffit pour le pairing |
| Coupe-circuit | **P1.11 (pin 4)**, logique **RISING edge** (jack normalement fermé) — firmware aligné |
| SW1 factory reset | **Obligatoire** sur P1.12 (pin 5), pull-up R15 100kΩ + anti-rebond C12 100nF. Distinct du RESET matériel. Firmware : appui court = statut, appui long + VBUS = factory reset |
| Détection USB | P1.13 (pin 6) via diviseur R16+R17 (100kΩ + 100kΩ → 2,5 V). Module pin VBUS (pin 32) en **NC**. D+/D− en **NC** |
| LED RGB | **P1.14 (R) / P1.15 (G) / P1.01 (B)** — pins 7/8/61. Ancienne attribution P0.13-P0.15 abandonnée |
| Quartz 32.768 kHz | **X2 acté** sur XL1 (P0.00, pin 17) + XL2 (P0.01, pin 18) avec 2 × 12 pF NP0 |
| Alimentation module | LDO externe XC6210 → VDD. Reg0 : DC/DC et LDO internes désactivés (section 8.3 datasheet Raytac). Pas de R32, pas de L4 sur DCCH |
| Pont diviseur ADC | Optimiser conso : MOSFET de coupure ou résistances 1MΩ |
| LED Vf vert/bleu | Ajuster R9/R10 si luminosité insuffisante |

---

## Checklist ERC (Electrical Rule Check) KiCad

- [ ] Toutes les pins VDD du module connectées à +3V3
- [ ] Toutes les pins GND connectées au plan de masse
- [ ] Découplage 100nF sur chaque VDD (C6, C7, C8) + réservoir 10µF (C9)
- [ ] Pull-up RESET 10kΩ présent (R5) — **aucun bouton câblé sur RESET**
- [ ] **SW1 obligatoire** sur P1.12 (pin 5) avec pull-up R15 (100kΩ) et anti-rebond C12 (100nF)
- [ ] Résistances CC USB-C 5.1kΩ présentes (R3, R4)
- [ ] Diviseur détection USB R16+R17 (100kΩ + 100kΩ) câblé vers P1.13 (pin 6)
- [ ] Pin VBUS module (pin 32) en **NC** explicite dans le schéma
- [ ] Pins D+ / D− module en **NC** (USB Disabled)
- [ ] Condensateurs accord NFC C15 et C16 (68-100 pF NP0) sur NFC1 (pin 52) et NFC2 (pin 54)
- [ ] Quartz X2 câblé sur XL1 (pin 17) et XL2 (pin 18) avec C13 + C14 (12 pF NP0)
- [ ] Pas de net flottant — no-connect explicites sur toutes les pins inutilisées
- [ ] Labels de net cohérents (VBAT, +3V3, GND)
- [ ] Footprints associés à tous les composants
- [ ] **Footprint MDBT50Q-1MV2** téléchargé depuis le dépôt officiel Raytac (GitHub `raytac/mdbt50q-series`) — ne pas utiliser le footprint du MDBT50Q-RX
- [ ] Pinout module vérifié sur datasheet officiel MDBT50Q-1MV2 avant saisie

---

## Note de routage — Contraintes certification modulaire

> Ces contraintes préservent la certification FCC/CE du module MDBT50Q-1MV2. Les violer impose une re-certification complète de l'end-product.

### Zone keep-out antenne BLE (module)
Le module comporte une zone « **No Ground-Pad Area** » de **3,8 mm** autour de l'antenne chip (chapitre 2.3 du datasheet Raytac). Dans cette zone :
- Aucun plan de masse (pas de copper fill GND)
- Aucune piste signal ni d'alimentation
- Aucun via
- **Placer le module vers le bord du PCB** afin que la zone antenne surplombe l'air (pas de substrat PCB sous l'antenne).

### Séparation antenne NFC / antenne BLE
- L'antenne boucle NFC rayonne perpendiculairement à son plan. Placer la boucle NFC **à l'opposé** de l'antenne chip BLE du module, ou sur la face inférieure du PCB.
- Distance minimale conseillée entre les zones actives des deux antennes : **≥ 10 mm**.
- Ne pas faire passer de piste d'alimentation ou de signal entre les deux zones d'antenne.

### Mention FCC / CE obligatoire sur l'end-product
Conformément aux exigences de certification modulaire, le produit final doit porter la mention :
```
Contains FCC ID: SH6MDBT50Q
Contains IC: 12208A-MDBT50Q
```
Vérifier les identifiants régionaux exacts (CE, MIC Japon, etc.) sur le certificat Raytac correspondant à la révision du module utilisé.

---

*Document Warden — Schéma KiCad — Juin 2026*
