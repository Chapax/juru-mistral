# juru-mistral — Cahier des charges firmware v1

Document de spécification pour l'implémentation.

Statut : architecture matérielle FIGÉE, périmètre logiciel FIGÉ.
Ce document est la référence unique. Aucun code ici — uniquement le QUOI
et le POURQUOI. Le COMMENT appartient à l'implémentation.

> **Révisions**
> - Juillet 2026 : version consolidée (14 sections).
> - Septembre 2026 : alimentation — MP1584 réglé à 5,4 V suivi d'une diode
>   Schottky 1N5817 anti-retour (la tension de seuil de la diode est
>   compensée par le réglage à 5,4 V). Voir §1.

---

## 0. Toolchain et organisation du code

- PlatformIO sous VS Code. Plateforme espressif32 récente, cœur Arduino
  3.x (pioarduino). PAS de sketch .ino monolithique.
- Structure imposée :
  - `lib/juru-core/` : le SOCLE réutilisable (voir §12) — aucune
    référence au séchage.
  - `src/` : le MÉTIER mistral (capteurs, régulation, presets) — ne
    touche jamais directement WiFi/serveur ; passe par le socle.
  - `platformio.ini` : versions de libs ÉPINGLÉES (fork ESP32Async par
    URL Git), `board_build.partitions = partitions.csv`.
  - `partitions.csv` versionné dans le repo : 2 slots app (~1,9 MB
    chacun, type "Minimal SPIFFS") + NVS. Décision irréversible sans
    effacement complet → figée dès le premier flash.
- `fw_version` injecté à la compilation depuis le tag Git ; visible
  dans /api/status, la page web et l'en-tête CSV.
- Compatibilité IDE Arduino : non prioritaire, documentée "best effort".
- Environnement `[env:native]` : tests unitaires (Unity) de la logique
  pure (PID, machine à états, humidité absolue) sur PC.

## 1. Architecture matérielle (figée)

```
Panneau solaire 12V (Voc ~18-22V, 5-9W estimés, À CONFIRMER par mesure)
        │
   [INA226, shunt R100, adresse I2C 0x40]   ← mesure V et I panneau
        │
   MP1584 réglé 5,4V  (entrée 4,5-28V — marge OK sur Voc panneau)
        │
   Diode Schottky 1N5817 (anti-retour ; ni 1N400x trop lente, ni SS34 surdimensionnée)
        │
   XIAO ESP32-C3, pin 5V
        ├── ETA4054 intégré : charge 18650 (~370mA), power path automatique
        └── pads BAT+/BAT- (dos de carte) ── porte-batterie 18650 protégée 3000mAh
                    │
              MT3608 réglé 12,0V ── Arctic P12 Pro PST
                                     (alim 12V permanente, vitesse par fil PWM)
```

Capteurs sur bus I2C unique (SDA/SCL) :
- Sonde SHT30 étanche inox, câble 2m — adresse 0x44 (fixe) — ÉCHAPPEMENT
- Module SHT30 PCB violet — adresse 0x45 (jumper AD à ressouder côté VDD) — ENTRÉE D'AIR
- INA226 — 0x40 — côté panneau, EN AMONT du MP1584

Ventilateur Arctic P12 Pro PST (ACFAN00307A, pack de 5) : 120 mm,
600-3000 RPM, pression statique 6,9 mmH2O, mode 0 RPM sous 5 % PWM.
4 fils : +12V/GND depuis MT3608, PWM (bleu) en 3,3V logique 25 kHz,
tachy (jaune) OPTIONNEL — si utilisé, pull-up 10k vers 3V3 uniquement,
JAMAIS vers 5V/12V.
L'ouverture d'extraction sera agrandie à 120 mm — la pièce 3D fixe la
découpe (gabarit de perçage dans hardware/3d/, imprimable ou PDF 1:1).

Placement physique :
- Sonde 0x44 : dans le flux d'échappement, 2-3 cm EN AMONT des pales
  (côté caisson), tête perpendiculaire au flux. Support = pièce 3D.
- Module 0x45 : à l'entrée d'air, abrité, hors soleil direct.
- Boîtier électronique : hors chambre chaude si possible.
- Condensateur réservoir ≥ 470 µF près du XIAO côté alim (brownout WiFi
  sur batterie basse).

## 2. Mapping GPIO (XIAO ESP32-C3)

