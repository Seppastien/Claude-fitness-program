# Programme Fitness Semaine

Application web autonome (fichier HTML unique) pour suivre un programme fitness hebdomadaire complet, avec synchronisation Google Drive.

## Fonctionnalités

### Planning hebdomadaire
- **Lundi / Mercredi / Vendredi** — Jours bureau : séance matin + vélo aller/retour (2 × 10 km) + décompression soir
- **Mardi / Jeudi** — Télétravail : 3 séances gainage (matin / midi / soir)
- **Samedi** — Grande séance 1h–1h30 : échauffement + gainage pré-muscu + musculation haltères/banc + étirements
- **Dimanche** — Yoga doux 45–60 min adapté (dos raide, ventre proéminent)

### Navigation intelligente (auto-ouverture)

L'interface adapte automatiquement ce qui est affiché selon l'avancement :

- **Sessions** : une seule session est dépliée à la fois — la prochaine à faire dans la journée. Les sessions terminées se replient automatiquement. Un clic sur l'en-tête d'une session permet de la déplier/replier manuellement.
- **Exercices** : à l'intérieur d'une session ouverte, seul le premier exercice non coché reste visible. Les exercices terminés se replient. À la fin d'un exercice en durée, l'exercice suivant s'ouvre automatiquement 800 ms après la dernière série.

### Suivi des séries

- **Exercices en durée** : timer automatique par série/côté avec une modale plein écran et 3 bips audio distincts
  - **Phase de préparation** : décompte 3 s avec bip à chaque seconde avant chaque série
  - Bip aigu court — début de série
  - Double bip médium — mi-temps
  - Triple bip grave — fin de série
  - Bip aigu montant — fin du temps de repos
  - **Bouton ▶ Lancer toutes les séries** : démarre la première série non terminée ; l'enchaînement automatique prend la suite (prep → work → rest → prep…)
  - À la fin de la dernière série : l'exercice se coche, la modale se ferme, l'exercice suivant s'ouvre (ou la séance se replie si c'était le dernier)
  - **Modale timer** (plein écran) : rappelle le nom de l'exercice, sa description, le libellé de série et la consigne (reps/durée). Trois actions sont exposées :
    - **⏸ Pause / ▶ Reprendre** — fige le décompte de la phase en cours
    - **⏭ Sauter** — termine immédiatement la phase courante (prep→work, work→rest+valide la série, rest→série suivante)
    - **⏹ Reset** — annule la série en cours et ferme la modale
