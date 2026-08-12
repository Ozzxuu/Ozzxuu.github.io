---
title: "Lexique"
slug: lexique
unlisted: true
draft: true   # 184 entrées qui citent des articles pas encore publiés → à publier plus tard
description: "Les termes qui reviennent d'un article à l'autre, définis une seule fois."
---

Les articles s'appuient sur un fond commun de vocabulaire — systèmes de fichiers, noyau,
formats binaires, réseau. Plutôt que de le redéfinir à chaque fois, tout est ici.
Utilisez la recherche du navigateur (`Ctrl+F`) pour retrouver un terme.

---

## ABI (Application Binary Interface)
Le contrat **binaire** entre des morceaux de code compilés séparément : conventions d'appel (quels registres portent les arguments, qui sauvegarde quoi), disposition des structures en mémoire, noms et versions des symboles exportés. À distinguer de l'**API**, qui est le contrat au niveau du *source* — deux bibliothèques peuvent offrir la même API et des ABI incompatibles. Rupture d'API : il faut modifier le code source ; rupture d'**ABI** : le code source va très bien, mais le binaire déjà compilé ne fonctionne plus, il faut le relinker. Une lib qui « garde son ABI » peut donc être remplacée sous un programme déjà compilé sans y toucher — c'est toute la promesse de `libc.so.6`. Voir [[versioning de symboles]] pour la mécanique qui rend ça possible.

## ADB (Android Debug Bridge)
Protocole et outil de dialogue entre un poste et un appareil Android, via USB ou TCP/TLS. Trois composants : le **client** (`adb` sur le poste), le **serveur** (démon local qui multiplexe, port 5037) et le démon **`adbd`** sur l'appareil. Donne un shell non privilégié (uid `shell`) qui possède néanmoins des droits que l'utilisateur de l'appareil n'a pas : lister et désinstaller des paquets **par utilisateur**, écrire des réglages système, geler une application. C'est le levier principal du « debloat sans root ». Depuis Android 11, une variante **sans fil** existe, authentifiée par un appairage TLS (voir mDNS / DNS-SD). Le userland qu'on y trouve est du **toybox**, pas du GNU coreutils.

## Adresse d'écoute (bind address)
L'adresse locale à laquelle un service attache sa socket. `0.0.0.0` signifie « toutes les interfaces », une adresse précise restreint l'écoute à celle-là, et la boucle locale (`127.0.0.0/8`) n'est joignable que depuis la machine. Deux pièges vus en pratique : un même service peut lier ses sockets **TCP et UDP sur des adresses différentes** (donc vérifier les deux séparément, `ss -tlnp` puis `ss -ulnp`), et lier `0.0.0.0` peut empêcher le programme de connaître sa propre adresse, ce qui casse toute logique applicative qui en dépend. Sur les distributions où le hostname résout vers `127.0.1.1` via `/etc/hosts`, un service peut aussi écouter là sans qu'on s'en doute, hors de portée de `localhost`.

