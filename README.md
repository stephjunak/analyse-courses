# Analyse courses — l'app

Web app de suivi des dépenses de courses, partagée entre Stéphanie et Frédéric.
Reproduit ce que faisait le classeur `Suivi-depenses-supermarche.xlsx` (onglets
Détail + Synthèse), en ligne et à deux.

> Ce fichier est une **notice de mise en place et de dépannage**. Le suivi du
> projet (état, sessions, backlog) vit dans `../README.md`, mis à jour via `/fini`.

**En ligne :** https://stephjunak.github.io/analyse-courses/
**Dépôt :** https://github.com/stephjunak/analyse-courses

---

## Architecture

Un seul `index.html`, CSS et JS inclus dedans, zéro build, zéro npm. Le patron
est celui de `fav-deco`.

- **Données : Firestore** (projet `analyse-courses-5580f`), en REST direct via
  `fetch`, sans SDK. Collection `depenses`, un document par article.
- **Graphiques : Chart.js v4**, rangé en dur dans `vendor/chart.umd.min.js`
  (aucun chargement externe).
- **Hébergement : GitHub Pages** depuis `main`.

| Fichier | Rôle |
|---|---|
| `index.html` | toute l'app |
| `vendor/chart.umd.min.js` | Chart.js 4.4.4 figé |
| `manifest.json` + `icon.svg` | ajout à l'écran d'accueil |

### Modèle d'un document `depenses`

`date` (`"2026-08-01"`), `magasin`, `categorie`, `article`, `montant` (nombre),
`source` (`migration` / `ticket` / `manuel`), `supprime` (booléen), `creeLe`
(horodatage ISO).

### Points techniques repris de fav-deco

- Firestore type chaque valeur (`{stringValue}`, `{doubleValue}`...) :
  `versChamps()` / `depuisDoc()` font l'aller-retour.
- Pas de `WHERE` côté serveur : on charge toute la collection, on filtre et on
  agrège en JS.
- **Un `PATCH` sans `updateMask.fieldPaths` remplace tout le document.** Chaque
  modification liste ses champs explicitement (`modifier()`, `supprimerDoux()`).
- Garde-fou de chargement : une réponse vide ne remplace jamais un affichage
  déjà rempli.
- Pas de règle `delete` côté Firestore : la suppression passe par le booléen
  `supprime`.

---

## Sécurité, choix assumés (option A)

- La clé API web est en clair dans le dépôt public. Sans risque pour les
  comptes : ce n'est pas un secret, c'est la règle de sécurité Firestore qui
  protège la base.
- Règles Firestore : lecture / création / modification ouvertes, **suppression
  interdite** depuis l'extérieur. Qui a l'URL peut lire et écrire. Compromis
  accepté pour une app de courses à deux, comme fav-deco.
- Si un jour il faut fermer l'accès : Firebase Authentication (connexion
  Google) restreinte aux deux comptes, et restriction de la clé au domaine
  `stephjunak.github.io` dans la console Google Cloud.

### Règles Firestore en place

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /depenses/{docId} {
      allow read: if true;
      allow create: if true;
      allow update: if true;
      allow delete: if false;
    }
  }
}
```

---

## Mettre à jour Chart.js

Remplacer `vendor/chart.umd.min.js` par la version voulue, par exemple :

```
curl -sL -o vendor/chart.umd.min.js https://cdn.jsdelivr.net/npm/chart.js@4.4.4/dist/chart.umd.min.js
```

Puis vérifier que les deux graphiques de l'onglet Synthèse s'affichent encore.

---

## Redéployer sur un autre projet Firebase

1. [console.firebase.google.com](https://console.firebase.google.com) → nouveau
   projet → Firestore en mode production, région `europe-west`.
2. Onglet Règles → coller le bloc ci-dessus → Publier.
3. Paramètres du projet → « Vos applications » → app Web `</>` → copier
   `apiKey` et `projectId`.
4. Reporter les deux valeurs en haut du `<script>` de `index.html`
   (`FB_PROJECT`, `FB_API_KEY`).

---

## Migration initiale depuis le classeur Excel

Faite une fois, le 27/08/2026, par `../migrer-vers-firestore.py` (script hors
dépôt, il lit le `.xlsx` du dossier parent). 88 lignes importées, total août
342,67 € vérifié. Le classeur est conservé en archive dans le dossier parent,
Firestore est désormais la seule source de vérité.
