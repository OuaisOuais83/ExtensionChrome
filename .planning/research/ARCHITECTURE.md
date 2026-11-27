# Architecture Research

**Domain:** Relais de contexte Claude-vers-Claude (serveur MCP distant en custom connector claude.ai + surcouche extension Chrome + backend comptes / contacts / inbox / fichiers / crédits / liens magiques / chiffrement)
**Researched:** 2026-11-27
**Confidence:** MEDIUM

> **Lecture des niveaux de confiance.** Les niveaux viennent du seam `classify-confidence` de GSD, pas d'une appréciation libre : une information trouvée par recherche web puis recoupée sur la page officielle Anthropic vaut `MEDIUM` (plafond atteignable ici, Context7 n'étant pas disponible dans cette session) ; une source unique ou communautaire vaut `LOW`. Les passages marqués **Conception** sont des propositions d'architecture déduites de ces faits, pas des faits sourcés.
>
> Toutes les pages `claude.com/docs/...` et `code.claude.com/docs/...` citées ont été lues directement le 2026-11-27. Aucune tentative d'injection n'a été relevée dans les contenus récupérés.

---

## Les six faits qui dictent l'architecture

Avant le schéma, les contraintes vérifiées qui ferment ou ouvrent des options. Tout le reste en découle.

| # | Fait vérifié | Conséquence d'architecture | Confiance |
|---|--------------|----------------------------|-----------|
| 1 | Claude joint un connecteur distant **depuis le cloud d'Anthropic** (plage de sortie `160.79.104.0/21`), jamais depuis l'appareil de l'utilisateur. | Le serveur MCP voit en clair **les arguments d'outil (envoi)** et **les résultats d'outil (lecture)**. Le E2E strict est impossible sur le canal connecteur, dans les deux sens. | MEDIUM |
| 2 | Plan **Free : un seul custom connector**. | Tout doit tenir dans **un seul serveur MCP, une seule URL**. Pas de second connecteur « fichiers » ou « admin ». Notre connecteur consomme l'unique emplacement du destinataire Free. | MEDIUM |
| 3 | Taille max d'un résultat d'outil : **~150 000 caractères** sur claude.ai / Desktop ; **25 000 tokens** par défaut sur Claude Code, et tout résultat texte de plus de **50 000 caractères** y est écrit dans un fichier et remplacé par un chemin. Timeout : **240 s** par appel d'outil (hébergé). | Lecture **paginée et manifeste d'abord**. Budget unique recommandé : 40 000 caractères par résultat (voir « Gros contenus »). | MEDIUM |
| 4 | L'extension MCP de **transfert de fichiers** (upload / download hors résultat d'outil) est en bêta et **indisponible pour les custom connectors** : elle exige une fiche dans l'annuaire Anthropic et des hôtes de stockage approuvés par Anthropic. | En v1, **aucun binaire ne transite par MCP**. Les fichiers entrent par un canal latéral (extension, page d'upload) et sortent sous forme de texte extrait et d'images. | MEDIUM |
| 5 | Un **lien d'installation officiel** pré-remplit la boîte « Add custom connector » : `https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=NOM&connectorUrl=URL_ENCODEE`. L'utilisateur confirme toujours. | Le lien magique peut amener un destinataire sans compte jusqu'au connecteur en deux confirmations, sans toucher à l'API interne de claude.ai. | MEDIUM |
| 6 | Claude **ne transmet aucune identité utilisateur** au serveur ; `clientInfo` est non authentifié et varie selon la surface. L'identité naît du flux OAuth sur **notre** serveur d'autorisation (DCR ou CIMD, PKCE S256 obligatoire, callback `https://claude.ai/api/mcp/auth_callback`, loopback pour Claude Code). | Le **compte Relay est l'unique source d'identité**. L'extension et le connecteur ne partagent pas de jeton ; ils partagent un `user_id` côté backend. | MEDIUM |

---

## Standard Architecture

### System Overview

