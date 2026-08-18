---
name: rfc-workflow
description: Workflow complet et économe en tokens pour travailler sur une RFC du projet — analyse ciblée, scope, plan, implémentation, tests, review, matrice de conformité, handoff, classement final, vulgarisation et reprise entre sessions. Ne réécrit jamais le corps du document RFC (ça c'est rfc-review) ; seule exception, le mode « vulgarise » qui insère en tête du document une section « En clair » expliquant la RFC en langage courant. Il classe en revanche la RFC sur laquelle il vient de travailler, dans implemented/ ou rejected/, et tient le registre des numéros — il IMPLÉMENTE ce que la RFC décide, avec un principe strict qualité/contexte consommé — pas de subagents par défaut, pas de scan global du repo, lectures chirurgicales, tests ciblés, handoff compact réutilisable dans une nouvelle session. Générique et réutilisable sur n'importe quel projet : les chemins, la commande de test et l'emplacement du suivi (fichiers versionnés ou issues GitHub) sont déclarés par un `.rfc-workflow.yml` à la racine du projet, jamais codés dans le skill. Couvre aussi l'inventaire des RFC (mode "statut", ex-skill rfc-status, fusionné ici) — tri des RFC nouvellement arrivées, détection de celles livrées mais restées non classées, registre des numéros. Utiliser quand l'utilisateur demande d'analyser, implémenter, reviewer, continuer, finaliser ou rejeter une RFC, de faire le point sur les RFC, de rendre une RFC compréhensible en langage courant, ou d'AUDITER une RFC déclarée livrée pour vérifier qu'elle l'est vraiment — livrables présents, mécanismes confrontés aux données réelles du dépôt, faux « OK » débusqués (ex. "Implémente RFC-0092", "Rejette RFC-0023", "Statut des RFC", "Vulgarise RFC-0029", "Audite RFC-0029").
argument-hint: "[analyse|implémente|review|audite|continue|finalise|rejette|statut|vulgarise] [RFC-XXXX]"
allowed-tools: [Read, Edit, Write, Bash, Grep, Glob]
---

# RFC Workflow

V1.3 — machine d'états, preuve réelle obligatoire, classification des
problèmes, pré-commit ; révisé le 2026-08-18 par l'audit documentaire
(sort des fichiers de travail, cible de la checklist de classement,
dossiers réellement balayés par le mode statut), puis rendu **générique**
le même jour : la méthode reste ici, les données du projet passent dans
`.rfc-workflow.yml`, et le suivi d'état s'adresse par sept opérations
(`references/trackers.md`) au lieu de chemins en dur.

Cible : `$ARGUMENTS`. Si le mode ou le numéro de RFC manque, demander —
ne pas deviner, ne pas scanner `docs/` pour proposer une liste.

**Numéroter une nouvelle RFC** : opération `next_number` (voir
`references/trackers.md`), ne jamais réutiliser un numéro même si son
document a disparu. Cette seule opération suffit — ne pas balayer
`<rfc>/` pour le redécouvrir.

**Toute RFC s'ouvre par une section « En clair ».** C'est une convention
du dépôt, pas une option du mode « vulgarise » : dès que le skill touche
une RFC qui n'en a pas — une nouvelle arrivée, un document ancien —, il
l'écrit selon la phase 9, sans attendre qu'on le demande. Le coût est
nul : à ce moment-là, la RFC vient d'être lue en entier. Le seul cas où
on s'abstient est un document appartenant à une autre session (modifié
et non commité) : on le signale au lieu de l'éditer.

## Principe directeur

**Architecture Restraint d'abord** (`CLAUDE.md`, ADR-0038). Dans tous les
modes : la solution la plus simple qui satisfait l'exigence gagne ; une
complexité supplémentaire — Event Bus, Outbox, Queue, worker, distributed
lock, cleanup d'orphelins, nouvelle couche, nouvelle base — n'entre dans le
chemin d'implémentation qu'avec une preuve (bug existant, test qui échoue,
invariant, contrainte explicite). Sans preuve : `Future Work`, et
l'implémentation continue. Chaque problème relevé est classé `REQUIRED` /
`RECOMMENDED` / `OPTIONAL` / `FUTURE` ; seul `REQUIRED` bloque. Quand les
exigences sont satisfaites, les invariants préservés, les risques critiques
couverts et les tests définis : **STOP**, on ne cherche pas de problèmes
hypothétiques supplémentaires.

Optimiser **qualité du travail / contexte consommé**, pas la quantité
d'analyse. Avant toute opération coûteuse (lecture d'un gros fichier,
recherche large, suite de tests complète), se demander : « apporte-t-elle
assez de valeur pour son coût en tokens ? » Si non, ne pas la faire.
Plus de contexte n'est jamais automatiquement mieux.

Règles transverses, valables dans tous les modes :

- **Pas de subagent par défaut.** Un subagent uniquement si la tâche est
  réellement indépendante, à périmètre fermé (fichiers nommés, question
  précise), et que son coût est justifié. Jamais de subagents multiples
  pour explorer le repo ou refaire la même analyse. Jamais de skills
  orchestrateurs (rfc-review, architecture-teardown) depuis ce skill.
- **Séquentiel, pas parallèle** : analyse → décision → implémentation →
  tests → review. Pas d'analyses concurrentes du même problème.
