# CLAUDE.md — Programme Fitness Semaine

Guide de développement pour Claude. Ce fichier décrit l'architecture, les conventions et les règles à respecter lors de toute modification du projet.

---

## Vue d'ensemble

Fichier unique : `programme_semaine.html` (~1260 lignes)
Aucune dépendance externe. Tout est en HTML/CSS/JS vanilla dans un seul fichier.

---

## Architecture du fichier

```
programme_semaine.html
├── <style>          CSS complet (variables, composants, series tracker)
├── <body>           Topbar + sync bar + overview rings + day tabs + panels
└── <script>
    ├── DATA         Constantes DAYS, TYPE_LABELS, SESSIONS
    ├── STATE        currentWeekOffset, activeDay, timers, serieTimers, driveState
    ├── GOOGLE DRIVE OAuth2, load/save Drive, scheduleSave, fallback localStorage
    ├── RENDER       getSessionsForDay, countExercises, renderAllPanels,
    │                renderDayPanel, renderSeriesTracker, renderRepTracker,
    │                renderSubExoTracker, renderOverview, renderTabs
    ├── INTERACTIONS toggleEx, toggleVelo, setStar, setFeeling, saveJournalEntry
    ├── TIMERS       beep*, parseReps, parseRepSets, isDurationEx,
    │                startSerieTimer, repSerieCheck, toggleSerieCheck
    └── WEEK NAV     changeWeek, updateWeekDisplay, init()
```

---

## Structure des données SESSIONS

```javascript
SESSIONS = {
  bureau: [ ...sessions ],   // Lun, Mer, Ven
  tele:   [ ...sessions ],   // Mar, Jeu
  weekend: {
    sam: [ ...sessions ],
    dim: [ ...sessions ],
  }
}
```

### Objet session

```javascript
{
  id: 'string',
  emoji: '🌅',
  title: 'string',
  desc: 'string',
  duration: '20 min',
  exercises: [ ...exercices ],
  note: 'string optionnel',   // bannière jaune affiché sous le titre
  isVelo: true,               // cas spécial — affiche la carte vélo
  isMuscu: true,              // cas spécial musculation
  isYoga: true,               // cas spécial yoga
}
```

### Objet exercice

```javascript
{
  id: 'string unique',        // ex: 't_m2', 's_mu1'
  name: 'string',
  desc: 'string',
  reps: 'string',             // ex: '3 × 12', '2 × 40 s /côté', '3 min'
  sec: number,                // durée TOTALE du timer (tous côtés, toutes séries)
  type: 'gainage|etirement|cardio|muscu|yoga',
  charge: 'string optionnel', // ex: '2×2 kg'
  serieInfo: 'string optionnel', // description affichée UNE FOIS en haut du bloc
  sets: number,               // uniquement pour subExos
  subExos: [                  // uniquement pour exercices combinés (Sphinx→Enfant)
    { name: 'string', sec: number },
  ]
}
```

---

## Types d'affichage des séries

Il existe **4 types** selon la nature de l'exercice. Ne pas les confondre.

| Type | Détection | Rendu | Exemple |
|------|-----------|-------|---------|
| **Durée simple** | `isDurationEx(ex)` = true, pas de `/côté` | Timer par série | Planche 3×20s |
| **Durée latérale** | `isDurationEx(ex)` = true + `/côté` dans reps | Gauche 1/N ▶ / Droite 1/N ▶ | Psoas 3×40s/côté |
| **Reps simple** | `isDurationEx(ex)` = false, pas de `/côté` | Cases à cocher + repos 60s | Curl 3×12 |
| **Reps latéral vrai** | `isDurationEx(ex)` = false + `/côté` | Gauche 1/N ☐ / Droite 1/N ☐ | Rowing 3×12/côté |
| **Alterné avec pause** | `isDurationEx(ex)` = false, PAS de `/côté` | Cases à cocher simples, serieInfo indique "alternés" | Bird Dog 3×8 |
| **Alterné continu** | `isDurationEx(ex)` = false, PAS de `/côté` | Cases à cocher simples, serieInfo indique "cycles alternés" | Bicycle 3×16 |
| **Combiné** | `ex.subExos` présent | Postures alternées avec timer individuel | Sphinx→Enfant |

