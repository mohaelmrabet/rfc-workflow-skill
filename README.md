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

```bash
git clone git@github.com:mohaelmrabet/rfc-workflow-skill.git \
  ~/.claude/skills/rfc-workflow
```

Le skill se charge au démarrage de la session suivante. Pour le partager avec
un autre CLI qui lit les skills au format Anthropic (Gemini CLI, par exemple),
pointer un lien symbolique vers le même dossier :

```bash
ln -s ~/.claude/skills/rfc-workflow ~/.gemini/config/skills/rfc-workflow
```

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
| « Statut des RFC » | 8 — inventaire de `docs/rfc/` |
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

Tout vit dans `docs/.rfc-workflow/` du projet cible :

- `RFC-XXXX-analysis.md` — objectif, écart, scope, plan, risques.
- `RFC-XXXX-handoff.md` — ≤ ~1000 tokens, suffisant pour reprendre dans une
  session vierge. Ses sections « Already Analyzed » et « Do Not Repeat »
  empêchent la session suivante de repayer l'exploration.

## Structure

```
SKILL.md                    workflow, machine d'états, phases 0-9
references/conventions.md   conventions du repo cible (RFC, tests, git)
references/templates.md     analyse, handoff, matrice de conformité
```

`references/conventions.md` décrit le dépôt sur lequel le skill travaille —
emplacement des RFC, commandes de test, règles de commit. C'est le fichier à
adapter pour porter le skill sur un autre projet.
