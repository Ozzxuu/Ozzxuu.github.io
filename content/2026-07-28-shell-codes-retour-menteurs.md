---
title: "Ce qu'un code de retour ne dit pas"
slug: codes-de-retour-menteurs
description: "Quatre façons pour une commande shell de mentir sur son propre succès — dont deux qui inversent le sens."
tags: [shell, unix, débogage]
draft: true   # mon brouillon d'intro — à réécrire, ne part pas en ligne tant que c'est true
---

En pleine nuit, occupé à réinstaller un téléphone, j'écris un script qui inventorie les
paquets installés. Il m'annonce que 50 des 54 paquets cherchés sont absents.

Ils étaient tous là.

Le script n'avait pas de bug de logique. Il lisait correctement une réponse — seulement,
ce n'était pas la réponse à la question que je croyais poser. Un code de retour ne dit
jamais « as-tu trouvé quelque chose ? ». Il dit « l'appel s'est-il déroulé normalement ? ».

Les deux questions ont l'air voisines. En une nuit, l'écart entre elles m'a menti de quatre
façons différentes, dont deux qui inversent complètement le sens.

---

## 1. `|` contre `||` — le même piège, six jours après

La confusion : `cat fichier || grep motif`, écrit en croyant enchaîner les données.

**La preuve, produite sans le vouloir**, en changeant le motif pour du charabia :

```bash
cat .zshrc || grep "source"     # affiche le fichier
cat .zshrc || grep "sourceee"   # affiche le fichier
cat .zshrc || grep "sgeee"      # affiche le fichier
```

Le motif n'avait **aucune influence** — ce n'était pas « grep ne trouve rien », c'était **grep ne s'exécute jamais**. `cat` réussit, donc la branche « sinon » n'est jamais prise.

Et l'inverse le confirme :

```bash
cat .zshrc && grep "sgeee"      # affiche le fichier, puis se fige → ^C
```

Là grep **s'est** lancé — sans argument de fichier. Un filtre Unix sans fichier lit **stdin**, donc il attend une saisie. Le blocage est la démonstration que grep n'avait aucune donnée en entrée.

| Opérateur | Ce qu'il transporte | Équivalent bas niveau |
|---|---|---|
| `\|` | **les données** — stdout de gauche → stdin de droite | `call a` → passer le pointeur du buffer → `call b` |
| `\|\|` `&&` | **le contrôle** — la droite s'exécute ou non selon le code de retour | `call a` ; `test eax, eax` ; `jz` / `jnz` |

Deux mécanismes sans rapport : l'un déplace de la mémoire, l'autre déplace le pointeur d'instruction.

> ### ⚠️ Ce que ça dit sur la méthode
> Cette confusion était **déjà** la correction n°1 du [socle des commandes](2026-07-21-commandes-linux-socle.md) du 2026-07-21, avec la **même** analogie `test`/`jz`. Six jours plus tard, elle est revenue dans un vrai script.
> Ce n'est pas un échec : c'est la démonstration nette que **lire une correction ne crée pas de fluence**. Le socle a servi d'index — il a fallu se planter en conditions réelles, avec un script qui ne marche pas, pour que le mécanisme s'ancre. C'est l'argument pour continuer à apprendre sur de vrais projets plutôt que par relecture.

---

## 2. awk : `MOTIF { ACTION }`, et le mot nu qui est une variable

### La structure

```
awk 'MOTIF { ACTION }'
```

- **hors accolades** = motif, il **sélectionne** des lignes. Action par défaut si omise : afficher la ligne.
- **dans les accolades** = action, elle **fait** quelque chose. Motif par défaut si omis : toutes les lignes.

Toutes les erreurs de syntaxe rencontrées venaient de cette frontière :

| Écrit | Pourquoi ça casse |
|---|---|
| `awk 'print $2'` | `print` est une **action** posée en position de motif → erreur de syntaxe |
| `awk '(print $2)'` | idem, les parenthèses n'y changent rien |
| `awk '{$2 == device}'` | **comparaison en position d'action** : évaluée, résultat jeté, rien affiché |

