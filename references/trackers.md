# Où vit le suivi d'état

À lire une seule fois par session, après `.rfc-workflow.yml`.

Le skill porte la méthode ; le projet porte ses données. Ce fichier dit
comment les huit opérations de suivi se traduisent selon le backend
déclaré par `tracker:`.

## Lire la configuration

Au démarrage : `cat .rfc-workflow.yml` à la racine du projet. **Absent,
c'est un cas normal, pas une erreur** — appliquer les défauts et
continuer sans le signaler. Une seule lecture par session ; ne jamais
redécouvrir un chemin par exploration alors que la config le nomme.

| Clé | Défaut | Sert à |
|---|---|---|
| `tracker` | `files` | choisir la colonne du tableau ci-dessous |
| `mirror` | `null` | projection facultative ; `null` ou vide = aucune |
| `paths.rfc` | `docs/rfc` | où vivent les documents |
| `paths.registry` | `<rfc>/REGISTRY.md` | `files` uniquement |
| `paths.work` | `docs/.rfc-workflow` | analyses et handoffs locaux |
| `paths.debt` | `docs/DEBT.md` | dette sortie des handoffs |
| `paths.evidence` | `docs/evidence` | mesures et scripts versionnés |
| `paths.adr` | `docs/adr` | décisions structurantes |
| `tests` | *aucun* | commande de test ciblé ; sans elle, **demander** |
| `describes_present` | `README.md` | documentation à relire au classement |

## Les huit opérations

Ce sont les seuls points par lesquels le skill touche au suivi. Les
phases 1 à 5, 9 et 10 n'en utilisent aucune : elles lisent du code et des
documents, jamais un statut.

| Opération | `tracker: files` | `tracker: github` |
|---|---|---|
| `next_number` | ligne « prochain numéro libre » du registre | `gh issue list --label rfc --state all --json title --limit 500`, max des `RFC-XXXX` du titre, + 1 |
| `read_status` | ligne de la RFC dans le registre | `gh issue list --search "RFC-XXXX in:title" --state all --json number,title,labels,state` |
| `set_status` | éditer la ligne du registre, l'en-tête du document, **et** `git mv` au classement | `gh issue edit N --add-label state:… --remove-label state:…` ; au classement, `gh issue close N` **et** `git mv` le document |
| `list_all` | lire le registre en entier | `gh issue list --label rfc --state all --json number,title,labels,state --limit 500` |
| `save_handoff` | écrire `<work>/RFC-XXXX-handoff.md` | `gh issue comment N --body-file <fichier temporaire>` |
| `read_handoff` | lire `<work>/RFC-XXXX-handoff.md` | `gh issue view N --comments`, prendre le **dernier** bloc `## Handoff` |
| `add_debt` | une ligne dans `<debt>`, **puis `mirror_push_debt`** | `gh issue create --label dette --title "…" --body "Origine : RFC-XXXX"` |
| `close_debt` | réécrire l'entrée de `<debt>` : réglée → section « Réglé depuis » avec le sha ; réduite → constat resserré à ce qui subsiste. **Puis `mirror_push_debt`** | `gh issue close N --comment "…"` si réglée ; `gh issue comment N` si seulement réduite |

### Quand le suivi s'écrit, et où

**À chaque avancement, pas seulement au classement.** La règle vaut pour
tout ce que le suivi affiche — statut de RFC comme entrée de dette — et
c'est la règle qui se perd en premier, parce qu'un avancement partiel ne
ressemble pas à une fin.

Un suivi qui ne bouge qu'à la fin ne suit rien : pendant les heures où le
travail a lieu, il affiche `proposée` sur une RFC qu'on est en train
d'écrire, `todo` sur un bug déjà corrigé, et celui qui regarde le tableau
n'y voit rien venir. Le coût n'est pas cosmétique : **c'est ainsi qu'on se
retrouve à traiter deux fois la même chose**, ou à retravailler ce qu'une
autre séance vient de livrer.

Trois corollaires, tous appris à leurs dépens :

1. **Une correction partielle se synchronise aussi.** Ne pas attendre
   qu'une entrée soit entièrement réglée pour la mettre à jour : l'écrire
   *réduite à ce qui subsiste* vaut mieux que la laisser mentir en entier.
2. **Ce qui grossit se synchronise autant que ce qui se ferme.** Un bug
   qui se révèle plus large qu'écrit — un executor menteur qui s'avère
   en être trente — doit changer de taille dans le suivi le jour où on
   le découvre.
3. **Pousser fait partie de la synchronisation.** Un dépôt local en
   avance de huit commits laisse le miroir décrire un passé, quels que
   soient les labels. Vérifier `git status -sb` avant de conclure qu'un
   suivi est à jour.