- **Lectures chirurgicales** : localiser d'abord par Grep/Glob ciblé sur
  les symboles que la RFC nomme, puis Read avec offset/limit sur la
  portion utile. Ne pas relire ce qui est déjà en contexte ou déjà
  consigné dans le fichier d'analyse.
- **Sorties filtrées** : `| head`, extraction du champ JSON utile, etc.
- **Question avant chaque exploration supplémentaire** : « cette lecture
  est-elle nécessaire pour le Next Action / la phase en cours ? » Si
  non, ne pas la faire. Ne pas relancer un test déjà prouvé sauf si le
  code qu'il couvre a changé depuis.

## Classification des problèmes

Tout problème découvert est classé **avant** d'agir :

- **RFC** — dans le périmètre : se corrige dans le workflow.
- **ENVIRONMENT** — infra locale (conteneurs, versions, réseau, droits).
- **DEPENDENCY** — composant ou service tiers dont la RFC dépend.
- **UNRELATED** — indépendant : Follow-up, jamais corrigé ici.

ENVIRONMENT et DEPENDENCY ne se corrigent que s'ils **bloquent
directement la validation de la RFC**, et avec le fix minimal (relancer
un conteneur, retag d'image…) — jamais un chantier d'infra. Si la
preuve reste impossible : état BLOCKED + blocage documenté. Ne jamais
élargir le périmètre de la RFC pour corriger un problème indépendant.

## Une RFC, une responsabilité (SOLID appliqué aux documents)

Un document d'architecture obéit aux mêmes règles que le code qu'il
décrit. Une RFC qui traite beaucoup de choses est un God Object en
Markdown : personne ne la relit en entier, ses sections divergent, et
elle devient impossible à implémenter par lots ou à classer, puisqu'une
partie est livrée pendant qu'une autre reste théorique.

**Test d'appartenance, à appliquer avant d'écrire.** Énoncer ce que fait
la RFC en **une phrase sans « et »**. Si la phrase a besoin d'un « et »,
d'un « ainsi que » ou d'un « & » dans le titre, il y a deux RFC.

Formulé autrement, en SRP : *combien de raisons distinctes cette RFC
a-t-elle de changer ?* Plus d'une → découper. Le modèle de données
change pour des raisons sémantiques, le chemin d'écriture pour des
raisons d'ingestion, le chemin de lecture pour des raisons
d'assemblage : trois horloges différentes, trois documents.

**Signaux de découpe** — aucun ne tranche seul, leur cumul si :

- le titre contient « & », « et », une virgule d'énumération ;
- plus d'une dizaine de sections de fond, hors annexes ;
- deux sections décrivent des composants qu'aucune livraison commune ne
  relie ;
- l'ordre d'implémentation interne comporte des étapes qui pourraient
  sortir seules et être testées seules ;
- une partie est livrée et l'autre non : la RFC ne peut alors ni être
  classée `implemented`, ni rester `proposed` honnêtement.

**Garde-fou inverse, tout aussi important.** Ne pas fragmenter pour
faire joli : chaque RFC issue d'un découpage doit être **livrable et
testable seule**. Si deux documents doivent obligatoirement être
implémentés dans le même commit pour que quoi que ce soit fonctionne,
c'est une seule RFC. Le découpage suit les coutures réelles du système,
pas la symétrie du plan (ADR-0038, règle 3).

**Quand on découpe**, appliquer « Frontières entre RFC » ci-dessous :
frontière énonçable en une phrase, `Non-goals` nommant le responsable,
dépendances qualifiées bloquantes ou facultatives, et vocabulaire aligné
sur le document le plus ancien.

## Frontières entre RFC (anti-chevauchement)

Deux documents qui décrivent le même contrat, c'est un contrat qui
n'existe nulle part : chacun suppose que l'autre fait foi, les noms
divergent, et l'implémentation n'a plus de référence. Le coût se paie
tard, quand un composant est écrit deux fois sous deux vocabulaires.

**Déclencheur.** Dès qu'on écrit une RFC nouvelle, qu'on en révise une,
ou qu'on s'apprête à en rejeter une comme « déjà couverte », faire ce
qui suit — c'est peu coûteux et ça ne se rattrape pas après coup.

1. **Recenser le voisinage.** Grep sur le domaine (`mémoire`, `intent`,
   `pipeline`…) dans `<rfc>/*/` et par `list_all`. Le suivi
   suffit à repérer les voisines ; il ne suffit pas à savoir ce
   qu'elles contiennent.
2. **Lire les voisines en entier, pas leur en-tête.** Une RFC se juge
   sur ses sections, pas sur son titre ni sur son « En clair » : c'est
   au milieu du document que se cachent les contrats dupliqués. Lire
   les 40 premières lignes et la table des matières ne suffit pas — les
   doublons vivent dans les sections tardives, celles qui descendent au
   niveau des interfaces.
3. **Produire une table de correspondance, décision par décision**, et
   la vérifier **dans les deux sens** : ce que le document reprend de
   l'autre, et ce que l'autre couvrait déjà sans qu'on le sache. Une
   décision orpheline, repérée dans ce passage, est exactement ce qui
   se perd quand on absorbe sur un résumé.
4. **Trancher la frontière par une phrase**, écrite dans les deux
   documents. Une frontière énonçable tient ; une frontière qui demande
   un paragraphe ne tient pas. Exemple éprouvé : *la mécanique
   opérationnelle appartient au RFC du chemin, la décision périodique
   au RFC de l'élection.*
5. **Les noms antérieurs gagnent.** Si deux documents nomment
   différemment la même opération, le document le plus ancien fixe le
   vocabulaire et le plus récent s'aligne — sans exception, y compris
   quand le nom récent semble meilleur. Deux noms pour une opération
   coûtent plus cher qu'un nom imparfait.
6. **Écrire un « Non-goals » qui nomme le responsable**, pas un simple
   « hors périmètre » : chaque ligne cite la RFC qui prend le relais.
   Et pour les documents à forte frontière, une section normative
   « Ce que ce RFC ne doit PAS faire ».
7. **Tracer les dépendances** : un diagramme et une table qui
   qualifient chaque lien de **bloquant** ou **facultatif**. Une
   dépendance non qualifiée devient un prérequis découvert au moment de
   l'implémentation.

**Une RFC ne se rejette jamais comme « déjà couverte » sans cette
table.** Le cas qui a fondé cette règle : RFC-0051 absorbée par
RFC-0056 et RFC-0060, où l'oubli explicite de l'utilisateur n'était en
réalité couvert par aucune des deux au moment du rejet.

## Configuration du projet

Le skill porte la méthode ; **les données du projet vivent dans le
projet**. Premier geste de toute session : `cat .rfc-workflow.yml` à la
racine. Il déclare où vit le suivi (`tracker: files` ou `github`), les
chemins et la commande de test. **Absent, c'est un cas normal** :
appliquer les défauts de `references/trackers.md` et continuer sans le
signaler.

Dans tout ce document, `<rfc>`, `<work>`, `<debt>`, `<evidence>` et
`<adr>` désignent les chemins de cette configuration. Le suivi d'état ne
se touche que par les opérations `next_number`, `read_status`,
`set_status`, `list_all`, `save_handoff` / `read_handoff` et `add_debt`,
définies une fois pour toutes dans `references/trackers.md`. Une phase
qui écrirait un chemin de registre en dur casserait le skill sur le
projet suivant — c'est la seule règle que la généricité ajoute.

## Localiser une RFC

Un seul Glob : `<rfc>/*/RFC-XXXX_*.md`. Ne rien lire d'autre.
Les conventions de méthode (sens des sous-dossiers, typologie des
documents, skills voisins) sont dans `references/conventions.md` — le
lire une seule fois par session.

## État persistant (reprise entre sessions)

En `tracker: files`, tout l'état de travail vit dans `<work>` ; en
`tracker: github`, le handoff est un commentaire de l'issue et le reste
demeure local :

- `RFC-XXXX-analysis.md` — produit de la phase d'analyse. Il **porte le
  plan** dans sa section `Implementation Plan` : ne pas écrire de
  `RFC-XXXX-plan.md` à côté. Le dépôt en compte huit pour cinquante-quatre
  analyses, tous doublons d'une section qui existait déjà — la règle
  « deux documents pour un contrat, c'est un contrat qui n'existe nulle
  part » vaut aussi pour les fichiers de travail.
- `RFC-XXXX-handoff.md` — handoff compact (≤ ~1000 tokens), réécrit à
  chaque fin de séance de travail sur la RFC. Écrit et relu par
  `save_handoff` / `read_handoff`, jamais par un chemin en dur.
- `RFC-XXXX-measurement.md` et `RFC-XXXX-scripts/` — instruments et
  chiffres, quand la RFC en produit.

### Sort de ces fichiers au classement (phase 7)

Trois catégories, une règle chacune, sans recouvrement :

| Fichier | Au classement |
|---|---|
| `-analysis.md` (et les `-plan.md` hérités) | **Rien à faire.** Le dossier n'est pas versionné : le fichier reste local et disparaît avec la machine. Un plan exécuté n'a plus de lecteur — le code en est la preuve. |
| `-handoff.md` | **Rien à faire non plus**, à une condition : ce qu'il porte de durable doit être sorti *avant*. Une dette, un piège, une décision non traitée → `add_debt`. Ce qui n'en sort pas est perdu au prochain poste de travail, et c'est assumé pour un récit de séance. |
| mesures et scripts | **Sortis vers `<evidence>`, versionnés.** Ce sont les seuls porteurs des chiffres qui ferment un mécanisme : ils survivent à la RFC et se citent depuis elle. Une mesure laissée dans le dossier de travail est une mesure perdue. |

Deux conséquences à tenir :

1. **`<work>` n'est pas versionné** (`.gitignore`, depuis le
   2026-08-18) : c'est un bloc-notes local, jamais une source. Rien de ce qui
   doit survivre à la session n'y reste — les chiffres vont dans
   `<evidence>`, la dette par `add_debt`, le statut par `set_status`.
