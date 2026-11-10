# WARDEN — Guide Schéma Hiérarchique KiCad 8
**Création de la structure multi-pages**

> Guide pas à pas pour créer la hiérarchie de feuilles du schéma Warden.

---

## Principe du schéma hiérarchique KiCad

Un schéma hiérarchique fonctionne comme un arbre :

```
warden.kicad_sch                    ← feuille RACINE (vue d'ensemble)
  ├── alimentation.kicad_sch        ← sous-feuille 1
  ├── nordic_nfc.kicad_sch          ← sous-feuille 2
  └── peripheriques.kicad_sch       ← sous-feuille 3
```

La feuille racine ne contient PAS de composants — uniquement des **blocs de feuille** (sheet symbols) qui pointent vers les sous-feuilles, reliés entre eux par les nets globaux.

---

## Étape 1 — Créer le projet

1. KiCad → **File → New Project** → nommez-le `warden_hw`
2. Le projet crée automatiquement `warden_hw.kicad_sch` (votre feuille racine)
3. Ouvrez l'éditeur de schéma (Eeschema)

---

## Étape 2 — Créer les 3 sous-feuilles

Sur la feuille racine, pour chaque sous-feuille :

1. Menu **Place → Add Hierarchical Sheet** (ou touche `S`)
2. Dessinez un rectangle (le bloc de feuille) en cliquant deux fois
3. Une boîte de dialogue s'ouvre :
   - **Sheet name** : `Alimentation`
   - **File name** : `alimentation.kicad_sch`
4. Cliquez OK

Répétez pour les 3 blocs :

| Sheet name | File name |
|---|---|
| Alimentation | alimentation.kicad_sch |
| Nordic_NFC | nordic_nfc.kicad_sch |
| Peripheriques | peripheriques.kicad_sch |

Vous avez maintenant 3 rectangles sur votre feuille racine.

---

## Étape 3 — Définir les connexions inter-feuilles

C'est l'étape clé. Les feuilles communiquent via des **Hierarchical Pins** sur les blocs, reliés aux **Hierarchical Labels** à l'intérieur des sous-feuilles.

### Nets qui traversent les feuilles

| Net | Origine | Destinations |
|---|---|---|
| +3V3 | Alimentation | Nordic_NFC, Peripheriques |
| GND | Toutes | Toutes (masse commune) |
| VBAT | Alimentation | (lecture ADC sur place) |

### Approche recommandée pour Warden — Labels globaux

Plutôt que de gérer des hierarchical pins pour l'alimentation et la masse (fastidieux), utilisez des **Global Labels** pour les nets d'alimentation :

- `+3V3`, `GND`, `VBAT` → **Global Labels** (touche `Ctrl+H`)
- Ces nets sont automatiquement connectés à travers TOUTES les feuilles sans câblage explicite sur la racine

> ℹ️ Les Global Labels sont parfaits pour l'alimentation et la masse qui sont partout. Réservez les Hierarchical Pins pour les signaux spécifiques entre deux feuilles précises (rare dans Warden).

---

## Étape 4 — Naviguer entre les feuilles

