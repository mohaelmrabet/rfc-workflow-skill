---
name: rfc-workflow
description: Workflow complet et économe en tokens pour travailler sur une RFC du projet — analyse ciblée, scope, plan, implémentation, tests, review, matrice de conformité, handoff, classement final, vulgarisation et reprise entre sessions. Ne réécrit jamais le corps du document RFC (ça c'est rfc-review) ; seule exception, le mode « vulgarise » qui insère en tête du document une section « En clair » expliquant la RFC en langage courant. Il classe en revanche la RFC sur laquelle il vient de travailler, dans implemented/ ou rejected/, et tient le registre des numéros — il IMPLÉMENTE ce que la RFC décide, avec un principe strict qualité/contexte consommé — pas de subagents par défaut, pas de scan global du repo, lectures chirurgicales, tests ciblés, handoff compact réutilisable dans une nouvelle session. Couvre aussi l'inventaire du dossier docs/rfc (mode "statut", ex-skill rfc-status, fusionné ici) — tri des RFC nouvellement arrivées, détection de celles livrées mais restées non classées, registre des numéros. Utiliser quand l'utilisateur demande d'analyser, implémenter, reviewer, continuer, finaliser ou rejeter une RFC, de faire le point sur docs/rfc, ou de rendre une RFC compréhensible en langage courant (ex. "Implémente RFC-0114", "Rejette RFC-0023", "Statut des RFC", "Vulgarise RFC-0029").
argument-hint: "[analyse|implémente|review|continue|finalise|rejette|statut|vulgarise] [RFC-XXXX]"
allowed-tools: [Read, Edit, Write, Bash, Grep, Glob]
---

# RFC Workflow

V1.1 — durci avec les enseignements de RFC-0114 : machine d'états,
preuve réelle obligatoire, classification des problèmes, pré-commit.

Cible : `$ARGUMENTS`. Si le mode ou le numéro de RFC manque, demander —
ne pas deviner, ne pas scanner `docs/` pour proposer une liste.

**Numéroter une nouvelle RFC** : lire `docs/rfc/REGISTRY.md`, prendre le
« prochain numéro libre », ne jamais réutiliser un numéro même si son
document a disparu. Ce fichier suffit — ne pas balayer `docs/rfc/` pour
le redécouvrir.

**Toute RFC s'ouvre par une section « En clair ».** C'est une convention
du dépôt, pas une option du mode « vulgarise » : dès que le skill touche
une RFC qui n'en a pas — une nouvelle arrivée, un document ancien —, il
l'écrit selon la phase 9, sans attendre qu'on le demande. Le coût est
nul : à ce moment-là, la RFC vient d'être lue en entier. Le seul cas où
on s'abstient est un document appartenant à une autre session (modifié
et non commité) : on le signale au lieu de l'éditer.

## Principe directeur

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

## Localiser une RFC

Un seul Glob : `docs/rfc/*/RFC-XXXX_*.md`. Ne rien lire d'autre.
Les conventions du repo (sens des sous-dossiers, suppression des
propositions implémentées, commandes de test) sont dans
`references/conventions.md` — le lire une seule fois par session.

## État persistant (reprise entre sessions)

Tout l'état vit dans `docs/.rfc-workflow/` :

- `RFC-XXXX-analysis.md` — produit de la phase d'analyse.
- `RFC-XXXX-handoff.md` — handoff compact (≤ ~1000 tokens), réécrit à
  chaque fin de séance de travail sur la RFC.

Ces fichiers sont supprimés avec la RFC quand elle est finalisée
(convention du repo : une proposition implémentée se supprime).

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
  pourquoi avec des chiffres**. Réservé à une hypothèse mesurée puis
  infirmée, jamais à un abandon par lassitude ou par manque de temps :
  sans mesure, l'état reste BLOCKED. Voir « Phase 7 ».
- **FILED** — document classé dans `docs/rfc/implemented/` ou
  `docs/rfc/rejected/`. Une RFC n'est finie que classée.

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
| "Continue RFC-XXXX" | 0, puis reprise à la phase indiquée par le handoff |
| "Finalise RFC-XXXX" | 3 → 4 → 5 → 6 → 7 |
| "Rejette RFC-XXXX" | prototype + mesure, puis 7 — jamais de rejet sans chiffres |
| "Statut des RFC" / "Fais le point sur docs/rfc" | 8 uniquement — aucune modification de code, aucun numéro de RFC requis |
| "Vulgarise RFC-XXXX" / "Explique-moi RFC-XXXX simplement" | 9 uniquement — aucune modification de code ; seule écriture autorisée : la section « En clair » en tête du document RFC |

