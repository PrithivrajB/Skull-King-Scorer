# Skull King Scorer — Documentation technique

## Objectif

Application web de décompte de points pour le jeu de cartes **Skull King**. Fonctionne entièrement hors-ligne dans un navigateur — aucun serveur, aucune dépendance à installer. Un seul fichier HTML autonome (`index.html`) à ouvrir directement.

---

## Table des matières

1. [Technologies](#technologies)
2. [Architecture](#architecture)
3. [Modèle de données](#modèle-de-données)
4. [Logique de scoring](#logique-de-scoring)
5. [Système d'alliances (Butin)](#système-dalliances-butin)
6. [Navigation entre manches](#navigation-entre-manches)
7. [Barre de stats de manche](#barre-de-stats-de-manche)
8. [Classement et ex æquo](#classement-et-ex-æquo)
9. [Persistance](#persistance)
10. [Graphique — Plugin Sonar](#graphique--plugin-sonar)
11. [Image téléchargeable](#image-téléchargeable-downloadresult)
12. [Interface & animations](#interface--animations)
13. [Fonctions principales](#fonctions-principales)
14. [Tests intégrés](#tests-intégrés)
15. [Ajouter une feature](#ajouter-une-feature)
16. [Charte graphique](#charte-graphique)
17. [Limites connues](#limites-connues)

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
│   ├── #global-bar         — barre sticky avec logo pirate, infos partie, menu hamburger
│   ├── #modal-confirm      — modale générique de confirmation (titre + message HTML + callback)
│   ├── #map                — modale ajout joueur en cours de partie
│   ├── #home               — écran d'accueil (nouvelle partie, reprendre, historique)
│   ├── #compte-col         — écran de décompte (écran principal)
│   └── #classement         — écran classement final + graphique + actions partage
└── <script>      — toute la logique JS
```

### Gestion des écrans

Trois écrans `div.screen` avec la classe `.hidden`. La fonction `showScreen(id)` bascule leur visibilité.

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
  "done": false,
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
| `done` | bool | `true` uniquement après clic "Voir le classement" sur la modale de fin |
| `playerOrder` | string[] | Ordre d'affichage des cartes (drag-reorder) |
| `cumuls[pid][i]` | int | Score cumulé du joueur `pid` après la manche `i` |
| `butins` | Alliance[] | Alliances actives pour cette manche |
| `rascal` | null \| 10 \| 20 | Valeur Rascal (null = inactif) |

---

## Logique de scoring

Fonction centrale : `calcScore(data, N, allJoueurs)`.

### Règles de base

**Mise réussie (`bidOk`) :**
- `mise === 0 && plis === 0` → `10 × N` points
- `mise >= 1 && plis === mise` → `20 × plis` points

**Mise ratée :**
- `−10 × |mise − plis|`
- `mise === 0 && plis > 0` → `−10 × N` (pénalité fixe indépendante du nombre de plis pris)

### Bonus (seulement si `bidOk`)

| Bonus | Valeur | Max | Exception mise=0 réussie |
|---|---|---|---|
| Pirate capturé par sirène | +30 chacun | 7 | ✗ ignoré |
| Skull King capturé par sirène | +40 | 1 | ✗ ignoré |
| Pirate capturé par Skull King | +20 chacun | 2 | ✗ ignoré |
| **Léviathan capturé** | **+20 chacun** | **3** | **✅ s'applique** |
| Carte 8 capturée | +5 chacune | 4 | ✗ ignoré |
| Carte 7 capturée | −5 chacune | 4 | ✗ ignoré |
| 14 couleur capturé | +10 chacun | 3 | ✗ ignoré |
| 14 noir capturé | +20 | 1 | ✗ ignoré |
| Alliance (Butin) | +20 par alliance | — | voir règles butin |
| Bonus manuel | valeur libre | — | ✅ s'applique |

> **Règle spéciale Léviathan :** si un joueur a réussi son pari à 0 (mise=0, plis=0), le bonus léviathan s'applique quand même. C'est la seule exception aux bonus sur mise=0.

### Rascal

- Si `bidOk` et `mise >= 1` : ajoute la valeur (10 ou 20)
- Si raté et `mise >= 1` : soustrait la valeur
- Ignoré si `mise === 0`

---

## Système d'alliances (Butin)

Une alliance lie deux joueurs pour une manche. Chaque joueur gagne **+20** si les deux ont respecté leur mise.

```js
manche.butins = [
  { playerA: "p1", playerB: "p2" }
]
```

### Règles d'application

| Situation | Bonus |
|---|---|
| A ok (mise≥1) + B ok (mise≥1) | +20 chacun ✅ |
| A bid=0 ok + B bid≥1 ok | +20 chacun ✅ |
| A raté + B ok (ou inversé) | 0 ❌ |
| **A bid=0 ok + B bid=0 ok** | **0 ❌ (règle de protection)** |

> **Règle de protection :** si les deux alliés ont une mise à 0 et aucun pli, le bonus d'alliance ne s'applique pas. Cela évite les abus où deux joueurs coordonnent leur mise à 0.

Les butins sont illimités par manche. Un joueur peut apparaître dans plusieurs alliances.

### Fonctions butin

```js
stepButin(delta)          // ajoute (+1) ou retire (−1) un butin, max 2
openAlliancePanel()       // ouvre le panneau avec animation max-height
closeAlliancePanel()      // ferme avec animation, vide le DOM après 320 ms
toggleAlliancePanel()     // bascule ouvert/fermé
renderAlliancePanel()     // reconstruit entièrement les lignes du panneau
removeAlliance(idx)       // supprime l'alliance à l'index idx
setAllianceSide(idx, side, val) // change playerA ou playerB d'une alliance
```

---

## Navigation entre manches

```js
nextRound()           // valide la manche courante et avance (ou ouvre modale fin)
proceedNextRound()    // avance réellement après validation
prevRound()           // recule d'une manche
nextRoundNav()        // navigation parmi les manches déjà jouées (sans avancer)
loadRound(n)          // charge directement la manche n
saveRound()           // sauvegarde + recalcule les cumuls
```

### Fin de partie

Le bouton de la dernière manche s'appelle **"Fin du voyage"** (au lieu de "Prochaine traversée").

`state.done` est mis à `true` **uniquement** lorsque l'utilisateur clique "Voir le classement" dans la modale de fin — pas au clic du bouton fin de manche. Cela permet de revoir les scores de la dernière manche avant de finaliser.

---

## Barre de stats de manche

Une rangée de 4 chips affichée entre le séparateur et les cartes joueurs, recalculée à chaque modification :

| Chip | Contenu |
|---|---|
| **MISES** | Somme des mises annoncées |
| **PLIS** | Total des plis pris (rouge si > N) |
| **BONUS** | Somme des points positifs gagnés ce tour |
| **MALUS** | Somme des pénalités ce tour |

```js
renderMancheStats()   // appelé depuis recalcManche() et renderCards()
```

---

## Classement et ex æquo

Ranking olympique (1-2-2-4, pas 1-2-2-3) :

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

- Toutes les parties sont listées sur l'accueil avec podium et date.
- `state.current = true` marque la partie active (bouton "Continuer").
- `state.done = true` marque une partie terminée (classement figé).
- Auto-save déclenché 300 ms après chaque saisie (debounce `asTimer`).

---

## Graphique — Plugin Sonar

Le graphique de progression utilise un plugin Chart.js custom qui remplace la grille cartésienne par des **arches concentriques** de type radar sous-marin, origine en bas au centre.

```js
const sonarPlugin = {
  id: 'sonar',
  beforeDraw(chart) {
    const { ctx, chartArea: ca } = chart;
    const cx = (ca.left + ca.right) / 2, cy = ca.bottom;
    const maxR = Math.hypot(ca.right - ca.left, ca.bottom - ca.top) * 1.05;
    for (let i = 1; i <= 7; i++) {
      ctx.arc(cx, cy, (maxR / 7) * i, Math.PI, 0);
      ctx.strokeStyle = `rgba(232,160,32,${0.03 + i * 0.025})`;
    }
    // 10 rayons radiaux
    for (let i = 0; i <= 10; i++) {
      const ang = Math.PI + (Math.PI / 10) * i;
      ctx.strokeStyle = 'rgba(232,160,32,0.04)';
    }
  }
};
```

**Tooltip :** fond sombre `rgba(4,14,28,0.97)`, bordure or, police Cinzel.

**Axes Y :** entiers uniquement (`callback: v => Number.isInteger(v) ? v : ''`).

**Ligne zéro :** tracée en pointillés or uniquement si les scores contiennent des valeurs négatives ET que la ligne zéro se situe entre 10 % et 75 % de la hauteur du graphique (évite de perturber le sonar en bas).

---

## Image téléchargeable (`downloadResult`)

Génère une image canvas 900×1600 px (ratio 9:16) téléchargeable en PNG.

### Structure de l'image (haut → bas)

1. **En-tête** — icône crâne, "Skull King" (Cinzel Decorative), `◆ SCORER ◆`
2. **Bloc vainqueur** — titre, nom du gagnant (couleur du joueur), score, citation pirate
3. **CLASSEMENT** — tableau avec badges rang circulaires, points colorés, scores
4. **ÉVOLUTION DES SCORES** — graphique sonar reproduit en canvas 2D avec courbes de Bézier monotones
5. **Légende** — multi-lignes par **bin-packing glouton** (mesure réelle de chaque label, jamais de chevauchement)
6. **Pied de page** — ligne or + date

### Points techniques

- La hauteur de légende est **pré-calculée avant** `cArea` pour ajuster `pad.bottom` dynamiquement.
- Les courbes utilisent l'algorithme de spline cubique monotone (identique à Chart.js `cubicInterpolationMode: 'monotone'`).
- Aucun emoji dans le canvas (non-rendu cross-platform) — tout en formes et texte Canvas 2D.

---

## Interface & animations

### Barre globale (`#global-bar`)

Sticky en haut. Contient : logo pirate SVG, nom de la partie + manche courante, bouton hamburger.

Le menu hamburger (`#hbg-panel`) s'ouvre via `.open` (opacity + translateY en 200 ms). Actions : Accueil, Recommencer, Quitter.

### Badges de rang

| Contexte | Composant | Format |
|---|---|---|
| Tableau classement & side panel | `.rb2` | Cercle 24×24 px |
| En-tête carte joueur | `.rank-badge` | Ovale avec `#N` |

Couleurs : or (#1), argent (#2), bronze (#3), gris translucide (autres).

### Animations

| Élément | Mécanisme | Durée |
|---|---|---|
| Modales | `.mo.open` → backdrop + `.mb` translateY+scale | 250–280 ms |
| Panneau alliance | `max-height 0→2000px` + `opacity` | 300 ms, puis `overflow:visible` |
| Side-panel classement | `max-height 0→800px` + `opacity` sur `.open` | 380 ms |
| Panneau bonus carte | `max-height 0→600px` + `opacity` sur `.open` | 350 ms |
| Menu hamburger | `opacity` + `translateY(-8px)→0` | 200 ms |

### Scrollbar

Stylisée à la charte : 4 px de large, quasi-invisible (`rgba(gold, .18)`), légèrement visible au hover. `scrollbar-width: thin` pour Firefox.

---

## Fonctions principales

### Rendu

| Fonction | Description |
|---|---|
| `renderCards()` | Re-génère toutes les cartes joueurs pour la manche courante |
| `updateCardScores(pid, ri)` | Met à jour score/cumul d'une carte sans re-render complet |
| `renderAllianceInfoOnCards()` | Affiche les alliances sur chaque carte |
| `renderAlliancePanel()` | Reconstruit entièrement les lignes du panneau butin |
| `renderMancheStats()` | Met à jour la barre de chips MISES/PLIS/BONUS/MALUS |
| `renderClassement()` | Remplit le tableau de classement + bannière victoire |
| `renderChart()` | Instancie/recrée le graphique Chart.js |
| `renderSidePanel()` | Met à jour le panneau classement inline (vue séparée) |

### Scoring

| Fonction | Description |
|---|---|
| `calcScore(data, N, allJoueurs)` | Calcule le score d'un joueur pour une manche |
| `calcCard(inp)` | Wrapper de `calcScore` avec contexte `state` courant |
| `recalcManche()` | Recalcule tous les scores de la manche courante + stats |
| `recalcCumuls(from)` | Recalcule les cumuls de la manche `from` jusqu'à la fin |
| `computeRanks(entries)` | Ranking olympique — retourne les entrées avec `.rank` |

### Navigation

| Fonction | Description |
|---|---|
| `loadRound(n)` | Charge la manche n |
| `nextRound()` | Valide la manche courante, avance ou affiche modale fin |
| `proceedNextRound()` | Avance réellement (appelé après validation) |
| `prevRound()` | Recule d'une manche |
| `saveRound()` | Sauvegarde + recalcule les cumuls |
| `checkWinner()` | Calcule et affiche la modale de fin de partie |

### Sessions

| Fonction | Description |
|---|---|
| `startNewGame()` | Crée un nouvel état et démarre |
| `loadSession(id)` | Charge une partie sauvegardée |
| `saveGame()` | Persiste `state` en localStorage |
| `deleteSession(id)` | Supprime une partie |
| `getAllSessions()` | Liste toutes les clés `skull_session_*` |
| `loadFinished(id)` | Ouvre une partie terminée en lecture seule |

---

## Tests intégrés

Le fichier contient **~68 assertions** organisées en groupes, désactivées par défaut (`ENABLE_DEV_TESTS = false`).

```js
// Activer en passant ENABLE_DEV_TESTS = true et en rechargeant la page
```

### Groupes de tests

| Groupe | Tests | Contenu |
|---|---|---|
| BASE | T01–T11 | bid ok/raté, mise=0 ok/raté, pénalités |
| BONUS | T12–T25 | 14 couleur/noir, pirate, SK, sirène, combos |
| LÉVIATHAN | T26–T31 | bid réussi + lev, bid raté, **exception mise=0** |
| BUTIN/ALLIANCE | T32–T36 | A+B ok, A raté, bid=0 mixte, **règle de protection** |
| SCÉNARIO S1 | 14 asserts | 3 joueurs, 4 manches, cumuls intermédiaires |
| SCÉNARIO S2 | 20 asserts | 10 joueurs, N=6, totaux de départ |
| VALIDATIONS | V01–V05 | cumuls négatifs, rascal bid=0, somme plis |

---

## Ajouter une feature

### Nouveau champ de bonus

1. Ajouter le champ dans `emptyJ(pid, nom)` avec valeur par défaut `0`
2. Ajouter une entrée dans `BOUNDS` avec les bornes min/max
3. Ajouter `sField(label, pid, 'monChamp', jd.monChamp)` dans `card.innerHTML` (section `.pc-bonus`)
4. Intégrer dans `calcScore` : `const v = Math.min(i(data.monChamp), BOUNDS.monChamp().max); if (ok) bon += v * VALEUR;`

### Nouveau type d'écran

1. Créer un `<div id="mon-ecran" class="screen hidden">` dans le HTML
2. Appeler `showScreen('mon-ecran')` pour y naviguer

### Modifier la charte graphique

Toutes les couleurs passent par les variables CSS dans `:root`. Modifier `--gold`, `--gold-d`, `--gold-l` pour changer l'ensemble du thème. Les couleurs des joueurs sont dans le tableau `PCOLORS`.

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
['#fbbf24','#ef4444','#2563eb','#059669','#7c3aed',
 '#ec4899','#f97316','#22d3ee','#b4f020','#c4b5fd']
```

---

## Limites connues

- **Pas de réseau** : tout est local. Pas de multi-appareil synchronisé.
- **Max 10 joueurs** : contrainte des couleurs prédéfinies.
- **localStorage** : limité à ~5 Mo par domaine. Des centaines de parties peuvent être sauvegardées.
- **Pas de PWA** : peut être ajouté avec un Service Worker et un `manifest.json`.
- **"Obtenir un lien"** : placeholder — fonctionnalité non encore implémentée.