## AEAD (Authenticated Encryption with Associated Data)
Mode de chiffrement qui produit, en une opération, le texte chiffré **et** un tag d'authentification : le déchiffrement échoue si un seul octet a été altéré. Évite la faute classique du chiffrement seul, où un attaquant peut modifier le message sans que le destinataire s'en aperçoive. Représentants courants : **AES-GCM** et **ChaCha20-Poly1305**. En analyse de malware, voir un AEAD importé signale que la configuration (souvent l'adresse du C2) est chiffrée et vérifiée, donc qu'elle ne traînera pas en clair dans le binaire.

## Affinité CPU (CPU affinity)
Le masque de cœurs sur lesquels l'ordonnanceur est autorisé à placer une tâche. Par défaut tous, ce qui laisse le noyau migrer librement — au prix de la localité de cache. `taskset -c <liste> <cmd>` au lancement, `taskset -pc <liste> <pid>` sur un processus vivant ; ne demande aucun privilège pour ses propres processus. Utile quand une tâche a une échéance régulière (émettre une trame toutes les N ms) ou sur une [[P-core / E-core (architecture hybride)]] où les cœurs ne se valent pas. Le compteur `nr_migrations` de `/proc/<pid>/sched` mesure le problème.

## Agrégation de liens (bonding / LACP / MPTCP)

Faire passer du trafic sur **plusieurs interfaces réseau** à la fois. Le **bonding** Linux (et le **LACP**/802.3ad côté switch) regroupe des liens **ethernet identiques** : répartition **par flux** (chaque connexion suit un lien) ou redondance — il ne **somme pas** le débit d'un *seul* flux, et gère mal le WiFi (au mieux `active-backup` = failover). Agréger *une* connexion sur des liens hétérogènes exige **MPTCP** (Multipath TCP), qui éclate un flux TCP sur plusieurs chemins — mais les **deux bouts** doivent le supporter (ou un proxy dédié, ex. OpenMPTCProuter). Et rien n'aide si les interfaces partagent le **même uplink WAN** : le goulot est en amont, pas sur le LAN.

## API socket (Winsock)
API réseau bas niveau (POSIX `sys/socket.h`, ou **Winsock**/`ws2_32.dll` sous Windows). Le *choix* des fonctions trahit l'intention : `socket`/`bind`/`listen`/`accept` = **serveur** (écoute les entrants) ; `connect`/`send`/`recv` = **client** ; `sendto`/`recvfrom` = **datagramme UDP** sans connexion (broadcast, découverte LAN) ; `getaddrinfo` = résolution nom→IP. En triage : un exfiltrateur est *client-only* (`connect`+`send` vers un **C2**) ; un pair P2P écoute ET se connecte. Voir la présence de sockets ne dit pas la direction — seul le choix des appels + les endpoints en dur la disent.

## Aplatissement de flot de contrôle (control-flow flattening)
Transformation d'obfuscation qui détruit la structure lisible d'une fonction : les blocs de base sont dispersés et réordonnés, et un **répartiteur** (souvent un `switch` sur une variable d'état, ou une table de pointeurs indexée) décide à chaque tour quel bloc exécuter ensuite. Le graphe de flot devient un peigne plat où tout part du répartiteur et y revient — d'où le nom. Coûteux à lire manuellement ; on le contourne en émulant plutôt qu'en lisant. Souvent combiné à du **code mort** et à des prédicats opaques.

## appops (App Ops)
Framework Android de contrôle fin des opérations sensibles, distinct — et complémentaire — du système de permissions. Là où une permission est accordée à l'installation ou par l'utilisateur, une *app op* se bascule à `allow` / `deny` / `default` / `ignore` **par opération et par paquet**, y compris pour des capacités sans permission classique. S'interroge sans root : `cmd appops query-op SYSTEM_ALERT_WINDOW allow` liste tous les paquets autorisés à dessiner par-dessus les autres applications — un des premiers réflexes face à un affichage parasite.

## argv (et paramètres positionnels)
Le tableau des arguments remis à un programme au moment de l'`execve()` : `argv[0]` = le nom d'invocation, le reste = les arguments. Côté shell on parle de **paramètres positionnels** (`$1`, `$2`…), collectivement `$@` — remplis par l'**appelant**, donc vides si le script est lancé sans argument. Distinction cruciale pour tout **wrapper** : `"$@"` (quoté) retransmet le tableau **tel quel** — même nombre d'éléments, espaces internes préservées ; `$@` nu le refait passer par le découpage en mots, qui peut **changer le compte d'arguments** (`"a b"` → deux arguments). Analogie : `"$@"` = repasser l'`argv`/`argc` reçus directement à l'appelé ; `$@` = le re-tokeniser au passage.

## ARP (et table de voisinage)
Protocole qui résout « qui a l'IP X sur ce LAN ? » en adresse MAC. Le noyau garde un **cache volatil** (`ip neigh show`) : entrée fraîche → MAC connue ; machine muette → `STALE` puis `FAILED` puis disparition. Personne ne journalise les MAC par défaut — seul le serveur DHCP (la box) a un historique.

## asar (archive Electron)
Format d'archive **non compressée** propre à Electron, qui empaquette le code applicatif d'une app dans `resources/app.asar` : un en-tête JSON décrivant les offsets et tailles, suivi des fichiers concaténés. Electron le charge par un `require` **sans contrôle d'intégrité ni de signature** — d'où son statut de point d'accroche privilégié pour les mods clients (Vencord, BetterDiscord) : renommer l'original en `_app.asar` et poser un stub à sa place suffit à s'injecter, puis le stub `require` l'original. C'est le pattern du **proxy DLL** transposé (voir *DLL*), avec le même corollaire de détection : la présence du backup renommé *est* la trace du hook. S'inspecte avec l'outil `asar` (npm).

## attestation d'intégrité (Play Integrity / SafetyNet)
Mécanisme par lequel une application vérifie, **via un service distant**, que l'appareil et le système n'ont pas été altérés. Le verdict a plusieurs niveaux ; le plus élevé exige bootloader verrouillé, système d'origine signé et composant sécurisé intact. Utilisé par les applications bancaires, les jeux compétitifs et les portefeuilles de paiement. Conséquence pratique : root, déverrouillage du bootloader ou OS alternatif dégradent le verdict — et le **paiement sans contact**, qui exige le niveau maximal, est le premier à tomber. Analogie : une *remote attestation* de TPM — ce n'est pas l'application qui juge, c'est un tiers qui signe un constat sur l'état de la plateforme.

## Authenticode
Format de signature de code de Microsoft, stocké dans le PE via l'entrée *Certificate* du répertoire de données (indexée par **offset fichier**, pas par RVA — piège classique). Un certificat **auto-signé** (`subject == issuer`) ne valide pas et n'apporte aucune confiance, mais donne au binaire une identité crédible et trompe les outils qui se contentent de constater « signé ». Sa date `notBefore` est souvent l'**heure réelle de compilation**, bien plus fiable que le `TimeDateStamp` du PE, qui est trivialement falsifiable. Voir [[RFC 3161 (horodatage)]].

## B-tree (arbre B)
Structure de données arborescente équilibrée, accès en O(log n). **Btrfs** (« B-tree FS ») range toute sa structure — fichiers, extents, sous-volumes — dans des B-trees. Un snapshot = un nouvel arbre qui partage les nœuds de l'original jusqu'à divergence.

## bail DHCP (et réservation / bail statique)
Le DHCP ne *donne* pas une IP, il la **loue** (bail à durée limitée, renouvelé tacitement). Une **réservation** (bail statique) associe MAC → IP fixe côté serveur DHCP : la machine reste « automatique » mais reçoit toujours la même adresse. Alternative : IP statique configurée sur la machine (le serveur DHCP ne la connaît alors pas du tout). Doctrine : statique machine pour les serveurs, réservation pour les clients.

## BAL (Background Activity Launch)
Capacité pour une application **en arrière-plan** de lancer une activité au premier plan. Fortement restreinte depuis Android 10 pour empêcher les applis de s'imposer par-dessus ce que fait l'utilisateur, mais assortie d'exemptions (service en premier plan, interaction récente, UID sur liste d'autorisation) qui apparaissent dans les journaux sous la forme `BAL_ALLOW_*`. Point clé pour le diagnostic : une application qui passe par là **n'a besoin d'aucune permission sensible** — ni superposition, ni accessibilité — et reste donc **invisible à un audit statique des permissions**. Elle ne se trahit qu'en action, dans les lignes `ActivityTaskManager: START`.

## base64
Encodage qui représente des octets **binaires quelconques** avec seulement 64 caractères ASCII « sûrs » (`A–Z a–z 0–9 + /`), par groupes de 3 octets → 4 caractères (d'où +33 % de taille). Sert à **faire passer du 8 bits / du binaire à travers un canal qui n'accepte que du texte 7 bits** : pièces jointes mail, en-têtes accentués (encoded-words), `data:` URIs, payloads collés dans du texte en RE. Variante « URL-safe » : `+ /` → `- _`. Réversible sans perte. À ne pas confondre avec du chiffrement (aucun secret). Voir aussi *encoded-word*.

## BMC (Baseboard Management Controller)
Ordinateur autonome soudé sur la carte mère des serveurs (Dell : **iDRAC**, HP : iLO, Supermicro : IPMI/BMC), alimenté dès que la prise l'est. Console distante, power on/off, montage d'ISO, capteurs, API REST (Redfish) — un canal *out-of-band* qui survit à la mort de l'OS. Analogie : un debugger JTAG à demeure.

## Btrfs
Système de fichiers Linux moderne, **Copy-on-Write**, avec sous-volumes, snapshots, sommes de contrôle et compression transparente. Unité de stockage : l'**extent** ; structure : des **B-trees**.

## Bufferisation de flux (stdio buffering)
La libC ne remet pas chaque écriture au noyau : elle accumule dans un tampon utilisateur. Le mode dépend de la destination — **ligne par ligne** vers un terminal, **par blocs de 4 Ko** vers un fichier ou un tube, jamais pour `stderr`. Conséquence piégeuse : un programme redirigé vers un fichier paraît figé alors qu'il travaille, et si on le tue par `SIGTERM` **le tampon non vidé est perdu**. Même nature qu'un cache d'écriture : les données existent mais ne sont pas encore descendues d'un étage. Contournements : `stdbuf -oL -eL <cmd>` pour forcer le tampon ligne (inopérant si l'exécutable n'a pas la même architecture que le shim `LD_PRELOAD` de `stdbuf`), `timeout -s INT` pour laisser une chance au vidage, ou interroger le programme par un autre canal que sa sortie standard.

## Bytecode (et machine virtuelle)
Code compilé vers le jeu d'instructions d'une machine **virtuelle** plutôt que d'un processeur réel, puis interprété ou JIT-compilé à l'exécution. Conséquence pratique : l'artefact est **indépendant de l'architecture et du système** — un même fichier tourne en 32 et 64 bits, sous Windows et Linux. Dans un écosystème mixte, la frontière utile n'est donc pas « plugin ou pas » mais « bytecode ou code natif » : seul le second est lié à la plateforme. Exemples : IL .NET, `.class` Java, `.smx` SourcePawn, `.pyc`.

## C2 (Command & Control)
Serveur distant qu'un malware contacte pour recevoir des ordres et/ou **exfiltrer** des données. Indices : une adresse/domaine **en dur** dans le binaire, ou résolution DNS + `connect`+`send` vers une cible fixe. Son absence (aucun endpoint, réseau uniquement en broadcast LAN) est un argument fort de bénignité.

## callback TLS (Thread Local Storage callback)
Fonction qu'un PE enregistre (tableau `AddressOfCallBacks` du directory TLS) pour s'exécuter **avant** le point d'entrée, à chaque attach de process/thread. Technique d'évasion : un analyste qui pose son breakpoint sur `DllMain`/`main` rate ce code. En triage : vérifier que le tableau est **vide**. À ne pas confondre avec la section `.tls` elle-même, qui peut ne contenir que des *données* thread-local inertes.

## Cargo / crate (Rust)
**Cargo** est le gestionnaire de paquets + build system de Rust (`cargo build`, `check`, `run`, `install`). Une **crate** est l'unité de compilation/distribution Rust (une bibliothèque ou un binaire), déclarée dans `Cargo.toml` et téléchargée depuis crates.io. Piège vu : une crate peut n'exister qu'en **pré-version** (`imap = "3"` = alpha refusée → prendre `"2"` stable) ; et un type utilisé peut vivre dans une **dépendance transitive** (ex. `Envelope` dans `imap-proto`, pas `imap`) qu'il faut alors déclarer explicitement. Source cachée dans `~/.cargo/registry/src/`.

## carving (file carving)
Récupérer des fichiers **sans le système de fichiers**, en scannant les octets bruts (image disque, espace libre) à la recherche des **[[magic bytes]]** de début (et de fin si le format en a un). Donne le **contenu**, jamais le **nom** ni le chemin (ceux-ci vivent dans l'index / la [[MFT]], absente ici). À distinguer de l'**[[undelete]]** qui lit une entrée d'index supprimée et garde le nom. Limites : formats sans marqueur de fin (extraire un chunk borné, ex. `.kdbx`), **texte brut incarvable** (aucune signature), données **[[TRIM]]**ées/écrasées définitivement perdues. Outil : PhotoRec, ou scan de signature maison. Analogie : scanner un dump mémoire pour des magic bytes = retrouver des fonctions sans leurs noms.

## CFG (Control Flow Guard)
Protection MSVC moderne contre le détournement du flot d'exécution (appels indirects vers une adresse non prévue, typiques des exploits). Le compilateur émet la liste des cibles d'appel légitimes dans la section **`.gfids`** ; le runtime vérifie chaque appel indirect. Sa présence = toolchain récente et légitime — un indice qu'un binaire n'est pas un blob bricolé.

## ChaCha20-Poly1305
Chiffrement [[AEAD (Authenticated Encryption with Associated Data)]] combinant le flux ChaCha20 et l'authentificateur Poly1305. Rapide sans accélération matérielle, contrairement à AES-NI — d'où son adoption en TLS et dans les binaires Go, dont la bibliothèque standard étendue le fournit (`golang.org/x/crypto/chacha20poly1305`).

## chiffrement hybride (enveloppe de clé)
Combiner symétrique (rapide, gros volumes) et asymétrique (échange de clé) : la **donnée** est chiffrée avec une clé **symétrique** (AES) tirée au hasard, puis cette clé AES est **enveloppée** avec une clé **publique** (RSA/ECC) — seul le détenteur de la clé **privée** peut la déballer. Motif universel : TLS, PGP, keyslots [[dm-crypt / LUKS]]… et les **ransomwares** (AES par fichier + enveloppe RSA → indéchiffrable sans la clé privée de l'attaquant, donc **pas de décrypteur**). Miroir défensif : détruire la clé = [[crypto-erase]]. Fil rouge forensique : *la donnée est là, la clé est hors d'atteinte*.

## code de retour (exit status)
Entier (0–255) que tout processus renvoie en se terminant : **0 = succès**, tout le reste = échec (la valeur précise est propre au programme). Le shell le stocke dans `$?` (bash) / `$status` (fish), et les opérateurs `&&`/`||` s'en servent pour brancher : `a && b` (b si succès), `a || b` (b si échec). Analogie : la valeur de retour dans `eax` après un `call`, testée par un `test eax,eax` + saut conditionnel.

## Compositeur (compositing window manager)
Le composant qui assemble les fenêtres en une image finale et la présente à l'écran, au lieu de laisser chaque application dessiner directement dans le framebuffer. Il impose son propre rythme de présentation, ce qui découple la cadence de rendu d'une application de ce qui est réellement affiché — d'où les problèmes de [[frame pacing (rythme de présentation)]] quand une application produit beaucoup plus d'images que l'écran n'en affiche. Sous Wayland le compositeur est obligatoire et fait aussi office de serveur d'affichage (`kwin_wayland`, `mutter`, `Hyprland`).

## Copy-on-Write (CoW)
On ne copie une donnée qu'au moment où elle est modifiée : les lecteurs partagent l'original, une écriture crée une copie privée. Employé par `fork()` (pages mémoire), Btrfs (extents), OverlayFS (fichiers). Rend snapshots et forks instantanés et quasi gratuits au départ.

## core dump
Image mémoire d'un processus au moment de son crash, écrite pour analyse post-mortem (au format [[ELF (Executable and Linkable Format)]] sous Linux). Deux réglages noyau décident de son sort, et ils ne se jugent **que ensemble** : `kernel.core_pattern` dit où va le core — un chemin de fichier, ou, si la valeur commence par `|`, un **pipe** vers un programme lancé en root ; `fs.suid_dumpable` dit si les binaires privilégiés (SUID) en produisent (`0` = jamais, `2` = oui). La valeur `2` est risquée avec un `core_pattern` **fichier** (de la mémoire privilégiée peut atterrir dans un répertoire lisible) et sûre avec un **pipe** (rien n'est écrit dans le répertoire courant du processus) — c'est pourquoi systemd la pose délibérément quand `systemd-coredump` est le handler. Un auditeur qui juge la valeur seule se trompe.

## CRC-32
Somme de contrôle de 32 bits, conçue pour détecter les erreurs de transmission — **pas** une fonction de hachage cryptographique : on fabrique facilement des collisions, elle ne prouve rien contre un adversaire. Utile en analyse comme **empreinte d'égalité rapide** : un ZIP stocke le CRC-32 de chaque entrée dans son sommaire, ce qui permet de comparer le contenu de deux archives **sans les télécharger**. Deux entrées de même nom et même CRC sont, en pratique, le même fichier.

## crypto-erase (effacement cryptographique)
Effacer un support en **détruisant la clé de chiffrement** plutôt que les données : un SSD auto-chiffré garde tout en ciphertext avec une clé matérielle interne ; la régénérer/jeter rend l'intégralité **illisible instantanément** (le ciphertext reste, orphelin). Base de l'**ATA Secure Erase** / **NVMe Sanitize** modernes et de l'« Effacer contenu et réglages » iPhone. Couvre ce que l'overwrite `dd` rate (over-provisioning, blocs remappés — voir [[FTL]]). Analogie : une clé scellée à un TPM (cf. [[OSCrypt / DPAPI / safeStorage]]) — sans elle, le blob ne vaut rien. Norme : NIST SP 800-88 (niveau « Purge »).

## CSPRNG (générateur pseudo-aléatoire cryptographique)
Générateur d'aléa dont la sortie reste imprévisible même pour un attaquant qui en connaît des morceaux — requis pour clés, tokens, IDs de session. Sous Windows : `RtlGenRandom`, exporté par advapi32 sous le nom **non documenté `SystemFunction036`**, moteur réel derrière `CryptGenRandom`. Sous Linux : `getrandom(2)` / `/dev/urandom`. Voir un CSPRNG importé est bénin (génération d'identifiants) — à distinguer d'un usage de chiffrement (`CryptEncrypt` → piste ransomware).

## CVE / avis de sécurité de distribution (AVG)
Une **CVE** identifie une vulnérabilité **en amont**, dans le code d'un projet. Une distribution la traduit en avis propre (Arch : **AVG**, *Arch Vulnerability Group*) qui porte un état supplémentaire décisif : *un correctif est-il empaqueté et disponible ?* D'où la distinction opérationnelle qui change tout — « paquet **vulnérable** » ≠ « paquet **corrigeable** ». Un outil qui liste les premiers alarme sans donner d'action ; seul le filtre sur les seconds (`arch-audit -u`) produit une liste actionnable. Confondre les deux fait paniquer sur des vulnérabilités qu'on ne peut, par définition, pas corriger — et fait manquer celles qu'on peut.

## D-Bus
Bus de messages inter-processus des systèmes Linux de bureau. Deux instances : un bus **système** (services privilégiés) et un bus **de session** (un par utilisateur connecté). Un service y détient un **nom** (`org.freedesktop.secrets`), expose des **objets** identifiés par un chemin, qui portent des **interfaces** avec méthodes, propriétés et signaux. S'introspecte sans écrire une ligne de code avec `busctl --user list` / `get-property` / `call`. Analogie : une table d'exports partagée à l'échelle du système — le nom joue le rôle du symbole, le chemin d'objet celui de l'adresse.

## Dispatch d'interface (Go) / itab
Appel d'une méthode à travers une **valeur d'interface** Go : il ne se compile pas en `call <adresse>` mais en appel **indirect** via une **itab** (*interface table*), structure qui apparie le type concret et les pointeurs de ses méthodes. Conséquence en reverse : chercher « qui appelle `crypto/cipher.NewGCM` ? » par **références directes** (xref) ne retourne **rien**, alors que le code s'en sert — l'adresse n'est pas dans l'opcode d'appel, elle est chargée depuis l'itab au runtime. Un « 0 appelant » signe donc une **limite de la méthode**, pas une absence. On retrouve la cible par la **construction** de l'itab ou par l'exécution/émulation. Complément de [[pclntab (Go)]], qui donne les fonctions mais pas ce graphe d'appels indirects. Analogues : la **vtable** C++ (dispatch virtuel), ou un **thunk d'IAT** — l'appel passe par une case mémoire, pas par une constante.

## DLL (Dynamic-Link Library)
Bibliothèque partagée Windows (équivalent du `.so` Linux), au format **PE**. Expose des fonctions via sa **table d'exports** ; les exécutables la consomment via leur **table d'imports**. Le loader Windows résout les imports **par nom de DLL + nom de fonction** → une DLL qui réexporte les mêmes noms est un remplaçant transparent (« drop-in », base du proxying/hooking).

## `dlopen()` / chargement à l'exécution
Fonction qui charge une bibliothèque partagée **pendant** l'exécution, par opposition aux dépendances déclarées dans l'[[ELF]] (`DT_NEEDED`) que le loader résout au lancement. C'est le mécanisme des **plugins** : le programme ne connaît pas la lib à la compilation. `RTLD_NOW` résout **tous** les imports immédiatement (un seul manquant ⇒ retour `NULL` tout de suite) ; `RTLD_LAZY` diffère chaque fonction à son premier appel (l'échec surgit alors plus tard et ailleurs). `RTLD_GLOBAL`/`RTLD_LOCAL` décide si les symboles chargés servent aux chargements suivants. Le message d'erreur ne s'obtient que par `dlerror()` — un `NULL` seul ne dit rien. **Point capital** : la résolution se fait contre le disque **à l'instant de l'appel**, pas à celui du lancement du programme ; les deux peuvent être séparés par des heures et une mise à jour. Analogie : `LoadLibrary` + `GetProcAddress`.

## dm-crypt / LUKS
Chiffrement de disque Linux. `dm-crypt` est le moteur (device-mapper) ; **LUKS** est le format d'en-tête standard qui stocke les métadonnées de déchiffrement (clés dérivées d'une passphrase). Le volume déchiffré apparaît en `/dev/mapper/luks-<UUID-LUKS>`. L'**UUID LUKS** identifie le conteneur chiffré — ce n'est **pas** la clé.

## DoH / DoT (DNS chiffré)
Deux façons de chiffrer les requêtes DNS. **DoT** (DNS-over-TLS, port 853) est un canal dédié, identifiable sur le réseau ; c'est ce qu'Android appelle « DNS privé », réglé **au niveau système** et donc appliqué à toutes les applications. **DoH** (DNS-over-HTTPS, port 443) se fond dans le trafic web ; les navigateurs l'embarquent souvent **par application**. Piège classique : un DoH navigateur actif **contourne** le résolveur système — le filtrage posé au niveau OS ne s'applique alors pas à l'application qui génère le plus de requêtes. Les deux réglages doivent être décidés ensemble, pas indépendamment. Troisième conséquence, souvent ignorée : **[[ECH (Encrypted Client Hello)]] exige un DoH *navigateur*** — un DoT système, si bon soit-il, ne peut pas récupérer l'enregistrement DNS de type HTTPS dont ECH a besoin. Le DoH navigateur n'est donc pas un simple doublon du DoT système.

## domaine de routage DNS (`~.`, search vs route-only)
Étiquette attachée à une configuration DNS (globale ou par interface) qui décide **quelles requêtes lui sont envoyées**. Un *search domain* est aussi suffixé aux noms d'un seul label ; un *route-only domain*, préfixé de `~`, ne sert qu'au routage. Le cas particulier **`~.`** correspond à tout nom, avec **zéro label**. La règle d'arbitrage est un **longest-prefix match** : pour chaque nom, on retient le domaine correspondant ayant le **plus de labels**, et la requête ne part qu'aux serveurs associés à celui-là. Deux conséquences pratiques opposées : une interface déclarant un domaine spécifique (`lan`, `home`) n'est jamais consultée pour un nom public — elle ne gêne personne ; deux configurations déclarant toutes deux `~.` sont à **égalité parfaite**, la requête part aux deux **en parallèle** et la première réponse *réussie* gagne. Un NXDOMAIN ne comptant pas comme succès, une négative attend l'autre scope — et bloque jusqu'au timeout si celui-ci ne peut pas répondre. Cette égalité ne se résout par aucun réglage : elle se lève en supprimant un `~.`. Inspection : `resolvectl status`.

