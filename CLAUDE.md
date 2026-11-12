# Plan — contexte pour Claude

App perso de suivi de patrimoine, d'objectifs et de conseils. Interface 100 % en français.
En ligne : https://onihxh.github.io/plan/ · Dépôt : https://github.com/OniHxh/plan (public, branche `main`, GitHub Pages à la racine).

## Règles de travail (à respecter à chaque session)

- **Ne jamais effacer les données de l'utilisateur.** Elles vivent dans le `localStorage` de son téléphone et de son Mac, pas dans le dépôt. Toute évolution du format doit rester compatible : `load()` et l'import fusionnent avec `blank()`, il faut garder ça.
- **Avant chaque publication (`git push`)** : prévenir l'utilisateur pour qu'il fasse un export depuis Réglages, attendre son « Export fait, publie », puis pousser.
- Après le push : attendre ~50 s, vérifier le site en ligne avec `curl` (grep d'une fonction nouvelle + version du cache dans `sw.js`).
- À chaque mise à jour publiée, incrémenter `CACHE='plan-vN'` dans `sw.js`.
- Tester dans le navigateur intégré avant de publier : grand écran (1440×900) **et** téléphone (375×812), console sans erreur, avec des données factices (jamais les vraies).
- Le dépôt est **public** : ne jamais y mettre de montants réels, de clé GitHub, d'e-mail ou d'info perso.
- L'utilisateur n'est pas développeur : expliquer simplement, en étapes courtes. Il aime les graphiques, la 3D, les animations et une expérience soignée, style sombre premium.

## Ce que l'utilisateur veut (et ne veut pas)

- Suivi du **patrimoine**, des **objectifs** et des **conseils** (« je peux me permettre ? », quand freiner). **Pas** de saisie des dépenses au quotidien.
- Comptes de dépenses : Crédit Mutuel et Revolut. Salaire fixe reçu le 1er du mois, revenus ponctuels possibles.
- Mise à jour **1 fois par mois** (« point du mois »), idéalement juste après la paie.
- Conseils par **règles claires**, sans IA. Plusieurs objectifs, feu vert / orange / rouge selon les objectifs.
- L'app doit **demander chaque info** (assistant de départ, puis « À compléter »).

## Architecture

- Une seule page : `index.html` (HTML + CSS + JS vanilla, aucune dépendance). Plus `manifest.webmanifest`, `sw.js` (network-first pour la navigation) et les icônes (`icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, logo 3 barres isométriques).
- Données : `localStorage['plan-v1']`, objet `S`. `save()` stocke, met à jour `S.updatedAt` et déclenche la synchro.
- Modèle :
  - `S.snaps[YYYY-MM][accId]` = soldes mensuels, avec report du dernier solde connu (`balAt`).
  - `S.accounts` (types `courant`, `epargne`, `invest`, `bien`, `dette`), `S.charges` (`kind` `fixe`/`abo`, `freq` 1/3/12, `day`), `S.budgets`, `S.goals` (ordre = priorité), `S.extras`, `S.wishes`, `S.stats`, `S.settings`.
- Calculs clés :
  - `deltas()` : variation entre mois (seulement les comptes présents dans les deux mois).
  - `observedRate()` : moyenne des 3 derniers mois. `rate()` = observé, sinon prévu.
  - `plan()` : financement des objectifs en cascade par priorité, avec `eta`, `late` et `req`.
  - `status()` : feu (`lvl`), `free = cap - req`. `afford()` : verdict « je peux me permettre ».
- Rendu : `render()` choisit la vue selon `ui.tab`. Actions par délégation `data-a` → objet `A`. Champs liés par `data-k` → `setK()`. Fenêtres via `openSheet()`.
- Vues :
  - Mobile : `vHome`, `vPatri`, `vGoals`, `vWish`, `vBudget`.
  - Grand écran (≥ 1024 px, `isDesk()`) : menu latéral, `vDesk` (tableau de bord), `vAna` (analyses + projection), `vHist` (historique éditable + export CSV), mise en 2 colonnes des autres vues (`deskCols`), raccourcis clavier (`?` pour l'aide).
- Visuels : donut 3D, bocaux 3D, jauge, barres isométriques, courbe avec curseur (`chart(W,H)` + `scrub`), aire empilée, bilan de l'année façon « story », badges, confettis.
- Autres fonctions : rappel calendrier (.ics mensuel), mode discret (masque les montants), liste d'envies anti-achat impulsif (délai de réflexion), simulateur « Et si… ».

## Synchronisation entre appareils

- Chiffrée de bout en bout : PBKDF2 (250 000 itérations) + AES-GCM 256 via WebCrypto.
- Stockée dans un **Gist privé** (`plan-sync.json`) via l'API GitHub, avec une clé `ghp_…` limitée à la case `gist`, que l'utilisateur crée lui-même.
- Clé et phrase secrète dans `localStorage['plan-sync']`, jamais exportées.
- La dernière écriture gagne (`updatedAt`). En cas de modif sur les deux appareils, l'app demande quelle version garder.
- Garde-fou : un appareil vide ne peut jamais écraser la copie en ligne. « Tout effacer » désactive la synchro.
- Claude ne doit jamais saisir ni créer la clé GitHub à la place de l'utilisateur.

## Historique

1. Création de l'app, assistant en 8 étapes, feux, objectifs, budget.
2. Renommage en « Plan », nouveau logo, graphiques et 3D.
3. Rappel auto, graphiques des dépenses, simulateur, mode discret, envies, bilan de l'année, badges.
4. Import possible dès l'écran d'accueil.
5. Synchronisation chiffrée téléphone ↔ Mac.
6. Version grand écran pour Mac : tableau de bord, Analyses, Historique, raccourcis clavier.
