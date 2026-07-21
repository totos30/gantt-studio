# CLAUDE.md — Gantt Studio

## Philosophie du projet

Gantt Studio est une application **mono-fichier HTML**, **100 % offline**, **sans authentification**, **sans serveur**, distribuable gratuitement. Elle doit rester légère, simple et autonome. Toute décision technique doit être évaluée à l'aune de ces principes :

- **Zéro infrastructure** : pas de backend, pas de base de données, pas d'API cloud.
- **Zéro inscription** : l'utilisateur ouvre le fichier, ça fonctionne.
- **Zéro dépendance runtime** : les bibliothèques tierces sont embarquées (inline ou lazy-loaded).
- **Un seul livrable** : tout réside dans `gantt-studio.html`.

---

## Stack technique

### Obligatoire
- **Vanilla JavaScript ES6+** : pas de framework (pas de React, Vue, Angular, Svelte).
- **SVG** pour toute visualisation (graphiques, diagrammes, exports). Pas de Canvas sauf conversion PNG.
- **CSS inline** dans le `<style>` du fichier HTML : CSS Grid, Flexbox, variables CSS (`--custom-property`).
- **DOM manipulation directe** : `document.getElementById`, `innerHTML`, `querySelector`.

### Dépendances autorisées
- Dépendances uniquement si elles peuvent être **embarquées en inline** ou **lazy-loadées** à la demande.
- Exemple accepté : SheetJS (XLSX) chargé uniquement lors d'un import/export Excel.
- Google Fonts via CDN est toléré (polices UI uniquement), avec une police système en fallback.
- Interdire tout CDN pour de la logique applicative.

### APIs navigateur utilisées
- `FileReader` — lecture fichiers utilisateur
- `Blob` + `URL.createObjectURL()` — téléchargement fichiers générés
- `localStorage` — persistance légère de l'état (optionnel)
- `SVGElement` — rendu vectoriel
- `ResizeObserver` / `window.resize` — layout adaptatif

---

## Architecture

### Structure du fichier unique
```
gantt-studio.html
├── <head>         meta, viewport, title, Google Fonts
├── <style>        CSS complet (design system, modals, composants)
├── <body>         Markup HTML (topbar, sidebar, zone principale, modals)
├── <script>       Bibliothèques embarquées (SheetJS minifié si nécessaire)
└── <script>       Logique applicative principale
```

### État global minimal
```javascript
let tasks  = [];      // tableau principal des données
let groups = [];      // groupes (mode roadmap)
let MODE   = 'gantt'; // vue active
let LANG   = 'fr';    // langue active
```

- **Deux tableaux font autorité** : `tasks` et `groups`. Pas de store, pas de proxy réactif.
- Les constantes de configuration (palette, échelles, tailles) sont déclarées en haut du `<script>` principal.
- Pas de classes ES6 pour l'état — préférer des objets plats et des fonctions pures.

### Pattern de rendu
- **Rendu complet à chaque changement** : reconstruire l'intégralité du SVG ou du HTML de la section concernée.
- Pas de virtual DOM, pas de diffing, pas de réconciliation.
- Les fonctions de rendu (`render()`, `renderSidebar()`, `buildSVG()`) sont idempotentes et déterministes.
- Utiliser `setTimeout(..., 0)` pour débouncer les re-rendus déclenchés par des événements rapides (resize).

### Internationalisation
- Système i18n maison : objet `I18N` avec clés communes, valeurs par langue.
- Fonction `t(key)` résout la chaîne selon `LANG`.
- Les éléments HTML portent `data-i18n="key"` ; `applyI18n()` les met à jour en masse.
- Langues supportées : **français (fr)** et **anglais (en)**. Le français est la langue par défaut.

---

## Modèle de données

### Objet `task`
```javascript
{
  id:          number,        // identifiant unique auto-incrémenté
  name:        string,        // nom de la tâche / jalon
  start:       Date,          // date de début
  end:         Date,          // date de fin
  progress:    number,        // 0–100
  milestone:   boolean,       // jalon ?
  responsible: string,        // responsable (optionnel)
  color:       string,        // couleur hex (#rrggbb)
  groupId:     number|null    // groupe roadmap
}
```

### Objet `group`
```javascript
{
  id:             number,
  name:           string,
  color:          string,
  statusOverride: string|null  // 'auto' | 'ok' | 'risk' | 'late'
}
```

Tout nouveau champ doit être **facultatif avec une valeur par défaut** pour ne pas casser l'import de données existantes.

---

## Règles de code

