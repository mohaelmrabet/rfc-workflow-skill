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

## En clair — inséré en tête du document RFC (phase 9)

Placement : juste après le titre `#` de la RFC, avant le statut et le
corps. **10 lignes de texte maximum**, une par entrée, lignes vides non
comptées. Si la section existe déjà, la remplacer entièrement — jamais
deux sections « En clair » dans un document.

**De la prose, pas un formulaire.** Pas de titres, pas de puces, pas
d'étiquettes en gras : deux courts paragraphes qu'on lit d'une traite.

```markdown
## En clair

RFC-XXXX consiste à <ce qu'elle fait, en mots de tous les jours>, mais
surtout à <le vrai enjeu, celui qu'on ne voit pas dans le titre>.

Et <ce que ça vaut dans l'ensemble> : <RFC voisine> <ce qu'elle apporte> ;
RFC-XXXX <ce qu'elle ajoute par-dessus>.
```

Le premier paragraphe est **une seule phrase**, en deux temps autour du
pivot « mais surtout » : la première moitié dit la fonction, la seconde
dit la vraie raison d'être. C'est le cœur de l'exercice — si cette phrase
demande deux phrases, c'est qu'on n'a pas encore trouvé l'enjeu. Le second
paragraphe situe la RFC — parmi ses voisines si elle est vivante, dans le
temps du projet si c'est une archive (`rejected/`, hypothèse falsifiée,
document conservé pour ce qu'il interdit de reproposer). C'est ce qui rend
une brique compréhensible, pas sa description isolée.

Modèle de ton (vulgarisation de RFC-0028) :

> RFC-0028 consiste à brancher le vrai cerveau sémantique de Concio, mais
> surtout à faire en sorte que Concio sache dire « je n'ai pas ce cerveau
> disponible » plutôt que de faire semblant de l'avoir.
>
> Et à mon avis, c'est une très bonne évolution de l'architecture :
> RFC-0027 apprend à Concio à s'abstenir ; RFC-0028 lui donne enfin un
> étage fiable auquel déléguer ces abstentions.

Un terme technique inévitable se traduit dans la phrase même, entre
tirets. Jamais de glossaire : un glossaire, c'est déjà trop long.

## Matrice de conformité

Statuts : `PASS` · `PARTIAL` · `FAIL` · `NOT_TESTED`.
La preuve est concrète : commande + résultat, ou fichier:ligne.

```markdown
| Critère | Statut | Preuve |
|---------|--------|--------|
| <critère d'acceptation de la RFC> | PASS | <commande → sortie, ou fichier:ligne> |
```
