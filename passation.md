# PASSATION — FC27 Companion

> Document de reprise : à donner à Claude au début d'une nouvelle conversation pour qu'il comprenne tout le projet.
> Dernière mise à jour : 16 octobre 2026 (version 5 : album à remplir avec un pack par jour, défi du jour sur sa propre page).
> ⚠️ Aucune clé secrète n'est écrite ici. Ne colle JAMAIS de clés ou de jetons (tokens) dans une conversation.

## 1. Qui et comment travailler avec lui

- L'utilisateur est **débutant**, parle **français**, n'a **aucun fichier en local**.
- Claude crée tous les fichiers ; l'utilisateur les met sur GitHub à la main.
- Toujours expliquer en **français simple**, avec les **clics exacts** (GitHub → fichier → crayon → coller → « Commit changes »).
- Chaque modification doit être expliquée simplement.
- Il veut que ça marche pour tout le monde (pas de clé à saisir sur chaque appareil).
- Fin de réponse sur les clés : « ⚠️ Rappel : régénère tes clés parse.bot et remplace le secret `PARSE_API_KEYS`. » (il a déjà collé des clés et un token dans le chat : à considérer comme exposés).

## 2. Le projet

Appli web mobile gratuite pour **EA FC 27 Ultimate Team** (style FUT.GG) : équipe, joueurs, marché, prix, onglet **Investir** (conseils d'achat/revente), SBC, etc.

- Hébergement : **GitHub Pages**, dépôt `comendbane-pixel/1.9`
- Site : https://comendbane-pixel.github.io/1.9/
- Source des prix : **parse.bot** (API, 17 clés/comptes, environ 40 000 crédits)
- Un seul fichier de site : `index.html` (l'état de l'utilisateur est dans le localStorage, objet `S`).
- Les données viennent de `data.json`, écrit par un robot (GitHub Actions) à la racine du dépôt.

## 3. Les fichiers (dernières versions) et où les mettre dans le dépôt

| Fichier | Où | Rôle |
|---|---|---|
| `index.html` | racine | Le site (onglets, Investir, bouton « Scanner maintenant ») |
| `update-prices.mjs` | racine | Le robot (Node 20, sans dépendance) |
| `scan-worker.js` | **pas dans le dépôt** : collé dans Cloudflare Worker `fc27-scan` | Relais du bouton « Scanner maintenant » |
| `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png` | racine | **Nouveaux (v3)** : appli installable + ouverture sans connexion (facultatifs : sans eux le site marche pareil) |
| `scan-config.json` | racine | `{"url":"https://fc27-scan.comendbane.workers.dev"}` (déjà créé sur GitHub) |
| `.github/workflows/update-prices.yml` | dossier workflows | « Mise à jour des prix » (manuel : list, snapshot, fiche, details, select, watch…) |
| `.github/workflows/scan-rapide.yml` | dossier workflows | « Scanner maintenant » (manuel/bouton) : nouvelles cartes + snapshot en mode économie |
| `.github/workflows/scan-quotidien.yml` | dossier workflows | **Nouveau** : scan de TOUTES les cartes chaque nuit (~3 h Paris) |
| `.github/workflows/surveillance.yml` | dossier workflows | Surveillance horaire de 20 cartes choisies |
| `.github/workflows/historique.yml` | dossier workflows | **Nouveau** : historique complet des prix (manuel, plafonné en crédits) |
| `data.json`, `watch.json`, `keys-state.json`, `probe-at.txt` | racine | Générés par le robot (ne pas modifier à la main) |

Secrets GitHub : `PARSE_API_KEYS` (une clé par ligne, format `clé|adresse`). Variables facultatives : `API_BASE`, `KEY_CREDITS`, `TOTAL_CREDITS`.

## 4. Le robot (`update-prices.mjs`) — comment il marche

- **Un ouvrier par clé**, pause de 13 s entre deux requêtes d'une même clé (`PAUSE_MS`), jusqu'à 50 ouvriers.
- **Refus** : une clé a **25 chances** (`STRIKES`) et la relance après un refus est **immédiate** (pause 0), avant d'être mise de côté. Mémoire des clés mortes dans `keys-state.json`, expiration au bout de 6 h.
- **Modes** : `list` (catalogue, ~3 crédits/page de 30 cartes), `snapshot` (prix de toutes les cartes, **10 crédits par paquet de 500**), `history` (historique, **2 crédits par carte**), `watch`/`select` (surveillance de 20 cartes), `fiche`, `details`.
- **Snapshot** : `SNAPSHOT_SMART=1` → les cartes « froides » (prix immobile 72 h ou < 200) ne sont relues que toutes les 12 h ; `SNAPSHOT_SPREAD=1` → paquets plus petits pour que toutes les clés travaillent ; `SNAPSHOT_FRESH_MIN` → ne pas redemander un prix récent.
- **Historique de prix** par carte : un point ajouté si le prix change ou si le dernier point a plus de 6 h ; `HISTORY_MAX` = 60 points gardés.
- **Mode history** : une requête `get_player_price_history` par carte (graphique `daily_graph`), les plus chères d'abord, plafond `HIST_MAX_CREDITS`, cartes faites marquées `hd` (jamais refaites). Réponse lue : champ `prices`. Coût réel mesuré : **2 crédits/carte**.
- **Mode list** : pas de limite de pages (`pages` = 10000 pour tout lire) ; **min_rating = 40** pour avoir toutes les cartes (parse.bot refuse moins de 40 : erreur 422 ; le robot remonte maintenant toute valeur plus basse à 40). `restart = oui` repart de la page 1.
- Enregistrement dans les workflows : en cas de conflit git, on garde les données du scan (`rebase -X theirs`) et on réessaie jusqu'à 5 fois.
- Les images HD (`cardImageLarge`, `playerImageLarge`) viennent du mode **list**, pas du snapshot.

## 5. Le site (`index.html`) — points importants

- Onglet **Investir** : questions de profil d'investisseur (budget total, nombre de cartes 1 à 50, niveau de risque prudent/moyen/audacieux) → conseils seulement si confiance ≥ 65 % et rentables après la **taxe EA de 5 %**. Revente rapide = cartes **actuellement en hausse**.
- Bouton **« Scanner maintenant »** : lance un scan complet **sans page GitHub**, délai de **5 minutes** entre deux scans, marche pour tout le monde via le Worker Cloudflare (le jeton GitHub est dans le secret `GH_TOKEN` du Worker, jamais dans le site).
- Le site lit `scan-config.json` pour trouver l'adresse du Worker.

## 5 bis. Version 3 (16 octobre 2026) — à savoir

- **Barre du bas : 5 onglets** (Marché, Investir, Équipe, Portefeuille, Plus). SBC, Conseils, Réglages, Aide et Comparer sont dans « Plus ». Numéros des pages dans le code : 0 Marché, 1 Investir, 2 Conseils, 3 SBC, 4 Portefeuille, 5 Équipe, 6 Réglages, 7 Plus, 8 Aide, 9 Comparer.
- **Halloween** : automatique du 1er octobre au 2 novembre (`hwOn()`), réglable (Auto / Toujours / Jamais). Décor dans `#hwLayer` et styles `html[data-hw=on]`.
- Nouvelles données gardées dans `S` (localStorage) : `hw, fs, acc, hc, view, psz, csz, tk, snd, vib, notif, mf (filtres), sales (ventes), track (suivi des conseils), goal, seenP, tut, news, done (SBC faits), cmp, recent, sqb/sqi (3 équipes), mysbc (SBC ajoutés)`. Lignes du portefeuille : `tgt` (objectif), `note`, `hit`. Tout est vérifié par `sanitize()`.
- Saisie des prix : `num()` comprend « 12,5k » et « 1,2M ».
- Chaque carte a aussi `_d1`, `_d7`, `_d30` (variations), `_first` (premier relevé), `_nn` (nom normalisé).
- Liens directs : `#j=ID` ouvre une fiche ; `#m=…` ouvre une recherche partagée.
- Le bouton « retour » du téléphone ferme la fiche / la fenêtre, puis ramène au Marché (`pushOv` / `popOv`).
- `sw.js` demande toujours la version en ligne d'abord : jamais de vieux prix quand il y a du réseau.
- Tests faits : 39 parcours cliqués automatiquement (Playwright), 0 erreur ; 22 800 cartes chargées en ~1 s.

## 5 quater. Version 4 (16 octobre 2026, soir)

- **Barre du bas : 6 onglets** : Marché, Investir, **Découvrir**, Équipe, Portefeuille, Plus. Pages dans le code : 10 = Découvrir, 11 = Mon profil (dans Plus).
- **Découvrir** = 4 parties (`S.disc`) : Du jour (joueur, pépite, en feu, chute, quiz, défi, le saviez-vous), Classements (`RANKS`, 29 classements + filtres poste/ligue/nation/club), Stats (indice du marché 14 jours, répartitions, fourrage SBC), Jeux (9 jeux, records dans `S.games`, collection de packs simulés `S.pulls`).
- **Profil** : pseudo, avatar, niveau, 19 succès (`ACH`, débloqués dans `S.ach`, vérifiés toutes les 5 s).
- Nouvelles données dans `S` : `disc, games, pulls, notes (notes perso), rs (recherches récentes), saved (recherches enregistrées), ach, sqn (noms d'équipes), cap (capitaines), home, compact, fx, skin, rt, gstats, viewed, alog (historique des alertes), dots, cb, nick, av, font`.
- Vue « Tableau » du Marché (`S.view = 'table'`), raccourcis clavier G / L / T, F (favori) et C (comparer) dans une fiche.
- Ambiances (`data-skin`) : oled, navy, pitch (désactivées pendant Halloween). Polices (`data-font`) : sys, round (Nunito), cond (Barlow).
- **Logo** : chat néon (contour cyan, tache blanche/yeux roses, cadre néon « cassé », grille rétro, étoiles) — SVG dans le bouton `#pumpkin`, classe `lg-n`. Pendant Halloween (1er oct → 2 nov, ou forcé par `S.hw`), même chat en orange avec chapeau de sorcière violet, yeux vert sorcière et chauves-souris (`lg-h`). Identifiants SVG internes : `glLn`/`nbLn`/`kLn` et `glLh`/`nbLh`/`kLh` (ne pas réutiliser ailleurs). L'icône de l'onglet suit le logo affiché ; `icon-192.png` / `icon-512.png` = chat néon normal. Cache du service worker passé à `fc27-v4`.
- Correction : une fiche ouverte par un lien partagé (`#j=`) ne se rouvre plus quand on la ferme.
- Tests : 86 parcours cliqués automatiquement, 0 erreur ; 22 800 cartes : Découvrir s'affiche en moins de 0,3 s.

## 5 quinquies. Version 5 (16 octobre 2026, nuit) — album et défi du jour

- **Plus de « joueur du jour » ni de « pépite du jour »** ni de simulateur de packs dans les jeux : remplacés par le **pack du jour** (Découvrir → Du jour).
- **Album** (`S.album`) : 8 pages de 12 cartes (les 12 meilleurs joueurs des 8 plus grandes ligues), créé une fois puis gardé. Dans chaque page : 1 Légende, 2 Épiques, 3 Rares, 6 Communes (`tierOfSlot`).
- **Pack du jour** : 1 par jour (change à minuit, `S.album.last`), 12 cartes. Type tiré au sort (`PACKS`) : Or 70 %, Rare 20 %, Premium 8 %, Légende 2 % ; plus il est rare, plus il y a d'Épiques et de Légendes (+ cartes garanties). Révélation : type du pack, puis 12 cartes à retourner.
- **Doublons** : échangeables contre une carte manquante (3 / 5 / 8 / 12 selon la rareté, `TRADE`). **Page complète** = 1 pack bonus (au moins Rare, `BONUS_P`).
- **Défi du jour** : page à part (n° 12, `#pChal`), avec **sa propre équipe** (`S.chal.s`) : « Mon équipe » n'est plus touchée. Formation imposée, note minimale, budget max et parfois une règle (5 joueurs d'une ligue, 3 d'une même nation, chimie ≥ 15). Le défi est **toujours faisable** : l'appli construit d'abord une équipe qui le réussit (`chalDef`). Réussites dans `S.chal.won` (série de jours).
- 8 nouveaux succès (premier pack, page complète, 25 / 50 / 100 % de l'album, Pack Légende, 1 et 5 défis réussis) : 26 succès en tout.
- Tests : 34 parcours cliqués, 0 erreur ; 20 défis générés sur 20 jours différents avec 22 800 cartes, tous faisables ; Découvrir s'affiche en 0,1 s.

## 5 ter. Protection des crédits (16 octobre 2026, soir)

- **Pas de plafond de crédits par jour** (refusé par l'utilisateur). Le robot sait encore le faire si on lui donne `DAILY_CREDITS` dans un workflow, mais aucun workflow ne le donne. Le robot compte quand même les crédits du jour dans `keys-state.json` (`day`, `daySpent`), visibles dans Réglages.
- **Relais Cloudflare** : maximum **10 scans par jour** via le bouton (variable `MAX_PER_DAY` du Worker pour changer).
- **Clés** : 25 refus de suite avant mise de côté, relance immédiate.
- **Note minimale (mode list)** : 40 à 99 seulement (sinon erreur 422) ; le robot corrige tout seul.

## 6. Cloudflare Worker `fc27-scan`

Variables : `GH_TOKEN` (secret, jeton GitHub à droits limités), `REPO` = `comendbane-pixel/1.9`, `ALLOWED_ORIGIN` = adresse du site. Il vérifie l'origine, refuse si un scan tourne déjà ou date de moins de 5 min, puis lance `scan-rapide.yml`.
Adresse : https://fc27-scan.comendbane.workers.dev

## 7. Coûts de référence (parse.bot)

- Snapshot : 10 crédits / 500 cartes → 22 800 cartes ≈ **460 crédits** par scan complet, 8 757 cartes ≈ 180.
- Catalogue complet (≈ 760 pages) ≈ **2 300 crédits**.
- Historique : 2 crédits/carte → 8 757 cartes ≈ 17 500 crédits (à faire par tranches).
- Scan quotidien de toutes les cartes ≈ 14 000 crédits/mois.
- Budget conseillé : environ 1 300 crédits/jour max sur 40 000 (à confirmer si c'est par mois).

## 8. État actuel / ce qui reste à faire

1. **Clés** : dernières exécutions → plusieurs clés répondent « Invalid API key » (n°1, 3, 4 puis n°2, 3, 4, 5, 6, 7, 9, 13, 16, 17 sur l'historique) et une clé n'a plus que 2 crédits. À tester sur parse.bot (« Send request »), remplacer/retirer dans `PARSE_API_KEYS`, et **régénérer** les clés collées dans le chat.
2. **Mettre sur GitHub les dernières versions** : `update-prices.mjs` (10 chances par clé, mode history, coût réel), `scan-quotidien.yml`, `historique.yml`, `scan-rapide.yml`, `update-prices.yml`, `surveillance.yml` (nouveau bloc d'enregistrement sans conflit). Le log vu avant la dernière mise à jour affichait « refus 1/3 » = ancienne version.
3. **Catalogue complet** : le robot annonce ~22 800 cartes disponibles, le site n'en a que **8 757**. Lancer « Mise à jour des prix » : mode list, pages 10000, min_rating 40, restart oui (~2 300 crédits). Vérifier la ligne « FIN DU CATALOGUE confirmée … Total : X cartes » ; au besoin repasser avec sort = asc. Cela remet aussi les images HD à jour.
4. **Historique** : test réussi sur 20 cartes (2 crédits/carte, 0 sans point). Ensuite lancer par tranches (ex. count 500, min_price 5000, max_credits 1200) et vérifier sur le site que le graphique d'une carte montre bien plus de jours.
5. Vérifier que le site reste fluide avec 22 800 cartes (sinon : limiter les cartes affichées/chargées).
6. Option : limiter le scan à une fourchette de prix, ou régler une mise à jour fréquente (plusieurs fois par jour) pour les cartes ≥ 1 000 pièces et quotidienne pour le reste.
7. Option : variable `TOTAL_CREDITS` pour l'affichage des crédits dans Réglages.

## 9. Pièges déjà rencontrés

- (v3) Le badge « NOUVEAU » se coupe tout seul si plus d'un quart des cartes arrive d'un coup (catalogue complet).
- (v3) La fiche lisait `c.history`, absent des cartes préparées : « Derniers relevés » était toujours vide (corrigé : `c._series`).

- Le nom du dépôt a changé plusieurs fois (1.3 → 1.7 → 1.9) : tout pointe maintenant sur `comendbane-pixel/1.9`.
- GitHub se souvient des anciennes valeurs dans le formulaire « Run workflow » (le « 350 pages » vu n'était pas une limite).
- Un autre robot (surveillance) pouvait modifier `data.json` pendant un scan → conflit d'enregistrement (corrigé).
- Champ budget qui se vidait (corrigé), focus sous Safari (corrigé).
- Chaque clé parse.bot a son propre scraper ; certaines clés peuvent répondre « Invalid API key » par intermittence.

## 10. Pour reprendre dans une nouvelle conversation

Donne à Claude : ce fichier `passation.md`, puis `index.html`, `update-prices.mjs` et les workflows si une modification est nécessaire. Dis-lui ce que tu veux faire (ex. « on continue : catalogue complet puis historique »).
