# Stack Research

**Domain:** Partage de contexte de Claude à Claude : serveur MCP distant (connecteur custom claude.ai, utilisable sur web, desktop, mobile et Claude Code) + extension Chrome MV3 sur claude.ai + backend (contacts, boîte de réception, crédits, liens magiques, chiffrement)
**Researched:** 2026-11-27
**Confidence:** MEDIUM (versions : HIGH ; faits de plateforme Claude : MEDIUM ; choix du serveur d'autorisation OAuth et format de chiffrement : MEDIUM, à valider par un spike)

> **Lecture des niveaux de confiance.** Ils viennent du seam `classify-confidence` de GSD, pas d'une impression.
> - **HIGH** : version lue aujourd'hui sur le registre npm, paquet jugé légitime par le seam, et usage recoupé par la doc officielle.
> - **MEDIUM** : fait lu aujourd'hui sur une page officielle (Anthropic, MCP, Cloudflare, Stripe, Chrome) et recoupé par une recherche. C'est le plafond du seam pour une source web. MEDIUM aussi quand la dernière version d'un paquet a moins d'un mois environ (signal `too-new` du seam) : ce n'est pas un doute sur le paquet, c'est une raison d'épingler la version.
> - **LOW** : source unique, source non officielle, ou connaissance non revérifiée aujourd'hui.
> - **[INFÉRENCE]** : raisonnement de conception à partir des faits vérifiés. À valider par un essai réel.

---

## Réponses aux deux vérifications demandées

### 1. Combien de connecteurs custom sur le plan Free de claude.ai ? Un seul. (MEDIUM, deux pages officielles concordantes)

Citation de l'article d'aide Anthropic 11175166 (lu le 27/11/2026) :

DATA_k7Qm2xVd_START
Free users are limited to one custom connector.
DATA_k7Qm2xVd_END

La doc développeur `claude.com/docs/connectors/custom/add-unlisted` dit la même chose. Les connecteurs custom existent sur Free, Pro, Max, Ultra, Team et Enterprise. Sur Team et Enterprise, seul un Owner peut en ajouter un pour l'organisation.

Conséquences pour la stack :
- **L'URL du serveur MCP doit être définitive avant le premier utilisateur externe.** Modifier un connecteur impose de le supprimer puis de le recréer. Pour un utilisateur Free, c'est son unique emplacement.
- **Un seul serveur MCP, une seule URL.** Pas de découpage en plusieurs connecteurs (envoi, lecture, contacts) : tout tient dans un endpoint `/mcp`.
- **Un lecteur web de repli est obligatoire** pour le destinataire Free dont l'emplacement est déjà pris, et pour le membre d'une organisation Team qui ne peut pas installer lui-même.
- **Un listing dans l'annuaire Anthropic sort de la catégorie « custom ».** C'est le seul moyen de lever cette limite et d'obtenir les transferts de fichiers. La soumission est ouverte à tout plan payant. La limite éventuelle pour les connecteurs d'annuaire sur Free n'est pas documentée.

### 2. claude.ai gère-t-il le préremplissage par URL (`claude.ai/new?q=`) ? Non documenté sur le web, fonctionne apparemment en préremplissage seul. (LOW)

| Surface | Format | Statut | Confiance |
|---------|--------|--------|-----------|
| Claude Desktop | `claude://claude.ai/new?q=...` | Documenté officiellement (article 14729294, mis à jour le 30/06/2026). Préremplit seulement, l'utilisateur relit et envoie. `q` tronqué vers 14 000 caractères. | MEDIUM |
| App mobile | `https://claude.ai/code/new?q=...` et `claude://code/new` | Documenté, mais uniquement pour l'onglet Code. Rien pour le chat. | MEDIUM |
| claude.ai dans le navigateur | `https://claude.ai/new?q=...` | **Aucune page officielle.** Des sources non officielles (blog de février 2026, générateurs de liens tiers) disent que `q` préremplit le champ, avec un contrôle de sécurité qui peut afficher une fenêtre « prompt non fiable ». L'envoi automatique par URL a cessé de marcher en octobre 2025 (issue GitHub close sans correctif). | LOW |

Décision de stack : **ne jamais faire dépendre un parcours de `?q=`.** Le bouton « Ouvrir dans Claude » fait trois choses dans cet ordre : il tente le préremplissage par URL, il copie le texte dans le presse-papiers, il affiche « colle ici ». Le texte prérempli doit rester court (par exemple « Lis mon message Relay n° 42 ») et renvoyer vers l'outil MCP. Il ne doit jamais contenir le brief lui-même : la limite de longueur, la fenêtre « prompt non fiable » et le passage en clair dans l'URL l'interdisent. À tester en vrai sur un compte connecté lors du spike de plateforme.

---

## Faits de plateforme qui dictent la stack

Tous lus aujourd'hui sur `claude.com/docs/connectors/*`, `support.claude.com` et `modelcontextprotocol.io`. Confiance MEDIUM sauf mention.

| Sujet | Fait | Effet sur la stack |
|-------|------|--------------------|
| Transport | Streamable HTTP. L'ancien HTTP+SSE est encore accepté mais en cours d'abandon. | Un seul endpoint `POST /mcp`. Pas de route `/sse`. |
| Réseau | Claude appelle le serveur depuis l'infrastructure d'Anthropic (plage `160.79.104.0/21`), jamais depuis l'appareil. HTTPS sur le port 443 uniquement, IPv4 uniquement (un enregistrement `A` public est requis), pas de redirection vers un autre hôte (l'en-tête `Authorization` est perdu). | Domaine public dédié, sans redirection apex vers `www`. Un WAF ou une protection anti-bot devant `/mcp`, `/token`, `/register`, `/.well-known/*` casse la connexion. |
| Révision du protocole | La spec en vigueur est **2026-07-28** : sans état, plus de poignée de main `initialize`, plus de `Mcp-Session-Id`, `server/discover` obligatoire, `resultType` sur chaque résultat. Claude Code négocie cette révision quand le serveur la propose, sinon il reste sur l'ancienne. Les docs Anthropic décrivent encore `initialize` pour les surfaces hébergées. | **Servir les deux époques.** Le SDK v2 le fait par défaut (`legacy: 'stateless'`). Ne jamais passer `legacy: 'reject'`. |
| Authentification | Types acceptés par défaut : `oauth_dcr`, `oauth_cimd`, `none`. Claude n'utilise CIMD que si les métadonnées du serveur d'autorisation annoncent `client_id_metadata_document_supported: true` **et** `"none"` dans `token_endpoint_auth_methods_supported`. Sinon il retombe sur DCR. La spec 2026-07-28 déprécie DCR au profit de CIMD. | Le serveur d'autorisation doit proposer CIMD **et** garder DCR en repli. |
| Redirections OAuth | Surfaces hébergées (web, desktop, mobile, Cowork) : `https://claude.ai/api/mcp/auth_callback`. Claude Code : `http://localhost/callback` et `http://127.0.0.1/callback` sur un port variable. | La comparaison des URI de redirection doit ignorer le port pour le loopback. |
| Découverte | Un `401` avec `WWW-Authenticate: Bearer resource_metadata="..."` est obligatoire pour lancer la connexion. Le champ `resource` doit être exactement l'URL saisie par l'utilisateur. Seule la première entrée de `authorization_servers` est lue. | Métadonnées RFC 9728 et RFC 8414 servies sur le même domaine que `/mcp`. |
| Jetons | PKCE S256 sur chaque demande. Le endpoint `/token` doit accepter `application/x-www-form-urlencoded`. Rotation des jetons de rafraîchissement pour les clients publics. Erreur `invalid_grant` quand un jeton de rafraîchissement n'est plus valide. | Ne pas écrire ce code à la main. |
| Latence | 10 s maximum pour la découverte, l'enregistrement et `/token`. 30 s pour un rafraîchissement. | Pas de base qui se réveille lentement derrière `/token`. |
| Limites d'outil | Environ 150 000 caractères par résultat et 240 s par appel sur claude.ai et Desktop. 25 000 tokens sur Claude Code. | Pagination et manifeste pour les dossiers. Jamais un seul bloc. |
| Fonctions non gérées par Claude | Abonnements aux ressources, sampling. | Aucun push vers Claude. La notification passe par l'extension. |
| Fichiers | Transferts de fichiers en bêta, **réservés aux connecteurs de l'annuaire** avec hôtes de stockage approuvés par Anthropic. Ils exigent la révision 2026-07-28 et `@modelcontextprotocol/server` 2.4.0. `@modelcontextprotocol/sdk` 1.32.2 ne les gère pas. Sur un connecteur custom : indisponible, sans message d'erreur. | En v1, les fichiers binaires passent par l'extension ou le site web. Choisir dès maintenant le SDK v2 pour ne pas réécrire le serveur au moment du listing. |
| Lien d'installation | `https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=NAME&connectorUrl=ENCODED_URL`. Il préremplit la boîte de dialogue, l'utilisateur confirme, puis se connecte en OAuth. | Le « un clic » du lien magique est en réalité trois ou quatre clics. Aucune bibliothèque nécessaire. |
| Annotations d'outil | `title` plus `readOnlyHint` ou `destructiveHint` sur chaque outil. Outils de lecture et d'écriture séparés. Noms de 64 caractères au plus. Exigé pour l'annuaire. | À respecter dès la v1 pour ne pas refaire les outils. |