### Le piège central : un mot nu est un identifiant

En awk, `device` (sans guillemets) n'est **pas** la chaîne « device » : c'est une **variable jamais initialisée**, donc vide.

La preuve, sur la sortie de `adb devices` (en-tête + une ligne appareil + une ligne vide finale) :

```bash
awk '$2 != device'    # affiche l'en-tête ET la ligne appareil, écarte la ligne vide
awk '$2 == device'    # n'affiche QUE la ligne vide
```

Le second semblait « ne rien sortir ». En réalité il sortait **exactement** la seule ligne dont le champ 2 valait la chaîne vide. La commande fonctionnait parfaitement — elle ne testait simplement pas ce qu'on croyait.

Il fallait `"device"`, avec des guillemets doubles.

### Deux idiomes à retenir

```bash
# compter, en forçant le contexte numérique
awk '$2 == "device" { n++ } END { print n+0 }'
```

`n+0` est obligatoire : si aucune ligne ne matche, `n` n'a jamais été touchée et vaut **la chaîne vide**. `print n` sortirait une ligne vide, et un `[ "$x" -eq 1 ]` derrière planterait sur une comparaison numérique avec du vide. En awk, « pas de valeur » n'est pas une erreur — c'est silencieusement vide ou zéro selon le contexte d'emploi.

```bash
# écarter explicitement en-tête et ligne vide
awk 'NR > 1 && NF > 1 { print $2 }'
```

`NR` = numéro de ligne courant, `NF` = nombre de champs. Écarter par une **condition explicite** plutôt que par coïncidence : dans le cas réel, l'en-tête `List of devices attached` donnait `$2 = "of"`, qui ne matchait rien *par chance*. Une protection qui tient par coïncidence est une protection qu'on ne peut pas maintenir.

---

## 3. Les quatre mensonges d'un code de retour

Le cœur de la session. Rencontrés dans cet ordre, ils mentent chacun différemment.

### ① Succès sans résultat

```bash
adb devices; echo "status = $status"   # → toujours 0, appareil branché ou non
```

`adb devices` a **réussi** dans tous les cas : on lui a demandé de lister les transports, il a listé. Zéro transport est une liste valide, pas une erreur.

> **Analogie** : `EnumProcesses` rend `TRUE` avec un compteur de sortie à 0. Le code de retour informe sur **l'appel**, le résultat est dans **la sortie**. Confondre « l'appel a marché » et « l'appel a trouvé » est le classique.

**Conséquence** : ici il faut parser la sortie, pas tester `$?`.

### ② Échec silencieux, déguisé en « rien trouvé »

```bash
find /sdcard -name "*.kdbx" 2>/dev/null    # → rien
```

…alors que le fichier existait bel et bien. Le `find` de **toybox** (userland Android) **abandonne** à la première permission refusée au lieu de continuer, et le `2>/dev/null` masquait l'erreur. Le résultat vide n'était pas « absent », c'était « la commande a échoué ».

**Deux conclusions fausses ont été tirées de là** avant que la mesure ne les corrige.

> **Règle générale** : un filtre qui ne sort rien mérite toujours qu'on vérifie **pourquoi** il ne sort rien. Et `2>/dev/null` sur une commande d'inspection est un choix à peser : il enlève le bruit *et* le diagnostic.

### ③ Échec rapporté alors que ça a réussi — `grep -q` sous `pipefail`

Le plus vicieux, et une variante de ce qui avait été anticipé.

```bash
set -euo pipefail
if ! adb shell pm list packages | tr -d '\r' | grep -qx "package:$p"; then
    echo "ABSENT"      # ← branche prise MÊME quand le paquet est présent
fi
```

Le mécanisme : `grep -q` **sort dès la première correspondance**. Il ferme le tube, ce qui envoie `SIGPIPE` aux commandes en amont, qui se terminent en erreur. Avec `pipefail`, le statut du pipeline devient **non nul** — alors que grep a trouvé.

