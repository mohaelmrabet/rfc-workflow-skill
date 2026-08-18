# rfc-workflow

Skill [Claude Code](https://claude.com/claude-code) qui pilote le cycle de vie
complet d'une RFC : analyse, implémentation, tests, review, matrice de
conformité, handoff, classement final.

Il **implémente ce que la RFC décide** ; il ne réécrit jamais le corps du
document. Seule exception, le mode « vulgarise », qui ajoute en tête du
document une section « En clair » — le reste du texte n'est pas touché. Son critère d'optimisation est **qualité du travail / contexte
consommé** : pas de subagents par défaut, pas de scan global du repo, lectures
chirurgicales, tests ciblés.

## Installation

Le skill est **générique** : un seul exemplaire, placé dans les skills
utilisateur, sert tous les projets.

```bash
git clone git@github.com:mohaelmrabet/rfc-workflow-skill.git \
  ~/.claude/skills/rfc-workflow
```

S'il est déjà versionné dans un dépôt projet, un lien symbolique évite d'en
tenir deux copies :

```bash
ln -s <dépôt>/.claude/skills/rfc-workflow ~/.claude/skills/rfc-workflow
```

Il se charge au démarrage de la session suivante, dans n'importe quel projet.
Pour le partager avec un autre CLI qui lit les skills au format Anthropic
(Gemini CLI, par exemple), pointer un second lien vers le même dossier :

```bash
ln -s ~/.claude/skills/rfc-workflow ~/.gemini/config/skills/rfc-workflow
```

## Configuration d'un projet

Le skill porte la méthode ; **chaque projet porte ses données**, dans un
`.rfc-workflow.yml` à sa racine. Le fichier est facultatif : sans lui, les
défauts s'appliquent et un projet neuf fonctionne tel quel.

```yaml
tracker: files          # files | github        (défaut: files)
paths:
  rfc: docs/rfc
  work: docs/.rfc-workflow
  debt: docs/DEBT.md
tests: "vendor/bin/phpunit tests/<chemin>"
```

`tracker: files` tient le suivi dans un registre versionné et des handoffs
locaux. `tracker: github` le tient dans les issues du dépôt — une issue par
RFC, un label par état, le handoff en commentaire, l'issue fermée au
classement — pilotées par la CLI `gh`. Un board GitHub Projects branché sur
le dépôt se range alors tout seul ; le skill ne l'écrit pas.

Sur les deux backends, les **documents** ne bougent pas : RFC, ADR et mesures
restent versionnés dans le dépôt. Un tracker porte un statut, pas un document
normatif qui doit se relire en diff à côté du code qu'il décide.

## Utilisation

Le skill se déclenche sur l'intention, ou explicitement via `/rfc-workflow`.

| Demande | Phases exécutées |
|---|---|
| « Analyse RFC-0114 » | 1 — aucune modification de code |
| « Implémente RFC-0114 » | 1 → 7 |
| « Review RFC-0114 » | 4 (+ 5 si critères vérifiables) |
| « Audite RFC-0114 » | 10 — la livraison annoncée existe-t-elle vraiment ? |
| « Continue RFC-0114 » | reprise au checkpoint du handoff |
| « Finalise RFC-0114 » | 3 → 7 |
| « Rejette RFC-0114 » | prototype + mesure, puis classement |
| « Statut des RFC » | 8 — inventaire des RFC du projet |
| « Vulgarise RFC-0114 » | 9 — section « En clair » en tête du document |

Toute RFC ouvre sur une section **« En clair »** : deux paragraphes en langage
courant, dix lignes au plus. Le skill l'écrit d'office dès qu'il ouvre une RFC
qui n'en a pas — une nouvelle proposition en repart vulgarisée sans qu'on le
demande. Le mode ci-dessus sert à la refaire.

Le mode ni le numéro ne se devinent : s'ils manquent, le skill demande.

## Machine d'états

```
ANALYSIS → IMPLEMENTATION → TESTING → READY_TO_COMMIT → DONE → FILED
                    ↘ BLOCKED        ↘ REJECTED
```

L'état vit dans le champ `## State` du handoff. `DONE` exige les cinq preuves :
critères vérifiés, tests ciblés verts, run réel vert quand la RFC en demande un,
diff final contrôlé, handoff à jour. Un seul point manquant maintient `TESTING`
ou `BLOCKED`.

`REJECTED` est réservé à une hypothèse **mesurée puis infirmée**. Sans chiffres,
l'état reste `BLOCKED` — un abandon n'est pas un rejet.

## Garde-fous

- **Preuve réelle** — un test mocké ne prouve pas le comportement réel. Quand la
  RFC exige un run e2e, `TARGETED_TESTS_PASS ≠ DONE`.
- **Classification des problèmes** — tout problème découvert est étiqueté avant
  d'agir : `RFC`, `ENVIRONMENT`, `DEPENDENCY`, `UNRELATED`. Les problèmes
  indépendants partent en Follow-up et n'élargissent jamais le périmètre.
- **Pré-commit** — jamais de commit automatique. Le skill inspecte le diff,
  signale les fichiers hors scope et demande confirmation.
- **CONTEXT WARNING** — au-delà d'une dizaine de fichiers lus, le skill s'arrête,
  écrit le handoff et recommande `/compact` ou une nouvelle session.

## État persistant

En `tracker: files`, tout vit dans le dossier `work` du projet cible
(`docs/.rfc-workflow` par défaut, non versionné) :

- `RFC-XXXX-analysis.md` — objectif, écart, scope, plan, risques.
- `RFC-XXXX-handoff.md` — ≤ ~1000 tokens, suffisant pour reprendre dans une
  session vierge. Ses sections « Already Analyzed » et « Do Not Repeat »
  empêchent la session suivante de repayer l'exploration.

## Structure

```
SKILL.md                    workflow, machine d'états, phases 0-10
references/trackers.md      les sept opérations de suivi, files | github
references/conventions.md   méthode : cycle de vie, typologie, skills voisins
references/templates.md     analyse, handoff, matrice de conformité
```

Aucun de ces fichiers ne nomme un projet. Le suivi d'état ne se touche que par
`next_number`, `read_status`, `set_status`, `list_all`, `save_handoff`,
`read_handoff` et `add_debt` — sept opérations traduites une fois pour toutes
dans `references/trackers.md`. Porter le skill sur un autre projet ne demande
donc rien d'autre que d'y déposer un `.rfc-workflow.yml`, quand les défauts ne
suffisent pas.