## drop-in (répertoire de configuration `*.d/`)
Convention où un logiciel lit un fichier principal **puis** tous les `*.conf` d'un répertoire voisin, par ordre lexicographique, chaque fichier surchargeant les précédents : `/etc/systemd/resolved.conf.d/`, `/etc/sysctl.d/`, `/etc/modprobe.d/`, `/etc/NetworkManager/conf.d/`, overrides d'unités systemd… Intérêt : ajouter un réglage **sans éditer** le fichier livré par le paquet, donc sans conflit à la mise à jour. Corollaire pour le diagnostic : la configuration effective n'est visible dans **aucun** fichier isolé — il faut la reconstituer, avec `systemd-analyze cat-config <répertoire>` qui affiche le résultat fusionné **en indiquant l'origine de chaque ligne**. C'est le seul moyen fiable de répondre à « qui pose cette valeur ? ».

## déréférencement (pointeur)
Passer d'une **adresse** à la **valeur** stockée à cette adresse. En C : `p` est l'adresse (le pointeur), `*p` la valeur pointée. En ASM x86 : `rdi` = l'adresse, `[rdi]` = la valeur en mémoire à cette adresse. Les crochets `[ ]` **sont** l'opérateur de déréférencement ; ce qu'il y a *dedans* (`[rdi + rsi*4 + 8]`) est de l'**arithmétique d'adresse** évaluée avant l'accès. À ne pas confondre avec `lea`, qui calcule l'adresse **sans** déréférencer (équivalent de `&expr`).

## désassembleur / décompilateur
**Désassembleur** : traduit les octets machine en mnémoniques assembleur (bytes → `mov`/`call`/`jmp`). Ex. `objdump -d`, **capstone**, la vue disasm d'IDA. **Décompilateur** : reconstruit un pseudo-code haut niveau (proche du C) à partir de l'assembleur — ex. Ghidra, Hex-Rays. Le premier est fidèle et toujours possible ; le second est une *reconstruction* heuristique, plus lisible mais faillible. Pour un triage « où regarder », le désassembleur suffit.

## Détour (hook, inline hook)
Détourner l'exécution d'une fonction existante sans modifier son code source ni recompiler le programme : on réécrit son prologue par un saut vers du code de remplacement, lequel peut ensuite rendre la main à l'original via un **trampoline** (une copie des instructions écrasées suivie d'un saut de retour). Technique centrale en instrumentation, en reverse engineering et dans les frameworks de plugins (Detours, MinHook, DHooks). Corollaire à ne pas oublier : « décharger le module qui a posé le détour » n'est **pas** une preuve que le détour est retiré — pour trancher il faut empêcher son chargement et redémarrer.

## ECDH / X25519 (échange de clés)
**ECDH** (*Elliptic Curve Diffie-Hellman*) : mise d'accord sur un secret partagé **sans le transmettre**. Chacun tire une paire de clés (privée/publique), s'échange la **publique**, puis calcule `secret = f(sa_privée, publique_de_l'autre)` — les deux tombent sur la même valeur, qu'un témoin du réseau ne peut pas reconstituer. **X25519** est la variante standard sur la courbe **Curve25519** (RFC 7748), clés de **32 octets**, omniprésente (TLS 1.3, WireGuard, Signal). Des clés **éphémères** (une paire par session) donnent la *forward secrecy* : casser une session ne casse pas les autres. Point capital : ECDH **n'authentifie pas** — sans signature ni [[Épinglage de certificat / de clé (pinning)]], un intermédiaire peut faire un ECDH avec chaque camp ([[MITM / homme du milieu (attaque de l'intermédiaire)]]). D'où son usage en couche applicative **par-dessus TLS** dans certains C2 : c'est précisément ce qui résiste au MITM TLS. Analogie : deux peintres mélangeant des couleurs publiquement mais gardant secrète la teinte finale.

## ECH (Encrypted Client Hello)
Extension TLS qui chiffre le **SNI** — le nom d'hôte que le client annonce **en clair** dans sa poignée de main pour que le serveur choisisse le bon certificat. Sans ECH, chiffrer le DNS ne cache donc pas quel site on visite : le nom repart en clair au premier paquet TLS. Mécanisme : le client va chercher la clé publique du serveur dans un enregistrement DNS de **type HTTPS (RR 65)**. Conséquence structurelle sous Linux — `getaddrinfo()` ne sait renvoyer que des adresses, pas un type d'enregistrement arbitraire, donc **seul un client DNS interne au navigateur** peut obtenir cette clé. Un DNS chiffré posé au niveau **système** ([[DoH / DoT (DNS chiffré)]]) ne permet **pas** ECH, quelle que soit sa qualité. Déploiement réel encore partiel (surtout derrière les grands CDN).

## EFLAGS / registre d'état
Registre CPU x86 dont chaque bit est un **flag** positionné en *effet de bord* par les instructions arithmétiques/logiques, selon leur résultat. Les principaux : **ZF** (zéro), **SF** (signe/négatif), **CF** (retenue — comparaisons non signées), **OF** (débordement signé). Un flag est la **photo instantanée du dernier résultat**, développée immédiatement (pas retardée). `cmp`/`test` existent *uniquement* pour poser ces flags ; les sauts conditionnels (`jz`, `jb`, `jl`…) ne font que les **lire**. C'est le canal découplé « une instruction pose la question, une autre y répond ».

## eFuse (fusible matériel)
Fusible intégré au silicium, grillé **une fois pour toutes** par une opération logicielle. Sert de compteur ou de drapeau **irréversible** : déverrouillage du bootloader, environnement d'exécution de confiance compromis, garantie logicielle consommée. Une fois grillé, aucun flash ni aucune réinstallation ne le réarme — c'est ce qui distingue un état « modifié » d'un état simplement « modifiable ». Explique pourquoi certaines fonctions (paiement sans contact, conteneur chiffré constructeur, santé) disparaissent **définitivement** après un root, même si l'on remet ensuite le système d'origine.

## Electron (modèle multi-processus)
Framework d'apps de bureau = **Chromium** (rendu web) + **Node.js**, dans un binaire. Séparé en processus **renderer** (l'UI, sans privilèges, `sandbox:true`, `nodeIntegration:false`) et **main** (backend Node : fs, réseau, crypto). Ils communiquent par **IPC** via un pont **preload** restreint (`contextBridge`). Modèle de sécurité analogue à **ring 3 / ring 0** : le renderer non privilégié « fait une syscall » au main, qui exécute après validation.

## ELF (Executable and Linkable Format)
Format des exécutables, bibliothèques partagées (`.so`), fichiers objets et core dumps sous Linux — l'équivalent du [[PE (Portable Executable)]] Windows. Magic bytes `\x7fELF`. Double lecture du même fichier : les **sections** (vue de l'éditeur de liens : `.text`, `.data`, `.dynsym`, `.dynamic`…) et les **segments** (vue du loader : quoi mapper, à quelle adresse, avec quels droits) — la même donnée décrite deux fois pour deux consommateurs différents. Pour l'exécution dynamique, deux structures comptent : `.dynamic`, dont les entrées **`DT_NEEDED`** listent les dépendances déclarées **par nom** (`libc.so.6`, pas `/usr/lib/libc.so.6`), et `.dynsym`, la table des [[symbole]] du runtime. Inspection : `objdump -p` (dépendances déclarées), `nm -D` (symboles), `readelf -h/-l/-S` (en-têtes, segments, sections).

## empreinte de navigateur (fingerprinting)
Identification d'un visiteur par la **combinaison** de ses attributs exposés (User-Agent, dimensions d'écran, fuseau horaire, polices installées, rendu canvas/WebGL, nombre de cœurs…) plutôt que par un cookie — donc insensible à l'effacement de l'état. La propriété qui gouverne toute la défense est contre-intuitive : ce qui identifie n'est pas la **quantité** d'informations exposées mais l'**unicité** de leur combinaison. Réduire les attributs exposés d'une façon originale **augmente** l'unicité ; la seule défense qui fonctionne est de rejoindre une **foule uniforme** (navigateur à empreinte standardisée, laissé non personnalisé). Second piège : mentir mal est pire que ne pas mentir — une **incohérence** entre l'OS annoncé et les API réellement présentes est un signal plus fort que la vérité. Analogie : reconnaître un binaire à sa signature de toolchain plutôt qu'à son nom ; un build custom expose moins et s'identifie mieux.

## encoded-word (MIME, RFC 2047)
Mécanisme pour mettre du **non-ASCII dans un en-tête de mail** (Subject, From…), qui ne transporte que de l'ASCII 7 bits. Forme : `=?charset?encodage?payload?=`, trois champs — le **charset** (caractère→octets, ex. UTF-8), le **transfer-encoding** (`B` = base64, `Q` = quoted-printable, octets→ASCII), et les données encodées. Ex. `=?UTF-8?B?UsOpY2xhbWF0aW9u...?=` → « Réclamation… ». Le caractère `é` est du **parfait UTF-8** ; le problème n'est pas l'encodage des caractères mais le **canal 7 bits**. Décodé par les bibliothèques mail (mail-parser). Voir aussi *base64*, *MIME*.

## Entitlement (macOS / iOS)
Autorisation déclarée dans la signature de code d'un binaire Apple, qui lui ouvre une capacité précise (accès réseau, à un dossier, à un service système). Stockée en plist XML et DER dans le blob de signature, donc **lisible statiquement**. Très parlante en analyse : `com.apple.security.get-task-allow` signifie que le processus accepte qu'on lui attache un débogueur — présent dans une build **Debug** d'Xcode, jamais dans une distribution soignée.

## entropie de Shannon
Mesure du désordre/imprévisibilité d'une suite d'octets, en bits par octet (0 = uniforme, 8 = maximal). Repères sur un binaire : texte ~4–5, code compilé ~6, **chiffré ou compressé** ~7.5–8. Heuristique de triage : une section de code à entropie >7.2 signe un **packer**/du chiffrement → RE statique bloquée. À l'inverse, des entropies « normales » confirment que le vrai code est lisible sur disque.

## ESP (EFI System Partition)
Petite partition en **vfat/FAT32** que le firmware UEFI lit au démarrage pour y trouver les bootloaders (`.EFI`) et souvent les noyaux. Montée sur `/boot` ou `/efi`. Étant en vfat, elle est **hors** de tout Btrfs → jamais snapshotée.

## exec / execve()
`execve()` remplace l'**image mémoire** du processus courant par un nouveau programme **sans créer de processus** : même PID, mêmes descripteurs de fichiers ouverts, mais code/données/pile écrasés. Il n'existe donc **aucune adresse de retour** — rien ne s'exécute après. À distinguer de `fork()` (voir l'entrée dédiée), qui duplique le processus : le lancement classique d'une commande = `fork()` + `execve()` dans l'enfant + `wait()` dans le parent, et le parent survit pour continuer. Le builtin shell `exec cmd` expose `execve()` **sans** le `fork()` : c'est ce qui permet à un script **wrapper** de « devenir » le programme qu'il enveloppe — un seul PID de bout en bout, arbre de processus propre, signaux du bureau reçus directement par la cible. Analogie : un `jmp` là où un appel normal serait un `call`.

## EXIF
Métadonnées embarquées **dans** un fichier image (JPEG/TIFF) : modèle d'appareil, **date de prise** (`DateTimeOriginal`, dans le sous-IFD Exif `0x8769`), réglages, **GPS**, parfois n° de série. Survit au [[carving]] (c'est du contenu, pas de l'enveloppe filesystem) → sert à **dater/attribuer** une photo sans nom. Lu par Pillow (`im.getexif()`), exiftool. Équivalent conteneur pour la vidéo : les tags `ffprobe` (`creation_time`, logiciel d'encodage — datable au build près).

## expansion (shell)
L'ensemble des transformations que le shell applique à une ligne **avant** de construire l'`argv[]` et de lancer quoi que ce soit. Les principales : **tilde** (`~` → `$HOME`, uniquement en début de mot **non quoté** — `"~/x"` reste un `~` littéral, et un `Exec=` de `.desktop` n'expanse rien du tout) ; **variable** (`$VAR`, fonctionne partout, guillemets compris — d'où la préférence pour `$HOME` dès qu'on quote) ; **substitution de commande** (`$(cmd)` : fork + pipe sur stdout, la sortie remplace l'expression — à ne pas confondre avec le préfixe `VAR=val cmd`, qui ne fait que remplir l'`envp[]` de l'enfant) ; **découpage en mots** (word splitting sur les espaces, c'est lui que les guillemets neutralisent) ; **globbing** (`*`, `?`, `[…]`). Corollaire : quoter change *ce que le programme reçoit*, pas l'esthétique. `sh -x` affiche les commandes **après** expansion — l'outil pour voir ce qui a réellement été construit.

## extent
Plage contiguë de blocs disque, l'unité d'allocation de Btrfs. Un fichier = un ou plusieurs extents. Le CoW opère à la granularité de l'extent : un write crée un nouvel extent, l'ancien reste tenu par les snapshots.

## Épinglage de certificat / de clé (pinning)
Le client **fige** (épingle) d'avance l'empreinte du certificat serveur attendu, ou mieux sa **clé publique** (SPKI), et **refuse tout le reste** — y compris un certificat pourtant « valide » signé par une AC de confiance. C'est la contre-mesure au [[MITM / homme du milieu (attaque de l'intermédiaire)]] par **AC injectée** (proxy d'entreprise, AC locale d'analyste) : même si le poste fait confiance à l'AC de l'attaquant, la clé présentée ne correspond pas à celle épinglée → connexion abandonnée. Vu en pratique : une charge dont la couche TLS était déchiffrée par une AC injectée s'est quand même **tue**, parce qu'un secret serveur épinglé côté applicatif ([[ECDH / X25519 (échange de clés)]]) ne pouvait pas être forgé. Épingler la **clé** plutôt que le certificat survit au renouvellement (même clé, nouveau certificat). Analogie : ne pas se contenter qu'une pièce d'identité soit officielle, mais vérifier que c'est **celle** d'une personne précise, mémorisée à l'avance.