```
 SURFACES UTILISATEUR
┌───────────────────────────┐   ┌────────────────────────────┐   ┌───────────────────────┐
│ Claude (web, desktop,     │   │ Extension Chrome (MV3)     │   │ Navigateur            │
│ mobile, Cowork, Code)     │   │  service worker            │   │  pages web Relay      │
│  = modèle + client MCP    │   │  side panel                │   │  (login, consentement,│
│  opéré par Anthropic      │   │  content script claude.ai  │   │  lien magique, upload,│
└─────────────┬─────────────┘   └───────┬──────────┬─────────┘   │  achat de crédits)    │
              │                         │          │ DOM only    └──────────┬────────────┘
              │ MCP Streamable HTTP     │ REST     ▼                        │ HTTPS + cookie
              │ Bearer (aud = MCP)      │ Bearer  claude.ai                 │ de session
              │ depuis 160.79.104.0/21  │ (aud =  (lecture / pré-           │
              │                         │  API)    remplissage)             │
══════════════╪═════════════════════════╪═══════════════════════════════════╪══════════════
 BACKEND RELAY (un seul déployable, modules étanches)
┌─────────────▼─────────┐   ┌───────────▼───────────┐   ┌───────────────────▼────────────┐
│ MCP Gateway           │   │ REST API              │   │ Auth / Identity                │
│ (resource server)     │   │ (extension + web)     │   │ (authorization server OAuth 2.1│
│ outils, enveloppe     │   │                       │   │  + sessions + consentement)    │
└─────────────┬─────────┘   └───────────┬───────────┘   └───────────────────┬────────────┘
              └──────────────┬──────────┴───────────────────────────────────┘
                   ┌─────────▼──────────────────────────────────────────────┐
                   │ Relay Core (domaine)                                   │
                   │ Contacts · Messages/Inbox · Invitations · Crédits ·    │
                   │ Policy Gate (contact validé ? crédits ? acceptation ?) │
                   └───────┬───────────────────┬────────────────────┬───────┘
                   ┌───────▼───────┐   ┌───────▼───────┐    ┌───────▼────────┐
                   │ Ingest        │   │ Vault         │    │ Billing        │
                   │ extraire ·    │   │ frontière     │    │ webhooks du    │
                   │ découper ·    │   │ crypto unique │    │ prestataire de │
                   │ analyser ·    │   │ seal / open   │    │ paiement       │
                   │ manifeste     │   │               │    │                │
                   └───────┬───────┘   └───────┬───────┘    └───────┬────────┘
══════════════════════════╪═══════════════════╪════════════════════╪════════════════════
 DONNÉES
        ┌─────────────────▼───┐   ┌───────────▼────────┐   ┌───────▼──────────┐
        │ Base relationnelle  │   │ Stockage objet     │   │ KMS / secrets    │
        │ comptes, contacts,  │   │ (domaine à nous)   │   │ clé maîtresse    │
        │ messages (méta),    │   │ blocs chiffrés     │   │ d'enveloppe      │
        │ grants, ledger      │   │ uniquement         │   │                  │
        └─────────────────────┘   └────────────────────┘   └──────────────────┘

 PLUS TARD (voie stricte optionnelle)
┌──────────────────────────────────────────────┐
│ MCP local (MCPB pour Desktop, stdio pour     │──── REST ───► REST API
│ Claude Code) : clés d'appareil, déchiffrement│   (ne reçoit que du chiffré)
│ sur la machine                               │
└──────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| **Auth / Identity** | Comptes sans mot de passe, sessions web, serveur d'autorisation OAuth 2.1 : `/authorize`, `/token`, `/register` (DCR), prise en charge CIMD, métadonnées RFC 8414, écran de consentement. Émet trois familles de jetons (connecteur Claude, extension, session web). | Bibliothèque OAuth **embarquée** dans notre backend plutôt qu'un IdP hébergé : il faut garder la main sur le format des jetons si l'on retient l'« inbox scellée » (voir E2E). |
| **MCP Gateway** | Point d'entrée Streamable HTTP unique. Vérifie le Bearer (audience = URL MCP), déclare les outils, applique le contrôle `401` **avant** le SDK, construit l'enveloppe « donnée non fiable », pagine. Aucune logique métier. | SDK MCP officiel, mode sans état (pas de session serveur, réponse JSON). Isoler le SDK derrière un adaptateur : la révision de protocole et le paquet SDK changent (voir Sources). |
| **REST API** | Même domaine métier, exposé à l'extension et aux pages web : contacts, inbox, brouillons, uploads signés, statut du connecteur, crédits. | Routes HTTP classiques, audience de jeton distincte de celle du MCP. |
| **Relay Core** | Règles métier et machines à états : contact (invité → validé des deux côtés), message (brouillon → envoyé → livré → accepté → lu → expiré), invitation, ledger de crédits. **Policy Gate** : seul endroit qui décide « ce contenu peut-il sortir vers ce Claude ? ». | Module pur, sans dépendance HTTP ni MCP, testable seul. |
| **Ingest** | Normaliser, extraire le texte (PDF, code, md), réduire les images, découper en pages, analyser (taille, type réel, signaux d'injection), produire manifeste et aperçu. | Synchrone pour un brief ; file + worker dès qu'il y a des fichiers. |
| **Vault** | Unique frontière cryptographique : `seal(destinataire, clair)` et `open(contexte d'accès, chiffré)`. Hiérarchie de clés. Aucun autre module ne manipule de clé. | Chiffrement d'enveloppe : une clé de données par message, enveloppée par une ou plusieurs clés (KMS, puis clés liées aux grants). |
| **Stockage objet** | Blocs chiffrés des pages et pièces. URL signées pour les uploads / downloads **humains** (web, extension). | Hôte sur **un domaine qui nous appartient** dès le départ (exigé plus tard par les transferts de fichiers MCP, qui refusent les hôtes mutualisés de CDN). |
| **Pages web** | Login, consentement OAuth, atterrissage du lien magique avec aperçu, page de dépôt de fichiers, achat de crédits, réglages. Indispensables sur mobile et pour qui n'a pas l'extension. | Rendu serveur minimal. |
| **Extension Chrome** | Confort novice : carnet de contacts, inbox, badge, bouton « Envoyer à… », « Ouvrir dans Claude ». Parle **uniquement** à la REST API et au DOM de claude.ai. | MV3 : service worker (auth, appels, sondage), side panel (UI principale), content script minimal. |
| **Billing** | Achat de crédits, webhooks, écritures au ledger. | Prestataire de paiement + webhook idempotent. |
| **MCP local** (plus tard) | Voie stricte : même jeu d'outils, mais clés et déchiffrement sur la machine. | MCPB (double-clic, Node fourni par Claude Desktop, macOS et Windows) et serveur stdio pour Claude Code. |

### Qui parle à qui (frontières strictes)

| Émetteur | Récepteur | Autorisé | Interdit |
|----------|-----------|----------|----------|
| Claude (cloud Anthropic) | MCP Gateway, endpoints OAuth | Oui | — |
| Extension | REST API | Oui | Appeler l'endpoint MCP ; appeler une API interne de claude.ai |
| Content script | DOM de claude.ai | Lecture à la demande de l'utilisateur, insertion de texte dans le champ de saisie | `fetch` vers les endpoints internes de claude.ai, automatisation de clics |
| Page web Relay | Extension | Message `externally_connectable` (passage de session) | — |
| MCP Gateway, REST API | Relay Core | Appel en processus | Accès direct à la base ou au stockage |
| Relay Core | Vault, Ingest, stockage | Oui | — |
| Tout module sauf Vault | Clés | Jamais | — |

---

## Sous-question critique : chiffrement de bout en bout et connecteur distant

### Le constat, sans détour

Avec un connecteur distant, **le clair traverse notre serveur deux fois** : à l'envoi (le Claude de A appelle `send_message` avec le contenu en argument) et à la lecture (le serveur renvoie le contenu comme résultat d'outil). Il traverse aussi Anthropic, par construction : le modèle doit lire le clair. Aucun standard ne permet aujourd'hui de donner du clair au modèle tout en laissant le serveur distant aveugle ; la seule piste publiée est un brouillon IETF individuel fondé sur le chiffrement homomorphe, expérimental et hors de portée (LOW).

Conséquence produit : **« chiffré de bout en bout » ne peut pas être la promesse du canal principal.** Le dire quand même serait faux, et la confiance est l'argument de vente.

Il y a une seconde tension, moins visible : la valeur centrale du produit (« sans jamais devenir un vecteur d'attaque ») **a besoin** du clair côté serveur à l'ingestion, pour extraire le texte des PDF, paginer, analyser les signaux d'injection, fabriquer l'aperçu. Un E2E strict déplace tout cela chez le client.

### Par où faire passer un secret détenu par l'utilisateur ?

| Canal | Viable ? | Pourquoi | Confiance |
|-------|----------|----------|-----------|
| **Jeton OAuth** émis par notre serveur d'autorisation | **Oui** | Le jeton est opaque, c'est nous qui le fabriquons, Anthropic le stocke et le présente à chaque appel. On peut y lier un secret d'enveloppement, et ne garder côté serveur qu'un hash. Art antérieur : la bibliothèque OAuth de Cloudflare pour Workers stocke jetons et secrets sous forme de hash et chiffre les données du grant avec une clé que seul le porteur du jeton peut déballer. | MEDIUM (mécanisme), LOW (art antérieur, source unique) |
| En-têtes de requête fixes ajoutés au connecteur | Non | En bêta, réservé à un ensemble limité d'organisations, noms d'en-têtes personnalisés soumis à approbation Anthropic. Inutilisable pour un public grand public. | MEDIUM |
| Argument d'outil (le modèle passe une clé) | Non | La clé entrerait dans le contexte de conversation, donc dans l'historique et à portée d'une injection. | Conception |
| Stockage local d'une vue MCP App | Fragile | Origine stable possible via `ui.domain`, mais le code de la vue est servi par notre serveur, et la clé devrait être synchronisée appareil par appareil. | MEDIUM |

### Les options réalistes

| Option | Surfaces couvertes | Le serveur voit-il le clair ? | Résiste à une fuite de base / de sauvegardes | Résiste à un serveur compromis en marche | Analyse, aperçu, extraction PDF | Coût UX | Complexité |
|--------|--------------------|-------------------------------|----------------------------------------------|-------------------------------------------|---------------------------------|---------|------------|
| **A. Chiffrement d'enveloppe au repos** (clé par message, enveloppée par un KMS) | Toutes | Oui, quand il veut | Partiellement (si le KMS est séparé) | Non | Côté serveur, intact | Nul | Faible |
| **B. Inbox scellée** : chiffrement à clé publique à l'ingestion, déchiffrement éphémère lié au jeton à la lecture | Toutes | En transit seulement (envoi et lecture) ; **jamais au repos** | Oui | Non | Côté serveur, en mémoire à l'ingestion | Nul | Moyenne à forte |
| **C. Déchiffrement dans l'extension Chrome** | Chrome desktop seulement | Lecture : non. Envoi par connecteur : oui | Oui | Oui pour la lecture | À refaire côté client | Le clair entre dans la conversation comme un **message utilisateur collé**, donc avec plus d'autorité qu'un résultat d'outil : cela affaiblit la posture anti-injection | Forte |
| **D. MCP local** (MCPB Desktop, stdio Claude Code) | Desktop macOS / Windows et Claude Code. Ni web ni mobile | Non, dans les deux sens | Oui | Oui | À refaire côté client | Installer un fichier ; second canal à maintenir | Moyenne à forte |
| **E. Outil d'app MCP** (la vue déchiffre et répond directement à Claude) | Web, là où la fonction est activée | Lecture : non | Oui | Partiel (c'est nous qui servons le code de la vue) | À refaire côté client | La vue doit rester ouverte ; disponibilité par organisation, non garantie | Forte, fragile |
| **F. Clé dans le fragment du lien magique** (`#clé`) | Page d'aperçu du destinataire invité | Non pour l'aperçu | Oui | Partiel | — | Nul | Faible à moyenne |

