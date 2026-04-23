# CLAUDE.md — Programme Fitness Semaine

Guide de développement pour Claude. Ce fichier décrit l'architecture, les conventions et les règles à respecter lors de toute modification du projet.

---

## Vue d'ensemble

Fichier unique : `programme_semaine.html` (~3130 lignes).
Aucune dépendance externe. Tout est en HTML/CSS/JS vanilla dans un seul fichier.

Deux modes utilisateur cohabitent :
- **Mode utilisateur** (par défaut) : programme + suivi + modale timer.
- **Mode admin** : éditeur de configuration (⚙ dans la topbar). La classe `body.admin-mode` masque `overview-bar`, `day-tabs`, `day-panels` et la modale timer, et affiche `.admin-container`.

---

## Architecture du fichier

```
programme_semaine.html
├── <style>          CSS complet (variables, composants, series tracker, repli,
│                    admin mode, timer modal plein écran)
├── <body>           Topbar (+ bouton ⚙ admin) + sync bar + overview rings +
│                    day tabs + day panels + timer-modal + admin-container
└── <script>
    ├── DATA         DEFAULT_CONFIG (days, typeLabels, typeColors, sessions),
    │                LIVE_CONFIG, CONFIG_STORAGE_KEY
    ├── CONFIG API   cloneDefaultConfig, isValidConfig, loadConfig, saveConfig,
    │                scheduleSaveConfig, loadConfigFromDrive, saveConfigToDrive
    ├── SCALES       SCALES_STORAGE_KEY, EXERCISE_SCALES, loadScales,
    │                getExerciseScale, saveScale (slider 50–150 %)
    ├── STATE        currentWeekOffset, activeDay, timers, serieTimers,
    │                modalState, driveState
    ├── GOOGLE DRIVE OAuth2, load/save Drive (progression + config séparés),
    │                scheduleSave, fallback localStorage
    ├── RENDER       getSessionsForDay, countExercises, renderAllPanels,
    │                renderDayPanel, renderSeriesTracker, renderRepTracker,
    │                renderSubExoTracker, renderOverview, renderTabs
    ├── INTERACTIONS toggleEx, toggleVelo, setStar, setFeeling, saveJournalEntry,
    │                syncExValidation, toggleSerieCheck, repSerieCheck,
    │                onExerciseScaleInput, onExerciseScaleChange
    ├── TIMERS       beep*, parseReps, parseRepSets, isDurationEx, fmtTime,
    │                scaledSec, scaledSubSec, scaledRepsString,
    │                startAllSeries, startSerieTimer, tickSerieTimer,
    │                transitionToWork, finishWorkPhase, finishRestPhase,
    │                finalizeExerciseCompletion, resetSerieTimer
    ├── TIMER MODAL  openTimerModal, openRepRestModal, closeTimerModal,
    │                updateTimerModal, pauseModalTimer, skipModalTimer,
    │                resetModalTimer, findExerciseInfo, modalState, PREP_SECS
    ├── REPLI        toggleSession, toggleExCollapse
    ├── WEEK NAV     changeWeek, updateWeekDisplay
    ├── ADMIN MODE   toggleAdminMode, renderAdminTree, renderAdminEditor,
    │                adminAdd/Move/Duplicate/Delete (Session|Exercise),
    │                adminSubExo*, adminConvertToCombined/Simple,
    │                adminExportConfig, adminImportConfig, adminResetConfig
    └── INIT         IIFE de boot : loadConfig → loadScales → rendu →
                     tryRestoreToken (différé 800 ms) → Wake Lock
```

---

## Structure des données SESSIONS

Les sessions vivent désormais dans `LIVE_CONFIG.sessions` (voir section *Système de configuration*). Toujours lire via `getSessionsForDay(dayId)` — **ne jamais** référencer une constante `SESSIONS` figée, elle n'existe plus en tant que globale mutable.

