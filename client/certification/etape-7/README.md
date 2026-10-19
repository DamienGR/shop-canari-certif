# Étape 7 — Isolation de deux boutiques déployées

Passée dans la nuit du 19 au 20 octobre 2026 sur le compte A, selon la fiche
`docs/certification/etape-7-commandes.md` du socle. `canari-certif`, montée de
`v0.3.2` en `v0.3.10` (pull request #8, CI 20/20, fusion sans conflit, trois
migrations 0043 à 0045), provisionnée et déployée en staging sur workers.dev, à
côté de `pilote-certif` en `v0.3.9` derrière Access : deux versions du socle
côte à côte sur un compte. Un jeton de service Access « pilote-certif étape 7 »
créé pour l'étape, révoqué à la fin. La canari a été supprimée au terme de
l'étape.

Le sous-domaine workers.dev du compte A est remplacé par `<sous-domaine>`
dans les rapports ; le dépôt est public.

## Les lignes de l'étape

| Ligne | Preuve | Résultat |
|---|---|---|
| deux Workers sur deux origines, deux bases, six buckets | `origines.txt`, `bases.txt`, `buckets.txt`, `kv.txt`, `liaisons-*.json` | la canari en 200 sous « Canari Certif », le pilote en 200 avec le jeton et 302 sans ; deux bases (`22a5eef5…` canari, `42136d04…` pilote), six buckets, deux namespaces de sessions ; chaque Worker lié à sa seule base, à ses buckets et à son namespace |
| un produit créé dans l'une n'apparaît pas dans l'autre | `produits-croises.txt`, `canari-produits.json`, `pilote-produits.json` | chaque croix en 200 chez elle, 404 chez l'autre, dans les deux sens ; un produit dans la canari, vingt et un dans le pilote ; chaque base ne connaît que sa croix |
| `catalog import` de l'une vers l'autre : refus | `import-croise.json`, `canari-apres-import.json` | code 4, « il décrit une autre boutique que ce manifeste », avant toute lecture du compte ; toujours un produit dans la canari |
| `destroy` : tout part, rien du pilote ; le refus de R2 constaté | `destroy-simulation.json`, `destroy-1.json`, `destroy-2.json`, `*-apres.*` | six ressources simulées ; premier passage en code 6 (Worker, namespace et bucket de fichiers partis, buckets d'images et de sauvegardes refusés par R2, base gardée) ; second passage en 0 après les avoir vidés ; il reste une base, trois buckets et un namespace, tous du pilote, sous les mêmes identifiants, et ses liaisons sont inchangées ; le pilote en 200 avec ses vingt et un produits, la canari sans Worker (ci-dessous) |

## Le provisioning et le déploiement

`provision-canari.json` : code 0, quatre créations (la base et trois buckets),
juridiction `eu` relue sur les quatre, `-files` et `-backups` privés, la règle
de 90 jours, les migrations appliquées. `AUTH_SECRET` et `SECRETS_KEK` restent
en attente tant que le Worker n'existe pas : le premier déploiement les pose,
par entrée standard. La canari ne déclare pas de compte, exprès :
`CLOUDFLARE_ACCOUNT_ID`, pris dans le manifeste du pilote, l'a posée sur A, et
`provision` l'a dit par un avertissement.

**Le premier déploiement a fini en code 6**, `deploy-canari-1-propagation.json`,
pour la raison de propagation que la fiche prévoit : le test de fumée a couru
avant que workers.dev ne serve le nouveau nom partout. `/` et `/products` en
404 (le second message accusant la liaison D1), `/admin` en 404 compté comme
fermé, `/robots.txt` et `/admin/setup` en 200. Une minute plus tard, chaque
page renvoyait vers `/admin/setup`.

`deploy-canari-1.json`, la relance avec `--skip-verify --skip-build` avant
l'assistant, est la preuve du correctif 1 de `v0.3.10` : code 0, et quatre
avertissements, `/`, `/products` et `/robots.txt` « renvoie vers /admin/setup :
la boutique n'est pas encore installée », plus l'assistant ouvert.
`deploy-canari-2.json`, après `/admin/setup` : code 0, les cinq contrôles
verts, l'assistant refermé (303).

## Les limiteurs

`limiteurs.txt` : 600 recherches en rafale de 50 sur la canari, 560 en 200 et
40 en 429 ; le pilote en 200 juste après, puis la canari elle-même en 200. Les
deux Workers déclarent les mêmes `namespace_id` (10001 à 10003, voir
`liaisons-*.json`).

La mesure ne tranche pas seule : la canari était déjà rouverte quand on l'a
relue, juste après le pilote, et rien ne dit qu'elle était encore fermée au
moment où le pilote a répondu. Ce qui sépare les compteurs est la clé, qui
porte le nom d'hôte de la boutique (`host:chemin:adresse`, voir
`src/features/auth/rateLimit.ts` du socle) ; la mesure est compatible avec elle,
sans la prouver.

## La suppression

Avant `destroy`, `backup.json` : une archive chiffrée de la canari copiée dans
son bucket de sauvegardes (trois objets), pour constater le refus que le plan
attend. Le premier passage a imprimé la commande pour vider chaque bucket
avec `--remote -J eu` (correctif 3 de `v0.3.10`) : une image (la croix) et
les trois objets de l'archive, supprimés un par un, puis le second passage.
L'archive locale est partie avec la canari.

Le namespace KV des sessions de la canari, créé par wrangler au premier
déploiement, figure dans la simulation et part au premier passage (correctif 2
de `v0.3.10`).

`origines-apres.txt` : la première lecture de la canari, depuis un second
terminal, n'a pas obtenu de réponse HTTP (`000`, connexion non établie) ;
relue aussitôt en IPv4, elle répond 404, `error code: 1042` : Cloudflare n'a
plus de Worker sous ce nom. Quelques minutes plus tôt, depuis le terminal
qui a lancé `destroy`, elle répondait déjà 404.

La croix du pilote a été supprimée dans son administration après ce relevé :
`/api/products/croix-test-pilote` répond 404, le pilote retrouve ses vingt
produits.

## Ce qui a accroché

- **Le premier déploiement sur workers.dev sort en code 6 tant que le nom n'a
  pas propagé**, avec un message qui accuse la liaison D1 et propose un
  `wrangler rollback`. La fiche le prévoyait ; `deploy` ne le distingue pas
  d'une vitrine cassée.
- **L'import croisé, lancé depuis le pilote en `v0.3.9`, conseille encore
  `prepare-client.mjs`** : c'est le conseil que `v0.3.10` corrige. Le refus,
  lui, est le même.
