# Conventions de méthode de rfc-workflow

À lire une seule fois par session. Ne pas redécouvrir ces faits par
exploration.

Ce fichier ne contient que de la **méthode**, valable sur n'importe quel
projet. Les chemins, la commande de test et l'emplacement du suivi
appartiennent à `.rfc-workflow.yml` du projet ; leur traduction en
opérations est dans `trackers.md`. Rien de spécifique à un dépôt n'a sa
place ici — c'est ce qui rendait ce skill inutilisable ailleurs.

## Emplacement et cycle de vie des RFC

Une RFC finit toujours classée dans l'un de ces dossiers — c'est le
dernier geste du travail, pas un rangement optionnel :

- `<rfc>/proposed/` — propositions à implémenter, tant qu'elles le sont.
- `<rfc>/implemented/` — **implémentée et prouvée**. Le document y est
  déplacé (`git mv`), il n'est plus supprimé : le repo garde la décision
  et ses critères d'acceptation lisibles sans fouiller git.
- `<rfc>/rejected/` — **proposition écartée**, avec dans le même
  fichier la raison qui l'écarte. Une RFC rejetée sur mesure vaut d'être
  conservée : elle empêche de reproposer la même piste sans savoir ce
  qu'elle a coûté. Voir `RFC-0023-tripartite-measurement.md`, qui réunit
  la proposition, le prototype, les chiffres et les enseignements.
- `<rfc>/living/` — documents vivants/postmortems : restent.
- `<rfc>/future/` — **sans chemin d'implémentation**, en attente d'un
  déclencheur nommé. Différent de `blocked/` : rien n'empêche de l'écrire,
  c'est le besoin qui n'existe pas encore. Une telle RFC porte une section
  listant les déclencheurs vérifiables qui la rouvrent, et un critère de
  sortie. Ne jamais l'implémenter « tant qu'on y est » : la laisser dormir
  est le résultat attendu (Architecture Restraint, règle 2). Modèle dans
  Concio : `RFC-0071_attachment_content_scanning.md`.

Ces cinq dossiers sont les seuls. Il n'y a ni `proposals/`, ni
`blocked/`, ni RFC à plat : balayer un dossier inexistant donne un mode
statut qui ne trouve jamais rien et qui rassure à tort.

Les dossiers absents se créent au moment du classement. Le déplacement
se fait fichier par fichier (`git mv`), jamais par répertoire : d'autres
sessions écrivent le même repo.
- Nommage : `RFC-XXXX_slug_descriptif.md`. Format interne libre — pas de
  template imposé, extraire les sections où qu'elles soient.
- **Une seule section imposée** : toute RFC ouvre sur `## En clair`,
  juste après le titre. Une RFC qui arrive sans en repart avec.
- `docs/.rfc-review/` — ledgers du skill rfc-review (pipeline
  d'amélioration du document). Distinct de `<work>` qui appartient à ce
  skill. Ne pas les mélanger.

## Reconnaître la catégorie d'un document

Les signaux sont dans les ~10 premières lignes (`Status`, `Board note`,
`Document class`) — il suffit de les lire pour classer, sans lire le
document entier :

- **Document vivant** — le `Board note` dit explicitement qu'il reste
  en place même une fois une partie livrée : c'est un contrat ou un
  registre de triggers, pas une proposition à sens unique. → `living/`
- **Postmortem permanent** — une ligne `Document class` dit que le
  document vaut comme archive indépendamment du code. → `living/`
- **Proposition classique** — aucun de ces signaux. C'est la seule
  catégorie qui transite par `proposed/` puis `implemented/` ou
  `rejected/`.

Cette typologie vient de l'ex-skill `rfc-status`. Sa règle d'origine —
supprimer une proposition une fois implémentée — est caduque depuis
l'introduction de `implemented/` : on déplace, on ne supprime plus.

## Skills voisins (ne pas dupliquer leur rôle)

- `rfc-review` — améliore le DOCUMENT RFC (multi-agents, coûteux,
  uniquement sur demande explicite de l'utilisateur).
- `rfc-workflow` (ce skill) — implémente la RFC dans le code, la classe
  en fin de travail, et tient l'inventaire du dossier (mode « statut »,
  ex-skill `rfc-status`, fusionné ici le 2026-08-13). Il ne touche au
  document que pour y insérer la section « En clair » (mode
  « vulgarise ») ; toute autre réécriture appartient à `rfc-review`.

## Ce que ce fichier ne dit pas

Les conventions de code, de tests et de Git appartiennent au projet, pas
au skill : elles sont dans son `agent.md` / `CLAUDE.md`, et la commande
de test ciblé dans `.rfc-workflow.yml`. Un skill qui les recopie devient
faux dès le deuxième projet.