```javascript
LIVE_CONFIG.sessions = {
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
  sec: number,                // durée TOTALE du timer (tous côtés, toutes séries) — exercices en durée uniquement
  type: 'gainage|etirement|cardio|muscu|yoga',
  charge: 'string optionnel', // ex: '2×2 kg'
  restSec: number,            // repos en secondes après chaque série (reps uniquement) — défaut 60 si absent
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

### Bips de phase (exercices en durée)

```javascript
beepStart()   // 880 Hz aigu court      — début de série (+ tick chaque seconde en phase prep)
beepHalf()    // 660 Hz double médium   — mi-temps de la phase work
beepEnd()     // 440 Hz triple grave    — fin de la phase work
beepRestEnd() // 880→1100 Hz montant    — fin du temps de repos
```

### Fanfares de fin (arpèges victorieux)

Jouées à la validation du **dernier** exercice d'une granularité donnée. `playExerciseCompleteSound(chk)` choisit automatiquement la fanfare en fonction de l'état global du jour :

```javascript
beepExerciseDone() // C-E-G-C        — fin d'un exercice (cas par défaut)
beepSessionDone()  // escalier + C maj — dernier exercice d'une séance
beepDayDone()      // grande fanfare + accord soutenu — dernier exercice du jour
```

### Règles

- Les bips de phase ne concernent **que** les exercices en durée (`isDurationEx(ex)`).
- Les exercices en reps déclenchent `beepEnd()` à la validation d'une série et `beepRestEnd()` en fin de repos.
- `playExerciseCompleteSound` est appelée par `syncExValidation` quand toutes les séries d'un exercice passent à `done` — elle doit toujours être le point unique qui décide de la fanfare.

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

Quand un exercice est terminé (durée ou reps), `finalizeExerciseCompletion(anchor)` est appelé après validation de la dernière série. Il replie l'exercice courant après 800 ms et déplie le suivant. Si c'est le dernier exercice de la séance, il replie également la carte de session :

```javascript
// finalizeExerciseCompletion(anchor)
setTimeout(() => {
  exItem.classList.add('collapsed');
  const next = exItem.nextElementSibling;
  if (next && next.classList.contains('ex-item')) {
    next.classList.remove('collapsed');
  } else {
    const sessionCard = exItem.closest('.session-card');
    if (sessionCard) sessionCard.classList.add('collapsed');
  }
}, 800);
```

---

## Validation bidirectionnelle séries ↔ exercice

La fonction `syncExValidation(chk)` maintient la cohérence entre l'état des cases série et l'état global de l'exercice :

- **Toutes les séries cochées** → l'exercice passe à `checked`, sauvegarde, met à jour badge/onglet/overview.
- **Au moins une série décochée** → l'exercice repasse à `unchecked` si il était coché.

Sens inverse (`toggleEx`) : cocher/décocher l'exercice directement (case principale) synchronise **toutes** les cases série du tracker (`serie-check`) dans le même état.

---

## Cycle d'une série en durée (3 phases)

`startSerieTimer` est le point d'entrée ; il délègue chaque tick à `tickSerieTimer`, qui aiguille vers les transitions selon `st.phase`.

| Phase | Durée | Son | Action en fin |
|-------|-------|-----|---------------|
| `prep` | `PREP_SECS` = 3 s | `beepStart()` à chaque seconde | `transitionToWork` → passe en `work` |
| `work` | `secPerSet` | `beepHalf()` à mi-temps, `beepEnd()` en fin | `finishWorkPhase` → coche la série, puis `rest` (ou fin si dernière série) |
| `rest` | `restSec` | `beepRestEnd()` en fin | `finishRestPhase` → auto-clic sur le bouton de la série suivante |

Chaque série d'un exercice en durée suit donc **prep → work → rest → prep (série suivante)…** entièrement automatique une fois le premier démarrage lancé. À la dernière série, `finishRestPhase` ferme la modale et appelle `finalizeExerciseCompletion` pour replier l'exercice après 800 ms et ouvrir le suivant (ou replier la séance si dernier exercice).

Le bouton **▶ Lancer toutes les séries** (généré par `renderSeriesTracker` et `renderSubExoTracker`) appelle `startAllSeries(dayId, exId)` : il trouve la première série non terminée et simule un clic sur son bouton — l'enchaînement automatique prend la suite.

L'enchaînement série suivante se fait via `.click()` sur le bouton du tracker :

```javascript
if (serieIdx + 1 < totalSets) {
  const allBtns = tracker.querySelectorAll('.serie-timer-btn');
  if (allBtns[serieIdx + 1]) allBtns[serieIdx + 1].click();
}
```

---

## Modale timer plein écran

La modale `#timer-modal` s'affiche par-dessus l'app dans deux cas (masquée en `body.admin-mode`) :
- **Pendant une série en durée** : phases prep / work / rest avec contrôles pause/skip/reset.
- **Pendant le repos d'une série en reps** : affiche uniquement la phase `rest` avec les mêmes contrôles.