## FBE (File-Based Encryption)
Chiffrement au repos d'Android (obligatoire depuis Android 10), **par fichier**, clés dérivées du code d'écran et gardées dans le **TEE**. Deux états : **BFU** (*Before First Unlock* — tout chiffré/illisible, même en labo) et **AFU** (*After First Unlock* — clés en mémoire, `/data` lisible). Conséquence forensique : sans le code d'écran (ou un état AFU + root), les identifiants (ex. `accounts_ce.db`) sont inaccessibles. Analogues : BitLocker (Windows), LUKS/[[dm-crypt / LUKS]] (Linux).

## firmware (blob)

Code exécuté par le **processeur d'un périphérique** (carte WiFi/BT, SSD, GPU…), pas par le CPU hôte. Souvent chargé à chaud par le driver au démarrage du device (`Direct firmware load for …`) depuis `/lib/firmware/`. Binaire **opaque**, signé/versionné par le constructeur ; sous Linux fourni par le paquet `linux-firmware` (ou extrait d'un driver constructeur quand il n'est pas encore redistribuable). Un driver présent **sans** son firmware échoue en `-2` (ENOENT) et peut boucler en reset. À distinguer du **driver** (côté noyau hôte) : les deux doivent être présents et **compatibles en version**.

## fork()
Appel système qui duplique un processus. Ne recopie pas la mémoire : l'enfant reçoit une copie de la **table de pages** pointant vers les **mêmes pages physiques**, marquées CoW (read-only). La copie réelle d'une page n'a lieu qu'à la première écriture (page fault). C'est le modèle de référence du CoW.

## Frame pacing (rythme de présentation)
La régularité des intervalles entre deux images effectivement affichées, distincte du **nombre** d'images par seconde. Une moyenne de 900 fps peut masquer des intervalles très inégaux, parfaitement visibles à l'œil comme des saccades. Le problème s'aggrave quand l'application produit beaucoup plus d'images que l'écran n'en présente : le [[Compositeur (compositing window manager)]] en jette la majorité, à des instants irréguliers. D'où l'utilité de plafonner la cadence au rafraîchissement de l'écran. Se mesure par le **frametime**, pas par les FPS moyens.

## fstab (`/etc/fstab`)
Table déclarative des montages à effectuer au boot : une ligne = source (de préférence `UUID=`), point de montage, type, options, et deux champs hérités (`dump`, ordre `fsck` — 0 0 pour Btrfs qui se vérifie seul). Sous systemd, le fichier est **compilé en unités `.mount`** au démarrage → après modification, `systemctl daemon-reload` puis validation par `findmnt --verify` et `mount <point>` (résolution réelle depuis fstab) **avant** tout reboot. L'option `nofail` évite qu'un disque absent/mort envoie le boot en mode urgence.

## FTL (Flash Translation Layer) / wear leveling
Couche du contrôleur SSD qui **traduit** les adresses logiques (LBA vus par l'OS) en adresses physiques NAND, en répartissant l'usure (*wear leveling*). Conséquence : « écrire par-dessus » un bloc logique écrit en réalité dans une **cellule neuve** et marque l'ancienne à recycler → les anciennes données restent physiquement présentes. Avec l'**over-provisioning** (7-28 % de NAND invisible), c'est pourquoi `dd` **n'efface pas** un SSD (il faut le firmware : Secure Erase / [[crypto-erase]]) et pourquoi le [[carving]] y est vain après [[TRIM]]. Analogie : un remapping d'adresses invisible, comme l'ASLR masque l'adresse réelle.

## Gatekeeper
Mécanisme macOS qui contrôle, **au lancement**, l'origine d'un exécutable. Il se déclenche sur l'attribut étendu `com.apple.quarantine`, posé par les navigateurs et clients mail sur les fichiers téléchargés, et exige alors une signature Developer ID valide et une [[Notarisation (Apple)]]. Deux angles morts exploités par les stealers macOS : `curl` ne pose **pas** l'attribut, et `xattr -c` l'efface. D'où la mode des attaques par copier-coller dans le Terminal — le seul chemin où Gatekeeper ne s'applique pas.

## glibc (GNU C Library)
L'implémentation de la bibliothèque C standard utilisée par la quasi-totalité des distributions Linux de bureau — donc la dépendance commune de presque tout le userspace. Elle est livrée en **plusieurs fichiers** (`libc.so.6`, `libm.so.6`, le loader `ld-linux-*.so.2`…) qui ne sont pas indépendants : ils forment **un ensemble cohérent issu d'un même build** et s'échangent des symboles internes sous la version [[versioning de symboles]] `GLIBC_PRIVATE`, sans aucune garantie de compatibilité d'une release à l'autre. Conséquence pratique : mélanger des fichiers glibc de deux versions dans un même processus casse. Depuis la **2.34** (2021), `libpthread`, `libdl`, `libutil`, `libanl` et le contenu de `librt` sont **fusionnés dans `libc.so.6`** ; les anciens `.so` subsistent comme **coquilles vides** (~14 Ko n'exportant que des `__lib*_version_placeholder`) pour que les vieux binaires portant `DT_NEEDED: librt.so.1` trouvent encore un fichier à charger. Corollaire : `-lpthread -lrt -ldl` dans un vieux `Makefile` sont désormais des no-op.

## Goroutine
Unité de concurrence du langage Go, beaucoup plus légère qu'un thread système : l'ordonnanceur du runtime multiplexe des milliers de goroutines sur quelques threads OS. Conséquence pour l'analyse : le `main()` d'un programme Go peut s'exécuter sur un thread **différent** de celui qui démarre, ce qui met en échec les émulateurs mono-thread — ils voient le runtime s'initialiser puis tourner à vide en attendant des threads qui n'existent pas.

## GPC / DNT (signaux d'opposition au pistage)
Deux tentatives successives de demander aux sites de ne pas pister : **DNT** (*Do Not Track*, en-tête `DNT: 1`) puis **GPC** (*Global Privacy Control*, en-tête `Sec-GPC: 1` plus une propriété JavaScript). Le point commun est décisif : ce sont des **demandes sans aucune contrainte technique** — le site est libre de les ignorer, et rien dans le navigateur ne l'en empêche. DNT a été si universellement ignoré que Mozilla a fini par retirer la case de son interface. GPC se distingue sur un seul point, juridique et non technique : il est un signal d'opposition **légalement opposable** sous CCPA/CPRA (Californie), alors que le RGPD ne le reconnaît pas formellement. À opposer aux protections **techniquement contraignantes** (partitionnement de cookies, blocage réseau) qui ne demandent rien et n'ont pas besoin d'être honorées pour fonctionner.

## GPT (GUID Partition Table)
Table de partitions moderne (héritière du MBR), requise par UEFI. Vit dans les premiers LBA du disque et garde une **copie de secours en fin de disque**. Chaque partition y a un type, un nom (cosmétique) et un GUID. Se crée avec `parted mklabel gpt` ou `sgdisk`. Les partitions se créent alignées (départ classique à 1 MiB) pour coller aux blocs physiques des SSD.

## HAL (Hardware Abstraction Layer)
Couche logicielle qui sépare le framework Android des pilotes propres à chaque puce. Une application ne parle jamais au capteur : elle passe par un service système, qui passe par le HAL du fabricant, qui parle au matériel. Conséquence pour le diagnostic : une panne peut se situer **sous** l'application sans que celle-ci ait le moindre défaut — et les couches ne lisent pas les mêmes choses. L'énumération des périphériques sert souvent des **métadonnées statiques** issues de la configuration du HAL, alors que la configuration effective des flux **alimente et interroge le matériel** ; un composant défaillant peut donc apparaître « présent » jusqu'à la seconde où on l'utilise vraiment.

## hash cryptographique (SHA-256…)
Empreinte de taille fixe calculée sur des données : le moindre bit modifié change tout le hash, et on ne peut pas fabriquer un fichier visant un hash donné (résistance aux collisions/préimages). Sert d'**identité de fichier** : comparer deux hashes = comparer les fichiers bit à bit. MD5/SHA-1 sont cassés pour la sécurité mais subsistent comme identifiants ; SHA-256 est le standard courant.

## hash perceptuel (perceptual hash : dHash/pHash/aHash)
Empreinte du **contenu visuel** d'une image (réduite en petit niveaux de gris, puis gradients/DCT), conçue pour que deux images **visuellement proches** (rééchantillonnées, recompressées, légèrement retouchées) aient des hash **proches** — mesurés par **distance de Hamming** (≤ ~6 = quasi-doublon). À l'opposé du [[hash cryptographique (SHA-256…)]] où un seul bit change tout le hash. Base de la dédup visuelle (Czkawka, VisiPics, `imagededup`). Rate les gros recadrages/rotations (là : embeddings CNN). Analogie : comparer deux binaires **après recompilation** — les octets diffèrent, la *forme* se ressemble.