| Fonction | Pin XIAO | GPIO |
|---|---|---|
| I2C SDA | D4 | GPIO6 |
| I2C SCL | D5 | GPIO7 |
| PWM ventilateur | D10 | GPIO10 |
| Tachy ventilateur (option) | D9 | GPIO9 |
| Coupure alim MT3608 (option, MOSFET high-side) | D3 | GPIO5 |
| Mesure tension batterie (pont diviseur 2×220k sur BAT+) | D0/A0 | GPIO2 |

La mesure de tension batterie passe d'option à RECOMMANDÉE (comportement
dégradé §8 et critère d'autonomie §10 en dépendent).

## 3. Contraintes plateforme ESP32-C3 (impératives)

Budget SRAM (400 KB total, ~320 KB utiles après WiFi/runtime) :
- WiFi + système : ~100 KB (incompressible)
- Serveur async + buffers TCP : ~40-60 KB en pointe (3 clients)
- Ring buffer graphe 60 min : ~30 KB (dimensionné à la COMPILATION)
- État système + scratch JSON : ~5 KB
- Session TLS (menu updates) : 40-50 KB TRANSITOIRES → mode exclusif §7

Règles :
- Aucune allocation dynamique répétitive ; pas de String Arduino dans
  les chemins chauds ; buffers statiques.
- ArduinoJson en sérialisation streaming vers la réponse HTTP.
- Télémétrie santé OBLIGATOIRE dans /api/status : free_heap,
  min_free_heap (canari de fuite), rssi, uptime_s, vbat_mv, fw_version.

Flash 4 MB : partition 2 slots OTA (~1,9 MB/slot). Firmware estimé
~1,2 MB : marge OK, surveiller à chaque release (taille dans la CI).

Mono-cœur : AUCUN yield()/delay()/travail lourd dans les callbacks du
serveur (ils retardent la pile radio). Boucle principale en scheduler à
ticks non-bloquant (millis) ; aucune attente active nulle part.

## 4. Bibliothèques imposées

- Adafruit SHT31 (pilote les SHT30, 0x44/0x45)
- INA226 de Rob Tillaart (PAS la lib Adafruit INA219)
- PID maison (~30 lignes, anti-windup maîtrisé) — testable en natif
- ESPAsyncWebServer + AsyncTCP : fork ESP32Async UNIQUEMENT
  (≥ 3.6.2 / ≥ 3.3.2), épinglés dans platformio.ini
- Preferences (NVS), ArduinoJson
- HTTPClient + certificats racine embarqués pour le menu updates (§7)
- PAS d'ElegantOTA (licence incompatible GPL-3.0) — handler Update natif

PWM ventilateur : LEDC, 25 kHz, 8 bits.

## 5. Boucle de régulation

### 5.1 PID humidité

- Entrée : RH échappement (0x44) ; consigne : RH du preset (défaut 40 %)
- Sortie : duty 0-100 % ; RH > consigne → duty augmente
- Échantillonnage 2 s ; anti-windup par clamp de l'intégrateur
- Seuil PWM minimum mécanique 15 % (jamais de duty entre 1 et 14 %)
- Sonde étanche : réponse RH 30-60 s → boucle LENTE, gains prudents

### 5.2 Garde thermique (hystérésis) — priorité sur le PID

Ventilateur AUTORISÉ seulement si :
- T_échappement > T_entrée + 3 °C ET T_échappement > 30 °C

Sortie : T_échappement < T_entrée + 1 °C OU < 28 °C.
But : accumulation thermique matinale, pas de brassage d'air froid.

### 5.3 Purge de fin de journée

Déclenchement quand la garde ferme en fin de journée, SI
humidité ABSOLUE échappement > humidité ABSOLUE entrée + marge.
Calcul en g/m³ (formule de Magnus depuis T et RH) — jamais de
comparaison de RH à températures différentes.
Comportement : duty fixe 25-30 % (PLAFOND ABSOLU 40 % — conso du P12
Pro), réévaluation 5 min, arrêt sur convergence (delta < 0,5 g/m³)
ou après 4 h.

### 5.4 Alarme surchauffe — double régime, priorité absolue

- T_échapp > 52 °C ET T_entrée < 48 °C : EXTRACTION FORCÉE (duty 100 %)
- T_échapp > 52 °C ET T_entrée ≥ 48 °C : COUPURE ventilateur
- Flag loggé + web ; hystérésis de sortie à 50 °C ; écrase tout.

### 5.5 Machine à états

IDLE → DRYING → PURGE → IDLE ; ALARM accessible de partout, retour
sous hystérésis. État exposé (web + CSV). Note tuning : par temps
faible, cycles marche/arrêt duty 0 ↔ 15-20 % = comportement NORMAL
(plancher 600 RPM du P12 Pro), documenté dans docs/02.