2. **Un handoff ne fait jamais foi sur l'état d'une RFC.** `read_status` est
   la seule source du statut ; un handoff dit ce qu'une séance a fait, à sa date.
   Un handoff conservé qui annonce `READY_TO_COMMIT` sur une RFC classée est un
   défaut à corriger, pas une information.

## Machine d'états

L'avancement d'une RFC est un état explicite, tenu dans le champ
`## State` du handoff :

`ANALYSIS → IMPLEMENTATION → TESTING → READY_TO_COMMIT → DONE → FILED`
plus `BLOCKED` et `REJECTED`, accessibles depuis n'importe quel état.

- **ANALYSIS** — phase 1 en cours ou à faire.
- **IMPLEMENTATION** — scope arrêté, code en cours (phase 2).
- **TESTING** — code écrit ; tests, review, matrice, preuve réelle en
  cours (phases 3-5).
- **BLOCKED** — quelque chose empêche de progresser ou de **prouver**.
  Documenter dans `## Blocking Issue` : cause, classification (voir
  « Classification des problèmes »), tentatives, condition de déblocage.
- **READY_TO_COMMIT** — toutes les preuves réunies, ne reste que le
  commit (voir « Pré-commit »).
- **DONE** — commité et prouvé.
- **REJECTED** — la proposition ne sera pas implémentée, et **on sait
  pourquoi**. Deux formes recevables, et deux seulement :
  - *infirmée* — hypothèse mesurée puis démentie, avec les chiffres
    contre la baseline en service ;
  - *absorbée* — le contenu est repris par d'autres RFC, avec la table
    de correspondance décision par décision qui le prouve (voir
    « Frontières entre RFC »).

  Un abandon par lassitude ou par manque de temps n'en est pas un :
  sans mesure ni table, l'état reste BLOCKED. Voir « Phase 7 ».