### Style
- Fonctions fléchées pour les callbacks et les fonctions courtes.
- `Array.map`, `filter`, `reduce` plutôt que des boucles impératives.
- Nommage en anglais (variables, fonctions). Commentaires en français autorisés.
- Pas de commentaires qui disent *quoi* — seulement *pourquoi* quand ce n'est pas évident.
- Séparateurs visuels `// ══════ SECTION ══════` pour délimiter les grandes sections du script.

### Fonctions
- Fonctions courtes et à responsabilité unique.
- Pas de classes — fonctions pures qui prennent des données et retournent un résultat ou mutent l'état global.
- Les fonctions de rendu SVG retournent une chaîne SVG, pas un effet de bord direct.

### SVG
- Générer le SVG comme une chaîne (`let svg = '<svg ...>'`) puis l'injecter via `innerHTML`.
- Utiliser `<clipPath>` pour le dépassement de texte.
- Calcul de contraste WCAG (luminance relative) pour choisir le texte blanc ou noir sur les barres colorées.
- Les exports PNG passent par `<canvas>` + `drawImage(img)` après conversion SVG → Data URL.

### Gestion des dates
- Helpers maison : `addDays(date, n)`, `diffDays(a, b)`, `parseDate(str)`, `toISO(date)`.
- Affichage localisé : `date.toLocaleDateString('fr-FR')` ou `'en-GB'` selon `LANG`.
- Pas de bibliothèque date (pas de date-fns, pas de moment.js).

### Import / Export
- **CSV** : génération manuelle (`headers.join(',')`, boucle sur `tasks`).
- **Excel** : SheetJS lazy-loadé au premier appel.
- **SVG** : sérialisation via `XMLSerializer` ou construction string.
- **PNG** : canvas temporaire, `canvas.toDataURL('image/png')`, lien `<a download>`.
- Toujours proposer un **template CSV téléchargeable** pour guider l'utilisateur.

---

## Accessibilité & UX

- Contraste texte/fond calculé programmatiquement (WCAG AA minimum sur les barres).
- Labels `aria-label` sur les contrôles icônes sans texte visible.
- Focus visible sur tous les éléments interactifs (pas de `outline: none` sans alternative).
- Responsive : le layout s'adapte au redimensionnement de la fenêtre sans rechargement.
- Pas de modal bloquant sans bouton de fermeture accessible (clic extérieur + bouton × + Échap).

---

## Performance

- **Pas de requêtes réseau** pendant l'utilisation (sauf Google Fonts au chargement initial).
- SVG régénéré en entier : acceptable car le nombre de tâches est borné (< 500 en pratique).
- Lazy-loading des bibliothèques lourdes (SheetJS) : injecter le `<script>` dynamiquement au premier besoin.
- Éviter les `setInterval` ; préférer des événements utilisateur ou un seul `requestAnimationFrame` si animation.
- Taille cible du fichier livrable : **< 2 Mo** (actuellement ~984 Ko).

---

## Ce qu'il ne faut PAS faire

| Interdit | Alternative |
|---|---|
| Framework JS (React, Vue…) | Vanilla JS + DOM direct |
| Backend / API | Tout dans le navigateur |
| Base de données distante | `localStorage` ou export fichier |
| Authentification | Aucun système de compte |
| NPM + bundler | Fichier HTML autonome |
| Dépendances CDN pour la logique | Embarquer inline ou lazy-load |
| Classes ES6 pour l'état | Objets plats + fonctions |
| Bibliothèques de dates | Helpers maison |
| `alert()` / `confirm()` natifs | Modals HTML custom |
| Commentaires décrivant le quoi | Commentaires expliquant le pourquoi |

---

## Déploiement & distribution

- **GitHub Pages** : le fichier `gantt-studio.html` à la racine du repo est servi directement.
- **Usage local** : l'utilisateur télécharge le fichier et l'ouvre dans son navigateur — ça fonctionne.
- **Pas de build step** : pas de `npm run build`, pas de Webpack, pas de Vite.
- Licence **MIT** : code open-source, réutilisable librement, sans restriction commerciale.

---

## Checklist avant toute modification

- [ ] La fonctionnalité fonctionne-t-elle **sans serveur** ?
- [ ] Le fichier reste-t-il **autonome** (un seul `.html`) ?
- [ ] Aucune nouvelle dépendance réseau n'est requise à l'exécution ?
- [ ] Le rendu SVG reste **exportable** (SVG + PNG) ?
- [ ] Les deux langues (fr / en) sont-elles couvertes dans `I18N` ?
- [ ] Les données existantes restent-elles **compatibles** (rétrocompatibilité du format CSV/Excel) ?
- [ ] La taille du fichier reste-t-elle **< 2 Mo** ?
