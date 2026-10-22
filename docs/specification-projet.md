# JURU — Spécification du projet

*JURU — Just Useful Reusable Units.* Version du 22 octobre 2026.

Ce document décrit le projet JURU dans son ensemble : pourquoi il existe,
les règles qu'il se fixe, l'architecture commune et les modules prévus.
La spécification détaillée du premier module (le séchoir) est dans
[cahier-des-charges.md](cahier-des-charges.md).

---

## 1. But

Automatiser des objets qui existent déjà — un séchoir solaire, une ruche,
un arrosage — au lieu de les remplacer par des produits neufs.

Un module JURU se greffe sur l'existant. Il ajoute des mesures
(température, humidité, poids, énergie…), une régulation quand il y en a
besoin, et une page web consultable depuis un téléphone.

## 2. Principes

1. **Partir de l'existant.** On automatise ce qu'on a déjà ; on ne
   reconstruit pas l'objet.
2. **Peu de pièces, peu chères.** Chaque composant doit justifier sa
   présence : le plus simple qui satisfait le besoin réel. Repère : le
   module séchoir revient à environ 35-40 € hors séchoir.
3. **Récupération d'abord.** Composants de récupération et stock existant
   avant tout achat (panneau, ventilateur, batterie…).
4. **À la portée de chacun.** Montage essentiellement par câblage, outils
   courants, documentation en français.
5. **Réutilisable.** Le même boîtier, la même carte et le même socle
   logiciel servent d'un projet à l'autre (le *R* de JURU).
6. **Autonome.** Fonctionne sans internet ni service en ligne ; pas
   d'application à installer, une page web suffit.
7. **Ouvert.** Code sous GPL-3.0, documentation et pièces 3D sous
   CC-BY-SA 4.0. Aucun secret dans le code publié.
8. **Spécifier avant de construire.** Chaque module a un cahier des
   charges écrit avant le code, et chaque choix important est justifié.

## 3. Architecture commune

### 3.1 Carte

Carte unique pour tous les modules : **Seeed XIAO ESP32-C3**. WiFi intégré,
chargeur de batterie Li-ion intégré, quelques euros, format timbre-poste.

### 3.2 Socle logiciel `juru-core` et partie métier

Séparation stricte, imposée par l'arborescence du code :

| Partie | Contenu | Règle |
|---|---|---|
| `lib/juru-core/` (socle) | WiFi (domestique puis point d'accès de secours), portail de configuration, réglages en mémoire (NVS), serveur web et API, suivi de l'alimentation (INA226), télémétrie santé, mises à jour sans câble, registre de grandeurs | Aucune référence à un module particulier |
| `src/` (métier) | Capteurs du module, régulation, réglages propres au module | Ne touche jamais au WiFi ni au serveur directement ; passe par le socle |

Le **registre de grandeurs** décrit chaque mesure ou réglage (nom, unité,
type, modifiable ou non, plage). La page web et l'API se génèrent depuis
ce registre : un nouveau module déclare ses grandeurs, il n'écrit pas
d'interface.

Le socle sera extrait en bibliothèque partagée quand un deuxième module
l'utilisera. Jusque-là, il vit dans le dépôt du séchoir.

### 3.3 Communication

- Page web embarquée, consultable depuis un téléphone, sans internet.
- API JSON (`/api/status`, `/api/config`) : tout ce que fait la page passe
  par l'API.
- Home Assistant : préparé dès la v1 (noms de grandeurs stables), branché
  en v2.
- Le transport est interchangeable : WiFi aujourd'hui, LoRa envisagé pour
  les modules isolés.

### 3.4 Alimentation

Panneau solaire et/ou batterie 18650, mesure de l'énergie par INA226 sur
chaque module. Le comportement en batterie basse est spécifié : le module
s'éteint proprement, sans redémarrer en boucle.

## 4. Le boîtier modulaire

*En conception.*

Le but est un petit boîtier qui contient la carte et ses composants, et
qu'on **clipse sur le support** : le séchoir, une ruche, ou autre chose.
On peut en installer un seul ou plusieurs.

| Élément | Choix actuel |
|---|---|
| Boîtier hôte | Paramétrique, au format du XIAO |
| Extensions | Modules qui s'accrochent au boîtier : batterie, régulateur de tension, autres à définir |
| Fixation des extensions | Rail en queue d'aronde avec cran de verrouillage (v1), évolution prévue vers des connecteurs à emboîtement direct |
| Connexions | XT30 pour l'alimentation, connecteurs mâle/femelle pour les signaux |
| Conception | OpenSCAD (fichiers paramétriques) |
| Fabrication | Impression 3D ; PETG pour les pièces exposées à la chaleur |

Décisions ouvertes :
- type de module 18650 ;
- disposition des extensions : en colonne ou sur plusieurs faces.

## 5. Modules

| Module | Rôle | État |
|---|---|---|
| **juru-mistral** | Automatisation d'un séchoir solaire alimentaire existant : ventilateur piloté selon la température et l'humidité, batterie tampon, purge du soir, sécurité surchauffe | Spécifié ; programme v0.1.0 en cours — [cahier des charges](cahier-des-charges.md) |
| **juru-apis** | Suivi de ruche : température du couvain, humidité dedans/dehors, poids, sons (détection d'essaimage) | Architecture définie, à construire |
| Arrosage automatique | Arrosage selon l'humidité de la terre | Idée |
| Station météo | Température, pluie, vent | Idée |

### 5.1 juru-apis — grandes lignes

- Fonctionne hors ligne : mesure, garde en mémoire, dort entre deux
  mesures pour tenir sur batterie.
- Récupération des données à la visite : le téléphone se connecte au
  point d'accès du boîtier et télécharge l'historique.
- Capteurs visés : température du couvain (DS18B20 ou I2C), humidité
  intérieure/extérieure (SHT), poids (jauges + NAU7802), micro + analyse
  de fréquence pour anticiper l'essaimage, INA226.
- Heure : module DS3231, ou heure fournie par le téléphone à la
  synchronisation.
- À trancher : la pesée fait-elle partie de la v1.

## 6. Conventions

- **Noms** : `juru-<nom>` pour chaque module (`juru-mistral`, `juru-apis`).
- **Versions** : SemVer. Série 0.x pendant le développement ; la v1.0.0
  gèle l'API, le schéma de réglages et les noms de grandeurs — c'est le
  contrat sur lequel s'appuient les autres modules.
- **Branches** : `main` = version propre à consulter ; `dev` = travail en
  cours, reporté sur `main` une fois relu.
- **Bibliothèques** : versions épinglées ; uniquement des bibliothèques
  maintenues et de licence compatible GPL-3.0.
- **Code embarqué autonome** : page web entièrement incluse dans le
  programme, aucune ressource téléchargée depuis internet.

## 7. Références

- [Séchoir solaire de David Mercereau](https://david.mercereau.info/) — base physique du séchoir
- [Hiveeyes / Open Hive](https://hiveeyes.org/) — projet ouvert de suivi de ruche, le plus proche de l'esprit JURU
- BroodMinder — modèle commercial « synchronisation à la visite » qui valide l'approche de juru-apis
