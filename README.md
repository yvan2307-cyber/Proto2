
PROJET MINECRAFT CONNECTÉ
Automatisation serveur Minecraft via Node-RED,
Discord, RCON et interface physique Raspberry Pi
Document de présentation et plan détaillé du projet
 1. Introduction et objectifs
Ce document présente l'architecture et le fonctionnement du projet, construit autour de Node-RED comme couche d'orchestration centrale. Le projet relie un serveur Minecraft à plusieurs interfaces de contrôle et de notification : un bot Discord, une console physique basée sur Raspberry Pi (boutons, GPIO, écran LCD), et le protocole RCON qui permet d'envoyer des commandes directement au serveur de jeu.
L'objectif est de permettre à des joueurs ou des animateurs de déclencher des actions sur le serveur Minecraft (changement de mode de jeu, gestion du cycle jour/nuit, distribution d'objets, invocation d'un boss, etc.) depuis plusieurs points d'entrée — Discord ou boutons physiques — tout en centralisant les retours d'information (confirmations, statut) vers Discord et vers un afficheur LCD.
1.1 Technologies identifiées
Composant	Rôle dans le projet
Node-RED	Moteur d'orchestration : reçoit les événements (Discord, GPIO), les traite (switch, function) et déclenche les actions (RCON, LCD, Discord).
Discord (Discord Message / Discord Message Manager)	Réception de commandes tapées dans Discord et envoi de messages de confirmation/notification vers un salon Discord.
RCON (nœud rcon)	Protocole d'administration à distance de Minecraft : exécute les commandes serveur (gamemode, time set, give, summon, etc.).
Raspberry Pi 5 + GPIO (PIN 35/36/38)	Boutons physiques câblés sur les broches GPIO, utilisés comme déclencheurs matériels alternatifs à Discord.
Écran LCD 16x2	Retour visuel local (statut, action en cours, liste des joueurs) affiché sur la console physique.
2. Architecture générale
Le système s'articule autour de deux grandes chaînes de traitement, correspondant aux deux flows Node-RED fournis :
●	Flow « Commandes & notifications Discord » : centré sur la réception de messages Discord et de boutons de commande (dashboard), avec double sortie vers RCON (exécution) et Discord (confirmation).
●	Flow « Interface physique & affichage » : centré sur les entrées GPIO du Raspberry Pi (boutons physiques), avec sorties vers RCON, l'écran LCD, et Discord.
Le schéma logique général est le suivant :
●	Entrées possibles : message Discord, bouton dashboard (Créatif/Survie/Jour-Nuit), bouton physique GPIO (PIN 35/36/38).
●	Traitement : nœud « commuter » (switch) qui aiguille selon le type de commande, puis nœud « définir msg.payload » (change) qui construit la commande RCON correspondante.
●	Sorties : nœud « rcon » (exécution sur le serveur Minecraft), nœud « Discord Message Manager » (notification), nœud « raspi5 lcd 16x2 » (affichage local).
3. Détail du Flow 1 — Commandes & notifications Discord
3.1 Bloc « Modes de jeu » (Bouton Créatif / Bouton Survie)
Deux boutons de tableau de bord (« Bouton Créatif » et « Bouton Survie ») déclenchent chacun un nœud « commuter », qui alimente deux nœuds « définir msg.payload » distincts. Chaque payload construit une commande de changement de mode de jeu (probablement /gamemode creative et /gamemode survival). Les deux sorties convergent ensuite vers :
○	le nœud « Discord Message Manager », qui envoie une confirmation dans Discord (« message sent »),
○	le nœud « rcon », qui exécute effectivement la commande sur le serveur (statut « success »).
3.2 Bloc « Contrôle Jour / Nuit »
Un bouton dédié « Contrôle Jour / Nuit » et un second flux « Discord Message » alimentent chacun un nœud « commuter » à deux sorties, reliés à deux nœuds « définir msg.payload » (probablement /time set day et /time set night). Comme pour le bloc précédent, chaque commande est à la fois exécutée via RCON et notifiée sur Discord.
3.3 Bloc « Commandes Discord multiples »
Un troisième « Discord Message » alimente un nœud « commuter » à quatre sorties, connecté à quatre nœuds « définir msg.payload » distincts. Ces quatre branches convergent vers un nœud « Discord Message Manager » commun et un nœud « rcon » commun, ce qui suggère un ensemble de quatre commandes différentes pouvant être invoquées depuis Discord (par exemple des commandes d'administration ou des raccourcis de jeu).
4. Détail du Flow 2 — Interface physique Raspberry Pi & affichage
4.1 Bloc « Discord vers écran LCD »
Un flux « Discord Message », combiné à un nœud « horodatage » (timestamp), alimente un nœud « définir msg.payload » puis un nœud « rcon ». La sortie du RCON est ensuite traitée par un nœud « function 1 » qui distribue le résultat vers trois destinations :
○	l'écran « raspi5 lcd 16x2 » (affichage local du résultat),
○	le nœud « Discord Message Manager » (confirmation dans Discord),
○	un nœud « Affichage Joueurs », probablement dédié à l'affichage de la liste des joueurs connectés.
4.2 Bloc « PIN 38 — bouton à deux actions »
L'entrée GPIO « PIN 38 » alimente un nœud « commuter » à deux sorties. Chaque sortie passe par un nœud « définir msg.payload » puis un nœud « function » dédié (function 2 et function 3), avant de rejoindre à la fois un écran « raspi5 lcd 16x2 » et un nœud « rcon » commun. Ce bouton physique permet donc de déclencher deux commandes différentes sur le serveur, avec retour visuel sur l'écran.
4.3 Bloc « PIN 36 — distribution d'objets (kit d'équipement) »
L'entrée GPIO « PIN 36 » alimente un nœud « commuter » qui se ramifie vers neuf nœuds de commande, chacun correspondant à un objet du jeu : Sword, Bow, diamond_Helmet, diamond_Chestplate, diamond_leggings, diamond_boots, goldenapple, shield, arrows. Toutes ces branches convergent vers un nœud « rcon » commun. Un seul appui sur ce bouton physique déclenche donc une série de commandes /give successives afin de doter un joueur d'un équipement complet (armure en diamant, épée, arc, flèches, bouclier, pomme dorée).
4.4 Bloc « PIN 35 — invocation du Wither »
L'entrée GPIO « PIN 35 » alimente un nœud « Wither », relié à un nœud « rcon » (exécution de la commande d'invocation, probablement /summon wither) ainsi qu'à un nœud « function 4 » qui met à jour l'écran « raspi5 lcd 16x2 » pour signaler l'événement.
5. Synthèse des fonctionnalités du projet
Fonctionnalité	Déclencheur	Effet(s) produit(s)
Changement de mode de jeu (Créatif/Survie)	Boutons dashboard	Commande RCON + notification Discord
Contrôle du cycle jour/nuit	Bouton dashboard / Discord	Commande RCON + notification Discord
Commandes personnalisées (x4)	Message Discord	Commande RCON + notification Discord
Affichage joueurs connectés	Message Discord + RCON	Écran LCD + Discord + module « Affichage Joueurs »
Action physique double (PIN 38)	Bouton GPIO	Commande RCON + écran LCD
Distribution de kit d'équipement	Bouton GPIO (PIN 36)	9 commandes /give successives via RCON
Invocation du Wither	Bouton GPIO (PIN 35)	Commande RCON + affichage LCD
6. Plan détaillé du projet
6.1 Objectifs pédagogiques / techniques
●	Créer une passerelle entre un environnement physique (boutons, GPIO) et un jeu vidéo en ligne (Minecraft).
●	Automatiser l'administration d'un serveur Minecraft via le protocole RCON.
●	Intégrer une communication bidirectionnelle avec Discord (commandes entrantes, notifications sortantes).
●	Offrir un retour d'information immédiat à l'utilisateur via un écran LCD local.
6.2 Architecture matérielle
●	Raspberry Pi 5 : hôte de Node-RED, du serveur Minecraft (ou client RCON) et de la gestion GPIO.
●	Écran LCD 16x2 : connecté au Raspberry Pi pour l'affichage des statuts.
●	Boutons physiques câblés sur les broches GPIO 35, 36 et 38.
6.3 Architecture logicielle
●	Node-RED : plateforme de flux (flow-based programming) hébergeant l'ensemble de la logique.
●	Nœuds utilisés : Discord Message / Discord Message Manager, rcon, switch (« commuter »), change (« définir msg.payload »), function, raspi5 lcd 16x2, horodatage, inject (boutons dashboard).
●	Serveur Minecraft avec RCON activé (fichier server.properties : enable-rcon=true).
●	Bot Discord relié au flow via un nœud de messagerie Discord.
6.4 Étapes de réalisation (proposition de plan)
●	Étape 1 — Mise en place du serveur Minecraft et activation du RCON.
●	Étape 2 — Installation de Node-RED sur le Raspberry Pi et configuration du nœud RCON.
●	Étape 3 — Création du bot Discord et intégration des nœuds Discord Message / Discord Message Manager.
●	Étape 4 — Développement des flows de commandes de base (mode de jeu, cycle jour/nuit).
●	Étape 5 — Câblage des boutons physiques (GPIO) et intégration de l'écran LCD.
●	Étape 6 — Développement des flows avancés (distribution de kits d'objets, invocation d'entités).
●	Étape 7 — Tests, ajustements et documentation finale.
6.5 Pistes d'amélioration possibles
●	Ajouter une authentification ou une limitation d'accès aux commandes sensibles (ex. invocation du Wither).
●	Journaliser les actions déclenchées (logs horodatés) pour un suivi des événements.
●	Étendre l'affichage LCD avec un défilement de texte pour les messages longs.
●	Ajouter un retour sonore ou lumineux (LED) lors des actions physiques.
7. Remarques
Ce document a été rédigé à partir de l'analyse visuelle des deux flows Node-RED transmis. Certains libellés de commandes (contenu exact des nœuds « définir msg.payload » et « function ») n'étant pas directement lisibles sur les captures, les interprétations proposées ci-dessus sont basées sur le nommage des nœuds et la logique du flux. N'hésitez pas à me transmettre le détail du contenu des fonctions ou des flows supplémentaires afin d'affiner et compléter ce document.