## here-string / here-document
Deux façons d'alimenter l'entrée standard d'une commande **sans tube**. Le *here-document* (`<<MARQUEUR … MARQUEUR`) injecte un bloc de texte littéral ; le **here-string** (`<<< "$VAR"`) injecte le contenu d'une variable. Intérêt au-delà de la concision : puisqu'il n'y a pas de tube, il n'y a **pas de SIGPIPE possible** — ce qui évite qu'un `grep -q` (qui s'arrête dès la première correspondance) fasse échouer un pipeline sous `pipefail`. Extension bash/zsh/ksh, **absente de POSIX `sh`**. **Et absente de fish**, qui échoue au *parsing* (`Expected a string, but found a redirection`) — donc la commande n'est jamais lancée et rien n'est écrit. Idiome de remplacement portable : `printf '%s\n' 'ligne1' 'ligne2' | <commande>`, en **quotant chaque ligne** puisque, hors guillemets, `#` démarre un commentaire en fish.

## HPKE (Hybrid Public Key Encryption, RFC 9180)
Schéma **standardisé** de chiffrement à clé publique « hybride » : un **KEM** asymétrique (souvent [[ECDH / X25519 (échange de clés)]]) établit un secret partagé, puis un **AEAD** symétrique ([[AEAD (Authenticated Encryption with Associated Data)]] : ChaCha20-Poly1305, AES-GCM) chiffre la charge. « Hybride » = asymétrique pour **convenir de la clé**, symétrique pour **chiffrer le volume** (l'asymétrique seul est lent et borné en taille). C'est la formalisation nommée de la brique qu'on voit bricolée à la main dans nombre de protocoles (dont des C2) : `X25519 + AEAD`. Employé par ECH, MLS. Analogie : l'**enveloppe numérique** — on scelle la clé de session avec la clé publique du destinataire, le contenu voyage en symétrique.

## Idempotence
Propriété d'une opération qu'on peut **rejouer** sans changer le résultat au-delà du premier passage. Un scan idempotent (dédup par identifiant déjà traité, ex. Message-ID) peut être relancé après interruption sans créer de doublons ni re-déclencher d'effets. Base de la robustesse : « reprendre » devient sûr.

## IMAP (Internet Message Access Protocol)
Protocole d'accès aux mails **côté serveur** (les messages restent sur le serveur — contrairement à POP qui télécharge puis efface). Permet de lister les **enveloppes** (expéditeur, sujet, date) **sans** télécharger le **corps** MIME : distinction clé pour scanner beaucoup de mails à moindre coût. Port 993 (TLS direct) ou 143 (STARTTLS).

## initramfs / initrd
Mini système de fichiers chargé en RAM par le bootloader avant le vrai système. Contient le strict nécessaire pour monter la vraie racine (déchiffrer LUKS, charger les modules, monter Btrfs/overlay), puis « pivote » vers elle. C'est là que tourne `overlayfs-setup`.

## inode
La structure qui **est** réellement un fichier pour le système : type, droits, dates, taille et localisation des blocs de données. Ce qu'un inode ne contient **pas**, c'est son nom — le nom vit dans un répertoire, qui n'est qu'une table `nom → numéro d'inode`. D'où deux conséquences : plusieurs noms peuvent désigner le même inode (liens durs), et `unlink()` ne fait que retirer une entrée d'annuaire. L'inode n'est détruit qu'une fois **toutes** ses références tombées — y compris les descripteurs ouverts et les [[mmap]]. C'est pourquoi un processus continue de lire tranquillement un fichier « supprimé » (marqué `(deleted)` dans `/proc/<pid>/maps`), et pourquoi remplacer un fichier sous un programme vivant ne le casse pas : `pacman` fait `unlink` + création, donc **nouvel inode**, l'ancien restant vivant tant qu'il est mappé. Corollaire à retenir : **un même chemin, à deux instants, peut désigner deux fichiers différents** — `stat -c '%i %n' <f>` pour trancher. Analogie : le nom est une entrée de table de symboles, l'inode est l'objet pointé ; le compteur de références évite le `free()` prématuré.

## inotify
Mécanisme du noyau pour surveiller en temps réel des évènements sur des fichiers/dossiers (création, modification…). Outil userspace : `inotifywait` (paquet `inotify-tools`).

## Interpolation & prédiction (netcode)
Les deux mécanismes qui masquent la latence dans un jeu en réseau. **Interpolation** : le client affiche le monde avec un retard volontaire, de façon à toujours disposer de deux états serveur encadrant l'instant rendu — le retard se règle en nombre d'intervalles de paquets (un seul ne laisse aucun coussin de gigue, deux est le compromis courant). **Prédiction** : le client simule immédiatement les effets de ses propres entrées sans attendre la confirmation du serveur, puis corrige si le serveur le contredit — une correction visible est un *rubber-banding*. Deux conséquences utiles : ces réglages vivent **chez le client** et ne sont pas clampables par le serveur ; et l'échantillonnage de l'interpolation se fait **au moment du rendu**, donc un [[frame pacing (rythme de présentation)]] irrégulier suffit à produire des saccades sans aucun problème réseau.

## IOC (Indicator of Compromise)
Élément observable qui trahit une compromission ou permet de la détecter : empreinte de fichier, URL, IP, nom de domaine, chemin de dépôt, clé de registre, label de service. Un bon IOC est **spécifique** (peu de faux positifs) et **durable**. D'où deux règles : ne jamais publier comme IOC un fichier légitime, même détourné — il serait présent sur toutes les machines saines ; et préférer un invariant du kit à une valeur qui change à chaque build.

## IPC (Inter-Process Communication)
Mécanismes par lesquels des processus **isolés** échangent (pipes, sockets, messages nommés, mémoire partagée). Dans Electron : `ipcRenderer.invoke(canal, args)` côté renderer → `ipcMain.handle(canal, fn)` côté main, en requête/réponse **asynchrone**. Analogie : un **appel système** — le nom du canal = numéro de syscall, les arguments = registres, la valeur de retour = `eax` au retour.

## IPMI (Intelligent Platform Management Interface)
Standard (pré-Redfish) pour parler à un BMC : depuis le réseau, ou **depuis l'OS hébergé** via un device local (`/dev/ipmi0`, modules noyau `ipmi_si`/`ipmi_devintf`). Outil : `ipmitool` (+ extensions constructeur, ex. `delloem`). Permet notamment de reconfigurer le réseau du BMC sans passer par le BIOS.

## LaunchAgent / LaunchDaemon (macOS)
Mécanisme de persistance de macOS, piloté par `launchd`. Un **LaunchAgent** s'exécute dans la session d'un utilisateur (`~/Library/LaunchAgents/*.plist`), un **LaunchDaemon** au niveau système (`/Library/LaunchDaemons/`). Le plist déclare un `Label` (identifiant unique), `ProgramArguments` (quoi lancer) et souvent `KeepAlive` (relancer si ça meurt). Le `Label` est un excellent IOC de nettoyage : il donne directement le nom du fichier à chercher.

## LOLBin (Living Off the Land Binary)
Binaire **légitime**, souvent signé par l'éditeur du système, détourné pour exécuter du code malveillant. L'intérêt pour l'attaquant est double : il n'a pas à faire signer son code, et les défenses qui jugent un processus sur la réputation de son exécutable ne voient rien d'anormal. Voir [[Sideloading de DLL]] pour la variante la plus courante sous Windows.

## LZMA
Algorithme de compression à haut taux (famille de 7-Zip / xz), fondé sur un dictionnaire glissant et un codeur arithmétique. Le format « LZMA_ALONE » historique tient en un en-tête minuscule : 5 octets de propriétés (dictionnaire, paramètres du modèle) puis la taille non compressée sur 8 octets, avant les données. Des formats conteneurs embarquent parfois du LZMA avec **leur propre en-tête**, différent : il faut alors reconstruire l'en-tête attendu par la bibliothèque avant de décompresser. Piège associé : un `strings` sur des données compressées ne montre rien, ce qui fait conclure à tort qu'elles sont vides.

## Mach-O
Format d'exécutable d'Apple (macOS, iOS), équivalent du [[PE]] Windows et de l'ELF Linux. Organisé en **load commands** décrivant segments (`__TEXT`, `__DATA`, `__LINKEDIT`) et sections. Un binaire **universel** (« fat ») empaquette plusieurs architectures dans un seul fichier, précédées d'un en-tête `0xcafebabe` qui donne l'offset et la taille de chaque tranche. `LC_FUNCTION_STARTS` fournit les adresses exactes des fonctions, très utile en analyse — plus fiable que toute heuristique sur les prologues.

## magic bytes
Octets de signature en tête d'un fichier qui identifient son **vrai** format (`MZ` pour un PE, `7z¼¯'` pour du 7z, `\x7fELF`…), indépendamment de l'extension. C'est ce que lit `file`. Règle d'or du triage : l'extension ment, les magic bytes non (mais un conteneur peut en cacher un autre).

## Maildir
Format de stockage de mails où **un fichier = un message** (vs **mbox**, un seul gros fichier par boîte). Trois sous-dossiers : `tmp/` (écriture en cours), `new/` (livrés non lus), `cur/` (vus, avec les flags encodés **dans le nom du fichier** : suffixe `:2,` puis `S`=Seen, `R`=Replied, `F`=Flagged, `T`=Trashed, `D`=Draft). Livraison **atomique** : écriture dans `tmp/` puis `rename()` vers `new/` (jamais de mail à moitié écrit visible). Utilisé par Dovecot. Analogie : un inode par message plutôt qu'un blob concaténé.

## Malpedia
Base de référence du Fraunhofer FKIE qui **normalise les noms de familles de malware**, au format `plateforme.famille` (`win.emotet`, `osx.amos`). Sert de vocabulaire imposé à plusieurs plateformes de partage, dont ThreatFox. En pratique : toujours vérifier le nom exact avant de soumettre, et utiliser `unknown` (valeur officielle) plutôt qu'une famille voisine mais fausse, qui polluerait les filtres de tous les consommateurs du flux.

## Manifest V2 / V3 (extensions de navigateur)
Les deux générations d'API d'extensions des navigateurs dérivés de Chromium, adoptées ensuite par Firefox. La rupture porte sur le **blocage réseau** : en MV2, une extension enregistre un gestionnaire `webRequest` **bloquant** qui inspecte et annule chaque requête en JavaScript ; MV3 le remplace par des **règles déclaratives** évaluées par le navigateur, plus rapides mais bornées en nombre et en expressivité. Conséquence concrète : les bloqueurs de contenu les plus complets perdent une partie de leur pouvoir sur les navigateurs qui n'offrent que MV3. Point de divergence en 2026 : Chrome a retiré MV2, Firefox maintient les deux et conserve `webRequest` bloquante — d'où des recommandations d'extensions **différentes selon le navigateur** pour un même besoin.

## mDNS / DNS-SD (découverte de services)
Résolution de noms et découverte de services **sans serveur central**, par multicast sur le lien local (domaine `.local`, port 5353). Implémentations courantes : Avahi, Bonjour. Sert notamment à l'appairage du débogage sans fil d'Android : l'appareil annonce un service `_adb-tls-pairing._tcp` **tant que la boîte de dialogue d'appairage est affichée**, ce qui permet à un client local d'en découvrir le port sans le connaître à l'avance. Corollaire piégeux : fermer la fenêtre retire l'annonce, et l'appairage échoue **en silence** — l'erreur n'apparaissant qu'à la connexion suivante, sous une forme (rejet de certificat TLS) qui ne dit rien de la cause.

## MFT (Master File Table, NTFS)
L'**index** central de NTFS : un enregistrement (`FILE`) par fichier/dossier, avec son nom, ses dates **MACB**, son propriétaire (**Owner SID**) et la **liste de ses clusters** (data runs). Supprimer un fichier rend l'entrée réutilisable et libère les clusters — mais l'entrée (donc le **nom**) et les données survivent jusqu'à réécriture : c'est ce qui rend l'[[undelete]] possible. Équivalent ext4 : la table des **inodes**. Journaux associés : `$UsnJrnl` (créations/suppressions, **circulaire** = récent seulement), `$LogFile`. Analogie : la **table des symboles** d'un binaire — elle nomme *et* localise.

## MIME (Multipurpose Internet Mail Extensions)
Format structurant le **corps** d'un mail en parties typées (texte, HTML, pièces jointes, encodages). Un parseur (`mailparser` / `simpleParser`) transforme le MIME brut en objet exploitable `{ text, html, from, to, subject, date }`. Sans lui, le corps est un blob d'en-têtes et de sections encodées.

## MITM / homme du milieu (attaque de l'intermédiaire)
Position où un tiers s'intercale entre deux interlocuteurs, **relaie** leur trafic et peut le lire ou l'altérer, chacun croyant parler directement à l'autre. Côté analyse, on l'emploie **délibérément et localement** : injecter une **AC** dans le magasin de confiance d'une VM permet de déchiffrer le TLS d'un malware et de lire son protocole C2 (outils : mitmproxy, Burp). Ses contre-mesures sont l'[[Épinglage de certificat / de clé (pinning)]] et une **crypto de couche applicative** ([[ECDH / X25519 (échange de clés)]]) posée **par-dessus** TLS : le tunnel s'ouvre, mais l'échange interne ne se laisse pas forger. Rien à voir avec un défaut de TLS — c'est le modèle de confiance (à qui appartient l'AC) qui décide. Analogie : un **proxy** qui termine puis ré-émet la connexion, ou un **détour** ([[Détour (hook, inline hook)]]) qui intercepte un appel au passage.

## mmap / mapping mémoire
`mmap()` projette un fichier (ou de la mémoire anonyme) directement dans l'espace d'adressage d'un processus : lire l'adresse revient à lire le fichier, sans `read()`, les pages arrivant à la demande par fautes de page. C'est ainsi que le loader installe les bibliothèques ([[édition de liens dynamique]]) — le code exécuté n'est donc qu'une **vue** d'un fichier. Point clé : le mapping référence un [[inode]], **pas un chemin** ; il survit donc intact à un `unlink()` ou au remplacement du fichier, et continue de servir l'ancienne version. Inspection : `/proc/<pid>/maps` (plage d'adresses, droits, offset, inode, chemin, et le marqueur `(deleted)` quand le fichier mappé n'a plus de nom), `/proc/<pid>/map_files/` pour les symlinks vers les inodes eux-mêmes (lecture soumise à `ptrace_scope`). Analogie : une table de pages, mais adossée à un fichier plutôt qu'au swap.

## mode d'adressage (x86 : `[base + index*échelle + déplacement]`)
Façon dont une instruction mémoire calcule l'adresse effective à lire/écrire. Forme générale : **base** (registre, adresse de départ) + **index** (registre, numéro d'élément) × **échelle** (1/2/4/8 = taille d'un élément) + **déplacement** (constante). `[rdi + rsi*4]` = « élément `rsi` d'un tableau d'int 32 bits à `rdi` » = `rdi[rsi]`. L'échelle convertit un **numéro d'élément** (index 0,1,2…) en **offset d'octets** ; l'offset est la distance depuis la base, et `adresse = base + offset`. Piège : raisonner en **index** (0-based), pas en **ordinal** (1er, 2e…).

## MUA / MTA / MDA (rôles du mail)
Les trois maillons de la chaîne mail. **MUA** (*Mail User Agent*) = le client vu par l'humain (Thunderbird, TriMail). **MTA** (*Mail Transfer Agent*) = achemine les mails entre serveurs (Postfix, Exim), parle **SMTP**. **MDA** (*Mail Delivery Agent*) = dépose le mail dans la boîte et le sert au client (Dovecot), parle **IMAP**/POP3. Règle : **envoyer** = SMTP (MUA→MTA) ; **lire** = IMAP (MUA↔MDA).

## Mutex / verrou (exclusion mutuelle)
Primitive de concurrence : un « jeton » qu'un seul fil d'exécution peut détenir à la fois. Prendre le verrou (*lock*) avant de toucher une **ressource partagée** (une variable, une connexion à une base…), le relâcher après → évite les *races* (conditions de course). En Rust : `Mutex<T>` où le `T` n'est accessible qu'en verrouillant (`.lock()`). Bonne pratique vue : **relâcher le verrou pendant une I/O lente** (appel réseau) plutôt que de le garder — sinon tous les autres attendent derrière une section critique inutilement longue. Analogie : la clé unique des toilettes d'une station-service.

## mémoïsation (cache de calcul)
Mémoriser le résultat d'un calcul coûteux pour le **réutiliser** au lieu de le refaire. Le calcul n'a lieu qu'à la **première** demande (pour une entrée donnée), puis on lit le cache. Nécessite une **clé** stable qui identifie l'entrée (ex. `UID` d'un mail). Exemple vu : classer un mail par LLM (~0,5 s) une seule fois, stocker le résultat en base, le relire aux ouvertures suivantes — sinon on reclasse des milliers de mails à chaque lancement. Proche du *cache* ; à distinguer d'une *valeur dérivée* qu'on RECALCULE volontairement (car elle dépend d'un contexte changeant, ex. « maintenant »).

## NetBIOS
Couche de nommage réseau héritée de Windows (noms de machines ≤ 15 car., port 139 pour SMB, résolution par broadcast). Largement remplacée : SMB moderne parle directement sur 445, la découverte passe par WS-Discovery. Reste utile en diagnostic : `nmblookup -A <IP>` demande son nom à une machine.

## nice / niceness (et ionice)
La `niceness` est le niveau de courtoisie d'un processus vis-à-vis de l'ordonnanceur : de **−20** (le plus privilégié) à **+19**, 0 par défaut. Un écart de quelques points change franchement le temps CPU obtenu sous contention. Seul un processus privilégié peut **baisser** sa niceness (`renice -n <n> -p <pid>`), l'augmenter est libre. `ionice` est l'équivalent pour les entrées/sorties (classe + priorité). À savoir sous CachyOS : le démon `ananicy-cpp` applique automatiquement des règles **par nom de processus** définies dans `/etc/ananicy.d/` — une priorité inattendue se cherche donc là avant de se chercher dans le programme, et un fichier `99-*.rules` surclasse les règles livrées.

## Notarisation (Apple)
Procédure par laquelle Apple analyse un logiciel signé avec un Developer ID et lui délivre un ticket, exigé par [[Gatekeeper]] au premier lancement. Elle suppose un compte développeur payant et révocable — c'est précisément ce qu'un opérateur malveillant ne peut pas obtenir durablement. D'où le recours à une **signature ad-hoc** (voir [[Signature ad-hoc (macOS)]]) et à des vecteurs qui contournent Gatekeeper.

## NSIS (Nullsoft Scriptable Install System)
Générateur d'installeurs Windows, très répandu et très détourné. Son stub PE fait quelques dizaines de Ko et les données suivent en **overlay**, précédées d'un en-tête reconnaissable (`0xDEADBEEF` puis la chaîne `NullsoftInst`). La section `.ndata`, déclarée avec une taille sur disque nulle, est une signature du format. Le champ `flags` de cet en-tête porte `FH_FLAGS_NO_CRC` : un installeur légitime vérifie son intégrité, un installeur qu'on veut pouvoir bourrer d'octets doit désactiver ce contrôle.

## NSRL (National Software Reference Library)
Base du NIST recensant les hashes de fichiers de **logiciels connus-sains** (OS, applications publiées). Usage forensique : filtrer le « bruit » connu pour ne garder que l'inconnu, ou **prouver** qu'un binaire est identique à l'original publié. Interrogeable via l'API publique [CIRCL hashlookup](https://hashlookup.circl.lu). Logique **allowlist** (connu-bon), complémentaire des bases antivirus (connu-mauvais).

## NSS (Name Service Switch)
Couche d'indirection de la [[glibc (GNU C Library)]] qui décide, pour chaque type d'information (`hosts`, `passwd`, `group`, `services`…), **quelles sources consulter et dans quel ordre** — configuré dans `/etc/nsswitch.conf` (ex. `hosts: files mymachines resolve dns`). C'est pourquoi une entrée dans `/etc/hosts` court-circuite le DNS : `files` est consulté avant. Conséquence pour le diagnostic : `dig` interroge **un serveur DNS** et ne voit donc rien de NSS, tandis que `getent hosts <nom>` traverse la pile **réelle** qu'utilisent les applications. Tester avec le mauvais des deux fait conclure à l'inverse de la réalité.

## OAuth / fédération d'identité (« Se connecter avec X »)
Mécanisme où un service **délègue l'authentification** à un fournisseur (Google, Apple, Facebook) : pas de mot de passe local, on prouve son identité chez le fournisseur qui délivre un **jeton**. Pratique, mais **single point of failure** : si le compte fournisseur meurt/est verrouillé, le login délégué est **cassé** et il n'y a **aucun mot de passe local à réinitialiser** → il faut le support du service ou une suppression RGPD. Analogie : une **dépendance transitive** — le maillon central tombe, tout ce qui pointe dessus casse.

## OSCrypt / DPAPI / safeStorage
Chiffrement de secrets **lié au contexte OS/session**. `safeStorage` (Electron) délègue à **DPAPI** (Windows, clé dérivée de la session), au trousseau **libsecret** (Linux : gnome-keyring / kwallet) ou au **Keychain** (macOS). Un blob chiffré est **indéchiffrable hors de son contexte** (autre OS/session) — par conception, comme une clé scellée à un TPM ou dérivée du CR3/ASID d'un processus. Un échec de déchiffrement après migration n'est donc PAS une corruption, c'est l'isolation qui fait son travail.

## OverlayFS
Système de fichiers d'empilement. Superpose une couche **lowerdir** (read-only) et une couche **upperdir** (read-write) : les lectures voient la fusion, les écritures vont dans l'upper **sans toucher** le lower. `workdir` = espace de travail interne requis. C'est du **CoW au niveau fichier / VFS**.

## P-core / E-core (architecture hybride)
Sur un CPU hybride, deux familles de cœurs coexistent : les **P-cores** (performance, fréquence élevée, souvent avec SMT) et les **E-cores** (efficience, plus lents, sans SMT). Rien ne les distingue dans `/proc/cpuinfo` par un champ dédié ; on les identifie par leur fréquence maximale (`/sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq`). Enjeu pratique : une tâche sensible à la latence placée sur un E-core, ou migrée entre les deux familles, subit une gigue que la charge CPU moyenne ne montre pas. Voir [[Affinité CPU (CPU affinity)]] pour l'épinglage.

## packer / packing
Outil qui compresse et/ou chiffre un exécutable ; un petit **stub** ajouté par le packer restaure le code original en mémoire au lancement. But : réduire la taille, mais surtout **entraver l'analyse statique** (le code sur disque est illisible) — d'où sa fréquence dans les malwares. Détection : entropie de Shannon élevée, noms de sections atypiques (`UPX0`, `.themida`), table d'imports quasi vide. Contre-mesure : dumper le process déballé (dynamique) ou un unpacker dédié.

## pagination
Ne charger/traiter qu'une **fenêtre bornée** d'un grand ensemble (une « page ») au lieu de tout d'un coup, et charger la suite **à la demande**. Fait passer un coût de **O(taille totale)** à **O(taille de la page)** — constant, quelle que soit la taille du corpus (mémoire, réseau, latence avant premier affichage). Attention à l'**adressage** des bornes : inclusif/exclusif, 1-based (ex. plages IMAP `start:end`, taille `end−start+1`) vs 0-based demi-ouvert (slicing C/Rust `a..b`, taille `b−a`). Analogie : un moteur de jeu ne charge que ce qui entoure le joueur, pas toute la map.

## passerelle / route par défaut
La table de routage décide interface par destination ; le **même sous-réseau** se livre en direct (via ARP), tout le reste tombe sur l'entrée `default` → la passerelle (la box). **Supprimer cette entrée coupe tout accès hors-LAN** sans toucher au LAN — l'« interrupteur internet » d'un serveur. Champ `proto` de `ip route` : qui a posé la route (`kernel`, `dhcp`, `static`, `ra`) — donc ce qui disparaît si son propriétaire meurt.

## PATH (résolution de commande)
Variable d'environnement listant, **dans l'ordre**, les dossiers où le shell cherche un exécutable invoqué **par son nom** ; le premier trouvé gagne (`which -a <cmd>` les liste tous, `command -v <cmd>` donne le retenu). Un fichier posé dans un dossier prioritaire **masque** donc silencieusement un binaire système de même nom — technique utile (surcharge locale via `~/.local/bin`) et piège classique : un **wrapper** portant le nom de sa cible se rappellera lui-même à l'infini s'il ne l'invoque pas par **chemin absolu** (avec `exec`, pas une fork bomb mais une boucle d'`execve` dans un seul PID). Ne concerne que la résolution par nom : un chemin absolu, ou l'`Exec=` d'un `.desktop`, contournent PATH. Analogie : l'ordre de recherche des DLL sous Windows, avec le même risque de détournement par précédence.

## pclntab (Go)
Table de correspondance PC → ligne/fonction embarquée dans tout binaire Go, indispensable à ses traces de pile. Elle survit à `-trimpath` et à l'obfuscation des identifiants : même quand les noms sont randomisés, on récupère la **liste complète des fonctions avec leurs adresses et tailles**. C'est le point d'entrée naturel pour analyser un malware écrit en Go.

## PE (Portable Executable)
Format des exécutables/DLL Windows (`.exe`, `.dll`, `.sys`). Structure : en-têtes (magic `MZ` puis `PE\0\0`), **sections** (`.text` code, `.data`/`.rdata` données, `.rsrc` ressources, `.idata`/`.edata` imports/exports…), et tables d'imports/exports. Les noms de sections non standards trahissent souvent la toolchain (`.itext`/`.didata` = Delphi) ou un packer (`UPX0`…). `objdump`/binutils sait le lire depuis Linux.

## pipefail / `set -e` (options de robustesse du shell)
`set -e` interrompt le script à la première commande qui échoue ; `set -u` traite l'usage d'une variable non définie comme une erreur ; `set -o pipefail` fait échouer un **pipeline** si *n'importe quelle* étape échoue, au lieu de ne regarder que la dernière. Le trio `set -euo pipefail` est l'en-tête standard d'un script sérieux — mais il est à double tranchant : sous `pipefail`, un `grep` qui ne trouve rien (code 1) **ou qui s'arrête tôt** (`-q` → SIGPIPE en amont) fait tomber le pipeline, parfois exactement dans le cas que le script devait traiter. Voir **SIGPIPE** et **here-string**.

## point d'entrée (entry point)
Adresse où l'exécution démarre quand l'OS charge un binaire. Champ `AddressOfEntryPoint` du header PE (`e_entry` en ELF). Pour un EXE, c'est le stub CRT menant à `main` ; pour une **DLL**, c'est `DllMain` (via `_DllMainCRTStartup`), appelé à chaque attach/detach de process/thread. Premier endroit à désassembler en triage — le malware aime agir dès le chargement. À compléter par les **callbacks TLS**, qui tournent, eux, *avant* le point d'entrée.

## polling vs interruption (événementiel)
Deux façons d'attendre un événement. **Polling** : vérifier soi-même l'état en boucle (« c'est prêt ? c'est prêt ? »), coûteux et imprécis. **Interruption / événementiel** : on est **notifié** quand l'événement se produit, sans occuper le CPU à attendre. Exemples web : écouter chaque event `scroll` et recalculer (polling) vs `IntersectionObserver` qui prévient quand un élément entre dans la vue (événementiel). Généralise le polling matériel vs les IRQ, ou `epoll`/`select` vs busy-wait.

## profil professionnel (Android work profile / profile owner)
Second profil utilisateur cloisonné sur le même appareil, conçu à l'origine pour la gestion de flotte en entreprise. L'application qui le crée devient **profile owner**, privilège élevé lui permettant d'installer, geler et cloisonner d'autres applications. Détourné par des outils grand public pour isoler des applications intrusives (contacts, fichiers et applis du profil principal deviennent invisibles). À peser : c'est un composant tiers **très privilégié**, posé sur une API qui évolue à chaque version d'Android — un profile owner non maintenu est lui-même une surface d'attaque. Son seul apport réellement unique est le **gel** des applications, qui s'obtient aussi par `pm disable-user --user 0`.

## PTE (Page Table Entry)
Entrée de la table des pages qui mappe une page virtuelle → une page physique, avec des bits de permission (présent, **RW**, user/kernel…). Un bit **RW=0** rend la page non-inscriptible : toute écriture *fault*, quelle que soit l'instruction. Analogue matériel de la propriété `ro` d'un sous-volume Btrfs.

## race condition (condition de course) / TOCTOU
Bug où le résultat dépend de l'**ordre/timing** d'exécutions concurrentes accédant à une ressource partagée sans coordination. Caractère : **intermittent, non-déterministe, fuyant** (dur à reproduire et à tester → passe les tests, se déclenche au hasard en prod). Cas classique **TOCTOU** (Time-Of-Check to Time-Of-Use) : l'état vérifié a changé au moment de l'usage. On s'en défend **structurellement** (verrou/mutex sur la section critique, opération atomique, drapeau « déjà en cours »), **pas** en espérant que les tests l'attrapent. Ex. vu : un `IntersectionObserver` qui relance N fetches de la même page avant l'incrément d'un compteur → garde-fou `if (enCours) return`.

## RC4
Chiffrement de flux historique, aujourd'hui cassé et proscrit en cryptographie sérieuse, mais omniprésent dans les obfuscateurs : il tient en vingt lignes, sans dépendance ni table constante. Sa mécanique se reconnaît d'un coup d'œil dans du code déobfusqué — un tableau de 256 octets initialisé à l'identité, une première boucle de permutation par la clé (KSA), puis une boucle de génération qui XOR le flux (PRGA).

## RCON (Remote Console)
Protocole d'administration à distance d'un serveur de jeu : on s'authentifie par mot de passe, puis on envoie des commandes de console et on lit leur sortie. La variante Source tient sur TCP, en paquets `taille | id | type | corps` (type 3 = authentification, 2 = commande). Deux propriétés utiles au diagnostic : c'est un **canal indépendant de la sortie standard** du serveur, donc immunisé à la [[Bufferisation de flux (stdio buffering)]] ; et il partage habituellement le numéro de port du jeu, mais en TCP — donc potentiellement sur une [[Adresse d'écoute (bind address)]] différente. Côté sécurité : une tentative RCON contre un serveur qu'on ne possède pas est traitée comme une attaque, et sanctionnée comme telle.

## RDAP (Registration Data Access Protocol)
Successeur de `whois`, en HTTP et JSON, donc interrogeable avec un simple `curl` sans installer de client. Donne le bloc d'adresses, l'organisation, les dates d'enregistrement et le **contact abus** faisant autorité. Points d'entrée : `rdap.db.ripe.net/ip/<IP>` pour l'Europe, `rdap.org` en redirecteur générique.

## Redfish
API REST standardisée (DMTF) des BMC modernes : tout l'état du serveur (BIOS, capteurs, disques, power) en JSON sous `https://<bmc>/redfish/v1/…`, actions par POST/PATCH authentifiés. Successeur scriptable d'IPMI — un serveur s'administre au `curl`.

## registre (x86) et aliasing
Case de stockage interne au CPU. Sur x86-64, les registres généraux s'emboîtent par largeur (héritage 16 bits) : `rax` (64 bits) ⊃ `eax` (32) ⊃ `ax` (16) ⊃ `ah`:`al` (2×8). Écrire dans `al`/`cl`… ne modifie que l'octet bas du registre 64 bits sous-jacent — d'où `test cl, 1` = « bit 0 de l'octet bas de rcx ». Conséquence RE : `rax`, `eax`, `al` désignent **la même case** vue à des largeurs différentes, pas trois registres distincts. (Cas particulier : écrire dans un registre 32 bits comme `eax` **met à zéro** la moitié haute de `rax` ; écrire dans `ax`/`al` non.)

## Requête HTTP Range
En-tête `Range: bytes=<début>-<fin>` permettant de ne demander qu'une portion d'une ressource ; le serveur répond `206 Partial Content`. Très utile en analyse : le sommaire d'un ZIP étant **en fin de fichier**, on lit l'inventaire complet d'une archive de centaines de Mo en transférant quelques centaines de Ko, puis on extrait une entrée précise par son offset. Piège : certains CDN annoncent `accept-ranges: bytes` mais refusent les **suffix-ranges** (`bytes=-N`) — calculer des bornes explicites depuis `Content-Length`.

## Result / opérateur `?` (Rust)
**`Result<T, E>`** est l'enum standard de Rust pour une opération faillible : soit `Ok(T)` (succès), soit `Err(E)` (échec) — une **union étiquetée** dont le compilateur **impose** le traitement (impossible de lire la valeur sans gérer l'erreur). L'opérateur **`?`** suffixe une expression `Result` : sur `Ok(x)` il déballe `x` et continue ; sur `Err(e)` il **quitte la fonction en renvoyant `Err(e)`** (retour anticipé, propagation vers l'appelant). Ce n'est pas un try/catch — `?` *propage* (comme un `throw`), il ne *gère* pas. Analogie : `test` + `jnz .erreur` généré après chaque `call` faillible. Différence clé avec le **code de retour** C : ignorer l'erreur est impossible par accident (contrainte du type, pas discipline).