- **Double-clic** sur un bloc de feuille → entre dans la sous-feuille
- Bouton **"Leave Sheet"** (flèche vers le haut dans la barre d'outils) → remonte à la racine
- Panneau de gauche → arborescence des feuilles pour navigation rapide

---

## Étape 5 — Contenu de chaque feuille

### Feuille Alimentation
```
Composants :
  - BT1 connecteur LiPo JST-PH
  - U2 MCP73831 (charge)
  - J1 USB-C
  - U3 XC6210 LDO 3.3V
  - Pont diviseur ADC (R13, R14, C11)
  - Condensateurs découplage

Global Labels à placer :
  - VBAT (sortie batterie)
  - +3V3 (sortie LDO)
  - GND
  - VBAT_SENSE → vers P0.03 (ou label hiérarchique vers Nordic)

Power flags :
  - PWR_FLAG sur +3V3, VBAT, GND
```

### Feuille Nordic_NFC
```
Composants :
  - U4 module Raytac MDBT50Q-RX
  - Découplage C6-C9
  - R5 pull-up RESET
  - Test points SWD
  - Antenne NFC + condensateurs accord

Global Labels à utiliser (connectés automatiquement) :
  - +3V3, GND (alimentation)

Labels hiérarchiques (signaux vers périphériques) :
  - P0.11 (goupille)
  - P0.29 (buzzer)
  - P0.13/14/15 (LED RGB)
  - P0.03 (ADC batterie)
```

### Feuille Peripheriques
```
Composants :
  - J2 jack 3.5mm + R6 pull-up (goupille)
  - Q2 BC847 + R7 + LS1 (buzzer)
  - D2 LED RGB + R8/R9/R10

Global Labels :
  - +3V3, GND

Labels à connecter aux GPIO :
  - P0.11, P0.29, P0.13, P0.14, P0.15
```

---

## Étape 6 — Connexion des GPIO entre Nordic et Périphériques

Pour les signaux GPIO (P0.11, P0.29, etc.) entre la feuille Nordic et la feuille Périphériques, deux options :

### Option simple — Global Labels (recommandée pour Warden)
Nommez chaque GPIO avec un **Global Label** identique des deux côtés :
- Sur feuille Nordic : Global Label `GPIO_GOUPILLE` sur la pin P0.11
- Sur feuille Périphériques : Global Label `GPIO_GOUPILLE` sur le jack

KiCad les relie automatiquement. Simple, lisible, pas de gestion de hierarchical pins.

### Noms de nets GPIO suggérés
| Signal | Global Label | Pin module |
|---|---|---|
| Goupille | GPIO_GOUPILLE | P0.11 |
| Buzzer | GPIO_BUZZER | P0.29 |
| LED Rouge | GPIO_LED_R | P0.13 |
| LED Verte | GPIO_LED_G | P0.14 |
| LED Bleue | GPIO_LED_B | P0.15 |
| Batterie ADC | VBAT_SENSE | P0.03 |
| NFC FD (si NT3H) | GPIO_NFC_FD | P0.04 |

---

## Étape 7 — Vérification ERC

Une fois les 3 feuilles remplies :

1. **Inspect → Electrical Rules Checker**
2. Corriger les erreurs dans l'ordre :
   - "Power input not driven" → ajouter PWR_FLAG
   - "Label not connected" → vérifier orthographe des Global Labels
   - "Pin not connected" → ajouter no-connect (X) sur pins inutilisées du module

---

## Astuces KiCad 8 spécifiques

**Importer le footprint Raytac**
1. Télécharger depuis github.com/raytac/MDBT50Q footprint + symbole
2. **Preferences → Manage Symbol Libraries** → ajouter la lib Raytac
3. **Preferences → Manage Footprint Libraries** → ajouter le .pretty Raytac

**Créer un symbole custom si pas trouvé**
1. **Tools → Symbol Editor**
2. Créer un nouveau symbole avec les pins du module selon datasheet
3. Associer au footprint Raytac

**Raccourcis utiles**
- `A` : ajouter un composant
- `P` : ajouter une alimentation (power symbol)
- `W` : tracer un fil (wire)
- `Ctrl+H` : Global Label
- `H` : Hierarchical Label (dans une sous-feuille)
- `S` : Hierarchical Sheet (sur la racine)

---

## Ordre de travail recommandé

1. Créer le projet + les 3 blocs de feuille sur la racine
2. Remplir **Nordic_NFC** en premier (importer footprint Raytac, placer le module)
3. Remplir **Alimentation** (rails +3V3, VBAT, GND définis ici)
4. Remplir **Peripheriques** (connecter les GPIO via Global Labels)
5. ERC global
6. Association footprints (Tools → Assign Footprints)
7. Passage au PCB (Tools → Update PCB from Schematic)

---

*Guide Warden — Hiérarchie KiCad — Juin 2026*