## Phase 0 — Reprise ("Continue")

1. Lire **uniquement** `docs/.rfc-workflow/RFC-XXXX-handoff.md`, en
   premier. S'il existe : déterminer l'état courant (`## State`), puis
   ne lire que les fichiers strictement nécessaires au « Next Action »
   et reprendre exactement à ce checkpoint. **Jamais d'analyse globale,
   jamais de relecture massive** des fichiers listés dans « Already
   Analyzed » ; respecter « Do Not Repeat » à la lettre.
2. S'il n'existe pas : lire `RFC-XXXX-analysis.md` s'il existe, sinon
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
3. Écrire `docs/.rfc-workflow/RFC-XXXX-analysis.md` selon le template
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

Écrire `docs/.rfc-workflow/RFC-XXXX-handoff.md` selon le template
« HANDOFF » de `references/templates.md`. Compact (≤ ~1000 tokens) :
il doit suffire à reprendre dans une session vierge sans relire
l'historique. « Already Analyzed » et « Do Not Repeat » évitent à la
session suivante de repayer l'exploration.

Aussi générer un handoff (même partiel) chaque fois qu'une séance de
travail s'interrompt avant la fin.

## Phase 7 — Classement (DONE ou REJECTED)

Dernier geste du travail, jamais optionnel. `git mv` fichier par
fichier — d'autres sessions écrivent le même repo.

**Mettre à jour `docs/rfc/REGISTRY.md`** dans le même commit : ligne de
la RFC (statut, nouveau chemin) et « prochain numéro libre ». Le
registre est la source des numéros ; c'est faute de l'avoir consulté
qu'une proposition a été numérotée RFC-0022 alors que le numéro était
pris, le 2026-08-13.

**Implémentée** → `docs/rfc/implemented/`. Ajouter en tête du document :
statut, date, et les commits qui la livrent. Le handoff et l'analyse de
`docs/.rfc-workflow/` restent : ils portent les mesures et les pièges,
que le document de décision ne contient pas.

**Rejetée** → `docs/rfc/rejected/`, **un seul fichier** réunissant :

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

Le rejet appartient à l'utilisateur. Le rôle du skill est de **mesurer
avant de conclure** : prototyper contre le harnais d'évaluation existant,
donner à la proposition sa meilleure configuration, et rapporter les
chiffres — y compris ceux qui contredisent l'attente initiale.

## Phase 8 — Inventaire (mode statut)

Le seul mode qui balaie `docs/rfc/` au lieu de lire chirurgicalement.
Il ne touche jamais au code : **aucun `Edit`/`Write` sur `src`, `tests`
ou `config`** — uniquement des lectures, et des `git mv` de documents
après confirmation explicite de l'utilisateur.

1. **RFC à plat dans `docs/`** (`ls docs/RFC-*.md`) — jamais triées.
   Lire les ~10 premières lignes (`Status`, `Board note`, `Document
   class`), classer, `git mv` vers le sous-dossier correspondant. Un
   fichier modifié ou non commité par une autre session
   (`git status --short`) **ne se déplace jamais** : le signaler.
   Ces ~10 lignes disent aussi si le document ouvre sur `## En clair`.
   Ne pas le rédiger ici — cela demanderait de lire chaque RFC en entier,
   ce que ce mode s'interdit : les lister dans la sortie comme « à
   vulgariser », et le faire à la prochaine phase 1 qui les ouvrira, ou
   sur demande.
2. **`docs/rfc/proposals/`** — relire chacune : son statut a pu changer
   depuis le dernier passage. Une proposition livrée mais restée là est
   une phase 7 oubliée par la session qui l'a traitée : proposer le
   déplacement vers `implemented/`, ne pas le faire d'office.
3. **`docs/rfc/blocked/`** — lecture rapide : le blocage cité a-t-il été
   levé ? Si oui, vers `proposals/`.
4. **`implemented/`, `rejected/`, `living/`** — ignorés par défaut, sauf
   si `$ARGUMENTS` cible l'une d'elles.
4bis. **`docs/rfc/REGISTRY.md`** — vérifier qu'il reflète les
   déplacements constatés et que « prochain numéro libre » est juste.
   C'est le seul fichier qu'un auteur de RFC a besoin de lire pour
   numéroter.
5. **Ne jamais conclure « implémentée » sur la foi du document.** Si le
   `Status` ne cite pas de sha, chercher dans
   `git log --oneline --all -- docs/rfc/*/RFC-XXXX*` et dans les commits
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
