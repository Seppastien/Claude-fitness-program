# CLAUDE.md — Programme Fitness Semaine

Guide de développement pour Claude. Ce fichier décrit l'architecture, les conventions et les règles à respecter lors de toute modification du projet.

---

## Vue d'ensemble

Fichier unique : `programme_semaine.html` (~1430 lignes)
Aucune dépendance externe. Tout est en HTML/CSS/JS vanilla dans un seul fichier.

---

## Architecture du fichier

```
programme_semaine.html
├── <style>          CSS complet (variables, composants, series tracker, repli)
├── <body>           Topbar + sync bar + overview rings + day tabs + panels
└── <script>
    ├── DATA         Constantes DAYS, TYPE_LABELS, SESSIONS
    ├── STATE        currentWeekOffset, activeDay, timers, serieTimers, driveState
    ├── GOOGLE DRIVE OAuth2, load/save Drive, scheduleSave, fallback localStorage
    ├── RENDER       getSessionsForDay, countExercises, renderAllPanels,
    │                renderDayPanel, renderSeriesTracker, renderRepTracker,
    │                renderSubExoTracker, renderOverview, renderTabs
    ├── INTERACTIONS toggleEx, toggleVelo, setStar, setFeeling, saveJournalEntry,
    │                syncExValidation, toggleSerieCheck, repSerieCheck
    ├── TIMERS       beep*, parseReps, parseRepSets, isDurationEx,
    │                startSerieTimer, renderSeriesTracker, renderRepTracker
    ├── REPLI        toggleSession, toggleExCollapse
    ├── WAKE LOCK    requestWakeLock() — empêche la mise en veille (Screen Wake Lock API)
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
  veloDir: 'aller|retour',    // uniquement si isVelo
  veloStats: { km, duree, cal, difficulte }, // uniquement si isVelo
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
  url: 'string optionnel',    // lien fiche technique externe — affiche ↗ dans le nom
  sets: number,               // uniquement pour subExos
  subExos: [                  // uniquement pour exercices combinés (Sphinx→Enfant)
    { name: 'string', sec: number },
  ]
}
```

---

## Types d'affichage des séries

Il existe **7 types** selon la nature de l'exercice. Ne pas les confondre.

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

## Logique d'ouverture automatique (sessions & exercices)

### Sessions

Au rendu (`renderDayPanel`), deux index sont calculés pour déterminer quelle session afficher ouverte :

```javascript
// Dernier index d'une session entièrement terminée
const lastDoneIdx = sessions.reduce((found, s, idx) => {
  if (s.exercises && s.exercises.every(ex => checks[ex.id])) return idx;
  return found;
}, -1);

// Première session incomplète APRÈS la dernière session terminée
const targetOpenIdx = sessions.findIndex((s, idx) =>
  idx > lastDoneIdx && s.exercises && !s.exercises.every(ex => checks[ex.id])
);
```

**Règle :** une session est rendue `collapsed` si elle est terminée (`allDone`) OU si son index n'est pas `targetOpenIdx`. Cela garantit qu'une seule session reste ouverte à la fois : la prochaine à faire.

- Les sessions vélo (`isVelo`) ne participent pas à ce calcul (pas de `exercises`).
- Quand toutes les sessions sont terminées, `targetOpenIdx` vaut -1 et toutes se replient.

### Exercices

À l'intérieur d'une session ouverte, seul **le premier exercice non coché** reste déplié. Tous les exercices cochés ET tous ceux qui viennent après le premier ouvert se replient :

```javascript
let firstOpenExFound = false;
// Pour chaque exercice :
const exCollapsed = !!checked || firstOpenExFound;
if (!checked && !firstOpenExFound) firstOpenExFound = true;
```

Quand un exercice en durée (`startSerieTimer`) atteint la dernière série, il se replie automatiquement après 800 ms et déplie le suivant :

```javascript
setTimeout(() => {
  exItem.classList.add('collapsed');
  const next = exItem.nextElementSibling;
  if (next && next.classList.contains('ex-item')) next.classList.remove('collapsed');
}, 800);
```

---

## Validation bidirectionnelle séries ↔ exercice

La fonction `syncExValidation(chk)` maintient la cohérence entre l'état des cases série et l'état global de l'exercice :