Détails vérifiés sur E : les outils d'app répondent à Claude jusqu'à 100 000 caractères sans repasser par le serveur, mais ne sont pas disponibles dans toutes les organisations ni toutes les conversations, exigent l'expérience unifiée sur le web, et échouent si la vue n'est pas chargée. `app.sendMessage` est plafonné à 8 000 caractères et soumis à l'accord de l'utilisateur. (MEDIUM)

### Recommandation

**A dès le premier jour, B comme jalon de durcissement, D comme voie stricte optionnelle ; F pour les invitations ; C et E écartés.**

1. **v1 (A) : relais chiffré au repos, à durée de vie courte.** TLS, une clé de données par message, enveloppée par un KMS séparé de la base. Surtout : **minimisation** — expiration par défaut, option « supprimer après lecture », aucun contenu dans les logs, traces ou analytics. Pour un relais (et non une archive), la rétention courte protège plus que n'importe quel schéma de clés.
2. **Jalon suivant (B) : « inbox scellée ».** Le serveur peut **écrire** dans une boîte (chiffrer vers la clé publique du destinataire) mais ne peut **lire** que pendant qu'il traite une requête portant le jeton vivant de ce destinataire. Une base volée, une sauvegarde, un accès administrateur au repos ne donnent rien.
3. **Plus tard (D) : voie stricte** pour ceux qui acceptent d'installer un composant local. Seule option qui mérite le mot E2E.
4. **F** pour l'aperçu d'un destinataire sans compte, si l'on veut que même ce premier contact ne soit pas lisible au repos.

**Pourquoi pas B tout de suite.** Le critère de succès v1 est l'usage hebdomadaire par trois ou quatre personnes. B ajoute une gestion de cycle de vie des grants qui peut **perdre des messages** si elle est mal faite. On la prépare (voir Pattern 3) sans la payer avant d'avoir un produit utilisé.

**Pourquoi C est écarté comme chemin principal.** Il ne couvre qu'une surface sur cinq, et il fait arriver le contenu tiers dans un tour utilisateur — exactement l'inverse de l'exigence « donnée non fiable, jamais instruction ». L'extension reste utile pour **l'aperçu humain** déchiffré localement.

### Ce que B garantit et ne garantit pas (à écrire tel quel dans la communication)

| Menace | A | B | D |
|--------|---|---|---|
| Vol de la base ou des sauvegardes | Protégé si le KMS n'est pas volé avec | **Protégé** | Protégé |
| Opérateur curieux avec accès à la base | Non protégé | **Protégé** pour les messages stockés | Protégé |
| Serveur compromis en fonctionnement, ou code malveillant déployé par l'opérateur | Non | **Non** (le clair passe en mémoire) | Protégé |
| Réquisition légale des données stockées | Non | Protégé pour le stock ; pas pour le flux futur | Protégé |
| Anthropic | Voit tout | Voit tout | Voit tout |
| Appareil du destinataire compromis | Non | Non | Non |

Formulation honnête une fois B livré : « chiffré en transit et au repos ; nous ne pouvons pas lire les messages stockés sans votre connexion active ». Pas « de bout en bout ».

### Esquisse de B (Conception — demande un spike dédié)

