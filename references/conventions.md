# Conventions du repo utiles à rfc-workflow

À lire une seule fois par session. Ne pas redécouvrir ces faits par
exploration.

## Emplacement et cycle de vie des RFC

Une RFC finit toujours classée dans l'un de ces dossiers — c'est le
dernier geste du travail, pas un rangement optionnel :

- `docs/rfc/proposals/` — propositions à implémenter, tant qu'elles le sont.
- `docs/rfc/implemented/` — **implémentée et prouvée**. Le document y est
  déplacé (`git mv`), il n'est plus supprimé : le repo garde la décision
  et ses critères d'acceptation lisibles sans fouiller git.
- `docs/rfc/rejected/` — **proposition écartée**, avec dans le même
  fichier la raison qui l'écarte. Une RFC rejetée sur mesure vaut d'être
  conservée : elle empêche de reproposer la même piste sans savoir ce
  qu'elle a coûté. Voir `RFC-0023-tripartite-measurement.md`, qui réunit
  la proposition, le prototype, les chiffres et les enseignements.
- `docs/rfc/living/` — documents vivants/postmortems : restent.
- `docs/rfc/blocked/` — bloquées : ne pas implémenter sans déblocage.

Les dossiers absents se créent au moment du classement. Le déplacement
se fait fichier par fichier (`git mv`), jamais par répertoire : d'autres
sessions écrivent le même repo.
- Nommage : `RFC-XXXX_slug_descriptif.md`. Format interne libre — pas de
  template imposé, extraire les sections où qu'elles soient.
- `docs/.rfc-review/` — ledgers du skill rfc-review (pipeline
  d'amélioration du document). Distinct de `docs/.rfc-workflow/` qui
  appartient à ce skill. Ne pas les mélanger.

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
  catégorie qui transite par `proposals/` puis `implemented/` ou
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

## Code et tests

- Backend : `php-backend/` (src/, tests/, config/capabilities/).
- Tests : depuis `php-backend/`, `vendor/bin/phpunit tests/<chemin>` —
  toujours cibler un fichier ou un répertoire, jamais la suite entière
  par défaut.
- Vérifier la contrainte PHP de `php-backend/composer.json` avant toute
  syntaxe dépendante de version.
- Comments/docblocks : anglais, 1-4 lignes max (cf. CLAUDE.md) ; le
  narratif détaillé va dans le message de commit.
- Certains fichiers (ex. config/capabilities côté runtime, vendor créés
  par Docker) peuvent appartenir à root — si un write échoue en
  permission denied, passer par `docker exec`/`docker cp` plutôt que
  sudo.

## Git

- Branche principale : `main`. Commits en anglais, format
  conventionnel (`feat(...)`, `fix(...)`), sans trailer Co-Authored-By.
- D'autres sessions peuvent avoir des modifications non commitées :
  vérifier `git status` avant d'attribuer un diff à la RFC courante.