- **Toutes les séries cochées** → l'exercice passe à `checked`, sauvegarde, met à jour badge/onglet/overview.
- **Au moins une série décochée** → l'exercice repasse à `unchecked` si il était coché.

Sens inverse (`toggleEx`) : cocher/décocher l'exercice directement (case principale) synchronise **toutes** les cases série du tracker (`serie-check`) dans le même état.

---

## Auto-démarrage de la série suivante (timers durée)

Dans `startSerieTimer`, à la fin de la phase de repos (reste ≤ 0), la série suivante démarre automatiquement via un `.click()` sur le bouton suivant dans le tracker :

```javascript
if (serieIdx + 1 < totalSets) {
  const allBtns = tracker.querySelectorAll('.serie-timer-btn');
  if (allBtns[serieIdx + 1]) allBtns[serieIdx + 1].click();
}
```

Cela crée une chaîne travail → repos → travail → repos … entièrement automatique une fois le premier démarrage lancé.

---

## parseReps vs parseRepSets

Deux fonctions distinctes selon le contexte :

| Fonction | Utilisée par | Retourne |
|----------|-------------|---------|
| `parseReps(reps, totalSec)` | `renderSeriesTracker` (durée) | `{ sets, secPerSide, restSec, lateral }` |
| `parseRepSets(reps)` | `renderRepTracker` (reps) | `sets` (nombre entier) |

`parseRepSets` utilise le pattern `/^(\d+)\s*[×x]\s*(\d)/` — il exige un **chiffre après le ×**. Cela évite que `"10 × chaque sens"` ou `"10 souffles"` soient interprétés comme 10 séries.

---

## États de repSerieCheck (reps)

La case d'une série reps a **trois comportements** selon l'état courant :

1. **Non cochée, pas de timer** → coche, lance le décompte de repos (sauf dernière série).
2. **Cochée, timer en cours** → annule le timer de repos, remet l'affichage à zéro.
3. **Cochée, pas de timer** → décoche (permet la correction manuelle).

---

## Système de repli (collapse)

- **Session** : `session-card.collapsed` cache `.ex-list` et `.note-banner` via CSS.
- **Exercice** : `ex-item.collapsed` cache `.series-tracker` via CSS.
- `toggleSession(event, dayId, sessionId)` : toggle sur la carte ciblée par `data-sid`.
- `toggleExCollapse(event, dayId, exId)` : toggle sur `#exitem_{dayId}_{exId}`.
- `event.stopPropagation()` dans les deux handlers pour éviter la propagation vers le parent.

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
- Sync : debounce 2 s après chaque action (`scheduleSave`)
- Fallback : localStorage si non connecté
- Au chargement : merge localStorage + Drive (`Object.assign({}, local, remote)` — Drive prioritaire)
- Token restauré silencieusement via `tryRestoreToken()` (appelé 800 ms après init)

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

### IDs dynamiques (séries)

| Pattern | Cas |
|---------|-----|
| `{dayId}_{exId}_s{i}` | Série simple |
| `{dayId}_{exId}_s{i}_G` / `_D` | Série latérale Gauche/Droite |
| `{dayId}_{exId}_s{i}_sub{j}` | SubExo (exercice combiné) |

Ces IDs sont utilisés comme clés dans l'objet `serieTimers` et comme suffixes des éléments DOM `schk_`, `stbtn_`, `stdisp_`, `sphase_`, `srow_`.

---

## Points d'attention lors des modifications

- **Ajouter un exercice** : vérifier que l'ID est unique dans tout le fichier
- **Modifier `reps`** : recalculer `sec` selon le tableau ci-dessus
- **Exercice `/côté`** : vérifier que `sec` est bien × 2 par rapport à la durée par côté
- **subExos** : le champ `sec` de l'exercice parent n'est pas utilisé pour le rendu (chaque `sub.sec` est indépendant)
- **Ne jamais ouvrir en `file://`** : l'OAuth Google requiert `http://localhost`
- **Après modification** : commit + push GitHub Desktop → attendre 1-2 min pour GitHub Pages
- **parseRepSets** : ne jamais mettre un format `"N × texte"` où N > 1 si c'est une seule série — ça créerait N fausses séries
- **Ajouter un subExo** : `sets` doit être défini sur l'exercice parent ; le repos entre cycles (fin de chaque cycle complet) est 15 s — entre les postures d'un même cycle : 0 s
