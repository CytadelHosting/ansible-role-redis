# Synchronisation Upstream - Fork Redis

## Pourquoi ce fork existe

Ce fork du role Redis existe pour maintenir un standard Cytadel homogene (securite, tuning systeme, conventions de release) tout en continuant a recuperer les evolutions upstream.

Concretement, ce fork permet de:

- moderniser la configuration Redis (`ACL`, `protected-mode`, `maxmemory`, `AOF`),
- gerer les prerequis systeme (`sysctl`, `THP`, `systemd override`) de maniere idempotente,
- conserver une base compatible avec les conventions de la collection interne.

Le fork permet de garder un role stable en prod tout en conservant de la latitude pour des evolutions futures.

## Perimetre du fork

- Upstream source: `https://github.com/geerlingguy/ansible-role-redis`
- Fork source: `https://github.com/CytadelHosting/ansible-role-redis`
- Collection consommatrice: `cytadel.webserver`

Patchs principaux a conserver:

- template Redis modernise et parametrique,
- tuning systeme Redis (`sysctl`, `THP`, limites systemd),
- convention de branche/release Cytadel (`cytadel-release`, tags `vX.Y.Z-cytadel.k`).

## Strategie de synchronisation

Objectif: recuperer les evolutions upstream sans perdre les patches Cytadel.

### 1) Recuperer upstream dans le fork

```bash
git clone git@github.com:CytadelHosting/ansible-role-redis.git
cd ansible-role-redis

git remote add upstream https://github.com/geerlingguy/ansible-role-redis.git
git fetch --all --tags
```

### 2) Determiner la branche upstream de synchro

```bash
git fetch upstream --prune
SYNC_BRANCH=$(git remote show upstream | rg "HEAD branch" | awk '{print $NF}')
echo "$SYNC_BRANCH"
```

Alternative robuste (si besoin):

```bash
SYNC_BRANCH=$(git symbolic-ref refs/remotes/upstream/HEAD | sed 's@^refs/remotes/upstream/@@')
echo "$SYNC_BRANCH"
```

### 3) Synchroniser la branche miroir read-only

La branche `main`/`master` du fork est un miroir read-only de la branche par defaut upstream:

```bash
git checkout "$SYNC_BRANCH"
git pull --ff-only origin "$SYNC_BRANCH"
git reset --hard "upstream/$SYNC_BRANCH"
git push origin "$SYNC_BRANCH" --force-with-lease
```

### 4) Rejouer upstream sur `cytadel-release`

```bash
git checkout cytadel-release
git fetch upstream --tags
git rebase "upstream/$SYNC_BRANCH"
```

### 5) Reappliquer/verifier les patches Cytadel

- verifier le template `templates/redis.conf.j2`,
- verifier les tasks `tasks/main.yml` (sysctl, THP, systemd override),
- verifier les defaults `defaults/main.yml`,
- s'assurer que le role reste compatible avec la collection `cytadel.webserver`.

### 6) Tests avant tag

Tests minimaux recommandes:

1. `ansible-lint` sur le role
2. run Ansible en `--check` sur un host de test
3. run reel sur un host de test
4. validation Redis (`redis-server --test-memory 2` et/ou demarrage service OK)
5. verification service (`systemctl status redis*` selon distribution)

## Identifier la version upstream de depart

Cette procedure permet de savoir de quel etat upstream votre fork est parti.

```bash
git fetch upstream --tags --prune
base_commit=$(git merge-base HEAD "upstream/$SYNC_BRANCH")

echo "$base_commit"
git describe --tags --abbrev=0 "$base_commit"
git rev-list --count "$(git describe --tags --abbrev=0 "$base_commit")..$base_commit"
```

Interpretation:

- `git describe --tags --abbrev=0` retourne la base upstream `X.Y.Z`.
- le compteur `rev-list --count` donne le nombre de commits apres ce tag.

## Convention de tagging du fork

Convention recommandee:

- `vX.Y.Z-cytadel.k`

Exemples:

- `v1.9.1-cytadel.1`
- `v1.9.1-cytadel.2`
- `v1.9.2-cytadel.1` (nouvelle base upstream, compteur local reset)

Creation/push:

```bash
git checkout cytadel-release
git pull --rebase origin cytadel-release
git tag -a vX.Y.Z-cytadel.k -m "Cytadel Redis role release vX.Y.Z-cytadel.k"
git push origin cytadel-release
git push origin vX.Y.Z-cytadel.k
```

## Integration dans la collection `cytadel.webserver`

Mettre a jour selon le mode choisi:

- mode submodule dynamique (`roles/...`),
- mode fige (`roles.lock.yaml`).

Exemple lockfile:

```yaml
roles:
  - role_name: "geerlingguy_redis"
    src: "https://github.com/CytadelHosting/ansible-role-redis.git"
    version: "vX.Y.Z-cytadel.k"
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