**Aggravé par une course** : SIGPIPE ne se déclenche que si l'amont écrit encore quand grep s'arrête. Les motifs situés **tard** dans le flux y échappaient. Sur 54 paquets, exactement **4** ont été correctement détectés — ceux dont l'entrée arrivait en fin de sortie. Un bug qui a l'air aléatoire et qui ne l'est pas.

**La parade** — et elle est meilleure sur tous les plans :

```bash
# une seule lecture, stockée, puis interrogée sans tube
INSTALLED="$(adb shell pm list packages | tr -d '\r')"
grep -qx "package:$p" <<< "$INSTALLED"
```

Un **here-string** (`<<<`) n'est pas un tube : pas de SIGPIPE possible. Bonus : 54 allers-retours ADB remplacés par un seul, exécution instantanée au lieu de six secondes.

### ④ Silence menteur — aucune erreur, aucun effet

```bash
pm revoke <paquet> android.permission.ACCESS_FINE_LOCATION   # → sortie vide
```

Sortie vide interprétée comme « accepté ». Vérification derrière :

```
ACCESS_FINE_LOCATION: granted=true, flags=[ SYSTEM_FIXED | GRANTED_BY_DEFAULT ]
```

`SYSTEM_FIXED` = permission verrouillée par le système, non révocable. La commande a été **acceptée**, n'a produit **aucun effet**, et n'a **rien signalé**. Cette fois ce n'est même plus un code de retour qui ment, c'est l'absence de sortie.

> **Règle** : ne jamais déduire un effet d'une absence d'erreur. Vérifier l'**état**, pas le retour.

---

## 4. Quoting : donnée ou référence ?

Même famille que le mot nu d'awk — le quoting décide si un token est une **valeur** ou une **référence**.

```bash
log '$ADB'    # journalise les 5 caractères "$ADB"
log "$ADB"    # journalise la valeur
```

Et le piège structurel :

```bash
echo $VAR     # découpe sur espaces ET retours ligne, puis recolle à l'espace
              # → toute la structure en lignes est détruite
echo "$VAR"   # structure préservée
```

Sur une variable qui contient la sortie multi-lignes d'une commande, l'oubli des guillemets réduit tout à une seule ligne — et le travail de séparation en lignes fait juste avant part à la poubelle.

Enfin, le test à un argument :

```bash
[ status ]     # toujours VRAI : teste si la CHAÎNE "status" est non vide (6 caractères)
[ "$status" ]  # teste si la VALEUR est non vide
```

---

## 5. À quoi une redirection se lie

```bash
if [ $(awk '…') = 'device' <<< "$DEVICES" ]; then      # ✗ ne marche pas
```

Deux raisons cumulées :

1. **Une redirection se lie à la commande**, ici `[`. Elle ne peut pas atteindre le `$( )` interne.
2. **Question d'ordre** : `$( )` s'exécute **avant** `[`. Le shell lance le sous-shell, attend sa fin, remplace `$( )` par son texte, *puis* construit `[` avec ses descripteurs.

Le here-string alimente donc un processus qui n'en a pas besoin, pour nourrir un processus **déjà terminé**. C'est préparer un argument pour un appel qui a déjà retourné. La redirection doit être **à l'intérieur** des parenthèses, collée à la commande qui lit.

Autre effet du même genre, dans une boucle :

```bash
while read -r pkg; do
    adb shell dumpsys package "$pkg" </dev/null   # ← sans ça, adb avale le flux
done < liste.txt                                   #   du while et la boucle saute des lignes
```

---

## 6. `rsync -a` réapplique les permissions de la source

Constaté après coup sur un transfert vers un serveur distant.

```bash
ssh <hôte> 'mkdir -p /mnt/backup/<dossier> && chmod 700 … && chown root:root …'
rsync -a ~/source/ <hôte>:/mnt/backup/<dossier>/
# → le dossier ressort en drwxr-xr-x, propriétaire uid 1000
```