## RFC 3161 (horodatage)
Protocole d'horodatage par un tiers de confiance : une autorité signe un condensat accompagné de l'heure, ce qui prouve qu'un document existait à cette date. Dans une signature de code, cette **contre-signature** permet à la signature de rester valide après expiration du certificat. En analyse, elle offre une datation **indépendante** de l'attaquant — bien plus solide qu'un `TimeDateStamp` de PE, qui se falsifie en éditant quatre octets.

## rsync
Outil de synchronisation qui ne transfère que les **différences** (algorithme delta par blocs roulants). Options structurantes : `-a` (archive : récursif + métadonnées), `--delete` (miroir strict : supprime côté destination ce qui a disparu côté source), `--backup --backup-dir=<dossier>` (au lieu de perdre les fichiers modifiés/supprimés, les **déplacer** dans un dossier — daté, ça donne un versionnage incrémental gratuit), `--exclude`. Le `/` final sur la source signifie « le contenu de » (piège classique).
⚠️ **`-a` inclut `-o -g -p`** : rsync **réapplique** propriétaire, groupe et permissions **de la source** sur la destination. Un `chmod`/`chown` posé sur la cible *avant* le transfert est donc écrasé — silencieusement, la copie réussissant par ailleurs. Pour préserver les permissions de la cible : `rsync -a --no-o --no-g --chmod=D700,F600`. Corollaire de méthode : vérifier les permissions **après** le transfert, pas seulement avant.