- **FILED** — document classé dans `<rfc>/implemented/` ou
  `<rfc>/rejected/`. Une RFC n'est finie que classée.

**DONE ne peut être déclaré que si tout ceci est vrai :**

1. critères d'acceptation vérifiés (matrice sans FAIL ni NOT_TESTED
   injustifié) ;
2. tests ciblés verts ;
3. run réel vert quand la RFC en exige un (voir « Preuve réelle ») ;
4. diff final contrôlé (phase 4) ;
5. handoff final à jour.

Un seul point manquant → l'état reste TESTING (ou BLOCKED), jamais DONE.

## Modes

| Intention utilisateur | Phases exécutées |
|---|---|
| "Analyse RFC-XXXX" | 1 uniquement — aucune modification de code |
| "Implémente RFC-XXXX" | 1 → 2 → 3 → 4 → 5 → 6 → 7 |
| "Review RFC-XXXX" | 4 (+ 5 si critères vérifiables) — aucune modification sauf demande explicite |
| "Audite RFC-XXXX" / "RFC-XXXX est-elle vraiment implémentée ?" | 10 uniquement — aucune écriture, ni code ni document ; produit un verdict et des gaps |
| "Continue RFC-XXXX" | 0, puis reprise à la phase indiquée par le handoff |
| "Finalise RFC-XXXX" | 3 → 4 → 5 → 6 → 7 |
| "Rejette RFC-XXXX" | prototype + mesure, puis 7 — jamais de rejet sans chiffres |
| "Statut des RFC" / "Fais le point sur les RFC" | 8 uniquement — aucune modification de code, aucun numéro de RFC requis |
| "Vulgarise RFC-XXXX" / "Explique-moi RFC-XXXX simplement" | 9 uniquement — aucune modification de code ; seule écriture autorisée : la section « En clair » en tête du document RFC |

## Phase 0 — Reprise ("Continue")

1. `read_handoff` **et rien d'autre**, en premier. S'il existe : déterminer l'état courant (`## State`), puis
   ne lire que les fichiers strictement nécessaires au « Next Action »
   et reprendre exactement à ce checkpoint. **Jamais d'analyse globale,
   jamais de relecture massive** des fichiers listés dans « Already
   Analyzed » ; respecter « Do Not Repeat » à la lettre.
2. S'il n'existe pas : lire `<work>/RFC-XXXX-analysis.md` s'il existe, sinon
   vérifier `git log --oneline --grep="RFC-XXXX"` et `git status` pour
   estimer l'état, puis seulement démarrer la phase 1.
3. Si le handoff est contredit par la réalité (fichier disparu,
   `git status` inattendu) : corriger le handoff sur ce point précis et
   continuer — pas de ré-analyse complète pour autant.

## Phase 1 — Analyse

Aucune modification de code pendant cette phase.

1. Lire la RFC **une seule fois**, en entier. Les RFC du repo sont en
   format libre : extraire objectif, motivation, décisions, contraintes,
   critères d'acceptation, dépendances et RFC liées où qu'ils soient.
1bis. Si le document n'ouvre pas sur une section `## En clair`, l'écrire
   maintenant (phase 9) — c'est le seul moment où la RFC est entièrement
   en contexte. Une RFC nouvellement arrivée en repart donc toujours
   vulgarisée.
2. Identifier le code réellement concerné : Grep sur les classes,
   fichiers, routes ou commandes que la RFC nomme explicitement.
   N'inspecter que ce code, portion par portion.
3. Écrire `<work>/RFC-XXXX-analysis.md` selon le template
   « RFC Analysis » de `references/templates.md` (Objective, Current
   State, Expected State, Gap, Scope, Out of Scope, Dependencies, Risks,
   Implementation Plan, Follow-up).

**RFC déjà (partiellement) implémentée** : si le Grep initial montre que
le code attendu existe déjà (symboles présents, `git log
--grep="RFC-XXXX"` non vide), ne pas repartir de zéro. L'analyse devient
une analyse de gaps : lister ce qui existe déjà (avec preuve
fichier:ligne), ce qui manque, et ne vérifier/implémenter que les gaps,
depuis l'état réel du repo.