`-a` est une abréviation qui **inclut `-o -g -p`** : rsync réapplique propriétaire, groupe et permissions de la **source** sur la destination, écrasant le `chmod` posé avant transfert. La donnée n'était pas exposée pour autant (le dossier parent était `drwx------ root`), mais la propriété de sécurité voulue avait disparu **en silence**.

```bash
# pour resynchroniser sans casser les permissions cibles
rsync -a --no-o --no-g --chmod=D700,F600 ~/source/ <hôte>:/dest/
```

> **Leçon** : le transfert réussit, les fichiers sont là, et la garantie qu'on croyait avoir n'existe plus. D'où la vérification **après**, jamais seulement avant.

---

## 7. L'auto-contrôle qui ne peut rien contrôler

Le script de debloat se terminait par :

```bash
# contrôle final : aucun paquet de la liste ne doit subsister
for p in "${PKGS[@]}"; do
    if adb shell pm list packages | grep -qx "package:$p"; then   # ← MÊME test cassé
        echo "SUBSISTE $p"
    fi
done
echo "Contrôle OK"     # ← toujours affiché
```

Le contrôle réutilisait **la primitive défaillante qu'il était censé vérifier**. Il ne pouvait donc structurellement rien détecter : il confirmait sa propre erreur et affichait « OK » sur un travail à moitié fait.

> **Principe** : un auto-contrôle doit vérifier par un **chemin différent** de celui qu'il contrôle. Sinon ce n'est pas un contrôle, c'est un écho.

Exemple correct utilisé le même soir — pour valider une copie, ne pas se fier au code de retour de `rsync`, mais **compter les fichiers des deux côtés** :

```bash
find ~/source -type f | wc -l
ssh <hôte> 'find /dest -type f | wc -l'
```

---

### Variante : la branche d'erreur que `set -e` rend inatteignable

Découverte le lendemain, dans le **même script**, et c'est la même famille.

```bash
set -euo pipefail
res="$(adb shell pm uninstall … "$p" 2>&1 | tr -d '\r')"
if [ "$res" = "Success" ]; then
    log "  RETIRE   $p"
else
    log "  ECHEC    $p -> $res"      # ← branche JAMAIS atteinte
fi
```

Quand la commande distante échoue vraiment, elle rend un code non nul, `pipefail` le remonte au pipeline, **l'affectation hérite de ce statut**, et `set -e` tue le script — *avant* d'arriver au `if`. La branche `ECHEC` est donc morte : au premier échec réel, le script s'interrompt **sans bilan ni contrôle final**, en ayant l'air d'avoir fini.

Le bug est resté invisible tant qu'aucun paquet ne résistait vraiment. Le jour où l'un a résisté, la sortie s'est simplement arrêtée au milieu — pas d'erreur, pas de message.

```bash
res="$(… | tr -d '\r' || true)"   # le || true rend la branche d'erreur atteignable
```

Une fois corrigé, le message caché est apparu immédiatement : `Failure [DELETE_FAILED_OWNER_BLOCKED]` — un diagnostic exploitable, perdu jusque-là.

> **Le point commun des deux cas** : du code écrit *pour* gérer les ennuis, et qui ne s'exécute pas quand les ennuis arrivent. Un `else`, un contrôle final, un `catch` — tout ce qui n'est emprunté qu'en cas de problème doit être **testé en provoquant le problème**, sinon on ne sait pas s'il existe.

---

## 8. Démarche : mesurer au lieu de supposer

La check-list qui a servi toute la nuit — chaque constat vient avec sa commande de vérification.