## 6. Presets aliments (NVS, éditables via web)

| Preset | T max garde | RH cible |
|---|---|---|
| Herbes / feuilles | 42 °C | 35 % |
| Fruits | 48 °C | 40 % |
| Légumes / champignons | 48 °C | 40 % |
| Custom | éditable | éditable |

"T max garde" dépassée → duty minimum 40 % (pré-régulation douce).
Pas de viande/poisson en v1 (documenté).

## 7. Connectivité, interface web, mises à jour

### 7.1 Réseau

- STA 15 s sur credentials NVS → fallback AP `juru-mistral-<mac6>`.
- Premier démarrage : AP ouvert de configuration, FORCE la définition
  d'un mot de passe. Aucun secret codé en dur (source GPL public).
- mDNS : http://juru-mistral.local (ou -<mac6> si collision).
- device_id unique dérivé MAC : `juru-mistral-a1b2c3` (SSID, mDNS,
  unique_id HA).

### 7.2 Page web embarquée

- Page unique, HTML/CSS/JS inline, PROGMEM gzip, ZÉRO CDN (mode AP =
  intranet sans internet). Favicon SVG inline (assets/favicon.svg).
- Contenu : valeurs temps réel (T/RH entrée+échappement, humidités
  absolues, duty, état, V/I/P panneau, santé §3), sélecteur preset,
  consigne + gains PID, bandeau alarme, graphe 60 min en canvas natif
  (ring buffer RAM, perdu au reboot — assumé).
- Rafraîchissement : polling fetch /api/status toutes les 2 s
  (choix justifié dans [05-architecture-web.md](05-architecture-web.md)).
- API : GET /api/status ; POST /api/config. RÈGLE : toute action
  possible depuis la page passe par l'API (aucune logique enfouie
  dans le JS) — condition de la commandabilité HA (§9).

### 7.3 OTA (exigence v1)

Niveau 1 — ArduinoOTA : upload_protocol espota depuis PlatformIO
(développement sans câble).

Niveau 2 — OTA web : endpoint /update protégé par le mot de passe
device, upload .bin multipart, handler Update NATIF (pas ElegantOTA).

Garde-fous communs :
- Rollback automatique : image "pending verify", validation santé
  (WiFi up + 3 capteurs répondent) sous 60 s sinon retour arrière.
- OTA REFUSÉ en état ALARM et en purge sur batterie < 3,5 V.
- fw_version partout (§0).

### 7.4 Menu de mises à jour GitHub (v0.3)

- GET /api/updates : le device interroge
  api.github.com/repos/Chapax/juru-mistral/releases, renvoie la liste
  parsée (tag, date, notes, url asset, courante marquée).
- UI : onglet "Mises à jour", patch notes en rendu Markdown MINIMAL
  (gras/listes, ~30 lignes JS), bouton installer par version.
- Téléchargement : HTTPS OBLIGATOIRE avec certificats RACINE embarqués
  (bundle 2-3 racines : api.github.com + CDN des assets). JAMAIS
  setInsecure(). Suivi des redirections 302 activé explicitement.
- Vérification SHA-256 de l'asset AVANT validation de l'image.
- Mode exclusif pendant l'opération (bandeau UI, TLS = 40-50 KB RAM) ;
  mêmes interdictions qu'en 7.3.
- Pipeline CI (GitHub Actions) : sur tag v*, build PlatformIO, asset
  nommé `juru-mistral-firmware.bin`, checksum SHA-256 attaché, taille
  du binaire vérifiée < 1,8 MB.

## 8. Comportement énergie

- Pas de deep sleep en v1 (serveur joignable + purge nocturne).
  Réservation v2 : sommeil paramétrable 23h-6h.
- Comportement dégradé batterie basse (vbat via A0) : sous 3,4 V →
  boost ventilo coupé, cadence de mesure réduite, page web en mode
  minimal. Le système meurt proprement, pas en boucle de reboot.
