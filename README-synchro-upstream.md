# Synchronisation upstream et releases Cytadel

Ce document définit le workflow Git standard pour ce fork du role Redis.

## Principes

- La branche `main` (ou `master` selon le projet source) est **read-only**.
- Cette branche sert uniquement a synchroniser l'etat du projet source (`upstream`).
- La branche de travail et de release Cytadel est `cytadel-release`.
- Aucune Merge Request n'est necessaire vers le projet source.

## Configuration initiale des remotes

Verifier les remotes:

```bash
git remote -v
```

Si necessaire:

```bash
git remote add upstream <UPSTREAM_GIT_URL>
git remote set-url origin <YOUR_FORK_GIT_URL>
```

- `upstream` = depot source du role
- `origin` = votre fork Cytadel

## Synchroniser la branche source read-only

Utiliser `main` **ou** `master` selon la branche par defaut du projet source.
Exemple avec `main`:

```bash
git checkout main
git fetch upstream --tags
git reset --hard upstream/main
git push origin main --force-with-lease
```

Exemple equivalent si la branche source est `master`:

```bash
git checkout master
git fetch upstream --tags
git reset --hard upstream/master
git push origin master --force-with-lease
```

Notes:

- Ce reset est volontaire ici car la branche est read-only et miroir de `upstream`.
- `--force-with-lease` est utilise uniquement pour realigner la branche miroir du fork.

## Maintenir la branche Cytadel

Initialisation (une seule fois):

```bash
# Si upstream suit main
git checkout -B cytadel-release upstream/main
git push -u origin cytadel-release
```

ou:

```bash
# Si upstream suit master
git checkout -B cytadel-release upstream/master
git push -u origin cytadel-release
```

Workflow courant:

```bash
git checkout cytadel-release
git fetch upstream --tags

# Rebase de la branche Cytadel sur la derniere source
# (choisir upstream/main ou upstream/master)
git rebase upstream/main

# Developpement local
git add .
git commit -m "Redis: <message clair>"

# Publication sur le fork
git push origin cytadel-release
```

## Convention de tagging Cytadel

Format impose:

- `vX.Y.Z-cytadel.k`

Ou:

- `X.Y.Z` = version upstream de reference (dernier tag source synchronise)
- `k` = numero d'ordre des releases Cytadel basees sur ce meme `X.Y.Z`

Exemples:

- `v7.2.5-cytadel.1` : premiere release Cytadel basee sur upstream `v7.2.5`
- `v7.2.5-cytadel.2` : deuxieme release Cytadel sur la meme base upstream
- `v7.2.6-cytadel.1` : nouvelle base upstream, compteur `k` reinitialise a `1`

## Procedure de release et tag

Toujours tagger depuis `cytadel-release`:

```bash
git checkout cytadel-release
git pull --rebase origin cytadel-release
git tag -a vX.Y.Z-cytadel.k -m "Cytadel Redis role release vX.Y.Z-cytadel.k"
git push origin cytadel-release
git push origin vX.Y.Z-cytadel.k
```

Optionnel (si plusieurs tags locaux a pousser):

```bash
git push origin --tags
```

## Regles de gouvernance conseillees

- Ne jamais developper directement sur `main`/`master`.
- Tout commit fonctionnel va sur `cytadel-release`.
- Un tag de release pointe toujours un commit de `cytadel-release`.
- Documenter dans le message de tag les changements Redis importants:
  - securite (`ACL`, `protected-mode`, commandes desactivees)
  - tuning systeme (`sysctl`, `THP`, `systemd`)
  - changements de comportements par defaut
# Synchronisation Upstream - Fork Nginx

## Pourquoi ce fork existe

Ce fork du role nginx a ete cree pour corriger un probleme de validation Ansible apparu avec les includes relatifs (`include fastcgi_params;`) dans les vhosts.

Concretement:

- le role upstream valide `nginx.conf` via `validate: nginx -t -c %s`,
- Ansible ecrit `%s` dans un dossier temporaire (`/tmp/...`),
- Nginx tente alors de resoudre les includes relatifs depuis ce dossier temporaire,
- ce qui provoque des erreurs du type `open() ".../fastcgi_params" failed (2: No such file or directory)`.

Le fork permet de garder un role stable en prod tout en conservant de la latitude pour des evolutions futures.

## Perimetre du fork

- Upstream source: `https://github.com/geerlingguy/ansible-role-redis`
- Fork source: `https://github.com/CytadelHosting/ansible-role-redis`
- Collection consommatrice: `cytadel.webserver`

Patch principal a conserver:

- validation Nginx robuste avec prefix explicite (`-p /etc/nginx/`) **ou**
- toute approche equivalente garantissant la resolution des includes pendant `validate`.

## Strategie de synchronisation

Objectif: recuperer les evolutions upstream sans perdre les patches Cytadel.

### 1) Recuperer upstream dans le fork

```bash
git clone git@github.com:CytadelHosting/ansible-role-redis.git
cd ansible-role-nginx

git remote add upstream https://github.com/geerlingguy/ansible-role-redis.git
git fetch --all --tags
```

### 2) Creer une branche de sync

```bash
git checkout main
git pull --ff-only origin main
git checkout -b chore/sync-upstream-<UPSTREAM_TAG>
```

Exemple: `chore/sync-upstream-3.3.1`.

### 3) Rejouer upstream

Choisir merge ou rebase selon ta politique:

```bash
# Option merge (lisible pour l'historique d'integration)
git merge --no-ff upstream/main

# Option alternative (lineaire)
# git rebase upstream/main
```

Si upstream tague une release precise:

```bash
git merge --no-ff <UPSTREAM_TAG>
```

### 4) Reappliquer/verifier les patches Cytadel

- verifier la zone de `tasks/main.yml` liee a `template.validate`,
- verifier les handlers/templating si upstream a bouge,
- s'assurer que le role reste compatible avec les vhosts de `apsys`.

### 5) Tests avant tag

Tests minimaux recommandes:

1. `ansible-lint` sur le role
2. run Ansible en `--check` sur un host de test
3. run reel sur un host de test
4. `nginx -t` OK
5. recharge nginx OK

## Convention de tagging du fork

Convention recommandee:

- `v<UPSTREAM>-cytadel.<N>`

Exemples:

- `v1.9.1-cytadel.1`
- `v1.9.1-cytadel.2`
- `v1.9.2-cytadel.1` (nouvelle base upstream, compteur local reset)

Creation/push:

```bash
git tag -a v1.9.1-cytadel.1 -m "Fork release based on upstream 1.9.1 (Cytadel patchset #1)"
git push origin v1.9.1-cytadel.1
```

## Integration dans la collection `cytadel.webserver`

Mettre a jour selon le mode choisi:

- mode submodule dynamique (`roles/...`),
- mode fige (`roles.lock.yaml`).

Exemple lockfile:

```yaml
roles:
  - role_name: "geerlingguy_nginx"
    src: "https://github.com/CytadelHosting/ansible-role-nginx.git"
    version: "v1.9.1-cytadel.1"
```

Puis:

```bash
make -f scripts/Makefile validate-lock
make -f scripts/Makefile build
```

## Checklist rapide de release

- [ ] sync upstream terminee
- [ ] patch Cytadel present et teste
- [ ] tag fork pousse
- [ ] `roles.lock.yaml` mis a jour
- [ ] build collection OK
- [ ] test AWX/CLI OK

