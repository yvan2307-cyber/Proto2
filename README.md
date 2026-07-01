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