Le statut s'affiche à trois endroits, et `set_status` les écrit **tous
les trois dans le même geste** :

| Où | Quoi |
|---|---|
| le suivi (`tracker`) | la ligne du registre, ou le label de l'issue — la **source** |
| l'en-tête du document RFC | le champ `**Statut**`, que lit quiconque ouvre le document |
| le miroir, s'il existe | `mirror_push` |

Un statut à jour ici et périmé là est un mensonge à retardement : le
lecteur croit le premier endroit qu'il ouvre. Les trois bougent ensemble,
ou l'opération n'est pas finie.

Le handoff suit la même règle : `save_handoff` à chaque fin de séance,
pas seulement à la dernière — c'est lui qui raconte *où on en est*.

### La dette se projette comme le reste

**`add_debt` et `close_debt` appellent `mirror_push_debt`, dans le même
geste.** La règle des trois endroits ci-dessus décrit le statut d'une RFC,
mais elle vaut mot pour mot pour une entrée de dette : un `<debt>` à jour et
un miroir périmé, c'est un tableau qui affiche `todo` sur un bug corrigé.

C'est la moitié du suivi qu'on oublie, parce qu'une dette n'a pas de
document à classer et que rien ne vient rappeler qu'elle vit ailleurs.
Constaté le 2026-08-18 : cinq entrées corrigées et commitées, et les cinq
issues encore ouvertes sur le miroir — dont une dont le titre décrivait un
executor corrigé la veille. Le coût est direct quand le miroir est l'endroit
où l'on suit des agents : la séance suivante retraite ce qui est fait.

| Opération | `mirror: github` |
|---|---|
| `mirror_push_debt` | l'issue existe (`gh issue list --label dette --state all`, repérée par son identifiant `B-7`, `M-5`… en tête de titre) → réglée : `gh issue close N --comment` ; réduite : `gh issue comment N`, et `gh issue edit N --title` si le titre ne décrit plus le constat ; absente : `gh issue create --label dette`. **Idempotente** |

Le miroir range les issues fermées en `Done` par lui-même ; ne pas écrire
dans le champ de projet. Une entrée réduite reste ouverte, et son
commentaire dit ce qui a été traité **et** ce qui subsiste — l'un sans
l'autre laisse croire à une fermeture ou à une absence de progrès.

## Les états, traduits

La machine d'états ne change pas ; seule son inscription change.

| État | `files` | `github` |
|---|---|---|
| ANALYSIS · IMPLEMENTATION · TESTING · BLOCKED · READY_TO_COMMIT | champ `## State` du handoff | label `state:analysis`, `state:implementation`, `state:testing`, `state:blocked`, `state:ready` |
| DONE | `## State` du handoff | label `state:done`, issue toujours ouverte |
| REJECTED | statut au registre | label `state:rejected` |
| FILED | statut au registre, document dans `implemented/` ou `rejected/` | **issue fermée**, document dans `implemented/` ou `rejected/` |

FILED ferme l'issue : c'est la seule traduction qui coïncide exactement
entre les deux mondes, et c'est ce qui rend le board lisible sans le
piloter — GitHub range les issues fermées tout seul.

## Ce que le tracker ne prend jamais

Sur les deux backends, sans exception :

- les **documents RFC** restent dans `<rfc>/`, versionnés, déplacés par
  `git mv` au classement ;
- les **ADR** restent dans `<adr>/` ;
- les **mesures et scripts** restent dans `<evidence>/`.

Un tracker porte un *statut*. Un document normatif de plusieurs centaines
de lignes doit se relire en diff, à côté du code qu'il décide : une issue
ne sait pas faire ça. `tracker: github` déplace le suivi, pas les
documents.

## Le miroir — regarder un second système sans lui rien confier

`mirror: <backend>` projette le suivi vers un autre système. Une seule
règle, et elle suffit à écarter le problème des deux vérités :

> **Le miroir s'écrit, ne se lit jamais.** `read_status`, `read_handoff`
> et `next_number` interrogent toujours `tracker`, jamais le miroir.

Une projection qu'on ne lit pas n'est pas une source concurrente : quand
elle diverge, on l'écrase depuis le tracker sans rien perdre. C'est ce
qui permet d'essayer un système en vraie grandeur — le voir vivre sur ses
propres données — sans lui remettre l'état du projet.

Deux opérations de plus. La première installe, une fois ; la seconde
projette, après chaque `set_status` et jamais avant :