---

## Recommended Stack

**Résumé en une phrase.** Un seul Worker Cloudflare en TypeScript (Hono) qui sert `/mcp`, l'OAuth, l'API de l'extension et le site ; D1 pour les métadonnées, R2 pour les contenus chiffrés ; Stripe Checkout pour les packs de crédits ; Resend pour les liens magiques ; une extension WXT + React ; un paquet de chiffrement partagé fondé sur le format age.

Pourquoi Cloudflare Workers plutôt qu'autre chose : c'est la plateforme de référence pour les serveurs MCP distants (la doc Anthropic la cite nommément, Cloudflare maintient la bibliothèque OAuth dédiée à MCP et nomme Claude parmi les clients conformes), le modèle sans état de la spec 2026-07-28 correspond exactement au modèle d'exécution d'un Worker, il n'y a pas de démarrage à froid devant `/token`, et un seul fournisseur couvre calcul, base, fichiers et clé-valeur. Pour un porteur de projet seul qui veut des cycles courts, moins de comptes à gérer compte plus que la base de données idéale.

### Core Technologies

| Technology | Version | Purpose | Why Recommended | Confiance |
|------------|---------|---------|-----------------|-----------|
| TypeScript | 7.0.2 (repli 6.0.3) | Langage unique pour le serveur, l'extension, le site et le chiffrement | Un seul langage permet de partager les schémas zod et le code de chiffrement entre les trois surfaces. La 7.0.2 est la `latest` depuis juillet 2026. Si un outil la refuse, revenir à 6.0.3. | HIGH (version), LOW (compatibilité de tout l'outillage avec la 7, non testée) |
| Node.js | 24 LTS (22.x minimum) | Outillage local | WXT 0.21.5 exige Node 22 ou plus. La machine actuelle a la 22.23.5, c'est suffisant. | HIGH |
| Cloudflare Workers + `wrangler` | wrangler 4.168.0 | Hébergement du Worker unique | Voir ci-dessus. Drapeaux requis : `nodejs_compat` et `global_fetch_strictly_public` (exigé par CIMD), `compatibility_date` récente. | MEDIUM (`too-new`) |
| `@modelcontextprotocol/server` | 2.4.0 | Serveur MCP (SDK officiel v2) | Implémente la révision 2026-07-28, sert aussi les clients de 2025 depuis la même fabrique, expose `createMcpHandler(factory)` qui renvoie un gestionnaire `fetch` standard utilisable tel quel dans un Worker. C'est la version exigée pour les transferts de fichiers. Node 20 ou plus, pair `zod ^4.2.0`. | HIGH |
| `hono` | 4.13.23 | Routeur HTTP du Worker (API, pages serveur, webhooks) | Standard de fait sur Workers. Le client typé `hc` donne des types de bout en bout à l'extension et au site sans génération de code. `hono/jsx` suffit pour les pages de connexion et de consentement. | MEDIUM (`too-new`) |
| `@cloudflare/workers-oauth-provider` | 1.4.0 | Serveur d'autorisation OAuth 2.1 devant `/mcp` | Seule bibliothèque vérifiée aujourd'hui qui couvre tout ce que Claude exige : CIMD (draft-00, celui que la spec MCP épingle), DCR, PKCE, métadonnées RFC 8414 et 9728, indicateurs de ressource RFC 8707, rotation des jetons de rafraîchissement, gestion du port loopback RFC 8252 pour Claude Code. Jetons stockés hachés dans KV. Mode un seul Worker avec `OAuthProvider` (`apiRoute: '/mcp'`). Vous n'écrivez que la page `/authorize`. | MEDIUM (`too-new` : la ligne 1.x date de fin septembre 2026) |
| `better-auth` | 1.7.28 | Identité de premier niveau : lien magique, connexion Google, sessions | Greffon `magicLink` éprouvé, connexion sociale, limitation de débit, gestion des sessions. Sert à savoir qui est l'utilisateur. La page `/authorize` lit la session Better Auth puis appelle `completeAuthorization()`. Adaptateur Drizzle pour D1. | MEDIUM (`too-new`) |
| Cloudflare D1 + `drizzle-orm` | drizzle-orm 0.45.4, drizzle-kit 0.31.11 | Base relationnelle : utilisateurs, contacts, messages, invitations, registre de crédits, événements Stripe, clés publiques | Zéro configuration, émulation locale intégrée à `wrangler dev`, pas de réveil lent. Les écritures sont sérialisées, ce qui rend le débit de crédits atomique simple. Limites : 10 Go par base, pas de transaction interactive (seulement `batch()`). Les contenus vont dans R2, donc 10 Go de métadonnées couvrent largement la v1. | HIGH (versions), MEDIUM (choix D1 plutôt que Postgres) |
| Cloudflare R2 | (service) | Stockage des contenus et pièces jointes, **chiffrés avant écriture** | Pas de frais de sortie, liaison directe depuis le Worker, compatible S3 pour les URL présignées. Le serveur ne stocke que du chiffré. | MEDIUM |
| Cloudflare KV | (service) | Stockage des autorisations OAuth (liaison `OAUTH_KV`) | Imposé par `workers-oauth-provider`. Ne rien y mettre d'autre. | MEDIUM |
| `wxt` | 0.21.5 | Cadre de l'extension Chrome Manifest V3 | Défaut du marché pour une extension neuve : manifeste généré, rechargement à chaud, aides d'injection d'interface (`createShadowRootUi`), gestion de la navigation SPA (`wxt:locationchange`), publication automatisée. Plasmo n'a plus de version depuis mai 2025. | HIGH |
| `react` + `react-dom` | 19.3.1 | Interface de l'extension et du site | Même bibliothèque d'interface des deux côtés, composants partagés. `@wxt-dev/module-react` 1.2.2 pour WXT. | HIGH |
| `stripe` | 23.2.0 | Achat de packs de crédits | Checkout Session en `mode: 'payment'` + webhook `checkout.session.completed` qui crédite un registre maison. Activer **Stripe Managed Payments** si l'éligibilité est accordée : Stripe devient marchand officiel et gère la TVA. La France figure parmi les pays d'établissement acceptés, les codes fiscaux SaaS sont éligibles, le surcoût annoncé est d'environ 3,5 %. Sinon Stripe Tax. | MEDIUM |
| `resend` + `react-email` | 6.40.1 / 6.11.17 | E-mails transactionnels : lien magique de connexion, invitation d'un destinataire sans compte | API simple, SDK compatible Workers, gabarits en React. Depuis React Email 6 (avril 2026), composants et rendu s'importent du paquet unique `react-email`. L'ancien `@react-email/components` est marqué déprécié sur npm. Offre gratuite lue aujourd'hui : 3 000 e-mails par mois et **100 par jour**. Ce plafond journalier bloquera une boucle d'invitation virale : prévoir l'offre à 20 dollars par mois au lancement public. | MEDIUM (versions), LOW (tarifs, source unique) |
| `age-encryption` | 0.3.1 | Format de chiffrement des contenus et fichiers, identique dans l'extension, le site et le Worker | Implémentation TypeScript du format age par son auteur. Chiffrement vers plusieurs destinataires X25519, flux (`ReadableStream`) pour les gros fichiers, destinataires hybrides post-quantiques, chiffrement par passkey (WebAuthn PRF). Ne dépend que des bibliothèques noble et utilise WebCrypto quand il existe. Évite d'inventer un format. | HIGH (version et légitimité), MEDIUM (maturité : version 0.x) |
| `zod` | 4.6.7 | Schémas d'entrée des outils MCP et de l'API | Pair requis du SDK MCP v2 (`^4.2.0`) et dépendance de Better Auth. Un seul jeu de schémas partagé. | HIGH |

