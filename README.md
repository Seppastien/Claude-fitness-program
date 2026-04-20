# Programme Fitness Semaine

Application web autonome (fichier HTML unique) pour suivre un programme fitness hebdomadaire complet, avec synchronisation Google Drive.

## Fonctionnalités

### Planning hebdomadaire
- **Lundi / Mercredi / Vendredi** — Jours bureau : séance matin + vélo aller/retour (2 × 10 km) + décompression soir
- **Mardi / Jeudi** — Télétravail : 3 séances gainage (matin / midi / soir)
- **Samedi** — Grande séance 1h–1h30 : échauffement + gainage pré-muscu + musculation haltères/banc + étirements
- **Dimanche** — Yoga doux 45–60 min adapté (dos raide, ventre proéminent)

### Suivi des exercices
- Cases à cocher par série avec état persistant
- **Exercices en durée** : timer automatique par série avec 3 bips audio distincts (début / mi-temps / fin) + timer de repos enchaîné automatiquement
- **Exercices en répétitions** : case à cocher par série + timer de repos (60 s) déclenché à la validation + bip de fin de repos
- Détails de chaque série affichés (nombre de reps, poids, tempo)
- Navigation par semaine avec historique

### Journal quotidien
- Notation de la séance (1 à 5 étoiles)
- Ressenti en un clic (Fatigué / En forme / Difficile / Parfait / Douleur)
- Zone de notes libre
- Anneaux de progression par jour

### Synchronisation Google Drive
- Stockage des données dans un fichier `fitness_programme_semaine.json` sur Drive
- Sync automatique après chaque action (debounce 2 s)
- Fallback localStorage si hors ligne
- Restauration silencieuse de session au rechargement

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
5. Coller le Client ID dans le fichier HTML à la ligne :
   ```javascript
   const GOOGLE_CLIENT_ID = 'VOTRE_CLIENT_ID.apps.googleusercontent.com';
   ```
6. Au premier lancement, cliquer **Connecter Drive** dans la barre de statut

Les données sont stockées uniquement dans le fichier `fitness_programme_semaine.json` de ton Drive (scope `drive.file` restreint — aucun accès aux autres fichiers).

## Mise à jour du fichier

Modifier `programme_semaine.html` localement, puis via GitHub Desktop :
- **Commit** avec un message descriptif
- **Push** vers `main`

GitHub Pages se met à jour automatiquement en 1–2 minutes.

## Stack technique

- HTML / CSS / JavaScript vanilla — aucune dépendance externe
- Web Audio API pour les bips
- Google Identity Services (OAuth 2.0)
- Google Drive API v3
- localStorage comme cache local et fallback offline
