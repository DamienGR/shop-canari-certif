# Étape 2 — Forge réelle (canari-certif)

Plan de certification du socle, étape 2, passée le 2026-10-11 sur le socle
`v0.3.1`. Pas de compte déclaré dans le manifeste : `CLOUDFLARE_ACCOUNT_ID` du
compte A exporté pour le seul `shopctl repo`. Pas de production.

| Preuve | Ce qu'elle montre |
|---|---|
| `repo-1.json` | `shopctl repo --public --reviewer DamienGR` : dépôt public créé, `main` poussée, `staging` et `production` créés, relecteur obligatoire sur `production`, secrets de `staging` posés |
| `repo-2.json` | la même commande relancée : rien de créé, rien de poussé, secrets réécrits |

Relu sur GitHub après les deux passages :

- `DamienGR/shop-canari-certif` public, `main` au même commit que l'arbre local ;
- `staging` : `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN`, `SHOPCTL_BACKUP_KEY` ;
- `production` : aucun secret, relecteur obligatoire `DamienGR`.

`origin` a été posé en SSH avant le premier passage, pour contourner le défaut
de protocole trouvé sur `pilote-certif` (voir son `client/certification/`).