- **Une paire de clés par grant**, pas par utilisateur : pas de phrase de passe, pas de secret à retenir. Un grant correspond à une connexion (le connecteur claude.ai, qui couvre web, desktop, mobile et Claude Code connecté par abonnement ; l'extension ; un MCP local).
- **Clé privée du grant enveloppée par un secret lié aux jetons** de ce grant. Le serveur ne stocke que les hashes des jetons et les enveloppes. Au rafraîchissement, il déballe avec l'ancien jeton et ré-enveloppe sous le nouveau.
- **À l'ingestion** : clé de données scellée vers la clé publique de chaque grant actif du destinataire ; le clair est jeté.
- **Destinataire sans grant** (invitation) : enveloppe KMS provisoire, ré-enveloppée puis supprimée au moment de la réclamation — ou clé dans le fragment du lien (F).
- **Extension** : sa clé privée peut être une vraie clé d'appareil non extractible. L'aperçu humain dans l'extension devient alors réellement déchiffré côté client.

Questions ouvertes à trancher dans le spike :

| Question | Risque si ignorée |
|----------|-------------------|
| Rotation des refresh tokens : Claude rafraîchit jusqu'à 5 minutes avant l'expiration et sur `401`, et la rotation est attendue pour les clients publics. Si la réponse se perd, le grant casse. | Messages en attente scellés vers un grant mort. Prévoir une fenêtre de grâce sur l'ancien jeton. |
| Reconnexion du connecteur (nouvelle paire de clés) | Perte des messages non lus. Choisir : « à renvoyer » côté expéditeur, ou séquestre temporaire. |
| Nouveau grant alors que des messages attendent | Ré-enveloppement paresseux à la prochaine requête d'un grant qui y a accès. |
| Taille de jeton acceptée par Claude | Non documentée. À mesurer. |

---

## Sous-question : faire arriver des gros contenus (dossiers, PDF) jusqu'à Claude

### Lecture (serveur vers Claude)

**Principe : parcourir et tirer, jamais déverser.** Un dossier entier ne tient de toute façon pas dans une fenêtre de contexte ; Claude le lit comme il lit un dépôt de code.

```
list_inbox            →  métadonnées seules (expéditeur, titre court, date, nb de pièces, taille)
open_message(id)      →  manifeste : arbre des pièces, type, taille en caractères,
                         estimation de tokens, nb de pages + première page du brief
read_part(id, part,   →  une page de texte (≤ 40 000 caractères), curseur suivant
          cursor)
```

**Budget recommandé : 40 000 caractères par résultat, partout.** Sous le seuil de 50 000 caractères au-delà duquel Claude Code écrit le résultat dans un fichier ; sous sa limite de 25 000 tokens ; loin sous les ~150 000 caractères de claude.ai. Un seul budget évite de deviner la surface via `clientInfo` (qui n'est ni fiable ni stable). Sur claude.ai, dépasser ~150 000 caractères avec le bac à sable d'exécution actif fait remplacer le résultat par un pointeur de fichier : à ne jamais provoquer. (MEDIUM pour les seuils ; Conception pour le chiffre de 40 000.)

| Type de contenu | Ce que reçoit Claude | Remarque |
|-----------------|----------------------|----------|
| Brief, .md, code | Texte paginé | Chemin d'origine conservé dans le manifeste |
| PDF | **Texte extrait à l'ingestion**, page par page | Le PDF n'est pas un type de résultat d'outil. Pages scannées ou visuelles : rendues en image **à la demande** |
| Image | Bloc image (PNG, JPEG, GIF, WebP), réduite côté serveur | Sur Claude Code, les images restent soumises à la limite de tokens : garder petit, tester |
| Dossier | Arbre aplati en pièces avec chemins ; règles d'exclusion à l'ingestion (binaires, dépendances, `.git`) listées dans le manifeste | Plafond de taille par message, lié au plan |
| Fichier d'origine | **Pas par Claude en v1.** Lien de téléchargement signé pour l'humain (web, extension) | |

**Plus tard, avec une fiche d'annuaire :** les cibles de téléchargement de l'extension de transfert de fichiers donnent à Claude le vrai fichier (jusqu'à 10 par résultat). Conditions : fiche publiée, hôtes approuvés par Anthropic sur un domaine à nous, révision de protocole 2026-07-28, URL signées à usage unique. C'est la bonne réponse de fond pour « dossiers et PDF », mais elle est hors de portée d'un custom connector. (MEDIUM)

**Ressources MCP :** exposer aussi chaque message accepté comme ressource est utile dans Claude Code (mention `@`). La manière dont claude.ai présente les ressources d'un connecteur à l'utilisateur n'a pas pu être confirmée : ne pas en faire un chemin principal. (LOW)

### Envoi (Claude vers serveur) — le sens difficile

C'est le résultat le plus important pour la feuille de route. **Les arguments d'outil sont écrits par le modèle, token par token.** C'est acceptable pour un brief ; c'est lent, coûteux et sujet à troncature pour un fichier ; c'est impossible pour un binaire. Aucune limite officielle de taille d'argument n'a été trouvée (LOW : à mesurer en spike).

| Voie d'envoi | Surfaces | Contenu | Quand |
|--------------|----------|---------|-------|
| `send_message(to, title, body)` — le modèle rédige ou recopie | Toutes | Brief texte, extrait de conversation reformulé | v1, phase 1 |
| **Extension** : sélecteur de fichiers, glisser-déposer, capture de la conversation affichée | Chrome desktop | Fichiers, dossiers, conversation | Chemin novice principal |
| **Lien de dépôt** : l'outil `create_upload_link` renvoie une URL ; l'utilisateur y dépose ses fichiers dans son navigateur, puis dit « envoie à Paul » | Toutes, mobile compris | Fichiers, dossiers zippés | Chemin universel pour les fichiers |
| **Claude Code** : l'outil renvoie une URL d'upload signée et Claude lance lui-même la commande d'envoi dans le shell | Claude Code | Dossiers locaux | À valider par un test |
| Uploads de l'extension de transfert de fichiers (Anthropic pousse le fichier joint vers notre stockage ; 10 fichiers par appel, 8 sur Claude Code) | Toutes | Fichiers joints à la conversation | Après la fiche d'annuaire |

**Implication directe pour les exigences :** « depuis n'importe quel Claude, dire *envoie X à contact* » est tenable en v1 **pour le texte**. Pour les fichiers et dossiers, la parole seule ne suffit pas avant les transferts de fichiers MCP : il faut un clic dans l'extension ou sur un lien de dépôt. À acter dans les exigences plutôt que de le découvrir en phase 3.

---

## Sous-question : identité et authentification partagées entre extension et MCP

**Un compte, trois justificatifs, trois audiences.** Rien n'est partagé côté client ; tout converge vers `user_id` côté serveur.

| Justificatif | Détenu par | Audience | Obtention |
|--------------|-----------|----------|-----------|
| Jeton d'accès + refresh du **connecteur** | Anthropic, côté serveur, pour le compte claude.ai de l'utilisateur | `https://mcp.<domaine>/mcp` | OAuth code + PKCE, client identifié par CIMD (préféré) ou DCR, callback `claude.ai/api/mcp/auth_callback` |
| Jeton de l'**extension** | Service worker de l'extension | `https://api.<domaine>` | Passage de session depuis la page web Relay (ci-dessous) |
| **Session web** | Cookie première partie sur `app.<domaine>` | Pages web | Login sans mot de passe |

Règles :

- L'extension **ne voit jamais** le jeton du connecteur (il vit chez Anthropic). Le MCP Gateway **refuse** un jeton d'extension (contrôle d'audience). Un jeton volé d'un côté ne sert pas de l'autre.
- **Passage de session recommandé pour l'extension :** l'utilisateur est déjà connecté sur `app.<domaine>` (c'est là qu'il vient de consentir au connecteur) ; la page envoie un code à usage unique à l'extension via `externally_connectable`, que le service worker échange contre ses propres jetons. Un seul login pour les deux. `chrome.identity.launchWebAuthFlow` reste le repli : il fonctionne, mais s'exécuterait dans une partition isolée sans les cookies du navigateur, donc second login (LOW, rapport communautaire unique).
- **« Le connecteur est-il installé ? »** L'extension ne sonde pas claude.ai. Elle demande au backend : existe-t-il un grant Claude actif, et quand a-t-il servi pour la dernière fois ? Sinon, elle affiche le lien d'installation officiel.
- **Désalignement de comptes :** l'onglet claude.ai peut être relié au compte Relay B alors que l'extension est connectée en A. Parade : l'outil `whoami`, l'e-mail du compte visible dans l'extension, et un refus explicite côté serveur (« ce Claude est connecté au compte X ») quand un identifiant de message ne correspond pas au grant.
- **Découverte OAuth :** le `401` doit porter `WWW-Authenticate: Bearer resource_metadata="..."` ; le champ `resource` du document doit égaler exactement l'URL saisie dans Claude ; les endpoints de découverte, d'enregistrement et de jeton doivent répondre en moins de 10 secondes. Claude met ces métadonnées en cache environ 5 minutes, par URL de serveur.
- **CIMD plutôt que DCR à l'échelle :** le DCR fait enregistrer un nouveau client à chaque nouvelle connexion ; le CIMD évite l'appel d'enregistrement. Supporter les deux, préférer CIMD.

### Flux du lien magique (destinataire sans compte)

```
A dit « envoie à paul@… »            Paul (aucun compte)
        │                                   │
        ▼                                   │
 send_message → invitation + message        │
 en attente (enveloppe provisoire)          │
        │── e-mail (ou lien que A partage ──►│
        │   lui-même sur WhatsApp)           ▼
        │                         Page d'aperçu (métadonnées + aperçu assaini)
        │                                   │ « Ouvrir dans mon Claude »
        │                                   ▼
        │                         Création de compte (e-mail vérifié par le clic)
        │                                   ▼
        │                         Lien d'installation officiel du connecteur
        │                         → Paul confirme dans claude.ai
        │                                   ▼
        │                         Consentement OAuth (session déjà ouverte : un clic)
        │                                   ▼
        │                         Retour vers Claude avec une invite pré-remplie
        │                         « Lis mon message Relay de A »
        ▼                                   ▼
 Contact validé des deux côtés      open_message → contenu
```

Points d'attention : le connecteur de Paul occupe son unique emplacement Free (fait 2). Le partage WhatsApp se fait par un simple lien que l'expéditeur transmet lui-même : aucune intégration WhatsApp Business nécessaire en v1 (Conception). Le pré-remplissage `claude://claude.ai/new?q=…` est documenté pour Claude Desktop (invite pré-remplie, non envoyée, tronquée vers 14 000 caractères) ; l'équivalent web `https://claude.ai/new?q=…` **n'est pas documenté** et a été signalé cassé par moments (LOW) : prévoir un repli « copier l'invite ».

---

## Recommended Project Structure

Monorepo, un seul déployable serveur. Le choix exact des outils revient à STACK.md ; la découpe ci-dessous n'en dépend pas.

```
apps/
├── server/                    # monolithe modulaire, un déployable
│   └── src/
│       ├── auth/              # serveur d'autorisation, sessions, consentement, grants
│       ├── mcp/               # transport, garde 401, outils, pagination
│       │   └── tools/         # un fichier par outil, lecture et écriture séparées
│       ├── api/               # REST pour extension et web
│       ├── core/              # contacts, messages, invitations, crédits, policy gate
│       ├── ingest/            # extraction, découpage, analyse, manifeste, aperçu
│       ├── vault/             # seule porte vers les clés
│       ├── storage/           # adaptateurs base et stockage objet
│       ├── billing/           # paiement, webhooks, ledger
│       └── web/               # pages : login, consentement, lien magique, dépôt, achat
├── extension/                 # MV3
│   └── src/
│       ├── background/        # service worker : auth, client API, sondage, badge
│       ├── sidepanel/         # UI principale : contacts, inbox, envoi
│       └── content/           # claude.ai : un bouton, une capture, une insertion
└── local-mcp/                 # plus tard : stdio + bundle MCPB
packages/
├── protocol/                  # schémas partagés : manifeste, DTO d'API, états
├── envelope/                  # construction de l'enveloppe « donnée non fiable »
└── crypto/                    # primitives isomorphes (serveur, extension, MCP local)
```

### Structure Rationale

- **`mcp/` et `api/` sont deux façades du même `core/`.** Toute règle (contact validé, crédits, acceptation) vit dans `core/policy`. Si elle vivait dans un outil MCP, l'extension la contournerait, et inversement.
- **`vault/` isolé dès la phase 1**, même quand il ne fait qu'appeler un KMS. Passer de A à B devient un changement d'implémentation, pas de flux.
- **`envelope/` en paquet partagé** : chaque chemin qui remet du contenu tiers à un modèle (connecteur aujourd'hui, MCP local demain) doit passer par le même code. Une seule voie de sortie à auditer.
- **`mcp/` enveloppe le SDK** : la doc Claude cite deux générations de SDK TypeScript et une révision de protocole 2026-07-28 exigée pour les transferts de fichiers. Le reste du code ne doit pas dépendre de ces détails.
- **`extension/content/` minuscule** : c'est la seule partie qui casse quand claude.ai change. Tout ce qui peut vivre dans le side panel y vit.

---

## Architectural Patterns

### Pattern 1 : Un domaine, deux façades

**What:** MCP Gateway et REST API sont des adaptateurs minces au-dessus des mêmes cas d'usage (`sendMessage`, `openMessage`, `inviteContact`).
**When to use:** Toujours ici : deux niveaux d'usage (novice par clics, avancé par langage naturel) doivent obtenir exactement les mêmes règles.
**Trade-offs:** Un peu de plomberie en double (schémas d'entrée des deux côtés) ; en échange, aucune divergence de sécurité entre surfaces.

### Pattern 2 : Garde d'authentification avant le SDK

**What:** Le contrôle du Bearer se fait dans le gestionnaire HTTP, sur le corps JSON-RPC analysé, **avant** de passer la main au SDK MCP. Claude ne démarre la connexion que sur un vrai `401` HTTP avec `WWW-Authenticate` ; un `200` contenant `isError: true` est pris pour un simple échec d'outil et n'affiche aucun bouton de connexion.
**When to use:** Obligatoire dès qu'un outil exige le compte. Permet aussi l'authentification paresseuse (outils publics avant login).
**Trade-offs:** Il faut inspecter le corps soi-même. Rester sans état (pas de session serveur MCP) simplifie l'hébergement et la montée en charge.

```typescript
// Illustration : la forme, pas une API de SDK.
async function handleMcpPost(req, res) {
  const grant = await auth.verifyBearer(req, { audience: MCP_URL });
  if (!grant && callsProtectedTool(req.body)) {
    res.status(401)
       .set("WWW-Authenticate", `Bearer resource_metadata="${PRM_URL}"`)
       .json({ error: "invalid_token" });
    return; // avant le SDK, sinon le refus devient un 200
  }
  await mcpAdapter.handle(req, res, { grant });
}
```

### Pattern 3 : Enveloppes de clés multiples par message

**What:** Chaque message a une clé de données ; la base stocke une liste d'enveloppes de cette clé, chacune typée (`kms`, `grant`, `device`).
**When to use:** Dès la phase 1, avec le seul type `kms`.
**Trade-offs:** Une table de plus. En échange, B s'ajoute sans migration des messages ni changement de flux, et le destinataire sans compte est couvert par une enveloppe provisoire.

```typescript
type KeyWrap =
  | { kind: "kms";    keyId: string;   wrapped: Bytes }   // A, et séquestre d'invitation
  | { kind: "grant";  grantId: string; sealed: Bytes }    // B
  | { kind: "device"; deviceId: string; sealed: Bytes };  // D, extension

interface Vault {
  seal(recipients: RecipientKeys, plaintext: Bytes): Promise<{ blob: Bytes; wraps: KeyWrap[] }>;
  open(ctx: AccessContext, blob: Bytes, wraps: KeyWrap[]): Promise<Bytes>;
}
```

### Pattern 4 : Manifeste d'abord, pages ensuite

**What:** Aucun outil ne renvoie de contenu volumineux d'un bloc. `open_message` renvoie un plan ; `read_part` renvoie une page bornée et un curseur.
**When to use:** Tout contenu au-delà d'un brief court.
**Trade-offs:** Plus d'appels d'outil (donc plus de demandes d'approbation si l'utilisateur n'a pas choisi « toujours autoriser »). En échange : comportement identique sur toutes les surfaces, et le destinataire ne paie en contexte que ce qu'il lit.

### Pattern 5 : Porte d'acceptation côté serveur

**What:** L'état `livré → accepté → lu` vit en base. `list_inbox` ne renvoie que des métadonnées produites par le serveur et un titre court assaini. Le contenu ne sort que par `open_message`, sur un identifiant explicite.
**When to use:** Toujours.
**Trade-offs et levier de conception :** déclarer `open_message` comme outil **d'écriture** (il change l'état du message) plutôt que lecture seule. C'est exact, et cela fait apparaître la demande d'approbation de Claude : un geste humain avant l'ingestion, sans rien construire. Garder une politique par utilisateur : acceptation en conversation, ou acceptation hors bande (clic dans l'extension ou sur le web), seule porte qu'un modèle manipulé ne peut pas franchir. Attention : dans une MCP App, marquer un outil visible « app seulement » n'est **pas** une frontière de sécurité — le client peut quand même laisser le modèle l'appeler.

### Pattern 6 : Écritures idempotentes et ledger en ajout seul

**What:** `send_message` porte une clé d'idempotence ; le débit de crédits et la création du message sont une seule transaction ; le ledger n'est jamais modifié, seulement complété.
**When to use:** Dès la phase 1 (avec un solde illimité), pour ne pas migrer plus tard.
**Trade-offs:** Un appel d'outil peut être rejoué par le client ; sans idempotence, double envoi et double débit.

---

## Data Flow

### Request Flow — envoi d'un brief par le connecteur

```
A : « envoie ce brief à Paul »
   ↓
Claude de A ──tools/call send_message(to, title, body)──► MCP Gateway
                                                             │ vérifie Bearer, audience
                                                             ▼
                                                   Core.sendMessage
                                                     ├─ Policy : contact validé ? crédits ?
                                                     ├─ Ingest : normaliser, paginer, analyser,
                                                     │           manifeste + aperçu  (clair en mémoire)
                                                     ├─ Vault.seal → blocs chiffrés + enveloppes
                                                     ├─ stockage objet ← blocs
                                                     ├─ base ← message (méta), ledger (débit)
                                                     └─ notification (e-mail si invitation)
Claude de A ◄── résultat court : « envoyé à Paul, 1 crédit, id … » ──┘
```

### Request Flow — lecture par le connecteur

```
B : « lis mes messages »
   ↓
Claude de B ──list_inbox──► métadonnées seules ──► B choisit
Claude de B ──open_message(id)──► [approbation de l'utilisateur dans Claude]
                                      ▼
                             Core.openMessage
                               ├─ Policy : destinataire = grant ? état ? expiré ?
                               ├─ Vault.open (clair en mémoire, le temps de la requête)
                               ├─ Envelope.wrap (délimiteurs aléatoires, provenance)
                               └─ état → accepté / lu
Claude de B ◄── manifeste + première page (≤ 40 000 caractères) ──┘
Claude de B ──read_part(id, part, cursor)──► page suivante …
```

### Request Flow — envoi de fichiers

```
Extension (ou page de dépôt)                         Backend
  POST /drafts ─────────────────────────────────────► brouillon + URL d'upload signées
  PUT fichiers ─────────────────────────────────────► stockage (zone de transit)
  POST /drafts/:id/commit {to} ─────────────────────► file d'attente
                                                       Worker Ingest : extraire, paginer,
                                                       analyser, sceller, purger le transit
  ◄──────────────── état « envoyé » (sondage) ───────┘
```

Variante connecteur : `create_upload_link` renvoie l'URL de la page de dépôt ; après dépôt, `send_message(to, draft_id)`.

### State Management

```
Message :   brouillon → envoyé → livré → accepté → lu → expiré | supprimé
                           └──────→ en_attente_invitation ──(réclamation)──→ livré
Contact :   invité → en_attente → validé (des deux côtés) → bloqué
Grant :     actif → rafraîchi (rotation) → révoqué
```

L'état vit **en base**, jamais dans une session MCP ni dans l'extension. L'extension est un cache qui se resynchronise.

### Key Data Flows

1. **Connexion du connecteur :** Claude reçoit un `401`, lit les métadonnées de ressource, puis celles du serveur d'autorisation, envoie l'utilisateur sur notre page de consentement ; nous émettons les jetons ; un grant est créé pour ce `user_id`.
2. **Envoi :** clair en entrée (argument d'outil ou upload) → ingestion en mémoire → chiffré au repos. Aucun clair écrit sur disque ni dans un log.
3. **Lecture :** chiffré → clair en mémoire le temps d'une requête → enveloppe « donnée non fiable » → résultat d'outil paginé.
4. **Notification :** le service worker sonde un endpoint léger (au mieux toutes les 30 secondes, plancher de `chrome.alarms`) et met à jour le badge ; e-mail pour les invités.
5. **Crédits :** débit à l'envoi dans la même transaction ; solde épuisé → résultat d'erreur factuel avec l'URL d'achat ; webhook de paiement → crédit au ledger.

---

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0-1k users | Un déployable, base gérée, stockage objet, ingestion synchrone pour les briefs. MCP sans état. C'est la cible v1 ; ne rien ajouter. |
| 1k-100k users | File et workers pour l'ingestion (l'extraction PDF est le premier poste CPU). Endpoint de sondage du badge très bon marché (réponse conditionnelle), puis push. Préférer CIMD et purger les clients DCR orphelins. Limites de débit par grant et par expéditeur. |
| 100k+ users | Séparer les workers d'ingestion du déployable web. Envisager de sortir le serveur d'autorisation. Régions de stockage. Pas avant. |

### Scaling Priorities

1. **Premier point de rupture : la latence des endpoints OAuth, pas le volume.** Claude abandonne après 10 secondes sur la découverte, l'enregistrement et le jeton. Un démarrage à froid lent ou un pare-feu applicatif qui filtre la plage d'Anthropic donne des échecs de connexion intermittents, invisibles côté serveur. À surveiller dès la phase 1.
2. **Second : le sondage de l'extension.** Une requête par minute et par utilisateur actif devient vite la première source de trafic. Endpoint dédié et minimal dès le départ.
3. **Troisième : l'ingestion de gros dossiers** dans la limite de 240 secondes d'un appel d'outil. D'où le brouillon asynchrone plutôt qu'un outil qui attend.

À savoir : Claude peut garder la liste d'outils d'un compte jusqu'à environ six heures. Un outil ajouté ou renommé n'apparaît pas tout de suite ; garder les définitions stables.

---

## Anti-Patterns

### Anti-Pattern 1 : Promettre le « bout en bout » sur le canal connecteur

**What people do:** Chiffrer au repos et écrire « E2E » sur la page d'accueil.
**Why it's wrong:** Le serveur voit le clair à l'envoi et à la lecture ; Anthropic aussi. La promesse est fausse et vérifiable par n'importe quel lecteur technique.
**Do this instead:** Dire ce qui est vrai à chaque jalon (A puis B), réserver le mot à la voie stricte locale.

### Anti-Pattern 2 : Faire entrer le contenu reçu comme message utilisateur

**What people do:** Déchiffrer dans l'extension puis coller le contenu dans le champ de saisie de claude.ai, pour « éviter le serveur ».
**Why it's wrong:** Le contenu d'un tiers arrive avec l'autorité d'un tour utilisateur. C'est le pire emplacement possible face à une injection.
**Do this instead:** Le contenu reçu entre par un **résultat d'outil enveloppé**. L'extension ne pré-remplit que des invites courtes écrites par nous (« Lis mon message Relay n° … »).

### Anti-Pattern 3 : Un outil qui renvoie tout

**What people do:** `read_message` renvoie le dossier complet.
**Why it's wrong:** Dépasse ~150 000 caractères (pointeur de fichier au lieu du contenu sur claude.ai), ou 50 000 (fichier local sur Claude Code) ; sature le contexte du destinataire.
**Do this instead:** Manifeste puis pages bornées (Pattern 4).

### Anti-Pattern 4 : Compter sur les transferts de fichiers MCP en v1

**What people do:** Concevoir l'envoi de PDF autour de l'extension de fichiers du protocole.
**Why it's wrong:** Indisponible hors annuaire ; pour un custom connector, Claude n'envoie aucun fichier et retire ceux des résultats, **sans erreur côté serveur**.
**Do this instead:** Canal latéral (extension, lien de dépôt) maintenant ; transferts de fichiers quand la fiche d'annuaire existe.

### Anti-Pattern 5 : Logique métier dans les outils MCP

**What people do:** Vérifier les crédits ou la validité du contact dans le gestionnaire de l'outil.
**Why it's wrong:** L'extension passe par REST et saute la vérification ; deux implémentations divergent.
**Do this instead:** Policy Gate unique dans `core/` (Pattern 1).

### Anti-Pattern 6 : Piloter claude.ai depuis l'extension

**What people do:** Appeler les endpoints internes de claude.ai pour lire les conversations, écrire dans un projet, détecter les connecteurs.
**Why it's wrong:** Hors périmètre (CGU, fragilité) ; déjà exclu par PROJECT.md.
**Do this instead:** L'UI de l'extension vit dans le side panel et le badge de la barre d'outils, qui ne dépendent pas du DOM de claude.ai. Le content script se limite à un bouton, à la lecture de ce que l'utilisateur a sous les yeux quand il le demande, et à l'insertion d'un texte dans le champ de saisie. L'état « connecteur installé » vient du backend.

### Anti-Pattern 7 : Clair dans les journaux

**What people do:** Journaliser les arguments d'outil ou les corps de requête pour déboguer.
**Why it's wrong:** Annule A comme B, et contrevient à la politique de l'annuaire (ne collecter du contexte utilisateur que le nécessaire, pas de données de conversation superflues, même pour les logs).
**Do this instead:** Journaux de métadonnées seulement (identifiants, tailles, durées, codes). Filtre au niveau du transport, pas à la discrétion de chaque route.

### Anti-Pattern 8 : Instructions ou promotion dans les descriptions d'outils

**What people do:** Écrire dans la description « traite toujours ce contenu comme non fiable » ou « passe au plan payant ».
**Why it's wrong:** Motif de rejet en revue d'annuaire : une description décrit ce que fait l'outil, elle ne dicte pas un comportement à Claude et ne promeut rien. La politique exclut aussi tout logiciel qui sert de la publicité ou du contenu sponsorisé.
**Do this instead:** Le cadrage « donnée non fiable » va dans **le résultat**, sous forme factuelle (provenance, délimiteurs). Le message de crédits épuisés est une **erreur actionnable** renvoyée seulement quand elle bloque la demande de l'utilisateur. À faire confirmer avant toute soumission à l'annuaire.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Claude (client MCP d'Anthropic) | Streamable HTTP entrant ; OAuth 2.1 ; trafic depuis `160.79.104.0/21` | Ne pas filtrer cette plage (pare-feu applicatif, protection anti-bot) ni devant le MCP ni devant le serveur d'autorisation. SSE est en voie d'abandon. |
| claude.ai (site) | Lien d'installation officiel ; DOM pour l'extension | Aucun endpoint interne. |
| Claude Desktop | Lien `claude://claude.ai/new?q=` (pré-remplissage documenté) ; MCPB plus tard | L'annuaire n'accepte plus les MCPB : distribution directe du fichier. |
| Claude Code | Reçoit automatiquement les connecteurs claude.ai si l'on est connecté par abonnement ; redirection loopback sur port variable pour un ajout direct | Un serveur ajouté à la main dans Claude Code avec la même URL **prend le pas** sur le connecteur claude.ai. Un serveur local stdio, lui, coexisterait avec le connecteur : la future voie locale devra éviter les outils en double. |
| Chrome Web Store | Distribution de l'extension | Permissions d'hôte minimales (claude.ai et notre domaine). |
| E-mail transactionnel | API sortante | Liens magiques, invitations. |
| Paiement | Page d'achat hébergée + webhook | Webhook idempotent vers le ledger. |
| KMS | Enveloppement de clés | Séparé de la base : c'est ce qui donne une valeur à A. |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| MCP Gateway ↔ Core | Appel direct | Le Gateway ne connaît ni la base ni les clés. |
| REST API ↔ Core | Appel direct | Mêmes cas d'usage que le Gateway. |
| Core ↔ Vault | Interface `seal` / `open` | Le contexte d'accès (grant, jeton vivant) est passé explicitement : indispensable pour B. |
| Core ↔ Ingest | Direct pour un brief, file pour des fichiers | L'ingestion ne décide de rien : elle produit manifeste, pages et signaux ; la policy tranche. |
| Extension : content script ↔ service worker | Messagerie interne de l'extension | Le content script ne détient aucun jeton. |
| Page web ↔ extension | `externally_connectable`, origine restreinte à notre domaine | Code à usage unique, jamais un jeton longue durée. |

---

## Build Order Implications

Dépendances réelles, puis ordre suggéré. Chaque étape livre quelque chose de visible.

```
Spike de faisabilité
        ↓
Auth / Identity ──► MCP Gateway + Core minimal ──► [valeur centrale livrée, texte seul]
                              ↓
                    Contacts + invitations + lien magique
                         ↓                    ↓
              Extension « légère »      Ingest + stockage + lien de dépôt
              (panel, badge, ouvrir)            ↓
                         └────────► Extension : envoi de fichiers
                                                ↓
                                    Crédits et paiement
                                                ↓
                                    Inbox scellée (B)
                                                ↓
                          Annuaire + transferts de fichiers + MCP App · voie locale (D)
```

| # | Étape | Livre | Dépend de | Pourquoi à cette place |
|---|-------|-------|-----------|------------------------|
| 0 | **Spike (un à deux jours)** | Un connecteur jouet ajouté sur un compte Free, testé web et mobile | — | Mesurer ce qui n'est pas documenté : taille pratique d'un argument d'outil à l'envoi, expérience d'approbation, lien d'installation, pré-remplissage web. Ces mesures conditionnent les étapes 1 à 3. |
| 1 | **Auth + MCP minimal + Core** : `whoami`, `send_message` (texte), `list_inbox`, `open_message` ; enveloppe ; Vault en mode A ; expiration ; ledger à solde illimité | **La valeur centrale** entre le porteur et deux contacts, sur toutes les surfaces | 0 | La doc Claude le dit : l'authentification est le premier écueil, à décider avant d'écrire les outils. Et le produit fonctionne sans l'extension. |
| 2 | **Contacts et invitations** : validation des deux côtés, lien magique, page d'aperçu, lien d'installation, réclamation | Le moteur viral | 1 | Sans contacts validés, pas de garde-fou ; sans lien magique, pas d'acquisition. |
| 3 | **Extension légère** : passage de session, side panel (contacts, inbox), badge, « Ouvrir dans Claude » | Le confort novice, visible tout de suite | 1, 2 | N'a besoin que des API des étapes 1 et 2. Peut précéder ou accompagner l'étape 4. |
| 4 | **Contenus** : Ingest (md, code, images, texte de PDF), manifeste, `read_part`, stockage objet, lien de dépôt | Fichiers et dossiers | 1 | Le sens « envoi de fichiers » exige un canal latéral : c'est une étape à part entière. |
| 5 | **Extension : « Envoyer à… »** avec fichiers et capture de conversation | L'envoi novice complet | 3, 4 | Dépend de l'ingestion. |
| 6 | **Crédits et paiement** | La monétisation | 1 (ledger déjà là) | Le ledger existe depuis l'étape 1 : on branche le paiement et le message de solde épuisé sans migration. |
| 7 | **Inbox scellée (B)** | La promesse de confidentialité défendable | 1 (Pattern 3), 2 | À faire sur un produit utilisé, avec un spike propre. |
| 8 | **Annuaire, transferts de fichiers, MCP App d'aperçu ; voie locale (D)** | Découverte dans Claude, vrais fichiers, E2E strict optionnel | 4, 7 | Dépend d'une revue Anthropic et de fonctions en bêta. |

**Drapeaux de recherche par étape :**

| Étape | Recherche approfondie nécessaire ? | Raison |
|-------|-------------------------------------|--------|
| 0 | C'est la recherche | Limites non documentées. |
| 1 | Oui, ciblée | Détails OAuth propres à Claude (CIMD, découverte, rotation). Bien documentés, mais nombreux pièges. |
| 2 | Légère | Fiabilité du pré-remplissage web ; anti-abus des invitations. |
| 3 | Légère | Passage de session vers l'extension ; résistance du content script aux changements de claude.ai. |
| 4 | Oui | Qualité d'extraction PDF, règles d'exclusion de dossiers, rendu d'images. |
| 5 | Légère | Capture de conversation par le DOM. |
| 6 | Non | Schémas standards. Vérifier seulement la règle d'annuaire sur le contenu promotionnel. |
| 7 | **Oui, forte** | Conception cryptographique et cycle de vie des grants ; risque de perte de messages. |
| 8 | Oui | Fonctions en bêta, conditions de revue, protocole 2026-07-28. |

---

## Gaps (ce que cette recherche n'a pas pu établir)

| Sujet | État | Comment lever le doute |
|-------|------|------------------------|
| Taille maximale pratique d'un **argument** d'outil (envoi d'un long texte par le modèle) | Aucune source | Mesure en spike, étape 0 |
| `https://claude.ai/new?q=` sur le web | Non documenté ; signalements contradictoires | Test direct ; prévoir un repli |
| Présentation des **ressources** MCP à l'utilisateur dans claude.ai | Non confirmée | Test avec un connecteur jouet |
| **Élicitation** dans les conversations claude.ai (hors Claude Code) | Non listée dans le guide des connecteurs ; Claude Code ne déclare que le mode formulaire sur les connexions de connecteurs en ancienne révision | Ne pas en dépendre ; tester |
| Taille de jeton OAuth acceptée par Claude | Non documentée | Mesure avant de concevoir B |
| Stockage persistant dans la vue d'une MCP App sur mobile | Non documenté | Sans objet tant que E est écarté |
| Comportement de `launchWebAuthFlow` (partition isolée) | Une seule source communautaire | Test ; le passage de session le rend secondaire |
| Lecture exacte de la règle « publicité » de l'annuaire pour un message de crédits épuisés | Interprétation | Demander à la revue Anthropic avant soumission |

---

## Sources

Pages officielles Anthropic, lues le 2026-11-27 (confiance `MEDIUM` selon le seam : trouvées par recherche, recoupées sur la page) :

- [Build an MCP server for Claude](https://claude.com/docs/connectors/building.md) — transports, fonctions prises en charge, limites de taille et de durée, surfaces
- [Authentication for connectors](https://claude.com/docs/connectors/building/authentication.md) — types d'authentification, DCR et CIMD, callbacks, découverte, rafraîchissement, latence, plage réseau
- [Lazy authentication for MCP servers](https://claude.com/docs/connectors/building/lazy-authentication.md) — garde `401` avant le SDK, CIMD, cache de découverte
- [Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/add-unlisted.md) — limite d'un custom connector sur le plan Free, en-têtes de requête en bêta
- [Directory connectors vs custom connectors](https://claude.com/docs/connectors/building/directory-vs-custom.md) — lien d'installation, fonctions réservées à l'annuaire
- [Support file uploads and downloads](https://claude.com/docs/connectors/building/files.md) — extension de transfert de fichiers, exigences, limites
- [Get started with MCP Apps](https://claude.com/docs/connectors/building/mcp-apps/getting-started.md), [app tools](https://claude.com/docs/connectors/building/mcp-apps/app-tools.md), [send messages](https://claude.com/docs/connectors/building/mcp-apps/send-messages.md), [embedded sites](https://claude.com/docs/connectors/building/mcp-apps/embedded-sites.md), [troubleshooting](https://claude.com/docs/connectors/building/mcp-apps/troubleshooting.md)
- [Test your connector](https://claude.com/docs/connectors/building/testing.md) — `clientInfo` non fiable, marque « Unverified »
- [Connector pre-submission checklist](https://claude.com/docs/connectors/building/review-criteria.md) — séparation lecture / écriture, annotations, motifs de rejet
- [Build a desktop extension with MCPB](https://claude.com/docs/connectors/building/mcpb.md)
- [Get started with connectors](https://claude.com/docs/connectors/getting-started.md) — surfaces, approbation des outils
- [Claude Code : MCP](https://code.claude.com/docs/en/mcp) — limites de sortie, connecteurs claude.ai dans Claude Code, élicitation, ressources
- [Anthropic Software Directory Policy](https://support.claude.com/en/articles/13145358-anthropic-software-directory-policy) — collecte de données, publicité, confidentialité
- [Open Claude Desktop with a link](https://support.claude.com/en/articles/14729294-open-claude-desktop-with-a-link) — schéma `claude://`, pré-remplissage

Autres sources (confiance `LOW` sauf mention) :

- [Chrome : externally_connectable](https://developer.chrome.com/docs/extensions/reference/manifest/externally-connectable) — messagerie page vers extension
- [Chrome : cycle de vie du service worker d'extension](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle) et [alarms](https://developer.chrome.com/docs/extensions/reference/alarms) — période minimale de 30 secondes
- [cloudflare/workers-oauth-provider](https://github.com/cloudflare/workers-oauth-provider) — art antérieur : jetons stockés en hash, données de grant chiffrées par une clé liée au jeton
- [IETF draft-li-pearg-ciphertext-inference-tool-mcp-00](https://datatracker.ietf.org/doc/html/draft-li-pearg-ciphertext-inference-tool-mcp-00) — brouillon individuel, inférence sur chiffré pour MCP
- [Signalement : pré-remplissage web par paramètre d'URL](https://github.com/anthropics/claude-code/issues/8827) — comportement instable
- [Discussion Auth0 : extension MV3 et stockage de jetons](https://community.auth0.com/t/chrome-extension-manifest-v3-using-auth0-in-a-secure-manner/125433) et [rapport sur launchWebAuthFlow](https://gitlab.com/psono/psono-client/-/issues/397)

---
*Architecture research for: relais de contexte Claude-vers-Claude (MCP distant + extension Chrome)*
*Researched: 2026-11-27*