- Option GPIO5 (coupure MT3608 à l'arrêt prolongé) : réservée, non
  implémentée v1 — le mode 0 RPM natif du P12 Pro garantit déjà
  l'arrêt franc à duty 0.

## 9. Préparation Home Assistant (v1 prépare, v2 implémente)

1. Séparation état/transport : struct SystemState UNIQUE ; le handler
   JSON et le futur publisher MQTT ne sont que des sérialiseurs.
2. Nommage des grandeurs stable dès la v1, snake_case avec unité :
   temp_exhaust_c, temp_inlet_c, rh_exhaust_pct, rh_inlet_pct,
   abs_humidity_delta_gm3, fan_duty_pct, panel_v, panel_i_ma,
   panel_power_w, state, alarm, vbat_mv, free_heap, min_free_heap,
   rssi, uptime_s, fw_version. Registre de grandeurs (§12) porte
   nom + unité + device_class HA + writable/plage.
3. NVS : clés réservées mqtt_host, mqtt_port, mqtt_user, mqtt_pass,
   mqtt_enabled (défaut off). Section UI "Intégrations" présente,
   grisée v2.
4. Commandabilité : preset, consigne, gains, ventilo forcé — tout via
   /api/config (le futur command_topic MQTT appellera les MÊMES
   fonctions internes).
5. unique_id = device_id MAC (§7.1).
6. Note v2 : payloads MQTT Discovery > 256 octets — dimensionner le
   buffer client (PubSubClient par défaut insuffisant ; préférer un
   client async type espMqttClient, à trancher en v2).

## 10. Critères d'acceptation (recette avant livraison)

1. Scan I2C : 0x40, 0x44, 0x45 présents ; test croisé des deux SHT30
   côte à côte avant installation : écart < 1 °C / < 4 %RH.
2. duty→RPM monotone de 15 à 100 %.
3. Garde thermique : immobile à froid, départ spontané à ambiant+3 °C.
4. PID : consigne 40 %RH tenue ±5 % en régime établi, sans oscillation
   entretenue.
5. Purge : déclenchement au crépuscule sur delta d'humidité absolue,
   arrêt autonome (convergence ou 4 h).
6. Alarme : décapeur thermique sur sonde → extraction forcée < 5 s,
   flag web, retour sous 50 °C.
7. Autonomie : nuit complète avec purge 4 h → vbat > 3,6 V au matin.
8. Web joignable en STA et AP ; presets persistants après coupure.
9. OTA : mise à jour via /update ET via ArduinoOTA réussies ; rollback
   vérifié (flash d'une image volontairement invalide → retour auto).
10. min_free_heap stable sur 72 h de fonctionnement continu.

## 11. Plan de versions

- v0.1.0 "ça mesure" : socle juru-core, 3 capteurs, CSV, ArduinoOTA.
  Critères 1-2.
- v0.2.0 "ça régule" : machine à états complète, presets, page web +
  graphe. Critères 3-6. (version des sessions de réglage ; utilisable
  telle quelle)
- v0.3.0 "ça se met à jour" : OTA web, menu GitHub Releases, CI.
  Critère 9. (le menu se teste de bout en bout à la release suivante,
  par construction)
- v1.0.0 "c'est livré" : AUCUNE fonctionnalité nouvelle. v0.3.x ayant
  passé la recette complète (critères 1-10 + un séchage réel réussi).
  Déclare STABLES : API JSON, schéma NVS, noms de grandeurs — contrat
  du socle pour les futurs modules JURU. Breaking change = v2.
- SemVer strict ; jalons en Milestones GitHub ; badge de release dans
  le README.

## 12. Architecture socle / métier (fondation écosystème JURU)

Frontière PHYSIQUE imposée par l'arborescence PlatformIO :

- `lib/juru-core/` : SystemState générique + REGISTRE DE GRANDEURS
  (nom, unité, device_class, writable, plage), WiFi STA→AP + portail
  config + mDNS, serveur web + /api/*, NVS à schéma versionné,
  télémétrie santé, device_id MAC, hooks MQTT Discovery (v2),
  scheduler à ticks, OTA + menu updates.
  → AUCUNE référence au séchage. La page web et le JSON se GÉNÈRENT
  depuis le registre.
- `src/` (métier mistral) : drivers capteurs, PID, machine à états,
  presets. Déclare ses grandeurs au registre ; ne touche jamais
  WiFi/serveur directement.

Extraction de juru-core en librairie partagée : PAS en v1 (un seul
consommateur). La frontière rend l'extraction mécanique le jour où
juru-apis ou juru-pluvio démarre.

## 13. Périmètre explicitement EXCLU de la v1

- Séchage viande/poisson (sécurité alimentaire non garantie)
- MQTT / Home Assistant actif (préparé §9, implémenté v2)
- Historique persistant (SD, LittleFS long terme)
- Deep sleep ; multi-ventilateurs ; OTA automatique non sollicité

## 14. Points restant à trancher (non bloquants pour le dev)

- Voc et Isc réels du panneau (valide la marge du MP1584 — si Voc > 26V,
  remplacer par un module supportant plus, improbable)