### Supporting Libraries

| Library | Version | Purpose | When to Use | Confiance |
|---------|---------|---------|-------------|-----------|
| `@noble/curves` | 2.4.0 | Signatures Ed25519 de l'expéditeur | Dès la phase chiffrement. **age ne prouve pas qui a envoyé** : il faut signer le chiffré et ses métadonnées pour que le destinataire vérifie l'expéditeur sans faire confiance au serveur. | HIGH |
| `@noble/hashes`, `@noble/ciphers` | 2.4.0 | HKDF, SHA-256, AEAD pour l'emballage de clés | Seulement si un besoin sort du format age (clé de lien dans le fragment d'URL, emballage d'une clé par une clé serveur). | HIGH |
| WebCrypto (natif) | (plateforme) | Clés non extractibles stockées dans IndexedDB | Pour garder la clé privée de l'appareil hors de portée du JavaScript. X25519 et Ed25519 sont livrés dans Blink, WebKit et Gecko, ainsi que dans Node. Disponibilité dans Workers non revérifiée aujourd'hui. | MEDIUM (navigateurs), LOW (Workers) |
| `idb-keyval` | 6.3.0 | Stockage des `CryptoKey` dans l'IndexedDB de l'extension | Côté service worker ou page d'extension. Jamais depuis un script de contenu. | HIGH |
| `@modelcontextprotocol/ext-apps` | 2.0.4 | Carte interactive dans la conversation (aperçu + Accepter / Refuser) | Phase ultérieure. Pairs : `@modelcontextprotocol/server ^2.0.0`, `zod ^4.2.0`. Sur un connecteur custom, l'app ouverte peut porter la mention « Unverified ». Demande une recherche dédiée. | HIGH (version), MEDIUM (comportement dans Claude) |
| `aws4fetch` | 1.0.20 | URL présignées R2 | Quand un fichier doit monter directement du navigateur vers R2, ou pour les transferts de fichiers de l'annuaire. Les URL présignées ne marchent que sur l'hôte S3 `<ACCOUNT_ID>.r2.cloudflarestorage.com`, **pas** sur un domaine personnalisé. | HIGH (version), MEDIUM (contrainte R2) |
| `@webext-core/messaging` | 4.0.2 | Messages typés entre script de contenu, panneau latéral et service worker | Dès que l'extension a plus d'un contexte. | HIGH |
| `tailwindcss` + `@tailwindcss/vite` | 4.3.5 | Styles de l'extension et du site | Attention dans un shadow root : les unités `rem` suivent la page hôte et les variables déclarées par `@property` ne s'y enregistrent pas. Prévoir un thème en `px` pour l'interface injectée. | MEDIUM (`too-new`), LOW (piège du shadow root, non revérifié aujourd'hui) |
| `react-router` | 8.5.3 | Routage du site | Pages : aperçu du lien magique, boîte de réception web, crédits, réglages. | HIGH |
| `@tanstack/react-query` | 5.108.4 | Cache et rafraîchissement des données côté site et panneau latéral | Boîte de réception, compteur de non-lus, solde de crédits. | HIGH |
| `@cloudflare/vite-plugin` | 1.70.2 | Servir le site (SPA Vite) comme ressources statiques du même Worker | Un seul déploiement pour l'API et le site. Pairs : `vite ^6.1 \|\| ^7 \|\| ^8`, `wrangler ^4.168.0`. | MEDIUM |
| `@sentry/cloudflare` | 11.19.0 | Suivi d'erreurs | Au lancement public. Avant cela, les journaux Workers suffisent. Ne jamais journaliser un contenu, un jeton ou une URL signée. | MEDIUM |

