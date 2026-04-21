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

- **Exercices en durée** : timer automatique par série/côté avec 3 bips audio distincts
  - Bip aigu court — début de série
  - Double bip médium — mi-temps
  - Triple bip grave — fin de série
  - Bip aigu montant — fin du temps de repos
  - **Enchaînement automatique** : à la fin du repos, la série suivante démarre sans intervention
  - À la fin de la dernière série : l'exercice se coche et l'exercice suivant s'ouvre
- **Exercices en répétitions** : case à cocher par série + timer de repos (60 s) déclenché automatiquement à la validation
  - Cliquer à nouveau pendant le repos annule le décompte
  - Cliquer sur une série déjà cochée (hors repos) la décoche (correction)
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

### Journal quotidien
- Notation de la séance (1 à 5 étoiles)
- Ressenti en un clic (Fatigué / En forme / Difficile / Parfait / Douleur)
- Zone de notes libre
- Anneaux de progression par jour dans la barre d'en-tête (jaune = en cours, vert = terminé)
- Navigation par semaine avec historique complet

### Synchronisation Google Drive
- Stockage dans un fichier `fitness_programme_semaine.json` sur Drive
- Sync automatique après chaque action (debounce 2 s — plusieurs actions rapides = 1 seul envoi)
- Fallback localStorage si hors ligne
- Restauration silencieuse de session au rechargement de page
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

Les données sont stockées uniquement dans `fitness_programme_semaine.json` sur Drive (scope `drive.file` restreint — aucun accès aux autres fichiers).

## Mise à jour du fichier

Modifier `programme_semaine.html` localement, puis via GitHub Desktop :
- **Commit** avec un message descriptif
- **Push** vers `main`

GitHub Pages se met à jour automatiquement en 1–2 minutes.

## Stack technique

- HTML / CSS / JavaScript vanilla — aucune dépendance externe
- Web Audio API pour les bips
- Screen Wake Lock API pour empêcher la mise en veille de l'écran pendant une séance
- Google Identity Services (OAuth 2.0)
- Google Drive API v3
- localStorage comme cache local et fallback offline
