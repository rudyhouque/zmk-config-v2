# tapis-zmk-v2

Firmware ZMK pour **Le Spot 0x01** — un tapis de sol interactif équipé d'une carte NRF52840 Pro Micro (compatible nice!nano).

## Vue d'ensemble

Ce dépôt contient un shield ZMK personnalisé (`tapis`) qui transforme la carte NRF52840 Pro Micro en périphérique HID Bluetooth **3 touches HID + 1 touche Bluetooth**. Le firmware est compilé automatiquement en `.uf2` via GitHub Actions.

## Validations du 2026-05-06

### Compilation
- Le firmware compile correctement via GitHub Actions (pipeline ZMK v0.3).
- L'artefact `.uf2` est disponible dans l'onglet **Actions** après chaque push sur `main`.

### Matériel utilisé
- Clone **Nice!Nano v2 AliExpress** (vendu sous le nom "SuperMini NRF52840" ou "Pro Micro NRF52840").
- Compatible avec le target `nice_nano_v2` dans ZMK — aucun ajustement nécessaire.

### Flash
- Passer en mode bootloader : double-appui rapide sur Reset → le volume **NICENANO** apparaît dans le Finder.
- Glisser-déposer le fichier `.uf2` sur le volume → redémarrage automatique.

### Pins validées physiquement (2026-05-06)

| Sérigraphie carte | Pin Zephyr | Rôle validé |
|-------------------|------------|-------------|
| P0.11 | `gpio0 11` | Touche A — produit 'a' sur Mac AZERTY |
| P1.00 | `gpio1 0`  | Touche B — produit 'b' |
| P0.06 | `gpio0 6`  | Touche C — produit 'c' |
| P0.08 | `gpio0 8`  | Mode appairage BT (BT_CLR) |

> **Important — P0.13 inutilisable comme GPIO clavier** : sur le firmware `nice_nano_v2`, la pin P0.13 est réservée en interne pour le contrôle d'alimentation externe (`EXT_POWER`). Elle ne peut pas être utilisée comme entrée clavier. Utiliser P1.00 (D6) à la place pour la touche B.

### Bluetooth
- Connexion validée sur **macOS** sous le nom **TapisDuel**.
- Reconnexion automatique à l'appareil déjà couplé au démarrage.
- **Mode appairage** : court-circuiter GND et P0.08 → efface le bond BT en cours → la carte devient découvrable pour un nouvel appareil.

### AZERTY
- Le keymap utilise `&kp Q` (et non `&kp A`) pour produire la lettre **a** sur un Mac configuré en AZERTY.
- ZMK envoie des scancodes HID basés sur les positions QWERTY. Le Mac AZERTY interprète la position Q (QWERTY) comme la touche A (AZERTY). `&kp B` et `&kp C` sont identiques dans les deux layouts.

---

## Prochaine étape

Souder les capteurs de pression du tapis sur les pins P0.11 (A), P1.00 (B) et P0.06 (C), puis tester la détection de frappe avec le **Raspberry Pi CM4**.

---

## Matériel

| Composant | Détail |
|-----------|--------|
| Carte | Clone Nice!Nano v2 AliExpress (SuperMini NRF52840) |
| SoC | Nordic nRF52840 |
| Connectivité | Bluetooth LE (BLE HID) |
| Format firmware | UF2 (chargement via bootloader USB) |

### Câblage des touches

| Touche | Pin Zephyr | GPIO physique | Action ZMK | Mode |
|--------|------------|---------------|------------|------|
| A | `gpio0 11` | P0.11 (D7) | `&kp Q` ¹ | Actif bas, pull-up interne |
| B | `gpio1 0`  | P1.00 (D6) | `&kp B`   | Actif bas, pull-up interne |
| C | `gpio0 6`  | P0.06 (D1) | `&kp C`   | Actif bas, pull-up interne |
| BT | `gpio0 8` | P0.08 (D0) | `&bt BT_CLR` | Actif bas, pull-up interne |

¹ `&kp Q` produit 'a' sur Mac AZERTY (voir section AZERTY ci-dessus).

Toutes les touches sont câblées en **actif bas** avec résistance de pull-up activée côté firmware (pas de résistance externe nécessaire).