**Anti scope-creep** : toute découverte hors périmètre (dette technique,
problème architectural indépendant, refactoring tentant, autre RFC) va
dans la section `## Follow-up` de l'analyse — jamais dans le travail de
la RFC courante.

En mode "Implémente", présenter le plan en 5-10 lignes puis enchaîner ;
ne s'arrêter pour validation que si la RFC laisse un choix structurant
ouvert ou contredit l'état du code.

## Phase 2 — Implémentation

- Modifier uniquement les fichiers listés dans le Scope de l'analyse.
- Respecter les décisions de la RFC ; aucun changement architectural
  non prévu, aucun refactoring opportuniste, aucune modification
  d'autres RFC ou de composants sans rapport.
- Petits changements vérifiables : implémenter et tester incrément par
  incrément plutôt qu'un gros diff final.
- **Pas de faux « OK »** : jamais de statut codé en dur
  (`'status' => 'OK'`, succès par défaut) quand un résultat réel permet
  de le déterminer — l'assertion doit dériver du résultat effectif.

## Phase 3 — Tests

Escalade stricte, jamais toute la suite automatiquement :

1. tests directement concernés (fichier de test précis) ;
2. tests du composant (répertoire) ;
3. tests d'intégration pertinents ;
4. suite plus large **uniquement** si le changement le justifie
   (ex. modification d'un contrat partagé).

En cas d'échec : identifier la cause ; si elle est étrangère à la RFC,
la consigner en Follow-up sans la corriger ; sinon corriger, relancer le
test ciblé, puis élargir progressivement.

## Phase 4 — Review

Review ciblée du diff (`git diff` limité aux fichiers du Scope) :
conformité à la RFC, critères d'acceptation, régressions, tests
manquants, erreurs, cohérence architecturale directement liée à la RFC,
statuts codés en dur (faux « OK »). Quand la RFC porte précisément sur
la fiabilité/validation, vérifier que chaque assertion reflète
réellement le résultat qu'elle prétend vérifier.

Classement : **P0** bloque · **P1** important · **P2** amélioration ·
**P3** hors périmètre. Corriger P0 et P1 ; consigner P2/P3 en Follow-up
sans les corriger.

## Phase 5 — Matrice de conformité

Pour chaque critère d'acceptation de la RFC (ou, à défaut, chaque
décision vérifiable), produire la matrice du template : statut
PASS / PARTIAL / FAIL / NOT_TESTED avec preuve concrète (commande +
résultat, ou fichier:ligne). **Une RFC n'est jamais déclarée terminée
avec un FAIL ou un NOT_TESTED non justifié.**

## Preuve réelle

Un test unitaire ou mocké ne prouve pas automatiquement le comportement
réel. Quand la RFC exige un run réel (commande e2e, vrai service, vrai
environnement) : `TARGETED_TESTS_PASS ≠ DONE` — il faut aussi
`REAL_RUN_PASS`, consigné dans `## Evidence` (commande exacte + sortie).
Si l'environnement empêche le run réel : état **BLOCKED** avec le
blocage documenté précisément — jamais DONE. Avant de conclure à un
blocage, vérifier qu'on utilise le bon entrypoint (ex. client remote vs
console locale, mauvais conteneur).

## Pré-commit (READY_TO_COMMIT)

Jamais de commit automatique. À cet état :

1. `git status` — repérer les fichiers d'autres sessions/chantiers ;
2. `git diff --stat` — volume et fichiers réellement touchés ;
3. vérifier que chaque fichier modifié appartient au scope de la RFC —
   sinon le signaler explicitement, jamais l'embarquer en silence ;
4. afficher un résumé compact (fichiers + message de commit proposé) ;
5. demander confirmation à l'utilisateur avant le commit.

## Phase 6 — Handoff

`save_handoff` avec le template « HANDOFF » de
`references/templates.md`. Compact (≤ ~1000 tokens) :
il doit suffire à reprendre dans une session vierge sans relire
l'historique. « Already Analyzed » et « Do Not Repeat » évitent à la
session suivante de repayer l'exploration.

Aussi générer un handoff (même partiel) chaque fois qu'une séance de
travail s'interrompt avant la fin.

## Phase 7 — Classement (DONE ou REJECTED)

Dernier geste du travail, jamais optionnel. `git mv` fichier par
fichier — d'autres sessions écrivent le même repo.