### Development Tools

| Tool | Purpose | Notes | Confiance |
|------|---------|-------|-----------|
| `pnpm` 12 (workspaces) | Monorepo : `apps/server`, `apps/web`, `apps/extension`, `packages/shared` (schémas, types d'API, chiffrement) | Activer un âge minimal de publication (`minimumReleaseAge`) : plus de la moitié des paquets recommandés ont une dernière version de moins d'un mois (signal `too-new`). Toujours versionner le lockfile et épingler les versions exactes. | HIGH (version), LOW (nom exact du réglage, non revérifié) |
| `@modelcontextprotocol/inspector` 2.17.0 | Vérifier le protocole et le flux OAuth avant de brancher Claude | `npx @modelcontextprotocol/inspector --cli <url> --transport http --method tools/list` | MEDIUM (`too-new`) |
| `cloudflared` (tunnel) | Exposer le Worker local à Claude pendant le développement | Claude appelle depuis Internet : `localhost` ne suffit pas. Couper le tunnel après usage. | MEDIUM |
| `vitest` 5.1.2 + `@cloudflare/vitest-plugin` 1.9.2 | Tests du Worker avec D1, R2 et KV isolés par test | `@cloudflare/vitest-pool-workers` a été **renommé** en `@cloudflare/vitest-plugin` et ne reçoit plus de mises à jour. Le nouveau paquet accepte `vitest ^4.1.0 \|\| ^5.0.0`. Beaucoup de tutoriels citent encore l'ancien nom. | MEDIUM (`too-new`) |
| `@playwright/test` 1.65.1 | Tests de bout en bout de l'extension (Chromium avec l'extension chargée) | Indispensable pour détecter une casse après une mise à jour du DOM de claude.ai. | MEDIUM (`too-new`) |
| `@biomejs/biome` 2.6.9 | Lint et formatage | Un seul outil au lieu d'ESLint + Prettier. | MEDIUM |
| `drizzle-kit` 0.31.11 | Génération des migrations D1 | Appliquer avec `wrangler d1 migrations apply`. Rester sur la ligne 0.45 / 0.31 : la 1.0 de Drizzle est encore en bêta. | HIGH |
| Claude Code + greffon `mcp-server-dev` | Assistant de construction et de test du serveur MCP | Greffon officiel Anthropic cité par la doc de construction des connecteurs. | MEDIUM |

---

## Installation

```bash
# Racine du monorepo
pnpm init && pnpm add -Dw typescript@7.0.2 @biomejs/biome@2.6.9

# apps/server : Worker (MCP + OAuth + API + pages serveur)
pnpm add @modelcontextprotocol/server@2.4.0 zod@4.6.7 hono@4.13.23 \
  @cloudflare/workers-oauth-provider@1.4.0 better-auth@1.7.28 \
  drizzle-orm@0.45.4 stripe@23.2.0 resend@6.40.1 react-email@6.11.17 \
  aws4fetch@1.0.20
pnpm add -D wrangler@4.168.0 drizzle-kit@0.31.11 vitest@5.1.2 \
  @cloudflare/vitest-plugin@1.9.2 @modelcontextprotocol/inspector@2.17.0

# packages/shared : chiffrement et schémas (extension, site, Worker)
pnpm add age-encryption@0.3.1 @noble/curves@2.4.0 @noble/hashes@2.4.0 @noble/ciphers@2.4.0 zod@4.6.7

# apps/extension : extension Chrome MV3
pnpm add react@19.3.1 react-dom@19.3.1 @webext-core/messaging@4.0.2 idb-keyval@6.3.0 \
  @tanstack/react-query@5.108.4
pnpm add -D wxt@0.21.5 @wxt-dev/module-react@1.2.2 tailwindcss@4.3.5 @tailwindcss/vite@4.3.5 \
  @playwright/test@1.65.1

# apps/web : site (SPA servie par le même Worker)
pnpm add react@19.3.1 react-dom@19.3.1 react-router@8.5.3 @tanstack/react-query@5.108.4
pnpm add -D vite@8.4.0 @cloudflare/vite-plugin@1.70.2 tailwindcss@4.3.5 @tailwindcss/vite@4.3.5
```

Ressources Cloudflare à créer une fois : une base D1, un bucket R2, un espace KV lié sous le nom `OAUTH_KV`. Secrets à poser avec `wrangler secret put` : clé Stripe, secret de webhook Stripe, clé Resend, secret Better Auth, clé maître de chiffrement serveur.

---

## Détail par brique

### 1. Serveur MCP distant

