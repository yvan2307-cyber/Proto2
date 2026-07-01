# 🛠️ PROJET MINECRAFT CONNECTÉ
### *Automatisation serveur via Node-RED, Discord, RCON et interface physique Raspberry Pi*

Document de présentation et spécifications détaillées.

---

## 1. Introduction & Objectifs

Ce document présente l’architecture et le fonctionnement d’un écosystème connecté, articulé autour de **Node-RED** comme couche d’orchestration centrale. Ce projet jette un pont entre un serveur Minecraft et plusieurs interfaces de contrôle et de notification :

* **Un bot Discord** pour les commandes à distance et les alertes.
* **Une console physique (Raspberry Pi)** équipée de boutons poussoirs (GPIO) et d'un écran LCD.
* **Le protocole RCON** pour l'exécution instantanée des commandes sur le serveur de jeu.

### 🎯 Objectif principal
Permettre à des joueurs ou des animateurs de déclencher des actions dynamiques en jeu (*changement de mode, gestion du temps, distribution d'équipements, invocation de boss*) depuis différents points d'entrée (Discord ou boutons physiques), tout en centralisant les retours d'information (statuts, confirmations) sur Discord et sur l'afficheur LCD local.

### 💻 Technologies & Rôles

| Composant | Rôle technologique |
| :--- | :--- |
| **Node-RED** | **Cœur de l'architecture** : reçoit les événements, applique la logique métier et orchestre les sorties. |
| **Protocole RCON** | **Passerelle Minecraft** : exécute les commandes serveur (`/gamemode`, `/time`, `/give`, `/summon`). |
| **Discord API** | **Interface distante** : nœuds *Message* et *Message Manager* pour l'interaction textuelle. |
| **Raspberry Pi 5 + GPIO** | **Interface matérielle** : gestion des broches physiques (PIN 35, 36, 38) comme déclencheurs. |
| **Écran LCD 16x2** | **Afficheur local** : retour visuel en temps réel (statuts, actions, joueurs connectés). |

---

## 2. Architecture Générale du Système

Le système est segmenté en deux chaînes de traitement principales, matérialisées par deux *flows* distincts sous Node-RED :
### Logique de traitement standard
1. **Captation** : Événement entrant (Message Discord, appui sur bouton physique ou virtuel).
2. **Aiguillage & Formatage** : Un nœud `switch` oriente le flux, puis un nœud `change` prépare le `msg.payload` avec la syntaxe exacte de la commande Minecraft.
3. **Exécution & Notification** : Envoi simultané au nœud `rcon` et aux nœuds de rétroaction (Discord / LCD).

---

## 3. Spécifications du Flow 1 — Commandes & Notifications Discord

Ce premier flux gère les interactions logicielles, combinant l'interface Discord et un tableau de bord virtuel (*Dashboard*).

### 3.1 Bloc « Modes de jeu »
* **Déclencheurs** : Boutons "Créatif" et "Survie" du Dashboard Node-RED.
* **Logique** : Routage via un nœud `switch` vers deux nœuds `change` distincts configurés pour générer les payloads des commandes `/gamemode creative` et `/gamemode survival`.
* **Sorties** : Convergence vers le nœud `rcon` (exécution) et le nœud `Discord Message Manager` (confirmation dans le salon).

### 3.2 Bloc « Contrôle Jour / Nuit »
* **Déclencheurs** : Un bouton physique/dashboard dédié ET un nœud d'écoute Discord (`Discord Message`).
* **Logique** : Analyse de la commande reçue pour formater les payloads `/time set day` ou `/time set night`.
* **Sorties** : Exécution immédiate sur le serveur et notification de mise à jour sur Discord.

### 3.3 Bloc « Commandes Discord Multiples »
* **Déclencheur** : Écoute globale de mots-clés via un nœud `Discord Message`.
* **Logique** : Un nœud `switch` à 4 sorties filtre les messages pour dispatcher la demande vers 4 nœuds `change` spécifiques (commandes d’administration ou raccourcis).
* **Sorties** : Centralisation des retours vers un nœud RCON et un canal Discord uniques.

---

## 4. Spécifications du Flow 2 — Interface Physique & Affichage Hardware

Ce second flux gère l'interaction directe avec le Raspberry Pi 5 et ses périphériques câblés.

### 4.1 Bloc « Statut Serveur & Affichage LCD »
* **Déclencheur** : Requête automatique programmée (nœud `inject` / timestamp) ou via une commande Discord.
* **Logique** : Interrogation du serveur via RCON. La réponse brute est injectée dans un nœud `function` pour être triée et formatée.
* **Sorties multi-cibles** :
  * Mise à jour de l'écran `raspi5 lcd 16x2`.
  * Envoi d'un récapitulatif sur Discord.
  * Transmission à un sous-module spécifique "Affichage Joueurs".

### 4.2 Bloc « PIN 38 — Bouton à double action »
* **Déclencheur** : Appui sur le bouton relié à la broche GPIO 38.
* **Logique** : Un nœud `switch` bascule l'action selon l'état ou le contexte, distribuant le flux vers deux nœuds `function` distincts (`function 2` et `function 3`).
* **Sorties** : Exécution de la commande associée sur Minecraft et retour textuel sur l'écran LCD.

### 4.3 Bloc « PIN 36 — Kit d'Équipement Automatique »
* **Déclencheur** : Appui unique sur le bouton de la broche GPIO 36.
* **Logique** : Le signal est cloné et distribué en parallèle vers **9 nœuds de commande différents**, correspondants aux pièces du kit de combat standard.
* **Sorties** : Envoi d'une salve de 9 commandes `/give` successives via RCON pour équiper instantanément le joueur.

**Composition du Kit envoyé :**
* `Sword`, `Bow`, `Shield`, `Arrows`
* `Diamond Helmet`, `Diamond Chestplate`, `Diamond Leggings`, `Diamond Boots`
* `Golden Apple`

### 4.4 Bloc « PIN 35 — Événement Boss (Wither) »
* **Déclencheur** : Appui sur le bouton d'urgence de la broche GPIO 35.
* **Logique** : Routage direct vers un nœud custom "Wither" qui prépare la commande critique `/summon wither`.
* **Sorties** : Apparition du boss en jeu via RCON et déclenchement d'une alerte textuelle d'urgence sur l'afficheur LCD via le nœud `function 4`.

---

## 5. Matrice de Synthèse des Fonctionnalités

| Fonctionnalité | Déclencheur(s) | Action(s) RCON | Rétroaction / Sorties |
| :--- | :--- | :--- | :--- |
| **Changement de Mode** | Boutons Dashboard | `/gamemode` | RCON + Notification Discord |
| **Cycle Horaire** | Dashboard / Chat Discord | `/time set` | RCON + Notification Discord |
| **Commandes Custom (x4)**| Mots-clés Discord | Commandes d'admin | RCON + Message de confirmation |
| **Moniteur de Joueurs** | Automatisation (Timestamp) | `/list` | Écran LCD + Salon Discord |
| **Action Contextuelle** | Bouton Physique (**PIN 38**) | Commande variable | RCON + Affichage LCD local |
| **Distribution de Kit** | Bouton Physique (**PIN 36**) | 9x `/give` (Équipement) | RCON (Données en série) |
| **Alerte Boss Wither** | Bouton Physique (**PIN 35**) | `/summon wither` | RCON + Alerte LCD critique |

---

## 6. Feuille de Route & Plan de Déploiement

### 6.1 Objectifs Pédagogiques & Techniques
* Maîtriser la programmation par flux (*Flow-based programming*) avec Node-RED.
* Interfacer le monde physique (IoT / GPIO) avec une application logicielle tierce (Serveur de jeu).
* Manipuler des APIs tierces (Discord) et des protocoles de communication réseau (RCON).
* Gérer les flux de données asynchrones et le parsing de chaînes de caractères.

### 6.2 Plan d'Exécution

* **Étape 1** : Configuration Serveur Minecraft & RCON (`server.properties` -> `enable-rcon=true`).
* **Étape 2** : Déploiement de Node-RED sur Raspberry Pi 5 & Installation des nœuds requis (RCON, Discord).
* **Étape 3** : Création du Bot Discord (Developer Portal) & Authentification des nœuds.
* **Étape 4** : Développement du Core Logiciel (Flow 1 : Modes de jeu, temps, commandes basiques).
* **Étape 5** : Intégration Hardware (Câblage des boutons GPIO 35/36/38 et bus de l'écran LCD).
* **Étape 6** : Développement des Scénarios Avancés (Flow 2 : Macros de kits, monitoring LCD, Boss event).
* **Étape 7** : Phase de Recette (Tests de charge, gestion des erreurs RCON) & Documentation.

* Fin
