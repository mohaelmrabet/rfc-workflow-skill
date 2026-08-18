# Où vit le suivi d'état

À lire une seule fois par session, après `.rfc-workflow.yml`.

Le skill porte la méthode ; le projet porte ses données. Ce fichier dit
comment les sept opérations de suivi se traduisent selon le backend
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

## Les sept opérations

Ce sont les seuls points par lesquels le skill touche au suivi. Les
phases 1 à 5, 9 et 10 n'en utilisent aucune : elles lisent du code et des
documents, jamais un statut.

| Opération | `tracker: files` | `tracker: github` |
|---|---|---|
| `next_number` | ligne « prochain numéro libre » du registre | `gh issue list --label rfc --state all --json title --limit 500`, max des `RFC-XXXX` du titre, + 1 |
| `read_status` | ligne de la RFC dans le registre | `gh issue list --search "RFC-XXXX in:title" --state all --json number,title,labels,state` |
| `set_status` | éditer la ligne du registre **et** `git mv` le document | `gh issue edit N --add-label state:… --remove-label state:…` ; au classement, `gh issue close N` **et** `git mv` le document |
| `list_all` | lire le registre en entier | `gh issue list --label rfc --state all --json number,title,labels,state --limit 500` |
| `save_handoff` | écrire `<work>/RFC-XXXX-handoff.md` | `gh issue comment N --body-file <fichier temporaire>` |
| `read_handoff` | lire `<work>/RFC-XXXX-handoff.md` | `gh issue view N --comments`, prendre le **dernier** bloc `## Handoff` |
| `add_debt` | une ligne dans `<debt>` | `gh issue create --label dette --title "…" --body "Origine : RFC-XXXX"` |

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
| `mirror_setup` | crée ce qui manque, et rien d'autre : les labels (`rfc`, `dette`, `living`, `future`, `state:*`), le projet (`gh project create --owner <compte> --title <projet>`) et sa vue tableau, puis y verse les issues ouvertes (`gh project item-add`). **Idempotente** : ce qui existe est laissé tel quel |
| `mirror_push` | l'issue existe (`gh issue list --search "RFC-XXXX in:title"`) → `gh issue edit` ses labels ; sinon `gh issue create`. Au classement, `gh issue close`. **Idempotente** : relancer ne duplique rien |

`mirror_setup` demande le scope `project` sur le jeton, que `gh` n'accorde
pas par défaut. S'il manque, le dire et demander `gh auth refresh -s
project` — cette commande est interactive, elle appartient à l'utilisateur.
Le reste du miroir fonctionne sans ce scope : seul le tableau en dépend.

**Créer le tableau n'est pas le piloter.** `mirror_setup` l'installe une
fois ; ensuite les colonnes viennent du *group by* sur les labels, que le
board applique lui-même. Le skill n'écrit jamais dans un champ de projet —
il écrirait le même état à deux endroits, et le second finirait par mentir.

**Ce que le corps d'une issue contient** : le lien vers le document, son
statut, puis le document lui-même — c'est ce qu'on vient lire. Au-delà de
la taille maximale d'une issue, tronquer et renvoyer au fichier. Rien
d'autre : une note qui explique au lecteur le fonctionnement interne du
miroir occupe la place la plus visible de chaque carte pour ne rien lui
apprendre. Un pied de page suffit à dire où se fait la modification.

Un échec du miroir ne bloque **jamais** le travail : le signaler et
continuer. L'inverse ferait dépendre le dépôt d'un service distant pour
classer une RFC.

**`mirror` vaut `null` par défaut**, et une clé vide (`mirror:`) vaut
`null` : la projection est l'exception, jamais l'état normal. Couper
l'essai, c'est vider la valeur — pas remanier le fichier. Les réglages du
backend (`github.repo`…) restent alors en place, inertes, et l'essai
reprend en réécrivant un mot.

Rien à rapatrier au passage : les fichiers n'ont jamais cessé d'être la
source. C'est la contrepartie d'une projection qu'on ne lit pas.

## Le board, en `tracker: github`

Le skill n'écrit **pas** dans GitHub Projects. Il pose des labels sur les
issues ; un board Projects v2 branché sur le dépôt les range par
lui-même, et un board de compte agrège plusieurs dépôts sans que le skill
en sache rien. Piloter Projects v2 demanderait les identifiants opaques
de ses champs, projet par projet — de la configuration en échange
d'aucune capacité nouvelle.

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