### État

```javascript
const modalState = { sid: null, active: false, isRepRest: false };
```

`isRepRest` distingue les deux contextes d'utilisation : `skipModalTimer` et `resetModalTimer` branchent différemment selon cette valeur. Un seul timer peut être visible à la fois : `updateTimerModal(sid, ...)` ignore tout appel concernant un `sid` différent du `sid` actif.

### API

| Fonction | Rôle |
|----------|------|
| `openTimerModal(sid)` | Résout l'exercice via `findExerciseInfo`, remplit nom/desc/libellé série/reps, affiche la modale — utilisé pour les séries en durée |
| `openRepRestModal(sid)` | Même remplissage que `openTimerModal` mais pose `modalState.isRepRest = true` — utilisé par `repSerieCheck` pendant le repos |
| `closeTimerModal()` | Masque la modale, remet `modalState` à zéro (sid, active, isRepRest) |
| `updateTimerModal(sid, phaseType, remaining)` | Met à jour le libellé de phase (`prep`/`work`/`rest`) et le compteur. En `prep`, le chiffre est agrandi (classe `.big`) |
| `pauseModalTimer()` | Toggle `st.paused` — le tick ignore les secondes pendant la pause |
| `skipModalTimer()` | Force la fin de la phase courante. Si `isRepRest` : saute le repos et appelle `syncExValidation`/`finalizeExerciseCompletion` si c'était la dernière série. Sinon : `prep`→`transitionToWork`, `work`→`finishWorkPhase`, `rest`→`finishRestPhase` |
| `resetModalTimer()` | Si `isRepRest` : annule le timer de repos et ferme. Sinon : `resetSerieTimer(sid)` + ferme |
| `resetSerieTimer(sid)` | Nettoie l'interval d'une série en durée, remet le bouton du tracker à son état initial |

### Points d'attention

- La résolution de l'exercice passe par `findExerciseInfo` qui lit `getSessionsForDay(dayId)` — donc **dépend de `LIVE_CONFIG`** et non de constantes figées.
- Le libellé de série est **réutilisé** depuis le DOM (`.serie-label` de la ligne `#srow_{sid}`), pas recalculé — garantit l'alignement avec le tracker (ex : `Série 2/3`, `Gauche 1/3`, `Sphinx 2/3`).
- Skip en `work` **valide la série** (coche auto + `syncExValidation`) comme si le timer était allé au bout.
- Reset NE valide PAS la série et remet le bouton du tracker en état « prêt à démarrer ».

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

1. **Non cochée, pas de timer** → coche, lance le décompte de repos via `openRepRestModal` (modale overlay, même contrôles que le timer durée), sauf sur la dernière série où le repos est quand même lancé — c'est `finishRestPhase` équivalent qui appelle `syncExValidation` + `finalizeExerciseCompletion` en fin de repos.
2. **Cochée, timer en cours** → annule le timer de repos, ferme la modale, remet l'affichage à zéro.
3. **Cochée, pas de timer** → décoche (permet la correction manuelle).

