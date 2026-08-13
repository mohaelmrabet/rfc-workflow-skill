# Templates rfc-workflow

## RFC Analysis — `docs/.rfc-workflow/RFC-XXXX-analysis.md`

```markdown
# RFC Analysis — RFC-XXXX <titre court>

## Objective
<1-3 phrases : ce que la RFC veut obtenir>

## Current State
<état actuel du code concerné, avec fichier:ligne>

## Expected State
<état visé selon les décisions de la RFC>

## Gap
<différence concrète entre les deux, point par point>

## Scope
<liste fermée des fichiers à modifier, avec la raison, 1 ligne chacun>

## Out of Scope
<ce que la RFC ne demande PAS, pour s'en protéger explicitement>

## Dependencies
<RFC liées, services, migrations, config requis>

## Risks
<risques concrets de l'implémentation, 1 ligne chacun>

## Implementation Plan
<étapes numérotées, petites et vérifiables, avec le test associé>

## Follow-up
<découvertes hors périmètre, à traiter ailleurs — jamais ici>
```

## HANDOFF — `docs/.rfc-workflow/RFC-XXXX-handoff.md`

Compact : ≤ ~1000 tokens. Doit permettre de reprendre dans une session
vierge sans relire l'historique ni refaire l'exploration.

Sections minimales obligatoires : State, Objective, Implemented,
Evidence, Blocking Issue, Remaining Issues, Next Action, Files Changed,
Already Analyzed, Do Not Repeat. Decisions et Follow-up si pertinents.

```markdown
# HANDOFF — RFC-XXXX

## State
<ANALYSIS | IMPLEMENTATION | TESTING | BLOCKED | READY_TO_COMMIT | DONE>

## Objective
<1 phrase>

## Implemented
<ce qui est fait, 1 ligne par élément>

## Evidence
<preuves concrètes : commande exacte → résultat, pour les tests ciblés
ET pour le run réel s'il est requis ; distinguer mocké vs réel>

## Blocking Issue
<si State=BLOCKED : cause précise, classification
(RFC/ENVIRONMENT/DEPENDENCY/UNRELATED), tentatives faites, condition de
déblocage. Sinon : « aucun »>

## Remaining Issues
<P0/P1 ouverts, échecs de tests non résolus>

## Next Action
<LA première action de la prochaine session — commande exacte, fichiers
exacts : exécutable sans aucune nouvelle exploration>

## Files Changed
<chemin — raison, 1 ligne chacun ; préciser commité ou non>

## Decisions
<choix non évidents pris pendant l'implémentation + pourquoi>

## Follow-up
<hors périmètre consigné (P2/P3, dette découverte, classification)>

## Already Analyzed
<fichiers/zones déjà compris, avec la conclusion en 1 ligne>

## Do Not Repeat
<explorations/tests déjà faits qu'il est inutile de refaire, pièges
connus (mauvais entrypoint, conteneur inadapté…)>
```

## Matrice de conformité

Statuts : `PASS` · `PARTIAL` · `FAIL` · `NOT_TESTED`.
La preuve est concrète : commande + résultat, ou fichier:ligne.

```markdown
| Critère | Statut | Preuve |
|---------|--------|--------|
| <critère d'acceptation de la RFC> | PASS | <commande → sortie, ou fichier:ligne> |
```