- **Fabrique sans état.** `createMcpHandler(() => buildServer(user))` construit un serveur neuf par requête. Aucun état de session. C'est le modèle recommandé à la fois par le SDK v2 et par Cloudflare, dont l'ancienne classe `McpAgent` est gelée et dépréciée.
- **Deux époques servies.** Garder la posture par défaut `legacy: 'stateless'`. Les exemples de certaines bibliothèques (dont la doc MCP de Better Auth) montrent `legacy: 'reject'` : à ne pas recopier, cela couperait les clients Claude qui parlent encore la révision 2025.
- **Rôle de serveur de ressources uniquement.** Le SDK v2 ne fournit plus de serveur d'autorisation. Les anciens assistants vivent, gelés, dans `@modelcontextprotocol/server-legacy`.
- **Pas besoin du paquet `agents` de Cloudflare.** Le `createMcpHandler` du SDK officiel tourne tel quel dans un Worker. `agents` 0.34.1 épingle des versions exactes du SDK et ajoute une dépendance de plus. À n'ajouter que si son assistant `getMcpAuthContext()` simplifie vraiment le câblage. [INFÉRENCE]
- **Message « plus de crédits ».** L'outil renvoie un résultat normal avec un texte factuel et un lien vers une page du site (par exemple `/credits`), qui crée la session Stripe à la demande. Ne pas renvoyer une URL de session Checkout brute : elle expire. Ne pas mettre de texte promotionnel dans les descriptions d'outils : la politique de l'annuaire rejette les descriptions qui font la promotion de produits et les logiciels qui servent de la publicité.
- **Authentification paresseuse.** Claude sait appeler des outils publics sans connexion et n'afficher la carte « Connect » qu'au premier outil protégé (réponse HTTP `401`, jamais un `200` avec `isError`). Utile pour le parcours du destinataire sans compte. À décider en phase lien magique.

### 2. Serveur d'autorisation OAuth : le point le plus risqué de la stack

Deux options crédibles ont été examinées jusque dans le code publié sur npm.

| Critère | `@cloudflare/workers-oauth-provider` 1.4.0 + Better Auth pour l'identité (**recommandé**) | Better Auth seul, greffons `@better-auth/mcp` + `@better-auth/cimd` 1.7.28 |
|---------|-------------|-------------|
| CIMD | Oui, draft-00 (celui que MCP épingle), activé par `clientIdMetadataDocumentEnabled: true` | Oui, profil `mcp-2026-07-28`, mais hors Node il faut **fournir soi-même un transport `fetch` durci** contre le SSRF |
| DCR | Oui | Désactivé par défaut, double option à activer |
| Port loopback de Claude Code | Géré, documenté | Présent dans le code, non documenté |
| Mention de Claude | Sa doc nomme Claude comme client conforme | Aucune mention |
| Nombre de pièces | Deux bibliothèques à relier par la page `/authorize` | Une seule |
| Stockage | KV (jetons hachés, propriétés chiffrées) | Base SQL, jetons JWT vérifiables sans lecture |
| Ancienneté | Ligne 1.x depuis le 24/09/2026 | Greffon MCP depuis janvier 2026, remanié pour la spec de juillet |

Recommandation : **`workers-oauth-provider` pour tout ce que Claude touche, Better Auth pour savoir qui est l'utilisateur.** La compatibilité avec le client OAuth de Claude est le chemin critique du produit. Mieux vaut la pièce conçue pour cela, quitte à relier deux bibliothèques.

**Ce choix n'est pas prouvé contre le vrai Claude.** Aucune des deux options n'a été testée aujourd'hui. Le premier livrable de la feuille de route doit être un spike : un outil `ping`, la connexion OAuth complète, vérifiée sur claude.ai web, l'app mobile et Claude Code. Si le spike échoue, basculer sur la colonne de droite, puis sur un fournisseur hébergé qui livre CIMD (WorkOS, Clerk, Stytch).

### 3. Extension Chrome (Manifest V3)

- **Interface injectée** : script de contenu sur `https://claude.ai/*`, `createShadowRootUi` avec `cssInjectionMode: 'ui'` pour isoler les styles. Écouter `wxt:locationchange` car claude.ai est une SPA. Utiliser les aides `ctx.*` de WXT pour survivre aux mises à jour de l'extension.
- **Panneau latéral** (`chrome.sidePanel`, permission `sidePanel`, Chrome 114 ou plus) pour le carnet de contacts et la boîte de réception. Il ne dépend pas du DOM de claude.ai, donc il ne casse pas quand Anthropic change son site. `sidePanel.open()` exige un geste de l'utilisateur. (MEDIUM)
- **Ancrage minimal dans le DOM de claude.ai.** Un seul bouton « Envoyer à… », avec un bouton flottant de secours si le sélecteur d'ancrage ne trouve rien. [INFÉRENCE]
- **Pastille de notification** : `chrome.alarms` (période minimale de 30 s depuis Chrome 120) + appel du compteur de non-lus + `chrome.action.setBadgeText`. Recréer l'alarme à chaque démarrage du service worker. Le Web Push (possible sans notification visible depuis Chrome 121) demande la permission `notifications`, qui affiche un avertissement à l'installation : à garder pour plus tard. (MEDIUM pour le Web Push, LOW pour la période exacte des alarmes : deux pages Chrome se contredisent)
- **Connexion de l'extension** : en v1, l'utilisateur se connecte sur le site par lien magique, et le service worker appelle l'API avec `credentials: 'include'` grâce à une permission d'hôte sur le domaine de l'API. L'API doit vérifier l'en-tête `Origin`. Solution plus robuste si cela coince : `chrome.identity.launchWebAuthFlow` avec PKCE. (LOW, à valider en phase extension)
- **Secrets** : jamais dans `localStorage` depuis un script de contenu, car c'est le `localStorage` de claude.ai. Utiliser `chrome.storage` et l'IndexedDB de l'origine de l'extension.
- **Permissions à demander** : `storage`, `alarms`, `sidePanel`, et les hôtes `https://claude.ai/*` et le domaine de l'API. Rien d'autre. Aucun code chargé à distance.

### 4. Backend, base, fichiers, paiements, e-mails

