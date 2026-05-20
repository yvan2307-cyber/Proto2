# PROTO2 – Projet Technologique : Minecraft × Digilab × Node-RED × Discord
 
> Intégration en temps réel entre un serveur Minecraft, un tableau de bord Node-RED, un kit Digilab (LEDs + écran LCD) et un bot Discord.
 
---
 
## 📌 Description du projet
 
L'objectif est de créer un **système connecté et interactif** où les événements qui se produisent sur un serveur Minecraft déclenchent des réactions physiques et visuelles en temps réel, et où un **bot Discord permet de communiquer directement dans le jeu** :
 
- Les **16 LEDs rouges** du Digilab indiquent le nombre de joueurs connectés et l'état du serveur
- L'**écran LCD** affiche des informations dynamiques (joueurs, événements…)
- Un **tableau de bord Node-RED** centralise tout et permet de contrôler le système
- Un **bot Discord** relaie les messages écrits sur Discord directement dans le chat Minecraft
 
Tout est relié : **chaque événement influence quelque chose d'autre**. C'est ce qui fait la force du projet.
 
---
 
## 🔌 Architecture générale
 
```
      Discord
         |
   (Message libre)
         ↓
      Node-RED
     ↙    ↓    ↘
Minecraft Dashboard  Digilab
 (chat)  (UI web) (LEDs + LCD)
         ↑
  (Lecture des logs)
         |
   Serveur Minecraft
```
 
---
 
## 🎮 1. Données récupérées depuis Minecraft
 
On utilise la **lecture des logs du serveur** (`server.log` ou `latest.log`) — c'est la méthode la plus simple et la plus fiable, sans plugin supplémentaire.
 
### Événements détectés :
| Événement | Trigger dans le log |
|---|---|
| Joueur qui rejoint | `PlayerName joined the game` |
| Joueur qui quitte | `PlayerName left the game` |
| Mort d'un joueur | `PlayerName was slain / fell / drowned…` |
| Cycle jour/nuit | À calculer via timestamp ou commande RCON |
| Démarrage du serveur | `Done! For help, type "help"` |
 
### Méthode de lecture dans Node-RED :
- Nœud `file in` (surveillance continue du fichier log)
- Nœud `function` pour parser les lignes et détecter les événements
 
---
 
## 💡 2. Digilab – LEDs et écran LCD
 
### LEDs :
 
Le kit dispose de **16 LEDs rouges**. Elles fonctionnent comme une **barre de progression** représentant le nombre de joueurs actuellement connectés sur le serveur.
 
| Situation | Comportement des LEDs |
|---|---|
| 0 joueur connecté | 0 LED allumée |
| X joueurs connectés | X LEDs allumées à pleine intensité (sur 16 max) |
| 16 joueurs connectés | 16 LEDs allumées (barre complète) |
| Serveur hors ligne | LEDs restent dans leur état mais à **50% d'intensité** (effet veille) |
 
**Règle d'intensité :**
- **Serveur en ligne** → intensité PWM à 100% (`255`)
- **Serveur hors ligne** → intensité PWM réduite à 50% (`127`) — le nombre de LEDs allumées ne change pas, seule la luminosité diminue
 
Cela permet de distinguer visuellement d'un coup d'œil si le serveur tourne encore, sans avoir à consulter le dashboard.
 
Les LEDs sont contrôlées via le nœud `serial` de Node-RED (`node-red-node-serialport`), avec envoi de la valeur PWM et du nombre de LEDs à allumer.
 
### Écran LCD :
L'écran affiche en **rotation automatique** :
- Statut du serveur (en ligne / hors ligne)
- Nombre de joueurs connectés
- Dernier événement détecté
- Message personnalisé (depuis le dashboard ou depuis Discord)
 
---
 
## 📊 3. Dashboard Node-RED
 
Interface web accessible localement, construite avec `node-red-dashboard`.
 
### Éléments affichés :
- **Statut serveur** : en ligne / hors ligne
- **Compteur de joueurs** connectés
- **Journal d'événements** en direct (liste des dernières actions)
- **Aperçu LCD** : ce qui est actuellement affiché sur l'écran physique
- **Journal Discord** : historique des messages envoyés depuis Discord vers Minecraft
 
## 🤖 4. Intégration Discord → Minecraft
 
