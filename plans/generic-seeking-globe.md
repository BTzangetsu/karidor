# Drag & Drop des murs avec preview

## Context

Actuellement, les murs se placent par clic : on clique sur "Mur H" / "Mur V" puis on clique sur un slot. L'objectif est d'ajouter un drag & drop intuitif : l'utilisateur tire un mur depuis le bouton d'action et le dépose sur le slot du plateau, avec un aperçu visuel qui suit le curseur. Le clic existant reste fonctionnel (approche additive).

## Approche : Pointer Events

Utiliser les **Pointer Events** (`pointerdown`, `pointermove`, `pointerup`) pour unifier souris + tactile. Créer un élément fantôme (`.wpiece-ghost`) en position `fixed` qui suit le curseur. Détecter le slot de destination via `document.elementFromPoint()`.

## Fichiers à modifier

| Fichier | Changements |
|---------|-------------|
| `static/js/game.js` | Variables d'état drag, fonctions `startWallDrag`, `onDragMove`, `onDragEnd`, `updateGhostPosition`; ajout `data-r`/`data-c` sur les gaps; hook `onpointerdown` sur les boutons Mur H/V |
| `static/css/game.css` | Styles `.wpiece-ghost`, `.drag-hover`, `.dragging` |

## Étapes d'implémentation

### 1. État drag (`game.js`, après `let mode = 'move'`)

```js
let dragState = {
    active: false,
    wallType: null,       // 'h' ou 'v'
    startX: 0, startY: 0,
    ghostEl: null,
    highlightEl: null,
    minDistance: 5,
};
```

### 2. Rendre les boutons Mur H/V draggable (`renderActionRow()`, ligne ~274)

Ajouter `onpointerdown="startWallDrag(event, '${m.id}')"` et `style="touch-action:none;user-select:none;"` sur les boutons wall-h et wall-v. Le clic existant (`onclick="setMode(...)"`) reste intact — le seuil `minDistance: 5` distingue clic vs drag.

### 3. `startWallDrag(event, modeId)`

- Vérifie `isMyTurn()` et `hasWallsLeft()`
- Crée un `<div class="wpiece wpiece-ghost">` positionné `fixed`, `pointer-events: none`, `z-index: 1000`
- Dimensionne le fantôme selon le type (horizontale : large × WALL_GAP, verticale : WALL_GAP × haut)
- Ajoute les listeners globaux `pointermove` / `pointerup` / `pointercancel` en capture
- Ajoute la classe `dragging` au body

### 4. `onDragMove(event)`

- Met à jour la position du fantôme via `updateGhostPosition()`
- Cache le fantôme, appelle `elementFromPoint()`, le réaffiche
- Trouve le `.gap-h` / `.gap-v` sous le curseur via `closest()`
- Applique `.drag-hover` sur le slot cible (highlight visuel)
- Valide que le type de gap correspond au type de mur

### 5. `onDragEnd(event)`

- Nettoyage : suppression du fantôme, retrait `.drag-hover`, `.dragging`, listeners
- Si déplacement ≥ `minDistance` et un slot valide est sous le curseur → émet `socket.emit('play_wall', { r, c, vertical })`
- Récupère les coordonnées via `data-r` / `data-c` sur l'élément gap
- Réinitialise `dragState`

### 6. Attributs data sur les gaps (`renderBoard()`, lignes ~153 et ~170)

Ajouter `data-r="${r}" data-c="${c+1}"` sur `.gap-v` et `data-r="${r+1}" data-c="${c}"` sur `.gap-h`.

### 7. CSS (`game.css`)

```css
.wpiece-ghost { transition: none; }

.gap-h.drag-hover .wpiece,
.gap-v.drag-hover .wpiece {
    background: var(--wall-placed);
    opacity: 0.6;
}

.abtn[onpointerdown] { cursor: grab; }
.abtn[onpointerdown]:active { cursor: grabbing; }

body.dragging {
    user-select: none;
    -webkit-user-select: none;
}
```

## Zéro changement backend

Le flow drag émet le même événement `play_wall` que le clic. Le serveur (`app/sockets/events.py`) ne change pas. La réponse `game_update` re-rend le plateau via le listener existant (ligne 431).

## Vérification

1. Lancer le serveur Flask : `python run.py`
2. Ouvrir le jeu dans 2 onglets, créer une partie
3. **Drag souris** : tirer depuis "Mur H" vers un slot horizontal → le fantôme suit, le slot s'illumine au survol, le mur se place au relâchement
4. **Drag tactile** : tester sur mobile/simulation mobile
5. **Clic préservé** : cliquer "Mur H" + cliquer un slot devrait toujours fonctionner
6. **Edge cases** : drag hors du plateau (ne place rien), re-render pendant le drag (fantôme survive), drag sur slot de type incompatible (ne highlight pas)
