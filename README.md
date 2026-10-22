<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/juru-mistral-logo-dark.svg">
    <img src="assets/juru-mistral-logo.svg" alt="juru-mistral" width="140">
  </picture>
</p>

# juru-mistral

**Régulation intelligente pour séchoir solaire alimentaire.**
Une petite carte ESP32 mesure l'humidité et la température, pilote le
ventilateur d'extraction au plus juste, et affiche tout en temps réel sur
une page web. Fonctionne jour et nuit sur batterie 18650 rechargée par le
panneau solaire du séchoir.

Premier module de l'écosystème **JURU** — *Just Useful Reusable Units*.

*[English version below](#english)*

---

## État du projet

*Mis à jour le 22 octobre 2026.*

| Étape | État |
|---|---|
| Cahier des charges complet ([docs/cahier-des-charges.md](docs/cahier-des-charges.md)) | ✅ Terminé |
| Choix des composants | ✅ Terminé |
| Programme de la carte — v0.1.0 « ça mesure » | 🔧 En cours |
| Banc d'essai capteurs (fiabilité des mesures température/humidité) | 🔧 En cours |
| v0.2.0 « ça régule » : régulation complète, presets, page web | ⏳ À venir |
| Assemblage et essais au soleil | ⏳ À venir |
| v0.3.0 « ça se met à jour » : mises à jour sans câble | ⏳ À venir |
| v1.0.0 : version stable, recette complète validée | ⏳ À venir |

Les instructions de montage et d'utilisation seront ajoutées ici au fur et
à mesure.

**Branches.** `main` contient la version propre à consulter. Le travail en
cours se fait sur `dev` et n'arrive sur `main` qu'une fois relu.

---

## Le problème

Sur un séchoir solaire classique à ventilation forcée, le ventilateur est
branché directement sur le panneau : plus il y a de soleil, plus il tourne
vite. C'est un défaut de conception — au moment où le caisson pourrait
atteindre sa meilleure température, le ventilateur extrait à pleine
vitesse et le refroidit. Le séchoir se bride tout seul, et s'arrête net
au coucher du soleil en laissant l'air humide condenser sur les aliments.

## La solution

juru-mistral découple le ventilateur de l'ensoleillement. Une batterie
tampon et un régulateur PID pilotent l'extraction en fonction de ce qui
compte : l'état de l'air dans la chambre.

- **Montée en température plus rapide** — le ventilateur reste coupé tant
  que la chambre n'a pas atteint sa fenêtre utile (garde à hystérésis)
- **Séchage optimisé** — le PID module la vitesse pour extraire l'humidité
  sans évacuer la chaleur ; l'air à 48 °C absorbe deux fois plus d'eau
  qu'à 35 °C
- **Purge de fin de journée** — sur batterie, l'extraction continue après
  le coucher du soleil tant que l'air sortant est plus humide que l'air
  entrant, ce qui évite la condensation nocturne et les moisissures
- **Sécurité surchauffe** — au-delà de 52 °C : extraction forcée, ou
  coupure si l'air entrant est lui-même trop chaud
- **Presets par aliment** — herbes, fruits, légumes/champignons
  (température max et humidité cible), sélectionnables depuis la page web
- **Suivi en direct** — température et humidité entrée/sortie, vitesse
  ventilateur, production du panneau solaire, alarmes ; WiFi domestique
  ou point d'accès autonome, page `http://juru-mistral.local`

## Matériel

| Composant | Rôle | Prix indicatif |
|---|---|---|
| Seeed XIAO ESP32-C3 | Contrôleur, WiFi, chargeur Li-ion intégré | 5 € |
| Cellule 18650 protégée 3000 mAh + support | Autonomie jour/nuit | 8 € |
| MP1584 (réglé 5,4 V) | Abaisseur panneau → carte | 1 € |
| Diode Schottky 1N5817 | Anti-retour | < 1 € |
| MT3608 (réglé 12 V) | Élévateur batterie → ventilateur | 1 € |
| Sonde SHT30 étanche (0x44) | Température/humidité échappement | 7 € |
| Module SHT30 (0x45) | Température/humidité entrée d'air | 5 € |
| INA226 | Mesure de la production du panneau | 2 € |
| Arctic P12 Pro PST | Ventilateur d'extraction 120 mm, 4 fils PWM | 5 € |

Total ≈ 35-40 € hors séchoir. Le montage est essentiellement du câblage :
la charge, la protection batterie et la bascule panneau/batterie sont
intégrées au XIAO. Le panneau accepté va de 5 à 25 V.

Le séchoir lui-même n'est pas couvert ici : n'importe quel caisson à
capteur solaire et extraction forcée convient.

## Contenu du dépôt

    docs/cahier-des-charges.md     Spécification complète (matériel, régulation, web, OTA, recette)
    docs/05-architecture-web.md    Choix d'architecture de la page web embarquée, sourcés
    assets/                        Logo du module (clair et sombre)

À venir : `lib/juru-core/` et `src/` (programme PlatformIO), `hardware/`
(schéma de câblage, pièce 3D), guides d'installation et de réglage.

## Limites connues

- Conçu pour les végétaux (35-48 °C). Le séchage de viande ou poisson
  exige des températures qu'un séchoir solaire pur ne garantit pas.
- La mesure d'humidité reflète l'air, pas la teneur en eau des aliments ;
  la fin de séchage se confirme au toucher.

## Licences

Firmware : [GPL-3.0-or-later](LICENSE) · Documentation, schémas et pièces
3D : [CC-BY-SA 4.0](docs/LICENSE)

---

<a name="english"></a>

# juru-mistral — English

**Smart control for solar food dryers.**
A small ESP32 board measures humidity and temperature, drives the
extraction fan precisely, and reports everything in real time on an
embedded web page. Runs day and night on an 18650 battery charged by the
dryer's solar panel.

First module of the **JURU** ecosystem — *Just Useful Reusable Units*.

## Project status

*Updated October 22, 2026.*

| Step | Status |
|---|---|
| Full specification ([docs/cahier-des-charges.md](docs/cahier-des-charges.md), French) | ✅ Done |
| Component selection | ✅ Done |
| Firmware — v0.1.0 "it measures" | 🔧 In progress |
| Sensor test bench (temperature/humidity accuracy) | 🔧 In progress |
| v0.2.0 "it regulates": full control loop, presets, web page | ⏳ Planned |
| Assembly and outdoor testing | ⏳ Planned |
| v0.3.0 "it updates itself": over-the-air updates | ⏳ Planned |
| v1.0.0: stable release, full acceptance tests passed | ⏳ Planned |

**Branches.** `main` holds the clean, readable version. Ongoing work
happens on `dev` and reaches `main` once reviewed.

## The problem

On a typical forced-ventilation solar dryer, the fan is wired straight to
the panel: more sun, faster fan. That's a design flaw — right when the
chamber could reach its best temperature, the fan extracts at full speed
and cools it down. The dryer throttles itself, then stops dead at sunset,
leaving moist air to condense on the food overnight.

## The solution

juru-mistral decouples the fan from sunlight. A buffer battery and a
PID controller drive extraction based on what matters: the state of the
air inside the chamber.

- **Faster warm-up** — the fan stays off until the chamber reaches its
  useful window (hysteresis guard)
- **Optimized drying** — the PID modulates fan speed to extract moisture
  without dumping heat; air at 48 °C absorbs twice as much water as at
  35 °C
- **End-of-day purge** — running on battery, extraction continues after
  sunset as long as outgoing air is wetter than incoming air, preventing
  overnight condensation and mold
- **Overheat safety** — above 52 °C: forced extraction, or shutdown if
  incoming air is itself too hot
- **Food presets** — herbs, fruits, vegetables/mushrooms (max temperature
  and target humidity), selectable from the web page
- **Live monitoring** — inlet/outlet temperature and humidity, fan speed,
  solar panel output, alarms; home WiFi or standalone access point,
  page at `http://juru-mistral.local`

## Hardware

| Component | Role | Approx. price |
|---|---|---|
| Seeed XIAO ESP32-C3 | Controller, WiFi, built-in Li-ion charger | €5 |
| Protected 18650 cell 3000 mAh + holder | Day/night autonomy | €8 |
| MP1584 (set to 5.4 V) | Panel → board step-down | €1 |
| 1N5817 Schottky diode | Backfeed protection | < €1 |
| MT3608 (set to 12 V) | Battery → fan boost | €1 |
| Waterproof SHT30 probe (0x44) | Exhaust temperature/humidity | €7 |
| SHT30 module (0x45) | Inlet temperature/humidity | €5 |
| INA226 | Solar panel telemetry | €2 |
| Arctic P12 Pro PST | 120 mm 4-wire PWM extraction fan | €5 |

Total ≈ €35-40 excluding the dryer itself. Assembly is mostly wiring:
charging, battery protection and panel/battery power-path switching are
built into the XIAO. Accepts any panel from 5 to 25 V.

The dryer cabinet is out of scope: any box with a solar collector and
forced extraction will do.

## Repository contents

    docs/cahier-des-charges.md     Full specification (hardware, control, web, OTA, acceptance tests)
    docs/05-architecture-web.md    Embedded web page architecture choices, with sources
    assets/                        Module logo (light and dark)

Coming next: `lib/juru-core/` and `src/` (PlatformIO firmware),
`hardware/` (wiring diagram, 3D part), setup and tuning guides.

## Known limitations

- Designed for plant-based food (35-48 °C). Drying meat or fish requires
  temperatures a pure solar dryer cannot guarantee.
- Humidity readings reflect the air, not the food's water content; end of
  drying is confirmed by touch.

## Licenses

Firmware: [GPL-3.0-or-later](LICENSE) · Documentation, schematics and 3D
parts: [CC-BY-SA 4.0](docs/LICENSE)
