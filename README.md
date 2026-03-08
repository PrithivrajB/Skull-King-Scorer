# Skull King Scorer — Documentation technique

## Objectif

Application web de décompte de points pour le jeu de cartes **Skull King**. Fonctionne entièrement hors-ligne dans un navigateur — aucun serveur, aucune dépendance à installer. Un seul fichier HTML autonome (`index.html`) à ouvrir directement.

---

## Technologies

| Technologie | Usage |
|---|---|
| **HTML/CSS/JS vanille** | Architecture complète — pas de framework |
| **Chart.js** (CDN) | Graphique de progression des scores |
| **Google Fonts** | Cinzel Decorative, Cinzel, Crimson Text |
| **localStorage** | Persistance des parties entre sessions |

Dépendances CDN :
```
https://cdn.jsdelivr.net/npm/chart.js
https://fonts.googleapis.com/css2?family=Cinzel+Decorative...
```

---

## Architecture

### Structure du fichier

```
index.html
├── <head>        — imports CDN, styles CSS complets
├── <body>
│   ├── #global-bar         — barre sticky avec bouton hamburger (menu déroulant)
│   ├── #modal-confirm      — modale générique de confirmation
│   ├── #map                — modale ajout joueur en cours de partie
│   ├── #home               — écran d'accueil
│   ├── #compte-col         — écran de décompte (écran principal)
│   └── #classement         — écran classement + graphique
└── <script>      — toute la logique JS
```

### Gestion des écrans