**`set_status` et le registre ADR (`<adr>`)** dans le même commit :
- `set_status` : statut final et nouveau chemin. En `tracker: files`, cela veut dire la ligne de la RFC **et** « prochain numéro libre » du registre ; en `tracker: github`, le label d'état et la fermeture de l'issue.
- **Relire les réserves de lecture que la RFC vient de périmer** (`tracker: files` ; en `github`, les réserves vivent dans le corps de l'issue). Une RFC ne périme pas que du code : elle périme ce que le registre affirmait avant elle. RFC-0041 a retiré `php-vcr` du dépôt sans que la réserve RFC-0032 — « `php-vcr` **est** configuré dans `tests/VcrTestCase.php` » — bouge : le registre s'est contredit à soixante lignes d'intervalle. Grep le registre sur les symboles que la RFC supprime ou remplace, et corriger la réserve au lieu d'en ajouter une.
- Si la RFC introduit ou valide une décision structurante d'architecture (choix de composant, pattern d'élection, modèle d'apprentissage, etc.), **créer ou mettre à jour un ADR synthétique** dans `<adr>/ADR-XXXX-nom.md` et enregistrer sa ligne dans `<adr>/README.md`.
- **Une réserve qui explique une règle générale n'est pas une réserve, c'est un ADR.** Le suivi dit le statut d'une RFC ; quand un paragraphe y enseigne quelque chose qui vaut au-delà d'elle — « une section *Écarts avec le code actuel* se périme au moment où la RFC est implémentée » — il appartient à `<adr>`, et la réserve n'en garde qu'un renvoi.

**Vérifier la documentation qui décrit le présent, dans le même commit.**
Une RFC livrée périme souvent ce qui décrit l'état du système, et personne
ne s'en aperçoit avant que ça mente depuis dix RFC — `README.md` a décrit
« aucune fonctionnalité métier implémentée » jusqu'à RFC-0028,
`docs/architecture/README.md` décrivait encore le pipeline du vertical
slice RFC-0002, et `pipeline-execution-flow.md` a annoncé onze étapes
jusqu'à ce qu'un audit le lise : il n'était nommé dans aucune checklist,
donc il n'a jamais été relu.

La cible est donc **un dossier, pas une liste de fichiers** (ADR-0050) :
`describes_present` de la configuration, **y compris les fichiers créés
dans ces dossiers depuis l'écriture de cette checklist**. Pour Concio :
`README.md` et tout fichier de `docs/architecture/`.

Se poser trois questions, et n'écrire que si la réponse est oui :

1. l'état annoncé (« ce qui est en place / pas encore ») a-t-il changé ?
2. un schéma, un pipeline, une cascade, une liste de services ou
   d'endpoints décrits ailleurs ne correspond-il plus au code ?
3. la RFC ajoute-t-elle une commande, une variable d'environnement ou
   une étape d'exploitation qu'un nouvel arrivant doit connaître ?

Si rien n'a bougé, ne rien écrire : le bruit documentaire coûte autant
que le mensonge. Et ne pas amender une RFC ni un ADR pour les remettre au
présent — ils datent une décision ; quand une décision ultérieure les
renverse, on l'écrit **en tête** du document et on ne touche pas au corps
(patron : RFC-0003 §0). Décrire le code tel qu'il est **câblé en
production**, pas tel que la RFC l'espérait — un étage prévu mais passé
à `null` dans le wiring se signale comme non monté.

**Fermer le handoff avant de classer.** Un handoff qui survit au
classement en annonçant `READY_TO_COMMIT` devient une source de vérité
concurrente, et elle perd : quatorze l'ont fait dans ce dépôt, sur des RFC
déjà dans `implemented/`. Au classement, le handoff est soit supprimé, soit
mis à `FILED` avec le sha livreur. Il n'y a pas de troisième issue.

**Implémentée** → `<rfc>/implemented/`. Ajouter en tête du document :
statut, date, et les commits qui la livrent. Puis appliquer la table
« Sort de ces fichiers au classement » de la section *État persistant* :
l'analyse et le plan partent, le handoff part ou s'archive selon ce qu'il
porte, les mesures et les scripts restent — hors du dossier de travail.
Ce qui doit survivre à la RFC, ce sont les chiffres et les pièges, pas le
récit de la séance.

**Rejetée** → `<rfc>/rejected/`, **un seul fichier** réunissant :

1. la proposition d'origine, intacte — un lecteur doit pouvoir juger sur
   pièces, pas sur le résumé qu'en fait celui qui l'écarte ;
2. la mesure qui l'infirme : instrument, corpus, variantes balayées,
   chiffres contre la baseline en service ;
3. ce que l'expérience a démontré, y compris ce qui vaut au-delà de la
   proposition (une métrique mal construite, une prémisse fausse, un
   coût caché) ;
4. le levier réel, s'il a été identifié en chemin.

Une hypothèse falsifiée avec des chiffres est un acquis : elle empêche
de reproposer la même piste sans savoir ce qu'elle coûte. La supprimer
efface ce travail. `RFC-0023-tripartite-measurement.md` sert de modèle.

**Rejet par absorption** — même dossier, même exigence de conservation,
mais les points 2 à 4 sont remplacés par :

2'. la **table de correspondance** : chaque décision et chaque critère
    d'acceptation d'origine, en face de la RFC qui le reprend et de la
    section précise où il atterrit ;
3'. ce que l'inventaire a révélé, en particulier toute décision
    orpheline — celle que personne ne reprenait et qu'on aurait
    supprimée sans le savoir ;
4'. la frontière retenue entre les documents qui héritent, en une
    phrase.

Une absorption sans table n'est pas un rejet, c'est une perte.
`RFC-0051_memory_consolidation_and_reconciliation.md` sert de modèle.

Le rejet appartient à l'utilisateur. Le rôle du skill est de **mesurer
avant de conclure** : prototyper contre le harnais d'évaluation existant,
donner à la proposition sa meilleure configuration, et rapporter les
chiffres — y compris ceux qui contredisent l'attente initiale.

## Phase 8 — Inventaire (mode statut)

Le seul mode qui balaie `<rfc>/` au lieu de lire chirurgicalement.
Il ne touche jamais au code : **aucun `Edit`/`Write` sur `src`, `tests`
ou `config`** — uniquement des lectures, et des `git mv` de documents
après confirmation explicite de l'utilisateur.

Les sous-dossiers réels sont `proposed/`, `implemented/`, `rejected/`,
`living/`, `future/` — il n'y a ni `proposals/`, ni `blocked/`, ni RFC à
plat dans `docs/`. Balayer un dossier inexistant donne un mode statut qui
ne trouve jamais rien et qui rassure à tort : c'est ce qui a laissé cinq
lignes du registre annoncer « implémentée » avec un chemin
`docs/rfc/implemented/…` pendant que les documents dormaient dans
`proposed/`.

1. **`<rfc>/proposed/`** — le dossier qui compte, à relire à chaque
   passage : un statut a pu changer depuis le dernier. Une proposition
   dont le code est livré est une phase 7 oubliée par la session qui l'a
   traitée — proposer le déplacement vers `implemented/`, ne pas le faire
   d'office. Une RFC modifiée ou non commitée par une autre session
   (`git status --short`) **ne se déplace jamais** : le signaler.
   Les ~10 premières lignes disent aussi si le document ouvre sur
   `## En clair`. Ne pas le rédiger ici — cela demanderait de lire chaque
   RFC en entier, ce que ce mode s'interdit : les lister dans la sortie
   comme « à vulgariser », et le faire à la prochaine phase 1 qui les
   ouvrira, ou sur demande.
2. **RFC arrivées hors dossier** (`ls <rfc>/*.md` et le dossier parent)
   — jamais triées. Lire les ~10 premières lignes, classer, `git mv`.
   Zéro aujourd'hui : le contrôle reste, il coûte une commande.
3. **`<rfc>/future/`** — lecture rapide : le déclencheur nommé par le
   document s'est-il matérialisé ? Si oui, vers `proposed/`.
4. **`implemented/`, `rejected/`, `living/`** — ignorés par défaut, sauf
   si `$ARGUMENTS` cible l'une d'elles.
4bis. **`list_all`** — vérifier que le suivi reflète les déplacements
   constatés et que `next_number` rendrait le bon numéro. C'est la seule
   lecture dont un auteur de RFC a besoin pour numéroter. La cohérence *mécanique* — chemin cité qui existe, RFC du
   disque inscrite, ADR indexé, lien mort — n'est plus à relire : elle
   est tenue par `tests/Architecture/DocumentationConsistencyTest.php`
   (ADR-0050). Lancer la suite `architecture` vaut mieux que la relire.
   Ce que le test ne dit pas, et qui reste le travail de ce mode : si le
   **statut** annoncé correspond au code livré.
4ter. **Les fichiers de `describes_present`** — lecture rapide :
   l'état annoncé cite-t-il encore une RFC dépassée, un composant
   remplacé, un lien vers un dossier qui n'existe plus ? Le signaler
   dans la sortie comme dette documentaire, sans réécrire ici (c'est la
   phase 7 qui écrit).
5. **Ne jamais conclure « implémentée » sur la foi du document.** Si le
   `Status` ne cite pas de sha, chercher dans
   `git log --oneline --all -- <rfc>/*/RFC-XXXX*` et dans les commits
   `feat`/`fix` qui mentionnent le numéro ; au moindre doute, grep le
   code pour la classe ou la méthode annoncée comme livrée.

**Sortie** : un tableau — RFC | dossier | catégorie | statut réel
constaté | à classer (implemented / rejected / reste sur place) | sha
qui la livre si le code est confirmé. Puis attendre la confirmation
avant tout `git mv` et tout commit. Ne rien supprimer : la convention de
suppression des propositions implémentées ne s'applique plus, elles sont
déplacées dans `implemented/`.

Ce mode remplace l'ancien skill `rfc-status`, fusionné ici le
2026-08-13 : les conventions n'existent qu'à un seul endroit
(`references/conventions.md`), et elles avaient déjà divergé entre les
deux skills.

## Phase 9 — Vulgarisation (« En clair »)

Rendre la RFC compréhensible **sans connaissance préalable du projet**,
en insérant une section `## En clair` en tête du document, juste après
le titre `#` et avant tout le reste (statut, board note, corps).

C'est la **seule** écriture que ce skill s'autorise dans un document RFC.
Elle est additive : le corps de la RFC n'est ni reformulé, ni réordonné,
ni corrigé — une erreur repérée en chemin va en Follow-up, pas dans le
document.

1. Lire la RFC une seule fois, en entier. Aucune lecture de code : la
   vulgarisation explique ce que la RFC **dit**, pas ce que le code fait.
2. Écrire la section selon le template « En clair » de
   `references/templates.md` : **deux courts paragraphes de prose**, pas
   un formulaire à puces.
   - Le premier dit ce que la RFC fait en mots de tous les jours, puis
     bascule sur « **mais surtout** » pour donner le vrai enjeu — celui
     que le titre ne dit pas.
   - Le second **situe** la RFC, et assume un avis sur ce que ça vaut.
     Situer veut dire deux choses selon le document :
     - une RFC vivante se situe **parmi ses voisines** — « RFC-0027
       apprend à s'abstenir ; RFC-0028 donne l'étage auquel déléguer » ;
     - une RFC classée dans `rejected/` ou tenue comme archive se situe
       **dans le temps du projet** : pourquoi le document existe encore
       alors que rien n'en sera implémenté — « mesuré sur 3 247 cas, le
       prototype ne bat la version en service dans aucune configuration,
       si bien qu'on ne peut plus reproposer la piste sans savoir ce
       qu'elle coûte ».

   Une brique se comprend par sa place, jamais par sa description isolée.
3. Insérer ou remplacer. Si une section `## En clair` existe déjà, la
   **remplacer intégralement** — jamais en empiler deux. Opération
   idempotente : relancer le mode sur la même RFC ne change rien d'autre.
4. Afficher à l'utilisateur la section produite, et lui demander si elle
   lui parle avant de considérer le travail fini.

Contraintes de rédaction :

- **10 lignes de texte maximum, tout compris** (lignes vides non
  comptées), et viser 6. C'est la contrainte principale du mode :
  au-delà, on relit une seconde RFC au lieu de la comprendre.
- Pas de glossaire, pas de liste de « ce qui change / ne change pas » :
  un terme technique inévitable se traduit dans la phrase même, entre
  tirets. Le reste du vocabulaire disparaît.
- Si ça ne rentre pas, c'est qu'on explique des détails : couper les
  détails, pas la clarté. Les seuils, noms de classes et énumérations
  appartiennent au corps du document.
- Zéro terme technique non traduit dans la même phrase ou dans le jargon.
- Aucune invention : rien qui ne soit pas dans la RFC. Si un point est
  incompréhensible faute d'information dans le document, l'écrire tel
  quel (« la RFC ne dit pas comment… ») plutôt que de combler le trou.
- Pas de chiffres, seuils ou noms de classe recopiés pour faire sérieux :
  ils appartiennent au corps du document.

Cette phase se déclenche **d'office** dès qu'une RFC ouverte par le
workflow n'a pas encore sa section — voir la règle en tête de document.
Le mode « vulgarise » sert alors à la refaire quand la première ne
convient pas, ou à traiter une RFC qu'aucun autre mode n'a ouverte.

## Phase 10 — Audit d'une RFC déclarée livrée

Vérifier qu'une RFC annoncée implémentée l'est réellement. La phase 4
relit un diff qu'on vient d'écrire ; celle-ci instruit une livraison dont
on ne sait rien, souvent faite par une autre session ou un autre outil.
Elle n'écrit rien — ni code, ni document, ni classement — et se termine
sur un verdict et une liste de gaps que l'utilisateur arbitre.

1. **Ne rien croire sur parole.** Un statut « implémentée », une ligne de
   registre et un handoff triomphal sont des affirmations, pas des
   preuves. Chercher la livraison réelle : `git log --grep="RFC-XXXX"`,
   et `git status` — du code non commité peut porter la livraison, et
   c'est justement là que personne n'a relu.
2. Lire la RFC une fois, en extraire la **liste fermée** des livrables et
   des décisions vérifiables. Si la RFC est découpée en phases, n'auditer
   que la phase déclarée livrée, et dire des autres qu'elles sont hors
   audit plutôt que de les compter en échec.
3. Pour chaque livrable, localiser le symbole annoncé (Grep ciblé) et
   trancher entre trois issues, jamais deux : **absent**, **présent et
   juste**, **présent et faux**. La troisième est la seule qui coûte cher
   à découvrir tard.
4. **Confronter chaque mécanisme aux données réelles du dépôt, pas à ses
   fixtures.** C'est la règle qui trouve tout : un test qui fabrique ses
   propres cas prouve que le code s'exécute, jamais qu'il mesure quelque
   chose. Toute valeur sentinelle, tout nom de catégorie, toute clé codée
   dans le mécanisme doit exister dans le corpus, la config ou le schéma
   réels — un grep sur le jeu de données suffit à le savoir.
5. **Chasser les faux « OK »** : branche vide qui rend un succès (`return
   1.0` sur zéro cas observé), catégorie inconnue notée zéro, statut codé
   en dur, assertion qui ne dérive pas du résultat qu'elle prétend
   vérifier. Un instrument faux est plus dangereux qu'un instrument
   absent : il rassure.
6. Rejouer les preuves citées, et **classer les échecs avant de les
   attribuer** : reproduire sur un worktree de `HEAD` (`git worktree add`,
   `vendor` en lien symbolique) avant d'accuser le travail audité. Un
   échec qui préexiste est UNRELATED et va en Follow-up, jamais dans le
   verdict.
7. **Sortie** : un tableau — livrable | attendu par la RFC | constaté
   (`fichier:ligne` ou commande + résultat) | verdict PASS / PARTIAL /
   ABSENT / FAUX | gravité P0-P3 — puis un verdict global en une phrase,
   puis ce qu'il faudrait faire.
8. Ne corriger **rien** sans go explicite. Sur go, l'audit devient
   l'analyse de gaps de la phase 1 et le travail reprend en mode
   « implémente » à partir de l'état réel du repo.

Un audit qui ne trouve rien est un résultat : le dire en une ligne, avec
les preuves rejouées, et s'arrêter là.

## Gestion de session — CONTEXT WARNING

Émettre `CONTEXT WARNING` et arrêter l'exploration dès que :

- l'analyse dépasse ~10 fichiers lus ou relit des fichiers déjà vus ;
- l'historique de session contient plusieurs tâches indépendantes ;
- une phase s'achève avec un gros volume de sorties accumulées.

Alors : résumer l'état en quelques lignes, écrire/mettre à jour le
handoff, puis recommander **/compact** (même tâche qui continue) ou une
**nouvelle session** (RFC terminée, nouvelle RFC indépendante, ou
historique multi-tâches). Toujours recommander une nouvelle session
après un handoff final.