| Étape | Commande de vérification | Ce qu'on attend |
|---|---|---|
| Le code de retour est-il exploitable ? | `cmd; echo "status = $status"` (fish) / `$?` (bash) | s'il vaut 0 même sans résultat → parser la sortie |
| Quelle est la forme **exacte** de la sortie ? | `cmd \| cat -A` | `$` en fin de ligne, `^M` = CRLF, espaces visibles |
| Sur quel flux sort ce message ? | `cmd 2>/dev/null` | s'il disparaît → stderr ; sinon stdout |
| Ce filtre trouve-t-il vraiment 0 résultat ? | relancer **sans** `2>/dev/null` | une erreur cachée change tout le diagnostic |
| Le contenu de mon tableau est-il ce que je crois ? | `printf "[%s]\n" "${ARR[@]}"` | les crochets révèlent espaces et résidus |
| La commande a-t-elle eu un effet ? | relire **l'état**, pas le retour | ex. `dumpsys …` après un `pm revoke` |
| Le service est-il en cause, ou le client ? | rejouer la même opération avec un **autre outil** | ex. `secret-tool` vs crate Rust sur le même démon |
| Ma copie est-elle complète ? | compter les fichiers des **deux** côtés | l'égalité, pas le code de retour de `rsync` |

### Le diagnostic Secret Service, comme cas d'école

Symptôme : une application Tauri n'arrive pas à stocker sa clé dans le trousseau KDE.

1. Le service répond-il ? `busctl --user list | grep -i secret` → oui, `ksecretd` détient `org.freedesktop.secrets`.
2. La collection par défaut existe-t-elle ? `busctl --user call … ReadAlias s default` → oui.
3. Le trousseau est-il verrouillé ? `busctl --user get-property … Locked` → `false`.
4. L'entrée a-t-elle été écrite ? `secret-tool search service <app>` → **entrée créée, `secret = ` vide**.
5. Le démon est-il cassé, ou le client ? Même opération via **libsecret** :
   ```bash
   printf 'VALEUR_TEST\n' | secret-tool store --label=diag diag-test essai
   secret-tool lookup diag-test essai | cat -A     # → VALEUR_TEST$
   secret-tool clear diag-test essai
   ```
   Aller-retour **parfait**. Donc le service va bien : c'est le chemin client (crate Rust `keyring` 3.6.3, qui parle le protocole en direct) qui perd la charge utile.

**La méthode** : isoler la variable qui diffère. Deux écritures, même démon, même collection, à quelques minutes d'intervalle — l'une réussit, l'autre non. Ce qui change n'est ni le service ni la config, c'est la **bibliothèque cliente**.

> Note sur l'étape 5 : la première tentative, faite **interactivement**, avait donné un résultat ambigu — `secret-tool store` **masque la saisie**, donc un Entrée sur une saisie vide enregistre un secret vide, et le `lookup` renvoie alors légitimement une ligne vide. Rejouer en **injectant la valeur par un tube** a supprimé le facteur humain et tranché.

---

## 9. Les commandes vues (mémo)

| Commande | Rôle |
|---|---|
| `cmd \| cat -A` | Voir les **octets exacts** : `$` en fin de ligne, `^M` pour CRLF, `^I` pour tab |
| `tr -d '\r'` | Supprimer les CR — indispensable dès qu'on parse la sortie d'un autre système |
| `paste -sd' ' -` | Aplatir plusieurs lignes en une seule, séparées par un caractère choisi |
| `printf "[%s]\n" "${ARR[@]}"` | Inspecter un tableau bash élément par élément, délimiteurs visibles |
| `cmd <<< "$VAR"` | **Here-string** : alimenter stdin depuis une variable, sans tube ni SIGPIPE |
| `shred -u <fichier>` | Écraser puis supprimer (attention : inefficace sur SSD, cf. [lexique](lexique.md)) |
| `jq '.chemin \| length' f.json` | Interroger du JSON — ici compter les entrées d'une sauvegarde |
| `keytool -printcert -jarfile <apk>` | Empreinte du certificat signataire d'un APK (bloc de signature **v1** seulement) |
| `busctl --user list` | Lister les services D-Bus de session et qui détient quel nom |
| `busctl --user get-property <svc> <objet> <iface> <prop>` | Lire une propriété D-Bus |
| `busctl --user call <svc> <objet> <iface> <méthode> <sig> <args>` | Appeler une méthode D-Bus |
| `secret-tool store/lookup/search/clear` | Écrire/lire/chercher/effacer dans le trousseau Secret Service |
| `usermod -aG <groupe> $USER` | Ajouter au groupe (**`-a`** obligatoire, sinon on remplace tous les groupes) |
| `pacman -Ss '^paquet$'` | Chercher un paquet exactement nommé dans les dépôts |
| `pacman -Fl <paquet>` | Fichiers fournis par un paquet **non installé** (nécessite `pacman -Fy`) |
| `du -s <chemins>/* \| sort -rn \| head` | Classer par taille — `sort -h` n'existe pas partout (toybox) |
| `rsync -a --no-o --no-g --chmod=D700,F600` | Copier **sans** réappliquer propriétaire/groupe/permissions de la source |