La durée du repos est lue depuis `ex.restSec` si défini, sinon 60 s par défaut.

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

- Deux fichiers distincts, tous deux avec scope `drive.file` (accès restreint aux fichiers créés par l'app) :
  - `fitness_programme_semaine.json` — progression (checks + journal par semaine)
  - `fitness_programme_config.json` — configuration du programme (`LIVE_CONFIG`)
- Sync progression : debounce 2 s via `scheduleSave` / `saveToDrive`
- Sync config : debounce 2 s via `scheduleSaveConfig` / `saveConfigToDrive`
- Fallback : localStorage si non connecté
- Au chargement progression : merge localStorage + Drive (`Object.assign({}, local, remote)` — Drive prioritaire)
- Au chargement config : `loadConfigFromDrive` écrase `LIVE_CONFIG` + miroir localStorage si le fichier Drive passe `isValidConfig`
- Token restauré silencieusement via `tryRestoreToken()` (appelé 800 ms après init)

---

## Système de configuration (LIVE_CONFIG)

Le programme (jours, sessions, exercices) n'est plus figé dans le source — il vit dans `LIVE_CONFIG` et est éditable via le mode admin.

### Structure

```javascript
LIVE_CONFIG = {
  version: 1,
  meta: { createdAt, lastModified, source: 'default'|'user'|'import' },
  days: [...],          // 7 jours
  typeLabels: {...},    // mapping type → libellé FR
  typeColors: {...},    // mapping type → couleur
  sessions: {           // identique à l'ancienne constante SESSIONS
    bureau: [...],
    tele: [...],
    weekend: { sam: [...], dim: [...] }
  }
}
```

### Priorité de chargement (au boot)

1. `DEFAULT_CONFIG` cloné dans `LIVE_CONFIG` — programme livré par défaut
2. `localStorage[CONFIG_STORAGE_KEY]` s'il existe et passe `isValidConfig`
3. Drive (`fitness_programme_config.json`) s'il existe et passe `isValidConfig` — écrase les deux précédents

### API

| Fonction | Rôle |
|----------|------|
| `cloneDefaultConfig()` | Deep-clone de `DEFAULT_CONFIG` avec meta fraîche (`source: 'default'`) |
| `isValidConfig(c)` | Vérifie version, days (len 7), typeLabels, typeColors, sessions.bureau/tele/weekend.sam/dim |
| `loadConfig()` | Lit `localStorage`, fallback sur `cloneDefaultConfig()` si absent/invalide |
| `saveConfig(newConfig, source)` | Valide, met à jour `meta.lastModified`, assigne `LIVE_CONFIG`, miroir localStorage, planifie sync Drive |
| `scheduleSaveConfig()` | Debounce 2 s avant `saveConfigToDrive()` |
| `loadConfigFromDrive()` / `saveConfigToDrive()` | I/O sur `fitness_programme_config.json` |

### Règle critique

Toute fonction de rendu doit lire via `getSessionsForDay(dayId)` **qui délègue à `LIVE_CONFIG.sessions`** — ne jamais référencer une constante `SESSIONS` figée. Idem pour `TYPE_LABELS` : passer par `LIVE_CONFIG.typeLabels`.

---

## Slider d'intensité par exercice

Chaque exercice a un slider 50 %–150 % (pas 5 %, défaut 100 %) qui multiplie à la fois la **durée des séries** et le **nombre de reps** (pas le temps de repos). Visible sur la carte d'exercice (dans la zone dépliée, au-dessus du tracker) et via un badge `×NN%` affiché à côté de la note de reps ainsi que dans la modale timer plein écran.

### Données

- `SCALES_STORAGE_KEY = 'fitness_exercise_scales'` — clé localStorage + champ dans le fichier Drive progression.
- `EXERCISE_SCALES` — objet `{ exId → percentage }`. **Global par exercice**, persiste d'une semaine à l'autre. N'est **pas** dans `LIVE_CONFIG` (donc l'export/import config reste portable).
- Valeur à 100 → entrée supprimée de l'objet pour garder le store compact.