> **Règle critique** : le `/côté` dans `reps` est le seul signal qui déclenche l'affichage Gauche/Droite. Ne l'ajouter que pour les exercices où on s'arrête vraiment entre les deux côtés (étirements, side plank, rowing). Pour Bird Dog, Dead Bug, bicycle → pas de `/côté`.

---

## Règles sur le champ `sec`

Le champ `sec` représente toujours la **durée totale** de l'exercice, tous côtés et toutes séries confondus. Les fonctions de rendu divisent ensuite selon le nombre de séries et de côtés.

| Format reps | Calcul sec | Exemple |
|-------------|-----------|---------|
| `3 × 30 s` | 3 × 30 = **90** | sec:90 |
| `3 × 40 s /côté` | 3 × 40 × 2 = **240** | sec:240 |
| `2 × 45 s /côté` | 2 × 45 × 2 = **180** | sec:180 |
| `3 × 8 /côté` reps | estimation libre | sec:480 |

---

## Règles sur les reps

- Toujours un **nombre pair** pour les exercices avec latéralité
- Format standard : `N × M` (sets × reps ou sets × durée)
- `/côté` uniquement si l'exercice marque un arrêt réel entre côtés
- `serieInfo` affiché UNE SEULE FOIS en tête de bloc — ne pas répéter l'info par ligne

---

## Système audio (Web Audio API)

```javascript
beepStart()   // 880 Hz aigu court      — début de série
beepHalf()    // 660 Hz double médium   — mi-temps
beepEnd()     // 440 Hz triple grave    — fin de série
beepRestEnd() // 880→1100 Hz montant    — fin du temps de repos
```

Les bips sont déclenchés uniquement sur les exercices en durée (`isDurationEx`). Les exercices en reps déclenchent uniquement `beepEnd()` au clic de validation et `beepRestEnd()` en fin de repos.

---

## Persistance des données

### Structure localStorage / Drive

Clé : `fitness_{weekKey}_{dayId}_{checks|journal}`
Exemple : `fitness_w20260420_mar_checks`

```javascript
checks  = { 'exo_id': true/false, 'velo_aller': true, ... }
journal = { rating: 1-5, feeling: 'string', note: 'string' }
```

### Google Drive

- Fichier unique : `fitness_programme_semaine.json`
- Scope : `drive.file` (accès restreint au fichier créé par l'app)
- Sync : debounce 2 s après chaque action
- Fallback : localStorage si non connecté

---

## Conventions de nommage des IDs

| Préfixe | Contexte |
|---------|----------|
| `b_m*`  | Bureau matin |
| `b_s*`  | Bureau soir |
| `t_m*`  | Télétravail matin |
| `t_mi*` | Télétravail midi |
| `t_s*`  | Télétravail soir |
| `s_e*`  | Samedi échauffement |
| `s_g*`  | Samedi gainage pré-muscu |
| `s_mu*` | Samedi musculation |
| `s_p*`  | Samedi post-effort |
| `d_y*`  | Dimanche yoga |

Les IDs de séries sont générés dynamiquement : `{dayId}_{exId}_s{i}` pour les séries simples, `{dayId}_{exId}_s{i}_G/D` pour les latéraux, `{dayId}_{exId}_s{i}_sub{j}` pour les subExos.

---

## Points d'attention lors des modifications

- **Ajouter un exercice** : vérifier que l'ID est unique dans tout le fichier
- **Modifier `reps`** : recalculer `sec` selon le tableau ci-dessus
- **Exercice `/côté`** : vérifier que `sec` est bien × 2 par rapport à la durée par côté
- **subExos** : le champ `sec` de l'exercice parent n'est pas utilisé pour le rendu (chaque `sub.sec` est indépendant)
- **Ne jamais ouvrir en `file://`** : l'OAuth Google requiert `http://localhost`
- **Après modification** : commit + push GitHub Desktop → attendre 1-2 min pour GitHub Pages
