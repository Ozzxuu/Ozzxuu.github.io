# Mémo d'écriture

Six tics relevés dans les fiches d'origine. Ils sont excellents en prise de notes et coûteux
en lecture continue — c'est la seule différence entre les deux exercices.

Ce fichier n'est pas publié (il n'est pas dans `content/`).

## 1. Un gras par paragraphe, maximum

Le gras désigne le mot qu'on retiendrait si on ne retenait qu'un mot. Huit gras dans un
paragraphe, c'est zéro gras.

## 2. Pas de gras en début de paragraphe comme faux titre

`**La parade** — …`, `**Aggravé par une course** : …` : c'est une béquille de notes.
Une transition écrite fait le même travail. « Le plus déroutant, c'est que… »

## 3. Le tiret cadratin, une fois par paragraphe

Il est très bon pour la chute d'une phrase. À trois par phrase, il empile des propositions
au lieu de les articuler. Les autres deviennent des points, des virgules ou des points-virgules.

## 4. La voix active, au passé

« Sortie vide interprétée comme "accepté" » → « J'ai lu cette sortie vide comme un accord. »
Tu étais là quand ça s'est produit ; l'écrire au passif transforme un vécu en rapport d'incident.
C'est le plus gros levier des six.

## 5. Ouvrir sur la scène, pas sur la thèse

Les fiches ouvrent par `## 0. L'essentiel en une phrase`. C'est parfait pour se relire,
mauvais pour accrocher : le lecteur ne sait pas encore pourquoi la thèse compte.
Ordre qui marche : le symptôme concret → ce qu'on croyait → la thèse.
Garde la phrase de synthèse, déplace-la trois paragraphes plus bas.

## 6. Couper les méta-références au processus

« Fiche d'apprentissage », « cette session », « ce qui a été corrigé », « Remplace-les
mentalement par les tiens » : le lecteur n'a pas ce contexte. Ce qui reste utile, c'est
l'erreur elle-même — raconte-la comme une erreur, pas comme une ligne de bilan.

---

## Ce qu'il ne faut surtout pas lisser

Les analogies bas niveau. `test`/`jz` pour `||`, l'IAT pour une table de dispatch, le
brownout MCU, `free()` qui empoisonne. C'est ce qui distingue ces textes de n'importe quel
tutoriel et c'est ce qui rend le mécanisme mémorable.

Les erreurs, aussi. La correction n°1 du socle revenue six jours plus tard dans un vrai
script — un article qui montre ça vaut trois articles qui montrent une solution propre.

---

## Mécanique

Nouvel article :

    hugo new content 2026-08-13-mon-sujet.md

Le nom de fichier porte la date (elle n'est pas dans le front matter). Le champ `slug`
donne l'URL publique : `slug: codes-de-retour-menteurs` → `/codes-de-retour-menteurs/`.
`draft: true` tant que ce n'est pas prêt — visible en local, jamais publié.

Les liens entre articles se font sur le **nom de fichier** (`[…](2026-07-18-snapper-btrfs.md)`),
comme dans docs-perso. Le render hook les traduit en URLs. S'il cite un article non publié,
le build affiche `LIEN MORT` — c'est une alerte, pas une erreur.