Le bot Discord est une **composante centrale** du projet. Il permet à n'importe quel membre du serveur Discord d'envoyer un message libre qui apparaîtra directement dans le chat du serveur Minecraft.
 
### Flux de données :
 
```
Utilisateur Discord
       |
  Écrit un message
  dans #minecraft-chat
       ↓
  Bot Discord (Node-RED)
  reçoit le message via webhook
       ↓
  Node-RED formate la commande :
  /say [Discord] <pseudo> : <message>
       ↓
  Envoi au serveur Minecraft via RCON
       ↓
  Le message apparaît dans le chat du jeu
```
 
### Fonctionnement détaillé :
 
1. Un utilisateur écrit un message dans un channel Discord dédié (ex. `#minecraft-chat`)
2. Le bot Discord capte ce message via un **webhook entrant** configuré dans Node-RED
3. Node-RED formate le message en commande Minecraft : `/say [Discord] Pseudo : message`
4. La commande est envoyée au serveur Minecraft via **RCON** (Remote Console)
5. Le message s'affiche dans le chat du jeu pour tous les joueurs connectés
 
### Configuration dans Node-RED :
 
| Nœud | Rôle |
|---|---|
| `http in` | Reçoit les webhooks Discord |
| `function` | Formate le message en commande `/say` |
| `exec` ou RCON | Envoie la commande au serveur Minecraft |
 
### Exemple de message en jeu :
 
```
[Serveur] [Discord] Alex : Quelqu'un peut m'ouvrir la porte ?
```
 
### Configuration Discord requise :
 
- Créer un bot Discord via le [Discord Developer Portal](https://discord.com/developers)
- Activer les **intents de messages** (Message Content Intent)
- Configurer un channel dédié (ex. `#minecraft-chat`) pour limiter les messages relayés
- Ajouter le token du bot dans la configuration Node-RED
 
---
 
## 🧰 5. Outils et bibliothèques Node-RED utilisés
 
| Nœud | Utilisation |
|---|---|
| `node-red-dashboard` | Interface utilisateur web |
| `node-red-node-serialport` | Communication avec le Digilab |
| `file in` | Lecture continue des logs Minecraft |
| `function` | Analyse et traitement des événements |
| `http in` | Réception des webhooks Discord |
| `exec` | Envoi de commandes RCON au serveur Minecraft |
 
---
 
## 🧪 Scénario de démonstration
 
Voici deux exemples complets du fonctionnement du système :
 
### Scénario 1 — Joueur qui rejoint le serveur
 
1. Un joueur rejoint le serveur Minecraft
2. Node-RED détecte la ligne dans le log
3. Une **LED supplémentaire s'allume** à pleine intensité (ex. : 3 joueurs → 3 LEDs allumées)
4. L'**écran LCD** affiche : `Steve a rejoint !`
5. Le **dashboard** log l'événement et met à jour le compteur de joueurs
 
### Scénario 1b — Serveur qui s'éteint
 
1. Le serveur Minecraft s'arrête
2. Node-RED détecte l'absence de nouvelles lignes dans le log (timeout)
3. Les LEDs déjà allumées **restent allumées** mais leur **intensité est divisée par 2**
4. L'**écran LCD** affiche : `Serveur hors ligne`
5. Le **dashboard** met à jour le statut serveur
 
### Scénario 2 — Message depuis Discord
 
1. Un utilisateur écrit `Salut tout le monde !` dans le channel `#minecraft-chat` sur Discord
2. Node-RED reçoit le message via le webhook
3. Node-RED formate la commande : `/say [Discord] Alex : Salut tout le monde !`
4. La commande est envoyée au serveur Minecraft via RCON
5. Le message apparaît dans le chat pour tous les joueurs en jeu
6. Le **dashboard** log l'événement dans le journal Discord
 
➡️ Ce sont deux boucles systèmes complètes — exactement ce qui est attendu.
 
---
 
## 📁 Structure du dépôt
 
```
PROTO2-projet/
├── flows/          # Flows Node-RED exportés (.json)
├── digilab/        # Code Arduino / scripts Digilab
├── discord/        # Configuration et scripts du bot Discord
├── docs/           # Schémas, captures d'écran
└── README.md       # Ce fichier
```