- **Exercices en répétitions** : modale plein écran en **deux temps**
  - **1. Phase d'exécution** (`rep-exec`) — la modale affiche le nom de l'exercice, la consigne (reps/charge) et attend une action manuelle :
    - **✓ Valider** — coche la série, joue `beepEnd()`, puis enchaîne automatiquement sur la phase de repos
    - **✕ Fermer** — ferme la modale sans cocher la série (utile si on s'est trompé de clic)
  - **2. Phase de repos** (`rep-rest`) — décompte configurable (`restSec` sur l'exercice, 60 s par défaut), avec les mêmes contrôles que les exercices en durée :
    - **⏸ Pause / ▶ Reprendre**, **⏭ Sauter**, **⏹ Reset**
    - À la fin du repos : bip montant + auto-clic sur le bouton ▶ de la série suivante (ou fermeture + repli de l'exercice si c'était la dernière série)
  - Cliquer sur une série déjà cochée (hors modale) la décoche pour correction manuelle
- **Exercices latéraux** (`/côté`) : lignes séparées Gauche / Droite avec timer individuel par côté — le repos ne se déclenche qu'après le second côté
- **Exercices alternés continus** (crunch bicycle, superman, planche tap) : séries simples sans séparation gauche/droite — serieInfo indique le total de cycles
- **Exercices alternés avec pause** (Bird Dog, Dead Bug) : séries simples, alternance fluide gauche/droite au sein de chaque série — serieInfo précise le nombre de reps par côté
- **Exercices combinés** (Sphinx → Enfant) : postures alternées au sein d'un même exercice, chacune avec son propre timer — repos 15 s entre les cycles, 0 s entre les postures d'un même cycle
- Description de chaque série (reps, poids, tempo) affichée **une seule fois** en haut du bloc de séries

### Validation synchronisée

Les cases séries et la case principale de l'exercice restent toujours cohérentes :
- Cocher toutes les séries → l'exercice se valide automatiquement
- Décocher une série déjà validée → l'exercice repasse à non terminé
- Cocher/décocher la case principale de l'exercice → toutes les séries suivent

### Fiches techniques des exercices

Chaque exercice peut afficher un lien `↗` vers une ressource externe (vidéo ou guide) pour contrôler la forme. Le lien s'ouvre dans un nouvel onglet sans interrompre le chronomètre en cours.

### Slider d'intensité par exercice

Chaque exercice expose un slider 50 %–150 % (pas 5 %, défaut 100 %) qui multiplie à la fois la **durée des séries** et le **nombre de répétitions** (pas le temps de repos).

- Le badge `×NN%` apparaît à côté de la note de reps quand le slider est ≠ 100 %, et est repris dans la modale timer plein écran.
- Valeur **globale par exercice** : un exercice réglé à 110 % le reste d'une semaine à l'autre tant qu'on ne touche pas au slider.
- La configuration du programme (mode admin, import/export) reste indépendante du scaling : un fichier de config échangé entre utilisateurs ne transporte pas les scales personnels.
- Modifier le slider **pendant** une série active n'affecte pas la série en cours (la valeur est figée au démarrage) — la nouvelle intensité s'applique à la série suivante.

### Mode administration

Un bouton ⚙ dans la barre supérieure bascule vers un éditeur de configuration (jours, sessions, exercices, séries, vélo, subExos).

- Arborescence jour → session → exercice avec ajout / duplication / déplacement / suppression
- Éditeur de champ complet pour chaque exercice (id, nom, description, reps, durée, charge, type, fiche technique, subExos…)
- **Export** de la configuration courante en fichier JSON
- **Import** d'un fichier JSON (remplace la configuration active après validation de schéma)
- **Reset** — restaure la configuration par défaut (programme livré avec le fichier)
- La config utilisateur est stockée dans `localStorage` et dans un second fichier Drive `fitness_programme_config.json` (sync debounce 2 s). En ouverture, la config Drive prime sur la config locale.
- La modale timer et la progression sont masquées en mode admin pour éviter toute interaction parasite pendant l'édition.

### Journal quotidien
- Notation de la séance (1 à 5 étoiles)
- Ressenti en un clic (Fatigué / En forme / Difficile / Parfait / Douleur)
- Zone de notes libre
- Anneaux de progression par jour dans la barre d'en-tête (jaune = en cours, vert = terminé)
- Navigation par semaine avec historique complet

### Synchronisation Google Drive
- Deux fichiers distincts sur Drive :
  - `fitness_programme_semaine.json` — progression (checks + journal par semaine + slider d'intensité par exercice)
  - `fitness_programme_config.json` — configuration du programme (éditée via le mode admin)
- Sync automatique après chaque action (debounce 2 s — plusieurs actions rapides = 1 seul envoi)
- Fallback localStorage si hors ligne
- Restauration silencieuse de session au rechargement de page (prompt:'none')
- En cas de conflit : les données Drive sont prioritaires sur le cache local

## Équipement requis

- Banc de musculation inclinable + déclinable
- 4 haltères à main (2 kg chacune)
- Disques : 0,5 kg / 1 kg / 2 kg

## Installation et utilisation

### PC (Windows / Mac / Linux)

1. Cloner ou télécharger le repo
2. Lancer un serveur local dans le dossier du projet :
   ```bash
   # Python
   python -m http.server 8080

   # Node.js
   npx serve .
   ```
   Ou utiliser l'extension **Live Server** dans VS Code (clic droit → *Open with Live Server*)
3. Ouvrir `http://localhost:8080/programme_semaine.html` dans Chrome

> ⚠️ Le fichier doit être servi via `http://localhost` — l'ouverture directe en `file://` ne permet pas l'authentification Google OAuth.

### Android

Accéder à l'URL GitHub Pages depuis Chrome Android :
```
https://seppastien.github.io/Claude-fitness-program/programme_semaine.html
```
Puis : **⋮ → Ajouter à l'écran d'accueil** pour créer un raccourci application.

## Configuration Google Drive (première utilisation)

1. Créer un projet sur [Google Cloud Console](https://console.cloud.google.com)
2. Activer l'**API Google Drive**
3. Créer un identifiant **OAuth 2.0** (Application Web)
4. Ajouter dans les origines JavaScript autorisées :
   - `http://localhost`
   - `http://localhost:8080`
   - `https://seppastien.github.io`
5. Le Client ID est déjà intégré dans le fichier HTML
6. Au premier lancement, cliquer **Connecter Drive** dans la barre de statut

Les données sont stockées dans deux fichiers Drive créés par l'app (`fitness_programme_semaine.json` pour la progression, `fitness_programme_config.json` pour la configuration éditable). Le scope `drive.file` restreint l'accès à ces fichiers-là — l'app ne peut rien lire d'autre dans ton Drive.

## Mise à jour du fichier

Modifier `programme_semaine.html` localement, puis via GitHub Desktop :
- **Commit** avec un message descriptif
- **Push** vers `main`

GitHub Pages se met à jour automatiquement en 1–2 minutes.

## Stack technique

- HTML / CSS / JavaScript vanilla — aucune dépendance externe, un seul fichier
- Web Audio API pour les bips de phases et les fanfares de fin (exercice / séance / journée complète)
- Screen Wake Lock API pour empêcher la mise en veille de l'écran pendant une séance
- Google Identity Services (OAuth 2.0) avec restauration de session silencieuse
- Google Drive API v3 (scope `drive.file` — accès restreint aux fichiers créés par l'app)
- localStorage comme cache local, miroir de config et fallback offline