| Opération | `mirror: github` |
|---|---|
| `mirror_setup` | crée ce qui manque, et rien d'autre : les labels, le projet (`gh project create`), ses vues, puis y verse les issues ouvertes (`gh project item-add`, une par URL). **Idempotente** : réajouter une issue déjà présente ne la duplique pas |
| `mirror_push` | l'issue existe (`gh issue list --search "RFC-XXXX in:title"`) → `gh issue edit` ses labels ; sinon `gh issue create`. Au classement, `gh issue close`. **Puis déplacer la carte du kanban** (voir ci-dessous) : sans ce second geste, le label change et la colonne ne bouge pas. **Idempotente** : relancer ne duplique rien |

### Déplacer la carte du kanban — le geste qu'on oublie

**Le label et la colonne sont deux champs différents.** Un board Projects v2
range ses cartes par son champ natif `Status` (`Todo` / `In Progress` /
`Done`), qui n'a aucun rapport avec les labels `state:*` posés sur l'issue
et **ne bouge jamais tout seul**. Poser le label sans toucher `Status`
donne exactement ce que ce skill a longtemps produit : une RFC en cours
d'écriture qui reste en `Todo` sur le kanban pendant des jours, pendant
que son issue affiche `state:analysis`. Celui qui regarde le tableau — et
c'est pour ça qu'on l'a créé — n'y voit rien venir.

Le mapping, des états du skill vers les trois colonnes :

| Colonne | États |
|---|---|
| `Todo` | rien n'a commencé : RFC enregistrée, dette ouverte |
| `In Progress` | `ANALYSIS`, `IMPLEMENTATION`, `TESTING`, `BLOCKED`, `READY_TO_COMMIT` — **`BLOCKED` compris** : une carte bloquée est un travail en cours qui a besoin de quelqu'un, pas un travail qui n'a pas commencé |
| `Done` | `DONE`, `FILED`, `REJECTED` |

Les identifiants de Projects v2 sont opaques et propres à chaque projet :
les **relire**, jamais les coder en dur ni les recopier d'une autre session.

```
gh project field-list <n> --owner <o> --format json    # id du champ Status + id de chaque option
gh project item-list  <n> --owner <o> --format json    # id de la carte, via content.number
gh project item-edit --id <carte> --project-id <projet> \
  --field-id <champ Status> --single-select-option-id <option>
```

Ce geste demande le scope `project`, comme `mirror_setup`. S'il manque, le
dire et poser le label quand même : une colonne périmée est un défaut connu,
un miroir muet en est un autre.

`mirror_setup` demande le scope `project` sur le jeton, que `gh` n'accorde
pas par défaut. Le reste du miroir fonctionne sans lui : seul le tableau en
dépend. Pour savoir si le scope est là, ne pas se fier à `gh auth status`,
qui lit un état local : demander à GitHub lui-même —

```
gh api -i user --hostname github.com | grep -i '^x-oauth-scopes'
```

S'il manque, le dire et s'arrêter là. `gh auth refresh -s project` est
interactive et appartient à l'utilisateur ; insister au-delà de deux essais
coûte plus que le tableau ne vaut, et créer un projet à la main prend trois
clics pour le même résultat.

**Ce que l'API permet, et ce qu'elle ne permet pas.** Les vues se créent et
se filtrent par GraphQL — `createProjectV2View` (`name`, `layout` parmi
`BOARD_LAYOUT` / `TABLE_LAYOUT` / `ROADMAP_LAYOUT`), puis
`updateProjectV2View` avec `filter`. En revanche le **regroupement** des
colonnes et le workflow *auto-add* ne sont exposés nulle part : les poser
reste un clic dans l'interface, à annoncer plutôt qu'à promettre.

**Créer le tableau n'est pas le piloter — mais il faut le piloter.**
`mirror_setup` l'installe une fois. Les vues *groupées par label* se
rangent ensuite seules ; **la vue kanban, non** : sa colonne vient du champ
`Status`, que `mirror_push` doit écrire à chaque transition.

L'objection qui a fait renoncer — « écrire le même état à deux endroits, et
le second finira par mentir » — décrit un vrai risque, mais ne pas écrire
ne l'évite pas : cela garantit que le second ment **dès la première
transition**, au lieu de risquer qu'il mente un jour. Entre deux champs
qu'on synchronise et deux champs dont un est faux par construction, le
choix ne se discute pas.

**Ce que le corps d'une issue contient** : le lien vers le document, son
statut, puis le document lui-même — c'est ce qu'on vient lire. Au-delà de
la taille maximale d'une issue, tronquer et renvoyer au fichier. Rien
d'autre : une note qui explique au lecteur le fonctionnement interne du
miroir occupe la place la plus visible de chaque carte pour ne rien lui
apprendre. Un pied de page suffit à dire où se fait la modification.

Un échec du miroir ne bloque **jamais** le travail : le signaler et
continuer. L'inverse ferait dépendre le dépôt d'un service distant pour
classer une RFC.

