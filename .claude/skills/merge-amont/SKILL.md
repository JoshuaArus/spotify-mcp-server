---
name: merge-amont
description: Veille hebdo de l'amont marcelmarais/spotify-mcp-server, qui ouvre ou met à jour la PR de merge sur le fork et rend un verdict au contrat de la veille d'Iris.
disable-model-invocation: true
---

# merge-amont

Un passage de la veille de l'amont, lancé par `/veille <machine> /spotify-mcp-server /merge-amont`
depuis Péon `/Iris`. Il porte l'amont `marcelmarais/spotify-mcp-server` vers le fork
`JoshuaArus/spotify-mcp-server` par une **PR de merge**, que l'owner merge lui-même. Le
passage livre la PR et son verdict. Le merge sur `main`, la notification de l'owner et la
Scheduled Task restent à l'owner et à Iris.

- **La branche** de la PR s'appelle `merge-amont`. Elle appartient à la veille : chaque
  passage la reconstruit depuis `origin/main` et la pousse en `--force-with-lease`.
- **Le worktree jetable** : le working tree du dépôt peut être partagé par d'autres
  sessions. Tout le travail git se fait dans `/tmp/merge-amont`, créé à l'étape 2 et
  retiré à l'étape 5. Le working tree partagé ne reçoit que des `git fetch`.
- **`gh` vise toujours le fork** par `-R JoshuaArus/spotify-mcp-server` : dans un fork,
  `gh` vise par défaut le dépôt parent.
- **L'état** d'un passage à l'autre est la PR elle-même : aucun fichier d'état.

## Étape 1 : comparer

1. Lis la branche par défaut de l'amont :
   `gh api repos/marcelmarais/spotify-mcp-server --jq '.default_branch + " " + (.archived|tostring)'`.
   Un 404 ou `archived: true` : l'amont ne bougera plus, le verdict est `rien` avec
   `retrait: oui` (étape 5).
2. Assure le remote `upstream` (`git remote add upstream https://github.com/marcelmarais/spotify-mcp-server.git`
   s'il manque), puis `git fetch origin main` et `git fetch upstream <branche>`.
3. Compte les commits amont absents du fork :
   `git rev-list --count origin/main..upstream/<branche>`.
4. Lis la PR de la veille :
   `gh pr list -R JoshuaArus/spotify-mcp-server --head merge-amont --state all --json number,title,url,state,headRefOid,closedAt`.

- **Zéro commit** : une PR `merge-amont` encore ouverte n'a plus d'objet, ferme-la avec
  `gh pr close <n> -R JoshuaArus/spotify-mcp-server --delete-branch --comment "L'amont est déjà dans main."`.
  Le verdict est `rien` ; passe à l'étape 5.
- **Refusée** : la dernière PR est `CLOSED` (pas `MERGED`) et son `headRefOid` contient
  déjà le sommet amont (`git merge-base --is-ancestor upstream/<branche> <headRefOid>`,
  après `git fetch origin <headRefOid>`) : l'owner a refusé ce merge. Le verdict est
  `rien` jusqu'à ce que l'amont bouge ; passe à l'étape 5.
- **Sinon**, passe à l'étape 2.

L'étape est finie quand le nombre de commits et l'état de la PR sont connus.

## Étape 2 : merger dans le worktree

1. `git worktree add -B merge-amont /tmp/merge-amont origin/main` (un worktree resté
   d'un passage précédent s'enlève d'abord par `git worktree remove --force /tmp/merge-amont`).
2. Dans `/tmp/merge-amont` :
   `git merge --no-ff upstream/<branche> -m "Merge upstream marcelmarais/spotify-mcp-server <branche>"`.
3. **Conflit** : relève les fichiers en conflit (`git diff --name-only --diff-filter=U`),
   puis `git merge --abort`. Rien n'est poussé, la PR ouverte reste telle quelle. Le
   verdict est `alerte` (étape 5), avec ces fichiers et le lien
   `https://github.com/JoshuaArus/spotify-mcp-server/compare/main...marcelmarais:spotify-mcp-server:<branche>`.

L'étape est finie quand le merge est commité dans le worktree, ou le conflit relevé.

## Étape 3 : ouvrir ou mettre à jour la PR

1. Résume les commits amont : `git log --no-merges --format='- %h %s' origin/main..upstream/<branche>`.
2. `git -C /tmp/merge-amont push --force-with-lease origin merge-amont`.
3. Le corps de la PR est ce résumé, sous la phrase « Commits amont à merger, au <date> : ».
   - **PR ouverte** : `gh pr edit <n> -R JoshuaArus/spotify-mcp-server --body-file <fichier>`.
   - **Sinon** : `gh pr create -R JoshuaArus/spotify-mcp-server --base main --head merge-amont --title "Merge amont marcelmarais/spotify-mcp-server" --body-file <fichier>`.

L'étape est finie quand la PR ouverte porte le merge poussé, et que tu as son numéro et
son URL.

## Étape 4 : attendre la CI

La CI (`.github/workflows/ci.yml`) tourne sur l'événement `pull_request`. Lance son
attente en arrière-plan (Bash `run_in_background`), pour le commit poussé :

```bash
sha=$(git -C /tmp/merge-amont rev-parse HEAD)
until id=$(gh run list -R JoshuaArus/spotify-mcp-server --commit "$sha" --event pull_request --json databaseId --jq '.[0].databaseId') && [ -n "$id" ]; do sleep 20; done
gh run watch "$id" -R JoshuaArus/spotify-mcp-server --exit-status > /tmp/merge-amont-ci.txt 2>&1
echo "EXIT=$? RUN=$id" >> /tmp/merge-amont-ci.txt
```

Pendant l'attente, termine ton tour par une seule ligne, `en cours · CI de la PR de merge amont`,
déclaré par `quiet_turn` s'il existe. La fin de la commande te réveille.

- **`EXIT=0`** : le verdict est `nouveau`.
- **Sinon** : relève le job et l'étape en échec (`gh run view <RUN> -R JoshuaArus/spotify-mcp-server --log-failed | tail -40`).
  Le verdict est `alerte`.

L'étape est finie quand le run est conclu.

## Étape 5 : rendre le verdict

1. Retire le worktree, s'il existe : `git worktree remove --force /tmp/merge-amont`.
2. Ton dernier message est le verdict, au contrat de la veille d'Iris, et rien d'autre :

```text
<rien | nouveau | alerte>
- <nouveau | alerte> · <l'élément, en une ligne> ([<titre court de la source>](<url précise>))
retrait: oui
```

- **`rien`** tient sur sa seule ligne.
- **`nouveau`** porte une ligne : le nombre de commits amont, leur résumé en quelques
  mots, la CI verte, et la PR en source, `[Merge amont (#N)](<url de la PR>)`.
- **`alerte`** porte une ligne par blocage : les fichiers en conflit, avec le lien de
  comparaison en source ; ou le job et l'étape de CI en échec, avec le run en source,
  `[CI de la PR #N](<url du run>)`.
- **`retrait: oui`**, seulement quand l'amont est supprimé ou archivé (étape 1).

Le passage est fini quand le worktree est retiré et le verdict rendu.