> **Touche BT (mode appairage)** : court-circuiter GND et P0.08. Cela efface le couplage Bluetooth en mémoire (`BT_CLR`) et rend l'appareil à nouveau découvrable.

## Bluetooth

- **Nom de l'appareil** : `TapisDuel` (configuré dans `tapis.conf`)
- **Profil** : HID over GATT (clavier/touche générique)
- **Couplage** : premier appairage automatique, mémorisé en flash
- **Validé sur** : macOS

## Structure du dépôt

```
tapis-zmk-v2/
├── .github/
│   └── workflows/
│       └── build.yml               # Pipeline GitHub Actions (ZMK v0.3)
├── boards/
│   └── shields/
│       └── tapis/
│           ├── tapis.overlay        # Définition hardware : kscan GPIO direct (4 touches)
│           └── tapis.zmk.yml        # Déclaration du shield pour ZMK
├── config/
│   ├── west.yml                    # Manifest West — pointe vers ZMK v0.3
│   ├── tapis.conf                  # Kconfig utilisateur : nom BT, BLE activé
│   └── tapis.keymap                # Keymap utilisateur : kp Q/B/C + bt BT_CLR
├── build.yaml                      # Matrice de build (board + shield)
└── zephyr/
    └── module.yml                  # Déclare ce dépôt comme module Zephyr
```

> **Convention ZMK user config** : les fichiers `.conf` et `.keymap` vont dans `config/` (couche utilisateur).
> Les fichiers de définition hardware (`.overlay`, `.zmk.yml`) vont dans `boards/shields/<shield>/`.

## Fichiers du shield

### `boards/shields/tapis/tapis.overlay` — définition hardware

```dts
/ {
    chosen {
        zmk,kscan = &kscan0;
    };

    kscan0: kscan {
        compatible = "zmk,kscan-gpio-direct";
        label = "KSCAN";
        input-gpios
            = <&gpio0 11 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche A  — pin P0.11 (D7) */
            , <&gpio1  0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche B  — pin P1.00 (D6), P0.13 est réservé EXT_POWER */
            , <&gpio0  6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche C  — pin P0.06 (D1) */
            , <&gpio0  8 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* BT_CLR    — pin P0.08 (D0) = mode appairage */
            ;
    };
};
```

### `config/tapis.conf` — configuration utilisateur

```ini
CONFIG_ZMK_KEYBOARD_NAME="Le Spot 0x01"
CONFIG_ZMK_BLE=y
```

### `config/tapis.keymap` — keymap utilisateur

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>

/ {
    keymap {
        compatible = "zmk,keymap";
        default_layer {
            bindings = <
                &kp Q   &kp B   &kp C   &bt BT_CLR
            >;
        };
    };
};
```

### `build.yaml`

```yaml
---
include:
  - board: nice_nano_v2
    shield: tapis
```

## Build GitHub Actions

Le workflow `.github/workflows/build.yml` utilise le pipeline officiel ZMK v0.3 :

```yaml
jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

Chaque push sur `main` (ou déclenchement manuel via `workflow_dispatch`) produit un artefact `.uf2` téléchargeable depuis l'onglet **Actions** du dépôt GitHub.

## Flash du firmware

1. Brancher la carte NRF52840 en USB.
2. Double-appuyer rapidement sur le bouton **Reset** → le volume **NICENANO** apparaît dans le Finder.
3. Glisser-déposer le fichier `.uf2` sur le volume — la carte redémarre automatiquement.

## Authentification GitHub

Pour pusher depuis le terminal, utiliser le GitHub CLI :

```bash
gh auth login
# Choisir : GitHub.com → HTTPS → Yes → Login with a web browser
```

Ensuite le `git push` fonctionne normalement (pas de mot de passe, pas de token à gérer).

## Dépendances

- [ZMK Firmware](https://github.com/zmkfirmware/zmk) v0.3
- [West](https://docs.zephyrproject.org/latest/develop/west/index.html) (gestionnaire de modules Zephyr)
- Chaîne de compilation ARM (`arm-zephyr-eabi`) — gérée par le container GitHub Actions