### Le miroir jamais posé — pas seulement la carte oubliée

Le piège documenté ci-dessus (label posé, colonne du kanban oubliée)
suppose que `mirror_push` a été exécuté. Le piège plus fréquent est en
amont : la session classe la RFC — `git mv`, registre mis à jour,
commit — et **n'appelle jamais `mirror_push` du tout**. Rien n'échoue,
rien ne le signale ; le registre dit « implémentée » et l'issue reste
ouverte, ou n'existe pas. Constaté sur ce dépôt le 2026-08-20 : onze RFC
classées entre le 2026-08-18 et le 2026-08-20, `mirror: github` déjà actif
depuis le matin du 2026-08-18, aucune n'avait d'issue.

La vérification ne se fait pas RFC par RFC (`gh issue list --search
"RFC-XXXX in:title"` coûte un appel par ligne du registre) : diffé
`<rfc>/REGISTRY.md` contre un seul `gh issue list --search "RFC- in:title"
--state all` couvre tout le dépôt en un appel. Si le projet fournit un
script pour ça (repéré par convention : `bin/check-*-mirror*` ou
équivalent nommé dans `.rfc-workflow.yml`), le lancer à la fin de la
phase 7 et en phase 8 (inventaire) — il coûte une commande et rattrape
exactement ce piège. S'il n'existe pas encore et que le projet a `mirror:
github`, proposer de l'écrire une fois (voir `bin/check-rfc-github-mirror`
dans le dépôt `concio` comme référence) plutôt que de relire le registre à
la main à chaque session.

**`mirror` vaut `null` par défaut**, et une clé vide (`mirror:`) vaut
`null` : la projection est l'exception, jamais l'état normal. Couper
l'essai, c'est vider la valeur — pas remanier le fichier. Les réglages du
backend (`github.repo`…) restent alors en place, inertes, et l'essai
reprend en réécrivant un mot.

Rien à rapatrier au passage : les fichiers n'ont jamais cessé d'être la
source. C'est la contrepartie d'une projection qu'on ne lit pas.

## Le board, en `tracker: github`

Le skill pose des labels sur les issues **et déplace la carte du kanban**.
Il a longtemps fait le premier geste seulement, en tenant pour acquis
qu'« un board Projects v2 range les labels par lui-même » : c'est faux
pour la vue qui compte. Le kanban range par son champ natif `Status`,
étranger aux labels, et une carte y reste en `Todo` tant que personne ne
l'écrit. Le coût de l'omission a été payé sur RFC-0092, restée « en
attente » sur le tableau pendant qu'elle était en cours d'analyse.

Le raisonnement écarté — « les identifiants opaques de Projects v2, c'est
de la configuration en échange d'aucune capacité nouvelle » — se trompait
sur la capacité en jeu : ce n'est pas du confort, c'est la seule chose que
le tableau existe pour montrer. Les identifiants se **relisent** en deux
commandes (voir `mirror_push`), ce qui ne coûte aucune configuration.

### Les labels, et pourquoi une lettre ne suffit pas

Dans un document, `D-8` vit sous un titre de section qui l'explique. Dans
une liste d'issues, la lettre est nue et ne dit rien à personne. Chaque
rubrique porte donc un label, qui devient aussi une colonne du tableau :

| Préfixe | Label | Rubrique |
|---|---|---|
| `B-` | `bug` | bugs actifs |
| `M-` | `instrument-faux` | mesures qui rassurent à tort |
| `S-` | `frontiere-sdk` | frontière SDK, code mort, vocabulaire |
| `A-` | `analyse-statique` | PHPStan et tests d'architecture |
| `E-` | `exploitation` | migrations, purges, environnement |
| `D-` | `a-arbitrer` | questions ouvertes |

Règle générale : **tout ce qu'un document rend lisible par sa structure —
section, colonne, position — doit devenir un label en le projetant.** Une
projection qui garde le code sans le contexte qui le décode est illisible.

### Les vues du tableau

Un projet porte plusieurs vues, chacune avec son filtre et son *group by* :
une par nature de travail, plutôt qu'un board unique où tout se mélange.

| Vue | Filtre | Group by |
|---|---|---|
| RFC | `is:open label:rfc` | `state:*` |
| Dette | `is:open label:dette` | rubrique |
| Bugs | `is:open label:bug` | — |

Prérequis, à vérifier une seule fois : `gh auth status` répond, et les
labels ci-dessus existent (`gh label create`). Si `gh`
manque ou n'est pas authentifié, le dire et s'arrêter — ne jamais
retomber silencieusement sur `files`, ce serait écrire le statut à un
endroit que personne ne lit.