---

## 10. Récap des analogies bas niveau

| Concept shell | Équivalent bas niveau |
|---|---|
| `\|` (tube) | Passage d'un pointeur de buffer entre deux `call` |
| `\|\|` / `&&` | `test eax, eax` + saut conditionnel (`jz`/`jnz`) |
| `adb devices` exit 0 sans résultat | `EnumProcesses` → `TRUE` avec compteur à 0 |
| `grep -q` + `pipefail` | Un flag d'erreur levé par le nettoyage, pas par l'opération |
| `pm revoke` sur `SYSTEM_FIXED` | `WriteFile` qui rend `TRUE` avec `bytesWritten = 0` |
| `$( )` évalué avant la commande englobante | Argument préparé pour un appel déjà retourné |
| Mot nu en awk | Symbole non résolu qui vaut 0 au lieu de lever une erreur |

---

## 11. Ce qui a été corrigé (le plus instructif)

1. **`||` employé comme pipe** — déjà corrigé au socle du 21/07, revenu en conditions réelles le 27/07. La lecture indexe, seul l'usage rend fluent. (cf. encadré §1)
2. **Deux conclusions fausses tirées d'un `find` vide** — « pas de base KeePass sur le stockage partagé », « aucune sauvegarde Aegis existante ». Les deux étaient fausses : `find` avait **échoué**, pas cherché. Corrigées en refaisant la recherche en contournant l'arborescence protégée. Un résultat vide n'est pas une information tant qu'on n'a pas vérifié que la commande a abouti.
3. **Un script « qui marche » et qui ne retire que 4 paquets sur 54** — diagnostic mené jusqu'à `grep -q` + `pipefail` + SIGPIPE, avec l'explication de la répartition non aléatoire des 4 succès. Le piège avait été **anticipé à voix haute quelques heures plus tôt** dans sa forme simple (grep qui sort 1 quand il ne trouve rien) — c'est la forme inversée, où il « échoue » **précisément quand il réussit**, qui n'avait pas été vue.
4. **Un `chmod 700` annulé par le `rsync` qui suivait** — trouvé seulement parce qu'une vérification a été faite *après* le transfert, alors que tout semblait s'être bien passé.
5. **Un contrôle final qui validait un travail à moitié fait** — parce qu'il réutilisait la primitive cassée. Reformulé pour vérifier par un autre chemin.
5 bis. **Une branche `ECHEC` inatteignable**, tuée par `set -e` avant d'être atteinte (cf. §7). Trouvée seulement le lendemain, quand un paquet a enfin résisté — et le message d'erreur qu'elle cachait s'est révélé être le diagnostic le plus utile de la journée.
6. **Une hypothèse de diagnostic abandonnée après mesure** — « l'app n'a pas l'accès aux notifications » : vérification faite, l'application ne déclare aucun service d'écoute de notifications. Hypothèse plausible, fausse, écartée en une commande plutôt qu'en une heure de manipulation.

> **Fil rouge de la session** : à chaque fois que quelque chose a coincé, la sortie de secours a été la même — **mesurer l'état réel** plutôt que raisonner sur ce que la commande *devrait* faire.

---

## Sources
- `man` locaux (`awk`, `rsync`, `grep`, `secret-tool`, `busctl`) — décrivent les versions exactement installées.
- Vérifications faites en direct sur la machine et sur l'appareil, aucune conclusion tirée de mémoire.
