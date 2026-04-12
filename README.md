# tapis-zmk-v2

Firmware ZMK pour **Le Spot 0x01** — un tapis de sol interactif équipé d'une carte NRF52840 Pro Micro (compatible nice!nano).

## Vue d'ensemble

Ce dépôt contient un shield ZMK personnalisé (`tapis`) qui transforme la carte NRF52840 Pro Micro en périphérique HID Bluetooth **3 touches HID + 1 touche Bluetooth**. Le firmware est compilé automatiquement en `.uf2` via GitHub Actions.

## Validations du 2026-04-12

### Compilation
- Le firmware compile correctement via GitHub Actions (pipeline ZMK v0.3).
- L'artefact `.uf2` est disponible dans l'onglet **Actions** après chaque push sur `main`.

### Matériel utilisé
- Clone **Nice!Nano v2 AliExpress** (vendu sous le nom "Pro Micro NRF52840").
- Compatible avec le target `nice_nano_v2` dans ZMK — aucun ajustement nécessaire.

### Flash
- Passer en mode bootloader : double-appui sur Reset → le volume **NICENANO** apparaît.
- Glisser-déposer le fichier `.uf2` sur le volume → redémarrage automatique.

### Pins validées physiquement

| Sérigraphie carte | Pin Zephyr | Rôle validé |
|-------------------|------------|-------------|
| `011`             | `gpio0 11` | Touche A — contact détecté |
| `013`             | `gpio0 13` | Touche B — contact détecté |

Les pins ont été testées en court-circuitant GND avec le pad correspondant : la frappe est bien reçue côté hôte.

### Bluetooth
- Connexion validée sur **macOS** sous le nom **TapisDuel**.
- Appairage automatique au premier couplage, mémorisé en flash.

---

## Prochaine étape

Souder les capteurs de pression du tapis sur les pins `011` (gpio0 11) et `013` (gpio0 13), puis tester la détection de frappe avec le **Raspberry Pi CM4**.

---

## Matériel

| Composant | Détail |
|-----------|--------|
| Carte | Clone Nice!Nano v2 AliExpress (Pro Micro NRF52840) |
| SoC | Nordic nRF52840 |
| Connectivité | Bluetooth LE (BLE HID) |
| Format firmware | UF2 (chargement via bootloader USB) |

### Câblage des touches

| Touche | Pin Zephyr | GPIO physique | Action ZMK | Mode |
|--------|------------|---------------|------------|------|
| A | `gpio0 11` | P0.11 | `&kp A` | Actif bas, pull-up interne |
| B | `gpio0 13` | P0.13 | `&kp B` | Actif bas, pull-up interne |
| C | `gpio0 6`  | P0.06 | `&kp C` | Actif bas, pull-up interne |
| BT | `gpio0 8` | P0.08 | `&bt BT_CLR` | Actif bas, pull-up interne |

Toutes les touches sont câblées en **actif bas** avec résistance de pull-up activée côté firmware (pas de résistance externe nécessaire).

> **Touche BT** : appuyer sur cette touche efface le couplage Bluetooth en mémoire (`BT_CLR`) et rend l'appareil à nouveau découvrable. Utiliser le behavior ZMK `&bt BT_CLR`.

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
│   └── tapis.keymap                # Keymap utilisateur : kp A/B/C + bt BT_CLR
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
            = <&gpio0 11 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche A */
            , <&gpio0 13 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche B */
            , <&gpio0  6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche C */
            , <&gpio0  8 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>   /* Touche BT (BT_CLR) */
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
                &kp A   &kp B   &kp C   &bt BT_CLR
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

1. Brancher la carte NRF52840 Pro Micro en USB.
2. Double-appuyer sur le bouton **Reset** pour passer en mode bootloader (lecteur USB `NRF52BOOT` ou `NICENANO` apparaît).
3. Copier le fichier `.uf2` sur le lecteur — la carte redémarre automatiquement.

## Dépendances

- [ZMK Firmware](https://github.com/zmkfirmware/zmk) v0.3
- [West](https://docs.zephyrproject.org/latest/develop/west/index.html) (gestionnaire de modules Zephyr)
- Chaîne de compilation ARM (`arm-zephyr-eabi`) — gérée par le container GitHub Actions