Trois écrans `div.screen` avec la classe `.hidden`. La fonction `showScreen(id)` bascule leur visibilité et affiche/masque le bouton hamburger (caché sur l'accueil).

```js
showScreen('home')       // accueil
showScreen('compte')     // décompte
showScreen('classement') // classement
```

---

## Modèle de données

### Objet `state` (source de vérité unique)

```json
{
  "schemaVersion": 1,
  "id": "uuid-like-string",
  "name": "Partie du 08/03",
  "currentRound": 3,
  "maxReached": 5,
  "config": { "manches": 8 },
  "playerOrder": ["p1", "p2", "p3"],
  "players": [
    { "id": "p1", "nom": "Jack", "color": "#e8a020" }
  ],
  "manches": [
    {
      "N": 1,
      "butins": [{ "playerA": "p1", "playerB": "p2" }],
      "joueurs": [
        {
          "id": "p1", "nom": "Jack",
          "mise": 2, "plis": 2,
          "pirate": 1, "sk": 0, "sirene": 0, "leviathan": 0,
          "rascal": null,
          "got7": 0, "got8": 0, "got14color": 0, "got14black": 0,
          "bonus": 0,
          "score": 60
        }
      ]
    }
  ],
  "cumuls": {
    "p1": [60, 130, 190]
  },
  "current": true
}
```

### Champs clés

| Champ | Type | Description |
|---|---|---|
| `currentRound` | int | Manche affichée (1-based) |
| `maxReached` | int | Manche la plus haute jamais atteinte |
| `playerOrder` | string[] | Ordre d'affichage des cartes (drag-reorder) |
| `cumuls[pid][i]` | int | Score cumulé du joueur `pid` après la manche `i` |
| `butins` | Alliance[] | Alliances actives pour cette manche |
| `rascal` | null \| 0 \| 10 \| 20 | Valeur Rascal (null = inactif) |

---

## Logique de scoring

Fonction centrale : `calcScore(data, N, allJoueurs)`.

### Règles

**Mise réussie (`bidOk`) :**
- `mise === 0 && plis === 0` → `10 × N` points
- `mise >= 1 && plis === mise` → `20 × plis` points

**Mise ratée :**
- `−10 × |mise − plis|`

**Bonus (seulement si `bidOk`) :**
| Bonus | Valeur |
|---|---|
| Pirate capturé | +20 chacun (max 7) |
| Skull King capturé | +40 (max 1) |
| Sirène capturée | +30 chacune (max 2) |
| Léviathan capturé | +20 chacun (max 10) |
| Carte 8 capturée | +5 chacune (max 4) |
| Carte 7 capturée | −5 chacune (max 4) |
| 14 couleur capturé | +10 chacun (max 3) |
| 14 noir capturé | +20 (max 1) |
| Alliance (Butin) | +20 par alliance **si les 2 joueurs ont rempli leur mise** |
| Bonus manuel | valeur libre |

**Rascal :**
- Si `bidOk` : ajoute la valeur (0, 10 ou 20)
- Si raté : soustrait la valeur

```js
function calcScore(data, N, allJoueurs = []) {
  // ...
  const ok = (mise === 0 && plis === 0) || (mise >= 1 && plis === mise);
  let base = ok
    ? (mise === 0 ? 10 * N : 20 * plis)
    : -10 * Math.abs(mise - plis);
  // bonus seulement si ok...
  return base + bon + rv;
}
```

---

## Système d'alliances (Butin)

Une alliance lie deux joueurs pour une manche. Chaque joueur gagne +20 si **les deux** ont respecté leur mise.

```js
// Structure
manche.butins = [
  { playerA: "p1", playerB: "p2" }
]
```

Les butins sont illimités par manche. Un joueur peut apparaître dans plusieurs alliances. `calcScore` reçoit `allJoueurs` pour vérifier la mise du partenaire.

---

## Classement et ex æquo

Implémentation du **ranking olympique** (1-2-2-4, pas 1-2-2-3) :

```js
function computeRanks(entries) {
  const sorted = [...entries].sort((a, b) => b.score - a.score);
  let rank = 1;
  for (let i = 0; i < sorted.length; i++) {
    if (i > 0 && sorted[i].score < sorted[i - 1].score) rank = i + 1;
    sorted[i].rank = rank;
  }
  return sorted;
}
```

Exemples :
```
A30 B30 C28      → rangs 1 1 3
A35 B30 C30 D25  → rangs 1 2 2 4
A30 B30 C30 D30  → rangs 1 1 1 1
```

---

## Persistance

Clé localStorage : `skull_session_<id>`.

```js
function sk(id) { return 'skull_session_' + id; }
function saveGame() { localStorage.setItem(sk(state.id), JSON.stringify(state)); }
function loadSession(id) { state = JSON.parse(localStorage.getItem(sk(id))); }
```

- Toutes les parties sauvegardées sont listées sur l'accueil.
- `state.current = true` marque la partie active (bouton "Continuer").
- Auto-save déclenché 300ms après chaque saisie (timer debounce `asTimer`).

---

## Graphique — Plugin Sonar

Le graphique de progression utilise un plugin Chart.js custom qui remplace la grille cartésienne par des **arches concentriques** de type radar sous-marin.

```js
const sonarPlugin = {
  id: 'sonar',
  beforeDraw(chart) {
    const { ctx, chartArea: ca } = chart;
    const cx = (ca.left + ca.right) / 2, cy = ca.bottom;
    const maxR = Math.hypot(ca.right - ca.left, ca.bottom - ca.top) * 1.05;
    // 7 arches dorées semi-transparentes
    for (let i = 1; i <= 7; i++) {
      ctx.beginPath();
      ctx.arc(cx, cy, (maxR / 7) * i, Math.PI, 0);
      ctx.strokeStyle = `rgba(232,160,32,${0.03 + i * 0.025})`;
      ctx.stroke();
    }
    // 10 rayons radiaux
    for (let i = 0; i <= 10; i++) {
      const ang = Math.PI + (Math.PI / 10) * i;
      ctx.beginPath();
      ctx.moveTo(cx, cy);
      ctx.lineTo(cx + Math.cos(ang) * maxR, cy + Math.sin(ang) * maxR);
      ctx.strokeStyle = 'rgba(232,160,32,0.04)';
      ctx.stroke();
    }
  }
};
Chart.register(sonarPlugin);
```

---

## Charte graphique

| Token | Valeur | Usage |
|---|---|---|
| `--gold` | `#e8a020` | Couleur principale, titres, bordures |
| `--gold-d` | `#b07818` | Boutons primaires, hover |
| `--gold-l` | `#f5c84a` | Highlights, hover steppers |
| `--ember` | `#c0392b` | Hover danger, drag-over |
| `--text` | `#e8d5a8` | Texte courant |
| `--dim` | `#7a6545` | Labels secondaires, placeholders |
| Fond | `#01050d → #000508` | Abysses, quasi-noir |
| Cartes | `#07090d → #09111d` | Gradient bleu-nuit très sombre |

Polices : **Cinzel Decorative** (titres/scores), **Cinzel** (labels/boutons), **Crimson Text** (corps).

Couleurs joueurs (10 max) :
```js
['#fbbf24','#ef4444','#3b82f6','#10b981','#8b5cf6',
 '#ec4899','#f97316','#06b6d4','#84cc16','#a78bfa']
```

---

## Interface & animations

### Hamburger menu (barre globale)

Le bouton `#hbg-btn` remplace les 3 boutons individuels. Il affiche un panneau `#hbg-panel` via la classe `.open` (opacity + transform en 200 ms). Le clic en dehors le referme automatiquement via un listener global.

```js
toggleHamburger()   // ouvre/ferme
closeHamburger()    // ferme uniquement
```

Actions disponibles dans le menu : **Accueil**, **Recommencer**, **Quitter**.

### Animations

| Élément | Mécanisme | Durée |
|---|---|---|
| Modales (confirm, ajout joueur) | `.mo.open` → backdrop + `.mb` translateY+scale | 250–280 ms |
| Side-panel classement | `max-height 0→800px` + `opacity` sur `.open` | 380 ms |
| Panneau bonus (par carte) | `max-height 0→600px` + `opacity` sur `.open` | 350 ms |
| Menu hamburger | `opacity` + `translateY(-8px)→0` sur `.open` | 200 ms |

Toutes les animations utilisent `cubic-bezier(.34,1.4,.64,1)` pour un ressort léger. Aucun `display:none` direct — les éléments restent dans le DOM et sont masqués par `opacity:0` + `pointer-events:none`.

### Icône Butins

SVG fourni par l'utilisateur (coffre ouvert avec verrou et bretelles), traits recolorés en `var(--gold)` (`#e8a020`), taille 18×18 px, aligné verticalement avec `vertical-align:middle`.

---

## Fonctions principales

### Rendu

| Fonction | Description |
|---|---|
| `renderCards()` | Re-génère toutes les cartes joueurs pour la manche courante |
| `updateCardScores(pid, ri)` | Met à jour le score/cumul d'une carte sans re-render complet |
| `renderAllianceInfoOnCards()` | Affiche les alliances sur chaque carte |
| `renderAlliancePanel()` | Génère le panneau de gestion des alliances |
| `renderClassement()` | Remplit le tableau de classement et affiche la bannière victoire |
| `renderChart()` | Instancie/recrée le graphique Chart.js |
| `renderSidePanel()` | Met à jour le panneau classement inline (vue séparée) |

### Scoring

| Fonction | Description |
|---|---|
| `calcScore(data, N, allJoueurs)` | Calcule le score d'un joueur pour une manche |
| `calcCard(inp)` | Wrapper de `calcScore` avec le contexte `state` courant |
| `recalcCumuls(from)` | Recalcule les cumuls de la manche `from` jusqu'à la fin |
| `computeRanks(entries)` | Ranking olympique, retourne les entrées avec `.rank` |

### Navigation

| Fonction | Description |
|---|---|
| `loadRound(n)` | Charge la manche n |
| `nextRound()` | Valide la manche courante, avance (ou affiche le classement final) |
| `prevRound()` | Recule d'une manche |
| `saveRound()` | Sauvegarde + recalcule les cumuls |

### Sessions

| Fonction | Description |
|---|---|
| `startNewGame()` | Crée un nouvel état et démarre |
| `loadSession(id)` | Charge une partie sauvegardée |
| `saveGame()` | Persiste `state` en localStorage |
| `deleteSession(id)` | Supprime une partie |
| `getAllSessions()` | Liste toutes les clés `skull_session_*` |

---

## Ajouter une feature

### Nouveau champ de bonus

1. Ajouter le champ dans `emptyJ(pid, nom)` avec valeur par défaut `0`
2. Ajouter une entrée dans `BOUNDS` avec les bornes min/max
3. Ajouter `sField(label, pid, 'monChamp', jd.monChamp)` dans `card.innerHTML` (section `.pc-bonus`)
4. Intégrer dans `calcScore` : `const v = Math.min(i(data.monChamp), BOUNDS.monChamp().max); if (ok) bon += v * VALEUR;`

### Nouveau type d'écran

1. Créer un `<div id="mon-ecran" class="screen hidden">` dans le HTML
2. Ajouter `'mon-ecran'` dans la liste de `showScreen`
3. Appeler `showScreen('mon-ecran')` pour y naviguer

### Modifier la charte graphique

Toutes les couleurs passent par les variables CSS dans `:root`. Il suffit de modifier `--gold`, `--gold-d`, `--gold-l` pour changer l'ensemble du thème. Les couleurs des joueurs sont dans le tableau `PCOLORS`.

---

## Tests intégrés

Le fichier contient 12 tests unitaires désactivés par défaut (`ENABLE_DEV_TESTS = false`).

```js
// Activer dans la console :
// Modifier ENABLE_DEV_TESTS = true, recharger
```

Cas couverts : bid OK/raté, bonus ignorés sur échec, rascal positif/négatif, mise=0 plis=0 (N×10), Skull King, Sirène, alliance, recalcCumuls, isolation localStorage.

---

## Limites connues

- **Pas de réseau** : tout est local. Pas de multi-appareil synchronisé (sauf onglets du même navigateur via l'événement `storage`).
- **Max 10 joueurs** : contrainte des couleurs prédéfinies.
- **localStorage** : limité à ~5 Mo par domaine. En pratique, des centaines de parties peuvent être sauvegardées.
- **Pas de PWA** : peut être ajouté avec un Service Worker et un `manifest.json` si nécessaire.
