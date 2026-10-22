# Architecture de la page web embarquée

Ce document explique les choix d'architecture de l'interface web de
juru-mistral et leurs justifications. Il s'adresse aux contributeurs :
avant de proposer de remplacer un mécanisme par un autre « plus moderne »,
lisez pourquoi il est là.

## Vue d'ensemble

```
XIAO ESP32-C3 (400 KB RAM, mono-cœur RISC-V)
│
├── ESPAsyncWebServer (fork ESP32Async)
│     ├── GET  /            → page HTML unique (PROGMEM, gzip)
│     ├── GET  /api/status  → JSON temps réel
│     └── POST /api/config  → preset, consigne, gains PID, WiFi
│
├── mDNS → http://juru-mistral.local
└── WiFi STA (tentative 15 s) → fallback point d'accès
```

## 1. Serveur asynchrone — et pourquoi ce fork précis

Un serveur web classique (`WebServer.h`) est bloquant : pendant qu'il
sert une requête, le `loop()` est gelé. Or la boucle principale fait
tourner un régulateur avec des responsabilités de sécurité : l'alarme
52 °C doit réagir même si trois navigateurs sont connectés. Le serveur
asynchrone traite les requêtes sur sa propre tâche ; la boucle de
régulation n'est jamais suspendue par le réseau.

**Pourquoi le fork ESP32Async.** Ce n'est pas une préférence, c'est une
nécessité. Le dépôt d'origine (me-no-dev) est archivé depuis janvier 2025.
Avec les cœurs ESP32 récents, les anciennes versions provoquent l'erreur
`assert failed: tcp_alloc` et un redémarrage en boucle. La version
maintenue est le fork ESP32Async (ESPAsyncWebServer ≥ 3.6.2,
AsyncTCP ≥ 3.3.2), qui a absorbé les améliorations des forks
intermédiaires.

**Discipline imposée.** Le serveur ne tourne pas sur le thread de la
boucle : interdiction d'appeler `yield()`, `delay()` ou toute fonction
qui les utilise dans les callbacks. Les handlers lisent l'état partagé et
répondent, rien de plus. Sur un mono-cœur comme le C3, un callback lourd
retarde la pile WiFi elle-même.

## 2. Polling JSON à 2 s, plutôt que WebSocket ou SSE

Trois mécanismes existent pour rafraîchir la page : le polling HTTP, les
Server-Sent Events (SSE) et les WebSockets. Les deux derniers poussent les
données du serveur vers le client ; le polling redemande. juru-mistral
utilise le plus simple des trois, délibérément.

**L'argument décisif est le rythme des données.** Le push se justifie
quand les événements sont imprévisibles ou rapides (jeu, chat). Les
mesures de juru-mistral arrivent à cadence fixe de 2 s et la physique
sous-jacente (séchage) évolue en dizaines de minutes. Un
`fetch('/api/status')` toutes les 2 s délivre la même fraîcheur perçue
qu'un push, pour un coût d'architecture bien moindre :

- **Zéro état côté serveur.** Une connexion persistante (WS/SSE) doit
  être suivie et nettoyée : le fork expose `cleanupClients()`,
  `setCloseClientOnQueueFull()`, `SSE_MAX_QUEUED_MESSAGES` — autant de
  mécanismes qui n'ont pas lieu d'exister en polling (requête, réponse,
  connexion fermée).
- **Robustesse gratuite en usage jardin.** Le client type est un
  téléphone qui se verrouille, sort de portée, revient. Une connexion
  WebSocket rompue ne se rétablit pas seule — il faut l'implémenter.
  Le polling n'a rien à reconnecter : la prochaine requête passe ou
  échoue, et l'interface affiche « hors ligne » sur simple timeout.
- **Mémoire bornée.** Chaque socket persistant consomme des buffers
  lwIP sur les 400 KB du C3. Des onglets oubliés en polling = zéro
  socket permanent.

SSE serait le second choix honorable (implémentation serveur simple,
reconnexion native côté navigateur via `EventSource`) — noté comme
évolution v2 si un besoin sub-seconde apparaissait.

**Pourquoi JSON et pas du HTML régénéré.** Séparer données et
présentation permet à la même API de servir un jour un autre module
JURU, un script de logging externe ou Home Assistant, sans toucher
au firmware.

## 3. Page unique, tout inline, zéro CDN

La page est un seul document HTML avec CSS et JS inclus, stocké dans
le firmware, sans aucune ressource externe.

**Le mode AP est un intranet sans internet.** Un point d'accès ESP32
ne route pas vers l'extérieur : toute ressource CDN (bibliothèque de
graphes, police web) serait inaccessible — et l'AP est précisément le
mode « au fond du jardin ». Tout inline = comportement identique en STA
et en AP. Le graphe des 60 dernières minutes est donc dessiné en
`<canvas>` natif (~50 lignes de JS vanilla), pas avec une bibliothèque
externe.

**Un fichier = pas de filesystem à gérer.** L'alternative (LittleFS +
fichiers séparés) ajoute une étape de flash, un risque de
désynchronisation firmware/FS et de la complexité de build. La page vit
en `PROGMEM` et est servie directement depuis la flash. Pré-compressée en
gzip, une page de 25-30 KB tombe à 6-8 KB.

## 4. STA d'abord, AP en secours, mDNS

La séquence (15 s de tentative sur le WiFi domestique, sinon point
d'accès) suit le double usage réel : séchoir près de la maison → confort
du réseau existant ; séchoir éloigné → autonomie totale. Le mDNS
(`juru-mistral.local`) évite de connaître l'adresse IP.

Premier démarrage = point d'accès de configuration qui **force** la
définition d'un mot de passe ; aucun mot de passe par défaut dans le
code. Le firmware étant publié sous GPL, tout secret dans le source
serait public par construction.

## 5. Le graphe en RAM, perdu au redémarrage : assumé

Ring buffer de 60 minutes (1800 points × ~16 octets ≈ 30 KB). Écrire en
flash toutes les 2 s userait la NVS (endurance ~100 000 cycles par
secteur) pour un bénéfice marginal : l'historique long terme est hors
périmètre v1, et le CSV série existe pour les sessions de réglage.

---

Chaque choix suit la même logique que le matériel : **le composant le
plus simple qui satisfait le besoin réel, dans sa version maintenue**,
avec les chemins d'évolution notés mais pas construits.

## Sources

- [ESP32Async/ESPAsyncWebServer](https://github.com/ESP32Async/ESPAsyncWebServer) — fork maintenu, notes de migration
- [ESP32 Web Server — guide (Random Nerd Tutorials)](https://randomnerdtutorials.com/esp32-web-server-beginners-guide/)
- [ESP32 Server-Sent Events (Random Nerd Tutorials)](https://randomnerdtutorials.com/esp32-web-server-sent-events-sse/)
- [WebSockets vs SSE (Ably)](https://ably.com/blog/websockets-vs-sse)
- [SSE vs WebSockets (germano.dev)](https://germano.dev/sse-websockets/)