- **Débit de crédits atomique sur D1** : contrainte `CHECK (balance >= 0)` sur le solde, puis un seul `db.batch([débit, insertion du message, ligne de registre])`. Si le solde passerait sous zéro, tout le lot est annulé. Ne pas écrire `db.transaction()` : D1 ne le gère pas. [INFÉRENCE à partir de la doc D1 : seul `batch()` est documenté]
- **Webhook Stripe idempotent** : table des identifiants d'événement déjà traités, crédit et marquage dans le même lot. Dans un Worker, la vérification de signature doit utiliser la variante asynchrone du SDK. (LOW, à confirmer dans la doc Stripe au moment de la phase)
- **Fichiers** : en v1, envoi par l'extension ou le site vers le Worker, qui écrit le chiffré dans R2. Plafonner la taille par fichier. URL présignées seulement quand le besoin de gros fichiers apparaît.
- **Tâches différées** : `ctx.waitUntil()` pour l'envoi d'e-mails, un Cron Trigger pour purger les messages et invitations expirés. Les Queues peuvent attendre.
- **WhatsApp** : aucun service à intégrer. Un lien `https://wa.me/?text=...` ou l'API de partage du navigateur suffit, l'utilisateur envoie lui-même le lien. (LOW, non revérifié aujourd'hui)
- **Résidence des données** : D1 et R2 proposent une restriction de juridiction européenne. (LOW, non revérifié aujourd'hui ; à confirmer avant le lancement public)

### 5. Chiffrement de bout en bout : ce que les bibliothèques règlent et ce qu'elles ne règlent pas

Le choix de bibliothèque n'est pas le point dur. La contrainte est structurelle : un connecteur MCP distant doit rendre du clair à Claude, donc **sur le chemin connecteur, le serveur voit le clair au moment de la lecture**. Aucune bibliothèque ne change cela. Le vrai bout en bout n'existe que là où du code tourne chez l'utilisateur : extension, page web, ou plus tard un serveur MCP local.

Ce que la stack recommandée permet, avec le même paquet `packages/shared` partout :

| Mode | Qui déchiffre | Promesse honnête |
|------|---------------|------------------|
| Extension vers extension | L'appareil du destinataire. Le serveur ne stocke que du chiffré. | Bout en bout |
| Lien magique pour un destinataire sans compte | Le navigateur du destinataire, avec une clé placée dans le fragment de l'URL (`#...`), que le navigateur n'envoie jamais au serveur | Bout en bout tant que le lien n'est pas intercepté |
| Lecture par le connecteur (mobile, Claude Code, utilisateur sans extension) | Le Worker, avec une clé par message, au moment de l'appel d'outil | Chiffré en transit et au repos, **lisible par le serveur à la lecture** |

Piste d'architecture à trancher ailleurs : faire de l'acceptation du message par le destinataire (dans l'extension ou sur le site) l'acte qui libère la clé au serveur pour une durée courte. L'exigence « aperçu puis acceptation » et la contrainte de chiffrement se règlent alors par le même geste. [INFÉRENCE]

Points de conception imposés par les bibliothèques :
- **age chiffre mais n'authentifie pas l'expéditeur.** Ajouter une signature Ed25519 (`@noble/curves`) sur le chiffré et les métadonnées, vérifiée avec la clé publique du contact.
- **Clé privée** : générée sur l'appareil, stockée comme `CryptoKey` non extractible quand c'est possible, sinon emballée par une clé non extractible.
- **Perte d'appareil et second appareil** : aucune bibliothèque ne le règle. Le chiffrement par passkey de `age-encryption` (WebAuthn PRF) est la piste la plus adaptée à un public non technique, car la passkey se synchronise par le gestionnaire de mots de passe. À étudier dans une recherche de phase dédiée.

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Cloudflare Workers + Hono | Next.js 16.4.6 sur Vercel + `mcp-handler` 2.4.0 | Si vous déployez déjà tout sur Vercel et préférez un seul projet Next.js pour le site, l'API et `/mcp`. Prendre alors Better Auth `mcp()` comme serveur d'autorisation et Neon pour la base. Stack valable, davantage de fournisseurs. |
| Cloudflare D1 | Postgres chez Neon (`@neondatabase/serverless` 1.2.1) via Hyperdrive | Dès qu'il faut des transactions interactives, de l'analytique SQL sérieuse ou plus de 10 Go. Le schéma Drizzle se réécrit de `sqlite-core` vers `pg-core` : coût modéré tant que la base est petite. |
| `workers-oauth-provider` + Better Auth | Better Auth seul (`@better-auth/mcp` + `@better-auth/cimd`) | Si le spike montre que cela fonctionne avec Claude sur Workers. Une seule bibliothèque, une seule table d'utilisateurs. |
| `workers-oauth-provider` + Better Auth | WorkOS AuthKit, Clerk, Stytch | Si vous ne voulez exploiter aucun code d'authentification. Ils livrent CIMD. Coût mensuel et dépendance à un tiers pour le cœur du produit. |
| `@modelcontextprotocol/server` 2.4.0 | `@modelcontextprotocol/sdk` 1.32.2 | Uniquement pour un prototype jetable calqué sur le quickstart d'Anthropic, qui utilise encore la v1. Pas pour le produit. |
| `age-encryption` | HPKE avec `@hpke/core` 1.10.1 | Si vous voulez une primitive normalisée par l'IETF (RFC 9180) et acceptez de concevoir vous-même le découpage en blocs et le multi-destinataire. |
| `age-encryption` | `libsodium-wrappers` 0.8.4 | Si une équipe connaît déjà libsodium. C'est du WebAssembly : la CSP de l'extension doit alors autoriser `wasm-unsafe-eval`. |
| Stripe Checkout + registre maison | Polar (`@polar-sh/sdk` 1.3.3) | Si Stripe refuse l'éligibilité à Managed Payments et que vous voulez quand même un marchand officiel qui gère la TVA. |
| Resend | Postmark (`postmark` 5.1.1) | Si la délivrabilité des liens magiques devient un problème mesuré. |
| WXT | CRXJS (`@crxjs/vite-plugin` 3.0.0) | Si vous voulez seulement Vite et écrire le manifeste à la main. Chromium uniquement, aucune aide d'injection. |
| Sondage par `chrome.alarms` | Web Push dans le service worker | Quand la notification instantanée devient un argument produit et que l'avertissement de permission est acceptable. |
| Connecteur distant seul | Serveur MCP local empaqueté en `.mcpb` | Plus tard, pour offrir du vrai bout en bout sur Claude Desktop. Ne couvre ni le web ni le mobile. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `legacy: 'reject'` sur `createMcpHandler` | Refuse les clients qui ouvrent par `initialize`. Rien ne prouve que toutes les surfaces Claude parlent déjà la révision 2026-07-28. | La valeur par défaut `legacy: 'stateless'` |
| Transport HTTP+SSE, URL en `/sse` | Déprécié dans la spec, jamais servi par le SDK v2 | Streamable HTTP sur `/mcp` |
| `McpAgent` (paquet `agents`) et sessions MCP en Durable Objects | Déprécié et gelé par Cloudflare. Les sessions n'existent plus dans la spec. | `createMcpHandler(factory)` sans état |
| Serveur d'autorisation OAuth écrit à la main | Rotation des jetons, PKCE, CIMD, protection SSRF, port loopback : chaque détail casse la connexion ou ouvre une faille | `@cloudflare/workers-oauth-provider` |
| Serveur d'autorisation en DCR seul | DCR est déprécié par MCP 2026-07-28. D'après Anthropic, Claude enregistre un nouveau client à chaque nouvelle connexion. | CIMD activé, DCR en repli |
| Authentification du connecteur par en-tête statique (clé d'API) | En bêta, réservée à certaines organisations, une clé par organisation et non par personne | OAuth par utilisateur |
| Jetons dans la chaîne de requête (`?token=`) | Interdit par la spec MCP, fuite dans les journaux | En-tête `Authorization` |
| API interne de claude.ai, automatisation de l'envoi de prompts | Hors périmètre du projet, fragile, risque sur les conditions d'utilisation | Connecteur MCP + préremplissage que l'utilisateur valide |
| Dépendre de `claude.ai/new?q=` | Non documenté sur le web, déjà modifié une fois sans préavis | Préremplissage tenté + presse-papiers |
| Plasmo | Dernière version en mai 2025 | WXT |
| Manifest V2 | Plus accepté par Chrome | Manifest V3 |
| `localStorage` dans un script de contenu | C'est le stockage de la page claude.ai, lisible par elle | `chrome.storage`, IndexedDB de l'extension |
| `tweetnacl` | Aucune version depuis 2023, pas de flux, pas de format de fichier | `age-encryption` + noble |
| Protocole Signal (`@signalapp/libsignal-client`), MLS (`ts-mls`) | Conçus pour la conversation continue avec renouvellement de clés. Ici ce sont des envois ponctuels asynchrones. Complexité sans bénéfice en v1. | Enveloppe age + signature |
| OpenPGP (`openpgp`) | Lourd, format ancien, gestion de clés hostile à un public non technique | `age-encryption` |
| Greffon Stripe de Better Auth pour les crédits | Pensé pour les abonnements. Les packs de crédits sont des achats ponctuels. (LOW, non vérifié en détail) | SDK `stripe` + registre maison |
| Stripe Billing à l'usage (compteurs, crédits de facturation) | Surdimensionné pour des packs prépayés | Checkout `mode: 'payment'` |
| Lemon Squeezy | SDK JavaScript sans mise à jour depuis novembre 2024 | Stripe Managed Payments, sinon Polar |
| API WhatsApp Business | Compte professionnel, validation de gabarits, coût par message, pour un simple partage de lien | Lien `wa.me` |
| Cloudflare Email Service | Envoi encore en bêta, réservé au plan Workers payant | Resend |
| `@cloudflare/vitest-pool-workers` | Renommé, marqué déprécié sur npm, plus de mises à jour | `@cloudflare/vitest-plugin` 1.9.2 |
| `@react-email/components` et les paquets `@react-email/button`, `/html`, `/tailwind` | Marqués dépréciés sur npm depuis React Email 6 | Paquet unique `react-email` 6.11.17 (`@react-email/render` reste maintenu) |
| WAF, mode anti-bot ou redirection devant les routes appelées par Anthropic | Connexion refusée avant d'atteindre votre code, avec le message « Couldn't reach the MCP server » | Règle d'exception pour `/mcp`, `/token`, `/register`, `/.well-known/*` et la plage `160.79.104.0/21` |

---

## Stack Patterns by Variant

**Si le spike OAuth échoue avec `workers-oauth-provider` :**
- Passer à Better Auth `mcp()` + `cimd()` + `jwt()`, activer aussi DCR.
- Parce que c'est la seule autre option open source vérifiée aujourd'hui. Sur Workers, il faudra écrire le transport `fetch` durci pour CIMD.

**Si vous restez sur Vercel :**
- Next.js 16.4.6, `mcp-handler` 2.4.0, Better Auth, Neon, stockage R2 ou Vercel Blob.
- Parce qu'un seul projet Next.js couvre le site, l'API et `/mcp`. Vérifier que le pare-feu Vercel laisse passer la plage d'Anthropic.

**Si le connecteur est accepté dans l'annuaire Anthropic :**
- Activer les transferts de fichiers : extension `io.modelcontextprotocol/files`, serveur bas niveau `Server` du SDK v2, cibles d'envoi présignées sur l'hôte S3 de R2, hôtes à faire approuver par e-mail à Anthropic.
- Parce que c'est le seul chemin officiel pour qu'un PDF ou une image passe par Claude lui-même.

**Si la base dépasse ce que D1 sait faire :**
- Neon Postgres via Hyperdrive, Drizzle en `pg-core`.
- Parce que les transactions interactives et l'analytique deviennent nécessaires.

**Si le vrai bout en bout doit couvrir Claude Desktop :**
- Ajouter un serveur MCP local empaqueté en `.mcpb` qui déchiffre sur l'appareil.
- Parce que c'est le seul moyen de donner du clair à Claude sans passer par le serveur. Hors v1.

---

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| `@modelcontextprotocol/server@2.4.0` | `zod@^4.2.0`, Node >= 20 | Dépend de `@modelcontextprotocol/core@2.4.0` exactement |
| `@modelcontextprotocol/ext-apps@2.0.4` | `@modelcontextprotocol/server@^2.0.0`, `zod@^4.2.0`, React 17 à 19 | |
| `@modelcontextprotocol/hono@2.0.2` (facultatif) | `hono@^4.11.4`, `@modelcontextprotocol/server@^2.3.0` | Sa protection contre le DNS rebinding vise les serveurs locaux. Derrière un nom public, lister l'hôte dans `allowedHosts`. |
| `agents@0.34.1` (non recommandé) | `@modelcontextprotocol/server@2.4.0` exactement | Épinglage exact : monter le SDK oblige à monter `agents` |
| `@cloudflare/workers-oauth-provider@1.4.0` | Drapeau `global_fetch_strictly_public`, `compatibility_date` >= 2024-11-11, liaison KV `OAUTH_KV` | Sans le drapeau, CIMD n'est pas annoncé et Claude retombe sur DCR |
| `better-auth@1.7.28` | `drizzle-orm@^0.45.2`, `drizzle-kit@>=0.31.4`, React 18 ou 19 | Les bindings D1 n'existent que pendant une requête : créer l'instance d'authentification par requête |
| `wxt@0.21.5` | Node >= 22, `vite@^6.3.4 \|\| ^7 \|\| ^8`, `typescript@>=5.4` | |
| `@wxt-dev/module-react@1.2.2` | `wxt@>=0.19.16` | |
| `@cloudflare/vitest-plugin@1.9.2` | `vitest@^4.1.0 \|\| ^5.0.0` | Remplace `@cloudflare/vitest-pool-workers` |
| `react-email@6.11.17` | Node >= 20.19, React 18 ou 19 | `resend` accepte `@react-email/render` en pair facultatif |
| `@cloudflare/vite-plugin@1.70.2` | `vite@^6.1.0 \|\| ^7 \|\| ^8`, `wrangler@^4.168.0` | |
| `age-encryption@0.3.1` | `@noble/*@^2`, Node >= 20, navigateurs récents, ES2023 | |
| Claude (surfaces hébergées) | URI de redirection `https://claude.ai/api/mcp/auth_callback` | Port 443, IPv4, pas de redirection inter-hôtes |
| Claude Code | Redirection loopback sur port variable | Comparer `localhost` et `127.0.0.1` sans le port |

---

## Indications pour la feuille de route

1. **Spike de plateforme en tout premier.** Un Worker avec un outil `ping`, OAuth complet, testé sur claude.ai web, mobile et Claude Code, plus un essai réel de `claude.ai/new?q=`. Il valide le choix le plus risqué de cette stack avant d'écrire du produit.
2. **Fixer le domaine de production avant tout utilisateur externe.** Le champ `resource` et l'URL du connecteur ne doivent plus bouger.
3. **Le chemin texte par connecteur d'abord, les fichiers ensuite.** Les fichiers binaires dépendent de l'extension ou du site en v1, et de l'annuaire plus tard.
4. **Phases qui demandent une recherche dédiée** : serveur d'autorisation (après le spike), gestion des clés et récupération d'appareil, MCP Apps pour la carte d'acceptation, soumission à l'annuaire et approbation des hôtes de fichiers, éligibilité Stripe Managed Payments.
5. **Phases sur des schémas standards, sans recherche supplémentaire** : Checkout et webhook Stripe, e-mails Resend, schéma Drizzle sur D1, squelette WXT.

---

## Sources

Pages officielles lues le 27/11/2026 :
- https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp : plans, limite Free, réseau, réglages OAuth
- https://claude.com/docs/connectors/custom/add-unlisted : limite Free, options d'authentification
- https://claude.com/docs/connectors/building : transport, limites de taille et de durée, fonctions non gérées
- https://claude.com/docs/connectors/building/authentication : types d'authentification, CIMD et DCR, URI de redirection, rafraîchissement, latence, plage IP
- https://claude.com/docs/connectors/building/lazy-authentication : forme du `401`, exemple de métadonnées CIMD, port loopback
- https://claude.com/docs/connectors/building/directory-vs-custom : lien d'installation, différences annuaire et custom
- https://claude.com/docs/connectors/building/files : transferts de fichiers, versions de SDK testées
- https://claude.com/docs/connectors/building/testing et `/troubleshooting` : tunnel, Inspector, causes d'échec de connexion
- https://claude.com/docs/connectors/building/review-criteria et https://support.claude.com/en/articles/13145358-anthropic-software-directory-policy : annotations, contenus promotionnels
- https://claude.com/docs/connectors/getting-started : surfaces où les connecteurs fonctionnent
- https://support.claude.com/en/articles/14729294-open-claude-desktop-with-a-link et `/14898120-open-the-claude-mobile-app-with-a-link` : liens profonds et paramètre `q`
- https://code.claude.com/docs/en/mcp : ajout d'un serveur HTTP, négociation de la révision 2026-07-28
- https://modelcontextprotocol.io/specification/2026-07-28/changelog : changements et dépréciations
- https://ts.sdk.modelcontextprotocol.io/v2/ (paquets, autorisation, clients anciens, Hono) : SDK TypeScript v2
- https://developers.cloudflare.com/agents/model-context-protocol/ (guide de migration, autorisation, démarrage) : `createMcpHandler`, dépréciation de `McpAgent`
- https://developers.cloudflare.com/d1/platform/limits/ : limites D1
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/ : URL présignées et domaines personnalisés
- https://docs.stripe.com/payments/managed-payments/eligibility : pays et produits éligibles
- https://wxt.dev/guide/essentials/content-scripts.html : interface injectée, navigation SPA
- https://developer.chrome.com/docs/extensions/reference/api/sidePanel et `/how-to/integrate/web-push` : panneau latéral, Web Push
- https://better-auth.com/docs/plugins/mcp, `/cimd`, `/oauth-provider` : option alternative de serveur d'autorisation
- https://resend.com/pricing : offres (source unique)
- https://react.email/docs/getting-started/manual-setup et journal des versions de React Email 6 : paquet unique `react-email`

Registre npm, interrogé le 27/11/2026 : toutes les versions de ce document, les plages de dépendances paires et les dates de publication. Contenu des paquets `@cloudflare/workers-oauth-provider@1.4.0`, `@better-auth/mcp@1.7.28`, `@better-auth/cimd@1.7.28`, `@better-auth/oauth-provider@1.7.28`, `age-encryption@0.3.1` et `@hpke/core@1.10.1` lu directement (README, docs embarquées, code distribué).

Sources non officielles, confiance LOW :
- https://github.com/anthropics/claude-code/issues/8827 : arrêt de l'envoi automatique par URL en octobre 2025
- https://austegard.com/blog/claudeainew-unofficial-readme.html : paramètres non documentés de `claude.ai/new`
- Comparatifs WXT, Plasmo, CRXJS de 2026 (dev.to, merginit.com) : concordants sur WXT par défaut
- https://paulmillr.com/noble/ : historique des audits des bibliothèques noble

Non vérifié aujourd'hui, à confirmer avant de s'y fier : disponibilité de X25519 dans le WebCrypto de Workers, restriction de juridiction européenne pour D1 et R2, traitement des cookies `SameSite` pour les requêtes d'une extension, vérification asynchrone des webhooks Stripe dans un Worker, comportement de Tailwind 4 dans un shadow root, compatibilité de tout l'outillage avec TypeScript 7.

---
*Stack research for: partage de contexte de Claude à Claude (connecteur MCP distant + extension Chrome + backend chiffré)*
*Researched: 2026-11-27*