### API

| Fonction | Rôle |
|----------|------|
| `loadScales()` | Lit `localStorage[SCALES_STORAGE_KEY]`, fallback `{}` |
| `getExerciseScale(exId)` | Retourne le % courant, 100 par défaut |
| `saveScale(exId, pct)` | Clamp+arrondi à [50,150] pas 5, miroir localStorage, push Drive via `scheduleSave()` |

### Helpers de scaling (point d'entrée unique)

Tout affichage de reps/sec **doit** passer par :

| Helper | Consommateur |
|--------|--------------|
| `scaledSec(ex)` | Timer d'exercice en durée (`parseReps` reçoit cette valeur) |
| `scaledSubSec(ex, sub)` | Chaque posture d'un exercice combiné (`renderSubExoTracker`) |
| `scaledRepsString(ex)` | Affichage de `ex.reps` sur la carte et dans la modale timer |

Règle : **ne jamais référencer `ex.sec`, `ex.reps`, `sub.sec` directement pour l'affichage ou le timer** — toujours passer par les helpers. L'admin lit encore les valeurs brutes (mode admin non scalé, slider masqué via CSS).

### Handlers

- `onExerciseScaleInput(dayId, exId, value)` — appelé sur `oninput` pendant le drag, met à jour le label `%` en live (sans re-render).
- `onExerciseScaleChange(dayId, exId, value)` — appelé sur `onchange` au relâchement : `saveScale`, puis rafraîchissement ciblé du reps label, du badge, et du `.series-tracker` (remplacement `innerHTML`). Si la modale timer est active pour un sid de cet exercice, ses `tm-reps` / `tm-scale` sont aussi rafraîchis.

### Comportement pendant une série active

Le scale est **capturé** dans `serieTimers[sid].secPerSet` au démarrage de la série. Modifier le slider en cours de série ne change **pas** le timer en cours — la nouvelle valeur s'applique à la série suivante. Volontaire.

---

## Mode admin

Activé par `toggleAdminMode()` (bouton ⚙ en topbar). Ajoute `body.admin-mode` qui masque la vue utilisateur et affiche `.admin-container` (éditeur en deux colonnes : arborescence + éditeur de champ).

### Fonctions clés

| Fonction | Rôle |
|----------|------|
| `renderAdminTree()` | Dessine l'arborescence jour → session → exercice, avec boutons d'ajout |
| `renderAdminEditor()` | Formulaire contextuel (selon la sélection) avec tous les champs d'un exercice/session/vélo |
| `adminAddSession/Exercise` | Création avec ID auto-généré unique (via `adminNewId`) |
| `adminMoveSession/Exercise` | Déplacement haut/bas dans la liste parente |
| `adminDuplicateSession/Exercise` | Clone avec nouvel ID |
| `adminDeleteSession/Exercise` | Suppression (avec confirmation côté UI) |
| `adminSubExoAdd/Delete/Move/Update` | Gestion des `subExos` d'un exercice combiné |
| `adminConvertToCombined/Simple` | Bascule un exercice entre format simple et `subExos[]` |
| `adminExportConfig()` | Télécharge `LIVE_CONFIG` en JSON |
| `adminImportConfig(event)` | Lit un fichier JSON, valide via `isValidConfig`, appelle `saveConfig(..., 'import')` |
| `adminResetConfig()` | `saveConfig(cloneDefaultConfig(), 'default')` après confirmation |

### Règles éditeur