## rôle système (Android RoleManager)
Attribution **exclusive** d'une fonction système à un paquet : navigateur (`BROWSER`), SMS, téléphone (`DIALER`), assistant (`ASSISTANT`), écran d'accueil (`HOME`)… Le détenteur reçoit des privilèges et des intentions que les autres n'ont pas. Se lit avec `cmd role get-role-holders android.app.role.<RÔLE>`. Deux conséquences pratiques : **remplacer** une application système impose de faire prendre le rôle au remplaçant *avant* de retirer l'ancienne (sinon plus de navigateur, plus de clavier, plus d'écran d'accueil) ; et **détourner** un rôle est une technique d'adware qui ne nécessite **aucune permission** — l'application ne force rien, le système l'appelle de lui-même.

## seccomp (filtrage d'appels système)
Mécanisme du noyau Linux par lequel un processus **s'auto-restreint** : il installe un filtre (programme BPF) qui décide, pour chaque appel système ultérieur, s'il passe, échoue, ou tue le processus (`SIGSYS`). Irréversible et hérité par les enfants — un processus ne peut que se restreindre **davantage**, jamais se rendre des droits. C'est la base des sandboxes de Firefox et Chrome : le processus de rendu ou de plugin est enfermé **après** avoir chargé ce dont il a besoin, puisqu'une fois le filtre posé l'accès au disque disparaît. D'où un motif récurrent : le **préchargement** explicite des bibliothèques juste avant la fermeture du sandbox (et les bugs qui vont avec, quand la liste préchargée ne correspond plus à la réalité du système). Symptôme d'une violation : mort brutale et `seccomp sandbox violation` au journal. Analogie : abandonner volontairement ses propres privilèges avant de traiter de l'entrée hostile, mais au niveau de l'interface noyau.

## Secret Service (trousseau freedesktop)
Spécification **D-Bus** (`org.freedesktop.secrets`) définissant un trousseau de secrets partagé entre applications : des **collections** (dont un alias `default`) contenant des **items** faits d'attributs en clair et d'une valeur secrète. Implémenté par gnome-keyring, KWallet/ksecretd… Les clients y accèdent soit par **libsecret** (`secret-tool`), soit par une implémentation propre du protocole — et les deux chemins ne se valent pas toujours : une incompatibilité côté client peut produire un item **créé avec des attributs corrects mais un secret vide**, sans erreur. Le trousseau protège surtout **au repos** : déverrouillé automatiquement au login, il reste interrogeable par tout processus du même utilisateur.

## SEH (Structured Exception Handling)
Mécanisme de gestion des exceptions de Windows (dépiler proprement la stack et exécuter les handlers quand une erreur remonte). En x64, les tables de déroulement (*unwind info*) sont rangées dans la section **`.pdata`** du PE. Pour l'analyste : une `.pdata` bien formée signe du C/C++ compilé normalement, et ces tables aident à retrouver les frontières de fonctions au désassemblage.

## Sideloading de DLL
Détournement de l'ordre de recherche des bibliothèques sous Windows : on place une DLL malveillante portant le nom attendu à côté d'un exécutable **légitime et signé**, qui la charge à son insu. Pour que la substitution passe, la DLL doit **exporter les mêmes fonctions** que l'originale. Technique MITRE **T1574.001**, prisée parce que le processus visible reste un binaire de confiance. Voir [[LOLBin (Living Off the Land Binary)]].

## Signature ad-hoc (macOS)
Signature de code sans certificat : le binaire porte des condensats de ses pages mais aucune identité (drapeau `CS_ADHOC`, blob CMS vide). Elle satisfait l'exigence formelle de signature d'arm64 sans rien prouver, et ne passe ni Developer ID ni [[Notarisation (Apple)]]. Détail exploitable : sans bundle, `codesign` dérive l'identifiant du **nom du fichier de sortie** — ce qui trahit régulièrement le nom de projet de l'auteur.

## SIGPIPE
Signal envoyé à un processus qui **écrit dans un tube dont l'extrémité de lecture est fermée**. Comportement par défaut : terminaison. C'est le mécanisme qui rend `cmd | head -5` efficace — `head` s'arrête, `cmd` est tué au lieu de produire le reste inutilement. Effet de bord majeur : le processus amont se termine donc **en échec**, ce que `set -o pipefail` remonte comme un échec du **pipeline entier** — y compris quand l'aval a parfaitement réussi (`grep -q` qui a trouvé sa correspondance). Sensible à une **course** : pas de SIGPIPE si l'amont avait déjà fini d'écrire, d'où des bugs qui paraissent intermittents et ne le sont pas. Parade : voir **here-string**.

## SLAAC / RA (auto-configuration IPv6)
En IPv6, une machine peut se configurer **sans DHCP** : le routeur diffuse des **Router Advertisements** (RA) et chaque hôte se fabrique une adresse (SLAAC) + une route par défaut (`proto ra` dans `ip -6 route`). Conséquence piège : « couper internet » en IPv4 ne suffit pas — la pile IPv6 a sa propre vie. Désactivation : `net.ipv6.conf.all.disable_ipv6=1` (sysctl).

## SMB (Server Message Block) / CIFS
LE protocole de partage de fichiers Windows (« partages réseau », `\\serveur\partage`), servi côté Linux par **Samba**. Versions : SMB1/CIFS (mort, vulnérable), SMB2, **SMB3** (chiffrement, signature — imposer `server min protocol = SMB3`). Port : **445** direct (139 = héritage NetBIOS). La **signature** authentifie chaque message (anti-MITM) ; l'**authentification** est à part (NTLMv2/Kerberos) avec sa propre base de comptes côté Samba (`smbpasswd`). La *découverte* des serveurs est un protocole séparé : WS-Discovery.

## snapshot
Copie figée d'un état à un instant T. En CoW (Btrfs, LVM…), quasi instantané et peu coûteux car il partage les données avec l'original jusqu'à divergence. À distinguer d'une **sauvegarde** (backup) : un snapshot vit sur le même disque.

## SQLite
Base de données relationnelle **embarquée** : pas de serveur, toute la base tient dans **un seul fichier**, manipulée via une bibliothèque liée à l'application. Idéale pour du stockage local structuré et requêtable (index, `SELECT`, `WHERE`) là où un JSON en mémoire ne passerait pas à l'échelle. En Rust : crate `rusqlite`, dont la feature **`bundled`** compile SQLite *dans* le binaire (aucune dépendance système à installer → build autonome, pratique pour la CI multi-OS). À distinguer d'un SGBD client-serveur (PostgreSQL, MySQL) : SQLite = un fichier, zéro process.

## Stack strings
Technique d'obfuscation où une chaîne n'existe **nulle part** dans les données du binaire : elle est assemblée à l'exécution, octet par octet ou mot par mot, par une suite d'instructions écrivant des valeurs immédiates sur la pile (`mov byte ptr [rbp-0x20], 0x6e`, ou `movz`/`movk` en arm64). `strings` ne voit rien : il faut **rejouer le calcul**. Même logique quand une clé est stockée en fragments recombinés par XOR — le secret est dans l'opération, pas dans les octets.

## STARTTLS
Mécanisme qui démarre une connexion **en clair** puis la **bascule en TLS** à la demande, sur le port habituel (IMAP 143, SMTP 587) — via la commande `STARTTLS` émise **avant** l'authentification. Alternative au **TLS implicite** (chiffré dès la connexion : IMAP 993). Piège de sécurité : s'authentifier *avant* le passage TLS envoie les identifiants en clair — d'où l'option serveur `disable_plaintext_auth`.

