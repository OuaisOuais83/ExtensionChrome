# Project Research Summary

**Project:** Relay (nom provisoire)
**Domain:** Partage de contexte de Claude à Claude entre personnes : serveur MCP distant (connecteur custom claude.ai), surcouche extension Chrome MV3, backend comptes / contacts / boîte de réception / fichiers / crédits / liens magiques / chiffrement
**Researched:** 2026-11-27
**Confidence:** MEDIUM

> **Lecture des niveaux de confiance.** Ils sont repris des quatre fichiers de recherche (seam `classify-confidence` de GSD). MEDIUM = lu le jour même sur une page officielle et recoupé. LOW = source unique, non officielle, ou non revérifiée. « Inférence » = raisonnement de conception à valider par un essai réel. Aucun fait de plateforme Claude n'est HIGH : tous peuvent changer, et plusieurs ne sont pas documentés.

## Executive Summary

Relay est une messagerie ponctuelle de personne à personne dont le facteur est Claude : A dit « envoie ce brief à Paul », le Claude de Paul le lit. La recherche confirme que l'espace est libre aujourd'hui (aucune fonction native n'envoie du contexte à une personne nommée hors de son organisation) et que le transport lui-même est une commodité : Threadpacket, Patchcord et agent2agent exposent déjà `send / inbox / read`, tous pour des développeurs, aucun avec de la traction. La différence se joue ailleurs : carnet de contacts inter-organisations, parcours pour non-techniques, et une défense contre l'injection de prompt que personne ne documente. Le canal officiel (connecteur MCP distant avec OAuth par utilisateur) couvre bien web, desktop, mobile et Claude Code, mais il ne transporte que **du texte tapé par le modèle**. Cinq hypothèses de PROJECT.md sont fausses ou non vérifiées : fichiers par le connecteur, installation en un clic, écriture dans un Projet, chiffrement de bout en bout sur le canal principal, préremplissage `claude.ai/new?q=` sur le web.

L'approche recommandée : un seul Worker Cloudflare en TypeScript qui sert `/mcp`, l'OAuth, l'API et le site, sur une seule URL définitive (le plan Free n'autorise qu'un connecteur custom, et une URL de connecteur ne se modifie pas). Construire **connecteur d'abord, extension ensuite** : toute la boucle centrale et toutes les protections doivent fonctionner sans extension, parce que mobile, Desktop et Claude Code n'en ont pas. Le texte passe par les outils MCP ; les fichiers passent par un canal latéral (page de dépôt web, puis extension) ; les vrais transferts de fichiers par Claude n'arrivent qu'avec une fiche dans l'annuaire Anthropic. La promesse de confidentialité honnête est « chiffré en transit et au repos, expiration courte », pas « de bout en bout ».

Trois risques dominent. **L'injection de prompt** : étiqueter le contenu « non fiable » ne suffit pas ; l'aperçu et l'acceptation doivent se faire hors du modèle (page web, plus tard extension), imposés par le serveur, et l'outil d'envoi du produit est lui-même le canal d'exfiltration à verrouiller. **L'OAuth propre à Claude** : le client de claude.ai est plus strict que la spec et échoue avec un message générique ; il faut un spike sur de vrais comptes avant tout code produit. **La politique de l'annuaire Anthropic** : le message de crédits épuisés, « envoie cette conversation » et tout texte promotionnel touchent des clauses précises ; concevoir conforme dès le premier outil évite une réécriture. Le découpage en deux jalons proposé par l'orchestrateur (jalon 1 : cœur texte sur toutes les surfaces sans extension ; jalon 2 : extension, fichiers, crédits, inbox scellée, annuaire) est soutenu par la recherche, avec six ajustements listés plus bas.

## Key Findings

### Recommended Stack

Détail dans [STACK.md](./STACK.md). Un seul fournisseur (Cloudflare) pour calcul, base, fichiers et clé-valeur : c'est la plateforme que la doc Anthropic cite pour les serveurs MCP distants, le modèle sans état de la spec MCP 2026-07-28 correspond à celui d'un Worker, et il n'y a pas de démarrage à froid devant `/token` (Claude abandonne après 10 secondes). Un seul langage (TypeScript) permet de partager schémas et chiffrement entre serveur, site et extension dans un monorepo pnpm.

Plus de la moitié des paquets recommandés ont une dernière version de moins d'un mois : épingler les versions exactes, versionner le lockfile, activer un âge minimal de publication.

**Core technologies:**
- **Cloudflare Workers + `wrangler` 4.168.0** : hébergement du Worker unique — pas de démarrage à froid, drapeaux `nodejs_compat` et `global_fetch_strictly_public` requis (sans le second, CIMD n'est pas annoncé).
- **`@modelcontextprotocol/server` 2.4.0** (SDK v2) : serveur MCP sans état via `createMcpHandler(factory)` — sert la révision 2026-07-28 et les clients de 2025 ; version exigée plus tard pour les transferts de fichiers. Ne jamais passer `legacy: 'reject'`.
- **`@cloudflare/workers-oauth-provider` 1.4.0** : serveur d'autorisation OAuth 2.1 (CIMD + DCR, PKCE, RFC 8414 / 9728, port loopback pour Claude Code) — seule bibliothèque vérifiée qui couvre tout ce que Claude exige. **Choix non prouvé contre le vrai Claude : c'est l'objet du spike.**
- **`better-auth` 1.7.28** : identité de premier niveau (Google, lien magique, sessions) — la page `/authorize` lit sa session puis appelle `completeAuthorization()`.
- **`hono` 4.13.23** : routeur de l'API et des pages serveur, client typé `hc` pour le site et l'extension.
- **Cloudflare D1 + `drizzle-orm` 0.45.4** : métadonnées, contacts, registre de crédits — débit atomique par `db.batch()` et contrainte `CHECK (balance >= 0)` ; pas de `db.transaction()`.
- **Cloudflare R2** : contenus chiffrés avant écriture, sur un domaine qui nous appartient. **KV** : réservé aux autorisations OAuth.
- **`zod` 4.6.7** : schémas partagés, pair requis du SDK MCP v2.
- **`resend` 6.40.1 + `react-email` 6.11.17** : e-mails transactionnels (offre gratuite plafonnée à 100 par jour, LOW).
- **`wxt` 0.21.5 + React 19.3.1** (jalon 2) : extension MV3 — side panel indépendant du DOM de claude.ai, `createShadowRootUi` pour le peu d'interface injectée.
- **`stripe` 23.2.0** (jalon 2) : Checkout `mode: 'payment'` + webhook idempotent ; Managed Payments si l'éligibilité est accordée (TVA gérée par Stripe).
- **`age-encryption` 0.3.1 + `@noble/curves` 2.4.0** (jalon 2) : format de chiffrement et signature de l'expéditeur (age n'authentifie pas l'expéditeur).

**Repli si le spike OAuth échoue :** Better Auth seul (`@better-auth/mcp` + `@better-auth/cimd`), puis un fournisseur hébergé qui livre CIMD (WorkOS, Clerk, Stytch).

### Expected Features

Détail dans [FEATURES.md](./FEATURES.md). Le tuyau est une commodité ; la valeur est dans la confiance et la simplicité.

**Must have (table stakes):**
- Envoi d'un brief texte en langage naturel, avec clé d'idempotence et reçu (identifiant, taille, empreinte).
- Boîte de réception : liste (métadonnées seules) puis lecture.
- Contacts validés des deux côtés, révocables, blocage ; aucune note libre dans la demande de contact.
- Compte en deux clics relié à l'OAuth du connecteur.
- Installation guidée par le lien de préremplissage officiel, en disant la vérité sur le nombre d'étapes.
- **Page web de message** : aperçu, accepter ou refuser, lire, copier — à la fois barrière humaine, repli de lecture universel et cible du lien d'invitation.
- Contenu reçu sous enveloppe « donnée non fiable » à délimiteur aléatoire ; lecture refusée tant que non accepté.
- Notification e-mail à l'arrivée (Claude ne reçoit aucun push).
- Accusés livré / accepté / lu, expiration par défaut, rappel d'un envoi.
- Compteur d'envois visible ; plafonds de débit sur envois et invitations.

**Should have (competitive):**
- Carnet de contacts inter-organisations avec résolution par prénom côté serveur et désambiguïsation.
- Lien magique où le premier message est la demande de contact (message retenu, l'acceptation crée le contact).
- Acceptation par geste humain hors de portée du modèle — aucun concurrent ne le documente.
- Confirmation de sortie à l'envoi (« voici exactement ce qui part, à qui »).
- Brief structuré de passation (objectif, contexte, décisions, contraintes, questions ouvertes).
- Fidélité vérifiable (octets d'origine + SHA-256, « voir l'original » hors conversation).
- Extension Chrome : dépôt de fichiers, « Envoyer à… », badge.
- Invitations gratuites, recevoir toujours gratuit, récompense des deux côtés à l'activation réelle.

**Defer (v2+):**
- Lot « dossier » avec manifeste et lecture à la demande.
- Fiche dans l'annuaire Anthropic et transferts de fichiers par la conversation.
- Capture fidèle d'une conversation par l'extension.
- Bout en bout réel (déchiffrement local, ordinateur seulement).
- Carte MCP App d'acceptation dans la conversation (dépend d'une question ouverte).
- Espaces d'équipe, autres IA.

**Anti-fonctionnalités à ne pas construire :** ingestion automatique, conversation autonome Claude à Claude, écriture dans un Projet, mention « bout en bout » sur le canal connecteur, promotion dans les résultats d'outil, paiement dans la conversation, lecture payante, annuaire public d'utilisateurs, e-mails d'invitation avec texte libre de l'expéditeur.

### Architecture Approach

Détail dans [ARCHITECTURE.md](./ARCHITECTURE.md). Un monolithe modulaire, un seul déployable : deux façades minces (MCP Gateway pour Claude, REST API pour le site et l'extension) au-dessus d'un même domaine métier. Toute règle (contact validé, crédits, acceptation) vit dans une Policy Gate unique ; si elle vivait dans un outil MCP, l'extension la contournerait. L'état vit en base, jamais dans une session MCP ni dans l'extension. Un compte, trois justificatifs à audiences distinctes (connecteur, extension, session web) : un jeton volé d'un côté ne sert pas de l'autre.

**Major components:**
1. **Auth / Identity** — comptes sans mot de passe, serveur d'autorisation OAuth 2.1, consentement, grants.
2. **MCP Gateway** — point d'entrée Streamable HTTP unique ; garde `401` avant le SDK ; enveloppe « donnée non fiable » ; pagination. Aucune logique métier.
3. **REST API** — mêmes cas d'usage, pour les pages web et l'extension.
4. **Relay Core + Policy Gate** — machines à états (contact, message, invitation), registre de crédits en ajout seul, décision unique « ce contenu peut-il sortir vers ce Claude ? ».
5. **Vault** — seule frontière cryptographique (`seal` / `open`), enveloppes de clés multiples par message dès le départ.
6. **Ingest** (jalon 2) — extraction, découpage, analyse, manifeste, aperçu.
7. **Pages web** — connexion, consentement, page de message, lien magique, dépôt, crédits.
8. **Extension Chrome** (jalon 2) — side panel et badge ; content script minuscule ; ne parle qu'à la REST API et au DOM.

**Patterns à suivre :** un domaine, deux façades ; garde d'authentification avant le SDK (vrai `401` HTTP, jamais un `200` avec `isError`) ; enveloppes de clés multiples (type `kms` seul au départ, `grant` et `device` ensuite sans migration) ; manifeste d'abord, pages ensuite ; porte d'acceptation côté serveur ; écritures idempotentes.

### Critical Pitfalls

Détail dans [PITFALLS.md](./PITFALLS.md) (17 pièges, dont 11 critiques).

1. **Croire que l'étiquette « non fiable » est la défense.** L'aperçu renvoyé au modèle est déjà une ingestion ; un outil `accept` peut être appelé par du texte injecté ; l'approbation par appel de Claude n'est pas un contrôle que nous possédons (« toujours autoriser », mode Research). — Aperçu et acceptation sur une surface humaine hors modèle ; avant acceptation, métadonnées produites par le serveur uniquement ; corpus d'injection en régression contre un compte qui a Gmail et Drive connectés.
2. **Notre propre outil d'envoi est le canal d'exfiltration et de ver.** L'attaquant est par construction un contact validé. — Règle de sortie déterministe côté serveur après lecture d'un contenu tiers, plafonds de débit et de diffusion, pas d'outil qui liste tout le carnet, jamais de renvoi automatique.
3. **OAuth conforme à la spec mais pas à Claude.** `401` + `WWW-Authenticate`, `resource` égal à l'URL saisie, CIMD annoncé correctement, `/token` en form-urlencoded, 10 secondes maximum, pas de redirection inter-hôtes, pas de WAF devant la plage `160.79.104.0/21`, rotation des refresh tokens avec fenêtre de grâce, attaque de liaison de compte. — Ne pas écrire le serveur d'autorisation à la main ; spike sur chaque surface, compte Free et payant.
4. **Concevoir fichiers, dossiers et conversations sur un canal qui ne porte que du texte tapé.** Un long texte est régénéré token par token : lent, tronqué, paraphrasé — la promesse « intact » casse. — Périmètre connecteur = texte ; reçu avec empreinte ; canal latéral pour le reste.
5. **Collision tardive avec la politique de l'annuaire** (clauses 1.E, 1.G, 2.D/2.E, 2.F, 4.A, 4.C). — La traiter comme contrainte de conception dès le premier outil ; zéro clair dans les journaux ; écrire à `mcp-review@anthropic.com` avant de dépendre d'une fiche.
6. **Annoncer « bout en bout » quand le serveur déchiffre** (précédent Zoom / FTC). — Formulation unique, écrite une fois dans une page sécurité.
7. **Marque.** Ni « Claude » ni « Anthropic » dans le nom, le domaine, le nom affiché du connecteur, l'icône, l'écran de consentement. Domaine définitif avant le premier utilisateur externe.

## Implications for Roadmap

### Verdict sur le découpage en deux jalons

**La recherche soutient le découpage proposé.** Les quatre fichiers convergent : spike en premier, OAuth ensuite, cœur texte avec la porte d'acceptation avant toute extension, puis fichiers, crédits, inbox scellée, annuaire. Le critère de succès v1 (le porteur et 2 ou 3 contacts, chaque semaine) est atteignable avec le jalon 1 seul.

**Six points où la recherche demande un ajustement :**

| # | Point | Ce que dit la recherche | Recommandation |
|---|-------|-------------------------|----------------|
| 1 | Le jalon 1 ne mentionne ni la page web de message ni la notification e-mail | Ce sont des prérequis, pas des options. Sans page web il n'y a aucune surface d'acceptation hors modèle sur mobile, Desktop et Claude Code. Sans e-mail personne ne sait qu'un message existe (aucun push vers Claude, pas d'extension au jalon 1). | Les inscrire explicitement au jalon 1. |
| 2 | Crédits entièrement au jalon 2 | FEATURES et ARCHITECTURE veulent le **registre en ajout seul, l'idempotence et le compteur visible dès le premier envoi** (solde illimité). Seuls le blocage, le paiement et le parrainage attendent. | Registre + compteur au jalon 1 ; Stripe au jalon 2. Évite une migration. |
| 3 | Inbox scellée au jalon 2 | D'accord, à condition que le module Vault isolé et les enveloppes de clés multiples (type `kms`) existent dès le jalon 1, et que la formulation de confidentialité soit décidée **avant de figer le schéma de données**. | Décision « que veut dire chiffré » au début du jalon 1, implémentation B au jalon 2. |
| 4 | Lien magique au jalon 1 | Désaccord partiel entre fichiers. FEATURES place le lien magique complet (message retenu pour un non-inscrit) en v1.x, après validation de l'usage ; ARCHITECTURE le place en étape 2. Les deux veulent l'ajout de contact par lien d'invitation dès le début. | Garder les deux au jalon 1 mais en deux phases : contacts mutuels par lien d'abord (nécessaires à tout envoi), message retenu pour non-inscrit ensuite. La seconde peut glisser sans bloquer le critère de succès. |
| 5 | Confirmation de sortie à l'envoi | FEATURES : « avant toute ouverture au public ». PITFALLS : dans le même plan que le premier outil `send`. | Jalon 1 : plafonds de débit, plafond de diffusion, pas de liste complète des contacts, journal de métadonnées, coupe-circuit par compte. La confirmation humaine hors modèle après lecture d'un contenu tiers doit être livrée **au plus tard avec la phase lien magique**, qui ouvre le cercle au-delà des proches. |
| 6 | Fiche d'annuaire au jalon 2 | FEATURES la classe P3 / v2+ ; ARCHITECTURE en dernière étape. Elle dépend d'une revue Anthropic, de fonctions en bêta, et l'interprétation de la clause 1.G pour « envoie cette conversation » n'est pas tranchée. | La mettre en dernière phase du jalon 2, sans rien faire dépendre d'elle. Écrire à Anthropic tôt (dès le jalon 1) pour obtenir une réponse écrite. Le pipeline de fichiers du jalon 2 ne doit pas l'attendre : il passe par le lien de dépôt web et l'extension. |

Un septième point, mineur : le **pipeline de fichiers ne doit pas dépendre de l'extension seule**. ARCHITECTURE introduit un lien de dépôt web (`create_upload_link`), seul chemin de fichiers qui marche sur mobile. À construire avant ou avec l'envoi de fichiers par l'extension.

### Jalon 1 — Cœur de valeur sur toutes les surfaces, sans extension

### Phase 1: Spike de plateforme et décisions irréversibles
**Rationale:** Cinq hypothèses de PROJECT.md sont fausses ou non vérifiées, et le choix le plus risqué de la stack (serveur d'autorisation) n'a été testé contre aucun vrai Claude. Un à deux jours, sur de vrais comptes.
**Delivers:** Un Worker avec un outil `ping` et l'OAuth complet, connecté sur claude.ai web (compte Free neuf et compte payant), app mobile, Desktop et Claude Code. Mesures : taille pratique et fidélité d'un argument d'outil (5 000, 20 000, 50 000 caractères), expérience d'approbation, lien d'installation, comportement réel de `claude.ai/new?q=`, présence automatique du connecteur sur mobile, chargement des images Markdown dans un résultat d'outil. Décisions : nom commercial sans « Claude », **domaine de production définitif**, région d'hébergement européenne, formulation de confidentialité. Courriel à `mcp-review@anthropic.com`.
**Addresses:** Questions ouvertes 1, 3, 5, 6 de FEATURES ; lacunes d'ARCHITECTURE.
**Avoids:** Pièges 3, 5, 8 ; dette « domaine provisoire ».

### Phase 2: Identité, serveur d'autorisation et squelette du connecteur
**Rationale:** Tout dépend de l'OAuth. La doc Claude le dit : l'authentification se décide avant d'écrire les outils.
**Delivers:** Compte (Google + code ou lien par e-mail), page de consentement qui nomme le compte en clair, CIMD avec DCR en repli, garde `401` avant le SDK, outil `whoami`, page d'installation avec le lien de préremplissage et captures d'écran, page « appareils connectés » avec révocation, journaux de métadonnées seulement (filtre au niveau du transport).
**Uses:** Workers, `workers-oauth-provider`, Better Auth, Hono, D1 + Drizzle, KV, SDK MCP v2.
**Implements:** Auth / Identity, MCP Gateway, squelette du Core et du Vault.
**Avoids:** Piège 8 (dont l'attaque de liaison de compte : un jeton de lien magique n'est jamais un justificatif OAuth), piège 11 (région, journaux), dette DCR seul. Critère de sortie : la connexion survit 24 heures et un redémarrage sans nouvelle demande de connexion.

### Phase 3: Envoi et lecture de texte avec acceptation hors modèle
**Rationale:** C'est la valeur centrale, et le modèle de sécurité doit être livré avec le premier outil de lecture, jamais après. Les contacts mutuels sont un prérequis de tout envoi : ils entrent ici dans leur forme minimale.
**Delivers:** Contacts validés des deux côtés par lien d'invitation entre comptes existants, blocage, suppression. Outils `send_message` (écriture, idempotent, brief structuré, reçu avec identifiant, taille et empreinte), `list_inbox` (lecture seule, métadonnées produites par le serveur), `open_message` (refuse tout message non accepté). **Page web de message** : aperçu rendu à l'humain seulement, Accepter / Refuser, lecture de repli, copie, « voir l'original ». Enveloppe à délimiteur aléatoire, assainissement à l'ingestion et à la lecture (caractères invisibles, images et liens Markdown neutralisés). Notification e-mail. Accusés livré / accepté / lu, expiration par défaut, rappel. Registre en ajout seul et compteur visible, sans blocage. Plafonds de débit. Vault en mode chiffrement d'enveloppe au repos.
**Addresses:** Toutes les fonctionnalités P1 de FEATURES.
**Avoids:** Pièges 1, 2, 3, 12, 13, 14. Critères de sortie : avec « toujours autoriser » activé, un message non accepté n'expose aucun texte écrit par l'expéditeur ; deux contacts au même prénom forcent un choix ; une chaîne canari n'apparaît dans aucun journal ; la boucle complète marche sur mobile.

### Phase 4: Invitation d'un destinataire sans compte (lien magique)
**Rationale:** Moteur d'acquisition, mais première ouverture au-delà du cercle de confiance : elle vient après un modèle de sécurité complet.
**Delivers:** Envoi à une adresse inconnue = invitation avec message retenu ; **l'expéditeur transmet le lien lui-même** (WhatsApp, e-mail) ; page d'atterrissage avec métadonnées vérifiées seulement ; contenu visible après preuve de contrôle de l'adresse invitée ; l'acceptation crée le contact mutuel ; parcours création de compte, lien d'installation, consentement, invite courte à copier ; lecture web « sans rien installer » pour les destinataires bloqués (emplacement Free déjà pris, membre non Owner d'une organisation). Jeton consommé sur un POST avec code de secours. Plafond quotidien d'invitations. Confirmation de sortie hors modèle si elle n'a pas été livrée en phase 3.
**Avoids:** Pièges 10 et 11 (pas d'e-mail envoyé par la plateforme à des non-utilisateurs, pas de texte libre, adresse conservée le temps de l'invitation seulement), liaison de compte.

### Jalon 2 — Extension, fichiers, monétisation, durcissement

### Phase 5: Extension légère
**Rationale:** N'a besoin que des API du jalon 1 ; livrable visible rapidement.
**Delivers:** Passage de session depuis le site, side panel (contacts, boîte de réception, aperçu et acceptation), badge par `chrome.alarms`, bouton « Ouvrir dans Claude » (préremplissage tenté + copie dans le presse-papiers), état « connecteur installé » lu sur le backend.
**Uses:** WXT, React, `@webext-core/messaging`.
**Avoids:** Pièges 9, 15, 16. Critère : l'extension reste utilisable avec tous les sélecteurs de claude.ai cassés ; la boucle centrale passe toujours extension désinstallée.

### Phase 6: Pipeline de contenus
**Rationale:** Le sens « envoi de fichiers » exige un canal latéral ; c'est une étape à part entière, indépendante de l'extension.
**Delivers:** Ingest (Markdown, code, images, texte extrait des PDF), stockage R2 chiffré, manifeste, `read_part` paginé, lien de dépôt web (`create_upload_link`), téléchargement de l'original pour l'humain, liste blanche de types, plafonds de taille, contenu utilisateur servi depuis un domaine enregistrable distinct.
**Avoids:** Pièges 3 et 10 ; anti-pattern « un outil qui renvoie tout ».

### Phase 7: Extension — envoi de fichiers et capture
**Rationale:** Dépend des phases 5 et 6.
**Delivers:** Bouton « Envoyer à… », glisser-déposer, capture de la conversation affichée sur clic explicite avec aperçu de ce qui part.
**Avoids:** Piège 9 (aucun appel aux endpoints internes, aucune observation passive, aucun envoi automatique).

### Phase 8: Crédits et paiement
**Rationale:** Le registre existe depuis la phase 3 ; on branche le blocage et l'achat sans migration, une fois l'usage hebdomadaire établi.
**Delivers:** Dotation mensuelle, alerte avant zéro, envoi mis en attente à zéro (jamais perdu), erreur factuelle avec un seul lien vers la page de compte, Stripe Checkout + webhook idempotent, parrainage déclenché par une activation réelle.
**Avoids:** Pièges 4 et 17 ; débit seulement sur livraison confirmée ; recevoir reste gratuit.

### Phase 9: Inbox scellée
**Rationale:** À faire sur un produit utilisé : une gestion de grants mal faite perd des messages.
**Delivers:** Chiffrement vers une clé publique par grant, déchiffrement éphémère lié au jeton vivant, clé d'appareil non extractible pour l'extension, signature de l'expéditeur, avertissement de changement de clé, page sécurité avec la formulation exacte.
**Avoids:** Pièges 6 et 7.

### Phase 10: Durcissement public et annuaire
**Rationale:** Dépend d'une revue Anthropic, d'une revue Chrome Web Store et d'un avis juridique.
**Delivers:** Publication Chrome Web Store (prévoir deux semaines de marge), documents RGPD et revue par un juriste, opérations anti-abus (signalement, blocage, contact), soumission à l'annuaire, puis transferts de fichiers MCP et éventuellement carte MCP App.
**Avoids:** Pièges 4, 5, 9, 10, 11.

### Phase Ordering Rationale

- **Dépendances réelles.** OAuth conditionne tout ; les contacts conditionnent l'envoi ; la page web de message conditionne l'aperçu, le repli de lecture et le lien magique ; l'ingestion conditionne l'envoi de fichiers par l'extension ; le registre de crédits posé tôt évite une migration.
- **Sécurité avant portée.** Le modèle de sécurité (phase 3) est complet avant que le lien magique (phase 4) et l'extension (phases 5 et 7) n'élargissent l'audience. Chaque contrôle est construit côté connecteur d'abord ; l'extension ajoute du confort, jamais de la protection.
- **Irréversible en premier.** Nom, domaine, région d'hébergement et formulation de confidentialité se décident en phase 1 : une URL de connecteur ne se modifie pas, et un utilisateur Free n'a qu'un emplacement.
- **Cycles courts.** Chaque phase livre quelque chose de visible ; le jalon 1 tient en quatre phases et atteint le critère de succès v1 sans l'extension.
- **Rien ne dépend de l'annuaire.** Les bénéfices d'une fiche (fichiers par la conversation, découverte) sont réels mais hors de notre contrôle.

### Exigences de PROJECT.md à reformuler

Les six demandées par l'orchestrateur, puis quatre que la recherche ajoute.

| # | Formulation actuelle | Problème établi | Reformulation proposée |
|---|----------------------|-----------------|------------------------|
| 1 | « tu send les dossiers dans le bon projet de ton gars » (inspiration) ; « dossier / projet entier » | Un connecteur MCP ne peut pas écrire dans la base de connaissances d'un Projet. Le faire exigerait l'API interne, déjà hors périmètre. | « Le contenu arrive dans une conversation du destinataire. Un lot téléchargeable permet de l'ajouter soi-même à un Projet. » Le dire dans le produit. |
| 2 | « Chiffrement de bout en bout visé » | Le serveur voit le clair à l'envoi et à la lecture ; Anthropic aussi. La promesse est fausse sur le canal principal. | « Chiffré en transit et au repos, clé par message, expiration courte, aucun contenu dans les journaux. » Puis, après l'inbox scellée : « nous ne pouvons pas lire les messages stockés sans votre connexion active ». Le terme « bout en bout » est réservé à une voie locale optionnelle future. Passer la décision clé correspondante de « Pending » à tranchée. |
| 3 | « un clic installe le connecteur » | Le lien officiel préremplit seulement ; l'utilisateur relit, confirme, puis se connecte en OAuth, avec un avertissement. Comptes entre 3 et 6 étapes selon les fichiers. | « Installation guidée en quelques étapes via le lien de préremplissage officiel ; un destinataire qui ne peut pas installer lit le message sur le web. » |
| 4 | Bouton « Ouvrir dans Claude » | `https://claude.ai/new?q=` n'est pas documenté, a déjà changé sans préavis et a servi de point d'entrée à une attaque publiée. Seul `claude://` (Desktop) est documenté, en préremplissage sans envoi. | « Le bouton tente le préremplissage, copie une invite courte dans le presse-papiers et affiche "colle ici". L'invite ne contient jamais le brief. Aucun parcours n'en dépend. » |
| 5 | « Depuis n'importe quel Claude… dire "envoie X" et le contenu part » + « fichiers (.md, PDF, code, images) » | Vrai pour le texte seulement. Sur un connecteur custom, Claude n'envoie aucun fichier et retire ceux des résultats, sans erreur. | Scinder par canal : « texte et Markdown par la phrase, sur toutes les surfaces » ; « fichiers et dossiers par un lien de dépôt web ou l'extension » ; « fichiers par la conversation après la fiche d'annuaire ». Les PDF sont lus comme texte extrait. |
| 6 | « conversation ou extrait » | La clause 1.G de la politique d'annuaire interdit d'extraire l'historique ou les fichiers de l'utilisateur. L'interprétation pour un envoi demandé par l'utilisateur n'est pas tranchée. Un texte long régénéré par le modèle est en plus paraphrasé. | « L'utilisateur désigne explicitement ce qui part ; Claude rédige un brief structuré. La capture fidèle d'une conversation est une action de l'extension, sur clic, avec aperçu. » Aucun outil de type `export_conversation`. Obtenir une réponse écrite d'Anthropic. |
| 7 | « message "plus de crédits + lien d'achat" renvoyé par le MCP » | Clause 4.C et grille de revue : pas de promotion. Un lien d'achat dans la conversation habitue aussi les utilisateurs à cliquer sur un faux message injecté. | « Erreur factuelle, uniquement sur l'envoi qui échoue : limite atteinte, envoi mis en attente, un seul lien vers la page de compte. Jamais dans un résultat de lecture, jamais de lien de paiement direct. » |
| 8 | « Le destinataire voit un aperçu et accepte avant que son Claude ingère » | Un aperçu renvoyé comme résultat d'outil est déjà une ingestion ; une acceptation exposée comme outil peut être déclenchée par une injection. | « L'aperçu est montré à l'humain hors du modèle (page web, extension). L'acceptation est un geste humain imposé par le serveur ; aucun outil ne peut accepter. » |
| 9 | « badge de notification » comme seule notification | Aucun push vers Claude ; pas d'extension sur mobile, Desktop, Claude Code. | Ajouter « notification e-mail à l'arrivée » comme exigence de base ; le badge est un complément. |
| 10 | « connecteurs claude.ai synchronisés sur l'app » (hors périmètre mobile) ; « contacts validés » sans plus | La synchronisation vers le mobile n'est pas écrite noir sur blanc ; à vérifier en phase 1. Plan Free : un seul connecteur custom ; Team / Enterprise : seul un Owner installe. | Ajouter l'exigence « lecture de repli sur le web » et la contrainte « une seule URL de connecteur, définitive ». |

### Research Flags

Phases qui demandent une recherche dédiée pendant la planification (`/gsd-plan-phase --research-phase <N>`) :
- **Phase 1 :** c'est la recherche elle-même — limites non documentées, mesurées sur de vrais comptes.
- **Phase 2 :** ciblée — détails OAuth propres à Claude (CIMD, découverte, rotation, taille de jeton), câblage `workers-oauth-provider` + Better Auth selon le résultat du spike.
- **Phase 3 :** ciblée — surface d'acceptation hors modèle sur chaque appareil, corpus d'injection, règle de sortie.
- **Phase 6 :** oui — qualité d'extraction PDF dans un Worker, règles d'exclusion de dossiers, rendu d'images.
- **Phase 9 :** **forte** — conception cryptographique, cycle de vie des grants, matrice de support des passkeys PRF, risque de perte de messages.
- **Phase 10 :** oui — texte des politiques au moment de la soumission, fonctions en bêta, revue juridique.

Recherche légère :
- **Phase 4 :** anti-abus des invitations, base légale, liens consommés par les scanners de messagerie.
- **Phase 5 :** passage de session vers l'extension, cookies `SameSite` pour une extension.
- **Phase 7 :** capture de conversation par le DOM, formulation des politiques Chrome Web Store.
- **Phase 8 :** éligibilité Stripe Managed Payments, règles européennes sur les crédits prépayés (non recherchées), vérification asynchrone des webhooks dans un Worker.

Schémas standards, sans recherche supplémentaire : Checkout et webhook Stripe, e-mails Resend, schéma Drizzle sur D1, squelette WXT.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Versions lues sur le registre npm (HIGH). Faits de plateforme Claude lus sur pages officielles (MEDIUM). Le choix du serveur d'autorisation n'a été testé contre aucun vrai Claude. Compatibilité de l'outillage avec TypeScript 7, X25519 dans le WebCrypto de Workers, juridiction européenne D1 / R2 : non revérifiés (LOW). |
| Features | MEDIUM | Contraintes de plateforme recoupées sur pages officielles. Rien n'est HIGH. Analyse concurrentielle fondée sur des pages éditeur ; chiffres de conversion, risque natif à 6-12 mois et calibrage des crédits sont LOW. |
| Architecture | MEDIUM | Les six faits structurants sont officiels. Les propositions (budget de 40 000 caractères, inbox scellée, passage de session) sont des conceptions non testées. |
| Pitfalls | MEDIUM | Politiques Anthropic et Chrome lues sur pages primaires. Les pièges propres au produit (ver par l'outil d'envoi, liaison de compte, aperçu = ingestion) sont des inférences solides mais non testées. Le volet RGPD repose surtout sur des sources secondaires. |

**Overall confidence:** MEDIUM

Suffisant pour structurer la feuille de route. Insuffisant pour figer la phase 2 avant le résultat du spike.

### Désaccords entre fichiers de recherche

À trancher pendant la planification ; la recommandation est indiquée.

- **Budget par résultat d'outil :** 40 000 caractères (ARCHITECTURE, sous le seuil de Claude Code) contre « moins de 100 000 » (PITFALLS). Retenir **40 000**, un seul budget pour toutes les surfaces.
- **Bibliothèque de chiffrement :** `age-encryption` (STACK) contre libsodium (PITFALLS). Retenir **age + noble** : libsodium est en WebAssembly et impose `wasm-unsafe-eval` dans la CSP de l'extension. Décision à confirmer en phase 9.
- **Déchiffrement dans l'extension :** STACK et PITFALLS décrivent un palier « extension vers extension » en bout en bout réel ; ARCHITECTURE l'écarte comme chemin principal, parce que le clair entrerait dans Claude comme message utilisateur collé. Conciliation : l'extension déchiffre localement **pour l'aperçu humain seulement** ; ce que lit Claude passe toujours par un résultat d'outil sous enveloppe.
- **`open_message` déclaré outil d'écriture** pour faire apparaître l'approbation de Claude (ARCHITECTURE) contre « l'approbation par appel n'est pas un contrôle » (PITFALLS). Les deux tiennent : le déclarer en écriture est exact et utile, mais **ne remplace pas** la porte hors modèle.
- **Outil qui liste les contacts :** présent dans le MVP de FEATURES, déconseillé par PITFALLS. Retenir une **résolution un par un** côté serveur, avec candidats en cas d'ambiguïté.
- **Connexion de l'extension :** cookie de session avec `credentials: 'include'` (STACK, LOW) contre code à usage unique par `externally_connectable` (ARCHITECTURE). Retenir le **passage de session**, origine restreinte à notre domaine.
- **Permissions d'hôte de l'extension :** claude.ai + domaine de l'API (STACK) contre claude.ai seul (PITFALLS). À trancher en phase 5 selon le mécanisme d'authentification retenu.
- **Nombre d'étapes d'installation :** deux confirmations (ARCHITECTURE), 3 à 4 clics (STACK, FEATURES), 4 à 6 étapes (PITFALLS). À mesurer en phase 1 ; ne rien promettre avant.
- **Séparation de la clé maîtresse :** secret du Worker (STACK) contre KMS séparé de la base (ARCHITECTURE). Un secret Worker est déjà hors de D1 ; acceptable au jalon 1, à revoir en phase 9.

### Gaps to Address

- **Surface humaine dans Claude sur chaque appareil** (une MCP App peut-elle changer l'état serveur sur un clic sans appel d'outil du modèle ; l'élicitation existe-t-elle sur claude.ai) : non établi. Tant que ce n'est pas prouvé, l'acceptation reste sur la page web et dans l'extension. Essai en phase 1.
- **`https://claude.ai/new?q=` aujourd'hui** : non vérifié. Test de cinq minutes en phase 1 ; le repli presse-papiers existe quoi qu'il arrive.
- **Taille et fidélité d'un argument d'outil long** : aucune source. Mesure en phase 1 ; conditionne la promesse « intact ».
- **Synchronisation du connecteur vers le mobile et Claude Code** : non écrite noir sur blanc. Vérification en phase 1.
- **Lecture par Anthropic des clauses 1.G et 4.C** pour l'envoi demandé par l'utilisateur et le message de quota : seule Anthropic peut répondre. Courriel en phase 1, réponse attendue avant la phase 8.
- **Limite « un connecteur » du plan Free appliquée ou non aux connecteurs d'annuaire** : non précisé. Change la taille du marché adressable.
- **Chargement des images Markdown dans les résultats d'outil de claude.ai** comme canal d'exfiltration : non testé. Neutraliser par défaut, tester en phase 3.
- **Taille de jeton OAuth acceptée par Claude** : non documentée. Mesure avant de concevoir l'inbox scellée.
- **Usage descriptif de « Claude »** (« fonctionne avec Claude ») : la page marques ne le traite pas. Mention sobre avec « non affilié à Anthropic » ; avis juridique avant le lancement public.
- **Passkeys PRF** sur Chrome, Safari, Android, iOS : à établir en phase 9.
- **Règles européennes sur les crédits prépayés** (rétractation, TVA) et points RGPD : non recherchés ou secondaires. Juriste avant la phase 10.
- **Dotation gratuite et prix** : aucune donnée fiable. À calibrer sur l'usage réel du jalon 1.
- **Demande elle-même** : aucun analogue ne montre de traction, et le segment non technique n'a jamais été testé. Le critère de succès v1 est le bon test ; ne pas investir dans le jalon 2 avant qu'il soit atteint.

## Sources

Aucune source n'atteint HIGH pour les faits de plateforme ; seules les versions de paquets le sont.

### Primary (HIGH confidence)
- Registre npm, interrogé le 2026-11-27 — versions, plages de dépendances paires, dates de publication ; contenu lu directement pour `@cloudflare/workers-oauth-provider`, `@better-auth/mcp`, `@better-auth/cimd`, `age-encryption`, `@hpke/core`.

### Secondary (MEDIUM confidence)
Pages officielles lues le 2026-11-27 et recoupées.
- https://claude.com/docs/connectors/building — transport, limites de taille et de durée, fonctions non prises en charge
- https://claude.com/docs/connectors/building/authentication et `/lazy-authentication` — CIMD et DCR, redirections, découverte, latence, plage réseau, garde `401`
- https://claude.com/docs/connectors/building/directory-vs-custom — lien d'installation, fonctions réservées à l'annuaire
- https://claude.com/docs/connectors/building/files — transferts de fichiers, conditions
- https://claude.com/docs/connectors/building/review-criteria et https://support.claude.com/en/articles/13145358-anthropic-software-directory-policy — annotations, clauses 1.E, 1.G, 2.F, 4.A, 4.C
- https://claude.com/docs/connectors/building/mcp-apps/getting-started (et pages associées) — cartes dans la conversation, limites, mention « Unverified »
- https://claude.com/docs/connectors/custom/add-unlisted et https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp — limite du plan Free, Team / Enterprise, mode Research
- https://support.claude.com/en/articles/14729294-open-claude-desktop-with-a-link — schéma `claude://`, préremplissage
- https://code.claude.com/docs/en/mcp — limites de sortie, connecteurs claude.ai dans Claude Code
- https://modelcontextprotocol.io/specification/2026-07-28/changelog et https://ts.sdk.modelcontextprotocol.io/v2/ — révision du protocole, SDK v2
- https://developers.cloudflare.com/agents/model-context-protocol/ , `/d1/platform/limits/` , `/r2/api/s3/presigned-urls/` — `createMcpHandler`, limites D1, URL présignées
- https://developer.chrome.com/docs/webstore/program-policies/policies et https://developer.chrome.com/blog/cws-policy-updates-2026 — code distant, usage limité, clause IA
- https://www.anthropic.com/legal/consumer-terms et `/legal/trademark-guidelines` — marque, accès automatisé
- https://docs.stripe.com/payments/managed-payments/eligibility — éligibilité
- https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/ — modèle de menace d'injection
- Pages éditeur des produits comparables (Threadpacket, Patchcord, agent2agent, MCP Agent Mail, share-context-mcp) — existence et liste d'outils

### Tertiary (LOW confidence)
- https://github.com/anthropics/claude-code/issues/8827 et https://austegard.com/blog/claudeainew-unofficial-readme.html — comportement de `claude.ai/new?q=`, à tester
- https://oasis.security/blog/claude-ai-prompt-injection-data-exfiltration-vulnerability — chaîne d'attaque par URL préremplie
- https://invariantlabs.ai/blog/whatsapp-mcp-exploited et https://www.bold.security/blog/account-confusion-risk-in-claude-connectors-allows-corporate-data-exfiltration-to-personal-accounts — détournement d'un outil d'envoi, confusion de comptes
- https://resend.com/pricing — plafonds de l'offre gratuite (source unique)
- Sources RGPD et invitations (autorités irlandaise et belge, résumé de la position de la CNIL, IAPP) — à faire valider par un juriste
- Blogs de croissance et de facturation à crédits (OpenView, Andrew Chen, Chargebee, Schematic) — boucle virale, calibrage
- https://datatracker.ietf.org/doc/html/draft-li-pearg-ciphertext-inference-tool-mcp-00 — brouillon individuel, hors de portée
- Rapports communautaires sur `launchWebAuthFlow` et les liens magiques consommés par les scanners

Listes complètes dans [STACK.md](./STACK.md), [FEATURES.md](./FEATURES.md), [ARCHITECTURE.md](./ARCHITECTURE.md) et [PITFALLS.md](./PITFALLS.md).

---
*Research completed: 2026-11-27*
*Ready for roadmap: yes*