- Toute édition appelle `saveConfig(...)` — pas de bouton « Sauvegarder ». Le debounce Drive se déclenche automatiquement.
- Les IDs saisis sont vérifiés contre `adminCollectAllExerciseIds()` pour éviter les doublons.
- La modale timer est forcée à `display:none` en `body.admin-mode` pour éviter toute interaction parasite (règle CSS explicite : `body.admin-mode .timer-modal{display:none !important}`).
- `parseRepSets` / `parseReps` / `isDurationEx` restent les fonctions de référence : si on édite un champ `reps` côté admin, son interprétation côté rendu n'est pas recalculée — il faut que le format reste compatible avec ces parseurs.

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

## Ordre d'exécution au boot (fonction `init`)

Tout se joue dans l'IIFE `(function init(){…})()` à la fin du `<script>` :

1. **`loadConfig()`** — hydrate `LIVE_CONFIG` depuis `localStorage` (ou `cloneDefaultConfig()` en fallback). DOIT précéder tout rendu : les fonctions de rendu lisent `LIVE_CONFIG`, pas les constantes figées.
2. **`EXERCISE_SCALES = loadScales()`** — hydrate les scales utilisateur depuis `localStorage[SCALES_STORAGE_KEY]`. DOIT précéder tout rendu : les helpers `scaledSec` / `scaledSubSec` / `scaledRepsString` les lisent à chaque appel.
3. **Détection du jour courant** — `getDay()` (JS : 0=dim) est remappé via `dayMap = [6,0,1,2,3,4,5]` vers l'index dans `LIVE_CONFIG.days` (0=lun).
4. **Rendu UI** — `updateWeekDisplay`, `renderAllPanels`, `renderTabs`, `renderOverview`.
5. **Bind admin events** — `bindAdminTreeEvents()`, `bindAdminEditorEvents()` (nécessaires même en mode utilisateur : les containers admin existent déjà dans le DOM).
6. **Différé 800 ms : `tryRestoreToken()`** — laisse la lib `gsi/client` finir son chargement asynchrone. Si un token valide existe, `loadFromDrive()` écrase `LIVE_CONFIG` (config Drive prioritaire) puis fusionne la progression + les scales, puis re-rend l'UI.
7. **Wake Lock** — première demande + écoute `visibilitychange` pour redemander au retour d'onglet (le lock est automatiquement relâché quand l'onglet n'est plus visible).

> **Règle critique** : ne jamais déclencher un rendu avant les étapes 1 et 2. Sinon `LIVE_CONFIG` vaut son état post-module (un simple clone de `DEFAULT_CONFIG`) et les scales utilisateur sont ignorés jusqu'au prochain boot.

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
- **Modifier le temps de repos (reps)** : utiliser le champ `restSec` sur l'exercice (ex: `restSec: 30`). Sans ce champ, la valeur par défaut est 60 s. Le champ n'est pas éditable depuis l'admin — modifier directement dans `DEFAULT_CONFIG`.
- **Modifier les données par défaut** : éditer `DEFAULT_CONFIG` — c'est la seule source figée dans le code. `LIVE_CONFIG` est mutable et reflète la config utilisateur
- **Ajouter une phase au timer** : la transition doit être gérée dans `tickSerieTimer` + `transitionTo*`/`finish*Phase` ET dans `skipModalTimer` (sinon le bouton ⏭ devient incohérent)
- **Nouveau libellé de phase** : ajouter la clé dans l'objet `labels` de `updateTimerModal` et la classe CSS correspondante (`.tm-phase.<nom>`)
- **Champ éditable côté admin** : ajouter le contrôle dans `renderAdminEditor`, le handler dans `bindAdminEditorEvents`/`adminApplyFieldEdit`, et toujours appeler `saveConfig(...)` — ne jamais muter `LIVE_CONFIG` directement sans passer par `saveConfig`
- **Import/export** : le JSON échangé doit passer `isValidConfig` — si on étend le schéma, bumper `version` et mettre à jour `isValidConfig` en conséquence