## stdin / stdout / stderr (flux standard)
Les trois canaux d'E/S ouverts d'office pour tout processus Unix, identifiés par leurs **descripteurs de fichiers** : `0` = stdin (entrée), `1` = stdout (sortie normale), `2` = stderr (erreurs — canal **séparé** pour que les erreurs n'empoisonnent pas les données). Toute la plomberie shell manipule ces numéros : `|` branche le stdout d'un processus sur le stdin du suivant, `>` redirige le 1, `2>` le 2, `&>` les deux. Analogie : trois handles pré-chargés dans des « registres » convenus au démarrage du processus — la convention d'appel du monde Unix.

## strings (outil)
Extrait les suites de caractères imprimables d'un binaire. Piège Windows : la moitié des strings sont en **UTF-16LE** (wide chars) → invisibles par défaut ; `strings -el` les révèle. Premier outil du triage : noms d'API, URLs, chemins, messages d'erreur, fichiers de config — le « journal intime » involontaire d'un binaire.

## subvolume (Btrfs)
Système de fichiers indépendant *à l'intérieur* d'un volume Btrfs : montable, snapshotable, configurable séparément. Identifié par un `subvolid` et un chemin (`/@`, `/@home`…).

## superbloc (superblock)
Bloc de **métadonnées globales** d'un filesystem, écrit par `mkfs` en tête de partition (souvent avec des copies de secours) : type, taille, UUID, label, paramètres. C'est là que vivent l'UUID et le label — ils appartiennent donc au **filesystem lui-même** et suivent le disque partout, machine comprise. Analogie : les en-têtes d'un binaire (PE/ELF) — structure fixe à offset connu qui décrit tout le reste.

## superposition (overlay / `SYSTEM_ALERT_WINDOW`)
Capacité à dessiner une fenêtre **par-dessus** les autres applications. Permission dite « spéciale » : elle s'accorde par un écran dédié, pas par le dialogue de permissions ordinaire. Historiquement le vecteur privilégié des adwares et des attaques par *tapjacking*. À vérifier en priorité face à un affichage parasite (`cmd appops query-op SYSTEM_ALERT_WINDOW allow`) — mais ne plus s'y arrêter : voir **BAL** et **rôle système**, deux mécanismes qui produisent le même résultat visuel **sans cette permission**.

## symbole (table des symboles)
Un nom associé à une adresse dans un binaire — fonction ou donnée. Deux tables distinctes : `.symtab` (complète, pour le débogage, souvent absente des binaires distribués) et **`.dynsym`** (réduite, indispensable au runtime pour lier les bibliothèques entre elles). Chaque entrée est soit **fournie** par ce fichier, soit **réclamée** à quelqu'un d'autre. Lecture avec `nm -D` (`-D` = table *dynamique*, sinon on lit `.symtab` souvent vide) : `T` = code défini ici, `D`/`B` = donnée définie ici, `U` = *undefined* (un **import**, à fournir par un autre fichier), `W`/`w` = faible, c'est-à-dire **facultatif** (absent ⇒ adresse nulle, sans erreur) ; majuscule = global, minuscule = local. L'adresse affichée est vide sur un `U` — il n'y a rien à localiser ici. Un « undefined symbol » à l'exécution signifie donc : *ce fichier réclame un nom que personne dans le processus ne fournit*. Analogie directe : `U` = entrée d'IAT non résolue, `T`/`D` = entrée côté exports de la [[table d'imports / d'exports (PE)]].

## sysctl (paramètres noyau réglables)
Interface d'exposition des variables du noyau via `/proc/sys/`, lisibles et modifiables à chaud (`sysctl <clé>`). Persistance par [[drop-in (répertoire de configuration `*.d/`)]] dans `/etc/sysctl.d/*.conf`, appliqué par `sysctl --system`. Deux pièges d'interprétation, tous deux rencontrés en audit : une valeur **ne se juge pas isolément** (voir [[core dump]] — `fs.suid_dumpable` change de sens selon `kernel.core_pattern`) ; et une valeur qui diffère d'un profil de durcissement n'est pas forcément plus **faible** — elle peut être différemment forte (variante irréversible vs révocable), ou posée délibérément par la distribution pour une raison de fonctionnement.

## systemd — types d'unités
`Type=oneshot` : le service fait sa tâche puis se termine (`RemainAfterExit=yes` le fait apparaître « active » ensuite). `Type=simple` : démon qui reste en avant-plan. Unités connexes : `.timer` (déclenchement planifié), `.path` (déclenchement sur évènement fichier). `systemctl cat <unit>` montre la définition.

## sérialisation / marshalling
Transformer une donnée en mémoire (struct, objet) en une représentation **portable et auto-descriptive** (souvent du texte : JSON, ou binaire) pour la transmettre ou la stocker, puis la reconstruire ailleurs (*désérialisation*). Nécessaire dès qu'on franchit une frontière — réseau, IPC, fichier — car on **ne peut pas** copier la mémoire brute (`memcpy`) : les pointeurs, le padding et l'endianness n'ont de sens que dans le processus d'origine. On parcourt donc les champs et on émet leurs *valeurs*. En Rust : la crate **serde** + `#[derive(Serialize)]` génère ce code à la compilation. Synonyme bas niveau : **marshalling**.

## table d'imports / d'exports (PE)
L'**export table** d'une DLL liste les fonctions qu'elle offre ; l'**import table** (IAT une fois résolue en mémoire) liste ce qu'un binaire consomme (`kernel32!CreateFileW`…). Résolution par le loader **au chargement, par nom**. En analyse : les imports sont un **profil comportemental** lisible sans désassembler (réseau, registre, injection, crypto…) ; les exports identiques à une DLL système signent un remplaçant drop-in.

## test témoin (contrôle)
Reproduire une mesure sur un système **connu sain**, aussi proche que possible du système suspect, pour savoir si une valeur observée est réellement anormale. Sans référence, une valeur inhabituelle *ressemble* toujours à une cause — et on construit une hypothèse dessus. Équivalent du **diff de binaires** en rétro-ingénierie : on ne cherche pas ce qui paraît étrange dans le suspect, on cherche ce qui **diffère** du sain. Coûte quelques minutes et élimine des branches entières d'hypothèses ; c'est souvent le seul moyen de réfuter une intuition séduisante.

## Themida
Protecteur logiciel commercial (anti-débogage, anti-VM, virtualisation d'instructions), l'un des plus coriaces à dépaqueter. Se reconnaît à ses noms de sections par défaut, `.themida` et `.boot`, quand l'opérateur ne les randomise pas — même famille d'indice que `UPX0`/`UPX1` ou `.vmp0`/`.vmp1` pour VMProtect.

## Tick / tickrate (simulation à pas fixe)
Un serveur de jeu avance sa simulation par **pas de temps fixes** — le tick — et non à chaque image rendue, afin que la physique soit reproductible et indépendante de la puissance des machines. Le tickrate est le nombre de pas par seconde (66,7 Hz pour Source par défaut). Il borne la latence minimale : un aller-retour ne peut pas être plus rapide qu'un tick. À distinguer de la fréquence d'envoi des états au client, qui est un réglage séparé et peut être inférieure. Une valeur de tickrate légèrement différente de la fréquence d'envoi crée une fréquence de battement, donc des états périodiquement sautés.

## tmpfs
Système de fichiers monté en **RAM** (et swap). Rapide et **volatil** : tout disparaît au démontage / reboot. Utilisé pour `/tmp`, `/run`, et comme couche `upperdir` de l'overlay d'un boot-snapshot.

## TOTP / HOTP (codes à usage unique)
Second facteur d'authentification calculé **localement** à partir d'un secret partagé : HOTP incrémente un compteur, **TOTP** utilise l'horloge (fenêtre typique de 30 s). Le secret est le **seul** élément à sauvegarder — le code, lui, se recalcule. Deux conséquences pratiques : une sauvegarde de coffre 2FA se vérifie en comparant les **codes générés** côte à côte, jamais les noms d'entrées (un secret corrompu produit un nom correct et un code d'apparence parfaitement valide) ; et un décalage d'horloge suffit à invalider tous les codes d'un coup.

## toybox
Implémentation minimaliste des utilitaires POSIX regroupés dans **un seul binaire** (même idée que BusyBox, licence permissive), userland standard d'Android. Les commandes portent les mêmes noms qu'en GNU coreutils mais **pas les mêmes options ni toujours le même comportement** : pas de `sort -h`, et un `find` qui **abandonne** à la première permission refusée au lieu de continuer son parcours. Piège pour qui scripte à distance en supposant du GNU — un résultat vide peut signifier « commande interrompue », pas « rien trouvé ».

## TRIM / discard
Commande par laquelle l'OS signale à un SSD que des blocs ne contiennent plus de données utiles, pour que le contrôleur les recycle (le SSD ne « voit » pas les suppressions de fichiers sinon). `mkfs` fait un TRIM complet du volume ; en usage courant, Btrfs monte les SSD avec `discard=async` (TRIM différé, sans coût sur les écritures). Effet de bord forensique : des données trimées sont **irrécupérables** par carving.

## udev
Gestionnaire d'évènements périphériques de l'espace utilisateur : à chaque détection par le kernel, il sonde le matériel, remplit sa **base d'infos** (celle que lit `lsblk`) et crée les liens stables `/dev/disk/by-uuid/…`, `by-label/…`, `by-id/…` — des alias qui survivent à l'ordre d'énumération, contrairement aux noms `/dev/nvmeXn1`/`sdX`.

## undelete (récupération par l'index)
Récupérer un fichier supprimé en lisant son **entrée d'index encore présente** (enregistrement [[MFT (Master File Table, NTFS)]] ou inode marqué « deleted » mais pas réécrit) : on récupère le **contenu ET le nom/chemin**, contrairement au [[carving]] (signature brute, sans nom). Fonctionne tant que l'entrée n'a pas été réutilisée. Outils : TSK `fls -rd` (lister) + `icat` (extraire), `tsk_recover`. Sur SSD avec [[TRIM]], souvent mort d'avance. Analogie : lire un symbole encore listé dans la table mais tagué supprimé.

## union étiquetée (tagged union / somme)
Type qui contient **exactement une** valeur parmi plusieurs variantes possibles, avec un **discriminant** (tag) qui dit laquelle. En mémoire : un tag + la charge utile de la variante active. Le `union` du C n'a **pas** de tag (au programmeur de savoir laquelle est valide → source de bugs) ; les **enums** de Rust (ex. `Result`, `Option`) sont des unions étiquetées dont le compilateur **vérifie** qu'on traite chaque variante. Aussi appelé *type somme* ou *discriminated union*.

## UUID (Universally Unique IDentifier)
Identifiant de **128 bits générés aléatoirement** — l'espace est si vaste que deux tirages ne collisionnent jamais en pratique, ce qui rend l'unicité *probabiliste mais fiable* sans registre central. Un filesystem en reçoit un au `mkfs`, stocké dans son **superbloc** : il identifie le filesystem où qu'il soit branché. À distinguer du label (choisi, non unique) et du chemin device (attribué au boot, instable). Référence de choix dans fstab : résoudre **par identité**, pas par position.

## valeur dérivée vs stockée
Une donnée peut être **stockée** (persistée telle quelle) ou **dérivée** (recalculée à la volée à partir d'autres données). On dérive plutôt que stocker quand le résultat **dépend d'un contexte changeant** — typiquement « maintenant ». Exemple : l'urgence temporelle d'un mail = fonction de (échéance − date du jour) ; on stocke l'**échéance** (stable) mais on **recalcule** l'urgence à chaque affichage (sinon elle serait figée alors qu'un mail devient urgent en vieillissant). Analogie base de données : une **vue** (calculée à la lecture) vs une **colonne** (matérialisée). À ne pas confondre avec la *mémoïsation*, qui elle FIGE volontairement un résultat stable.

## versioning de symboles (`symbole@VERSION`)
Mécanisme [[ELF]] permettant à **plusieurs versions du même nom** de coexister dans une seule bibliothèque, afin de changer la sémantique d'une fonction sans casser les binaires déjà compilés (`memcpy@GLIBC_2.2.5` et `memcpy@GLIBC_2.14` vivent ensemble dans la même `libc`). Chaque binaire est figé sur la version qu'il a demandée à la compilation : c'est ce qui fait tourner un exécutable Linux de 2005 aujourd'hui. Conçu par Ulrich Drepper pour glibc 2.1 (1999), d'après le schéma de Sun/Solaris. Lecture avec `nm -D` : **`nom@@VERSION`** (double `@`) = version *par défaut* d'un symbole **défini** ; **`nom@VERSION`** (simple) = version non-défaut, ou — sur un `U` — la version **exigée** (donc un fichier qui définirait ce nom dans une autre version ne satisfait pas la demande). Cas particulier à connaître : **`GLIBC_PRIVATE`**, réservé aux symboles que les fichiers de la [[glibc]] s'échangent entre eux. Ce n'est pas de l'API publique et il ne porte **aucune promesse de compatibilité** entre versions — c'est donc, par conception, la première chose qui casse quand deux générations de glibc se retrouvent dans un même processus. Analogie : les exports non documentés de `ntdll` — ça marche jusqu'au prochain service pack.

## vfat / FAT32
Vieux système de fichiers Microsoft, sans permissions Unix ni CoW, mais lu universellement par les firmwares UEFI → utilisé pour l'**ESP** (`/boot`).

## VFS (Virtual File System)
Couche d'abstraction du noyau qui expose une API unique (open/read/write/mount…) au-dessus de systèmes de fichiers hétérogènes (ext4, Btrfs, overlay, tmpfs…). C'est elle qui permet à un OverlayFS de coexister avec du Btrfs en dessous.

## WFP (Windows Filtering Platform)
La couche de filtrage réseau du noyau Windows : le Pare-feu Windows n'en est qu'un **client** parmi d'autres — antivirus, VPN et pare-feux tiers y injectent leurs propres filtres (callouts), **au-dessus ou en dessous** des règles visibles. Conséquence : un paquet peut être droppé silencieusement sans qu'aucun outil natif (règles, audit 5152) ne le montre. Inspection : `netsh wfp show filters` / `netsh wfp capture`.

## WS-Discovery (wsdd)
Protocole de **découverte** d'appareils sur le LAN (multicast) utilisé par Windows moderne pour peupler « Réseau » dans l'explorateur — successeur de la découverte NetBIOS. Un serveur Samba est invisible dans cette liste sans un démon dédié (`wsdd2` sous Linux) ; le partage marche quand même par chemin direct `\\<IP>`. Découverte et partage = deux protocoles indépendants.

## XDG Base Directory
Spécification freedesktop qui range les fichiers utilisateur **par rôle** au lieu de les entasser dans le home : `~/.config` (configuration, `$XDG_CONFIG_HOME`), `~/.local/share` (données, `$XDG_DATA_HOME`), `~/.local/state` (état et logs applicatifs), `~/.cache` (jetable, supprimable sans perte), `~/.local/bin` (exécutables perso). Propriété clé, souvent ignorée : à **nom de fichier égal**, la version *utilisateur* l'emporte sur la version *système* — `~/.local/share/applications/<app>.desktop` surcharge `/usr/share/applications/<app>.desktop` sans toucher au système, sans root, et **survit aux mises à jour du paquet**. C'est le point d'extension **prévu** pour redéfinir le lancement d'une application ; patcher le fichier système à la place revient à modifier un fichier que le gestionnaire de paquets écrasera en silence (`pacman -Qii <pkg>` → `Backup Files : None` = aucune protection, pas même un `.pacnew`).

## xorstr (obfuscation de chaînes)
Famille de bibliothèques C++ qui chiffrent les littéraux de chaînes **à la compilation**, chacun avec **sa propre clé aléatoire**, texte chiffré et clé étant émis comme valeurs immédiates puis XORés à l'usage. Conséquence pratique : il n'existe **aucune clé unique à extraire**, et une section `__cstring` ou `.rdata` quasi vide face à des centaines de Ko de code en est le signe. On la casse en émulant, pas en cherchant une routine centrale.

## XWayland
Couche de compatibilité qui fait tourner des applications X11 dans une session Wayland, en les faisant dialoguer avec un serveur X intégré au compositeur. Fonctionne pour l'immense majorité des programmes, mais ajoute un intermédiaire dans le chemin de présentation des images — d'où des problèmes de [[frame pacing (rythme de présentation)]] et de latence d'entrée sur les jeux anciens, qui n'existent pas en natif Wayland. Contournement courant : encapsuler le jeu dans un compositeur imbriqué dédié (`gamescope`), qui reprend la main sur la cadence de présentation.
## Écriture atomique (via `rename`)
Technique pour ne jamais laisser un fichier à moitié écrit : on écrit dans un fichier temporaire, puis on le bascule sur la cible avec `rename()` — opération **atomique** au niveau du système de fichiers (l'entrée annuaire nom→inode change d'un coup). Un lecteur voit toujours l'ancienne version **complète** OU la nouvelle, jamais un état intermédiaire. Équivalent d'un échange de pointeur atomique (CAS / `xchg`) évitant un *torn read*. Souvent doublé d'un `.bak` de la version précédente.

## édition de liens dynamique / `ld.so` (le loader)
Un binaire dynamique ne contient pas le code de ses dépendances : il en porte les **noms** (entrées `DT_NEEDED`), pas les chemins. La traduction nom→fichier puis la résolution des symboles reviennent au **loader**, `/usr/lib/ld-linux-x86-64.so.2` sur x86-64, désigné par le segment `PT_INTERP` du binaire. Séquence réelle : le noyau mappe le binaire, lit `PT_INTERP`, mappe **aussi** le loader, et saute dans **le loader** — pas dans `main()` ; celui-ci mappe alors les bibliothèques (ordre de recherche : `DT_RPATH`/`RUNPATH`, `LD_LIBRARY_PATH`, `/etc/ld.so.cache`, dossiers par défaut), résout les symboles, et passe enfin la main au programme. Deux surprises : le loader est **lui-même un `.so`** visible dans `/proc/<pid>/maps`, et il **exporte** ses propres symboles (sous `GLIBC_PRIVATE`) que les autres bibliothèques consomment — il est donc à la fois chargeur *et* dépendance. Analogie : le loader PE de `ntdll`, mais en espace utilisateur.

## énumération USB (et reset)

Séquence par laquelle l'hôte USB **détecte** un périphérique branché, lui attribue une **adresse**, lit ses descripteurs et charge son driver. Un **reset** (ré-énumération) rejoue cette séquence — normal au branchement, **anormal en boucle** : signale soit un périph qui n'aboutit pas (firmware manquant → `failed to load firmware`), soit une **chute de tension** (*brownout* : le 5 V passe sous le seuil, le device redémarre). Les logs noyau (`journalctl -k`) tracent chaque reset avec l'adresse **bus-port** (`1-11`, `1-6.3`) ; leur **cadence** (constante = périph en échec, vs corrélée à une charge = électrique) distingue les causes.

## état éphémère vs persistant
Une donnée est **éphémère** si elle vit seulement en mémoire volatile (variable, DOM…) et disparaît/se recalcule à chaque relance ; **persistante** si elle est écrite sur un support qui survit (fichier, base). Question de conception centrale : *« où vit la donnée ? »*. Re-calculer une donnée coûteuse à chaque ouverture (ex. une classification IA de 10k éléments) est intenable → il faut un **store persistant** (souvent un cache clé→valeur, ex. `UID → catégorie`). Analogie : registre volatile (perdu à la coupure) vs mémoire non-volatile.

