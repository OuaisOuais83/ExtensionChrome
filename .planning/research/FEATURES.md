# Feature Research

**Domain:** Partage de contexte de Claude à Claude entre personnes (messagerie d'agent à agent via connecteur MCP distant + extension Chrome sur claude.ai), pour founders SaaS / e-commerce non techniques
**Researched:** 2026-11-27
**Confidence:** MEDIUM

> **Lecture des niveaux de confiance.** Les niveaux viennent du seam `classify-confidence` de GSD : une information trouvée par recherche web puis recoupée sur la page officielle Anthropic le jour même est **MEDIUM** (plafond du seam pour ce type de source) ; une source unique (page éditeur, blog) ou une déduction personnelle est **LOW**. Rien ici n'est HIGH. Les déductions sont signalées par « (analyse) ».

## Verdict en cinq lignes

1. **Aucune fonction native « envoyer au Claude de quelqu'un » n'existe** au 27/11/2026. Le natif le plus proche est le partage de chat par e-mail (tous plans, lecture seule, sans fichiers, impossible de continuer ou de copier) et le partage de Projets (Team/Enterprise, même organisation uniquement). L'espace visé est libre, mais Anthropic possède déjà la brique d'invitation par e-mail. (MEDIUM)
2. **Le transport est une commodité.** Threadpacket (hébergé, gratuit jusqu'à 10 personnes) expose déjà `send / inbox / read / revoke / upload_url / invite` en connecteur Claude. Patchcord, agent2agent, MCP Agent Mail font de même. Tous visent des développeurs, aucun n'a de carnet de contacts grand public, de lien magique pour non-inscrits, ni d'UX anti-injection documentée. La différenciation est là, pas dans le tuyau. (MEDIUM)
3. **Un connecteur custom ne transporte pas de fichiers.** Les transferts de fichiers (bêta du 24/11/2026) sont réservés aux connecteurs listés dans l'annuaire avec hôtes approuvés. En custom, Claude n'envoie aucun fichier et retire les fichiers des résultats d'outil. Conséquence directe : l'envoi de PDF / images / dossiers passe par l'extension ou une page web d'upload, pas par la phrase « envoie ce PDF à Paul ». (MEDIUM)
4. **« Un clic installe le connecteur » est faux tel quel.** Il existe un lien officiel qui préremplit la boîte « Add custom connector », mais l'utilisateur doit confirmer puis se connecter en OAuth. Compter 3 à 4 clics. Et le plan Free est limité à **un seul** connecteur custom. (MEDIUM)
5. **« Dans le bon projet de ton gars » n'est pas atteignable par le canal officiel.** Un connecteur MCP ne peut pas écrire dans la base de connaissances d'un Projet. Le contenu arrive dans une conversation. Il faut le dire clairement dans le produit plutôt que le promettre. (analyse, cohérent avec la décision « pas d'API interne »)

## Platform Constraints That Shape Features

Ces faits bornent ce qui est constructible. Ils priment sur toute envie de fonctionnalité.

| Contrainte | Fait vérifié | Conséquence sur les fonctionnalités | Confiance |
|------------|--------------|-------------------------------------|-----------|
| Fichiers via connecteur | Transferts de fichiers : annuaire uniquement, hôtes approuvés par Anthropic, MCP révision 2026-07-28. Custom : « Not available ». | Texte = par l'outil. Binaire = par l'extension ou le web. Lister le connecteur dans l'annuaire débloque les fichiers plus tard. | MEDIUM |
| Taille des résultats | ~150 000 caractères par résultat d'outil sur claude.ai / Desktop ; 25 000 tokens sur Claude Code ; 240 s par appel. | Dossiers = manifeste + lecture fichier par fichier, jamais un seul bloc. | MEDIUM |
| Types de retour | Résultats texte et image, ressources texte et binaires acceptés. Pas d'abonnement aux ressources, pas de sampling. | Le Claude du destinataire peut voir texte et images. **Aucun push** : Claude ne sait jamais seul qu'un message est arrivé. | MEDIUM |
| Plan Free | Un seul connecteur custom. Limite annuaire non précisée. | Un destinataire Free qui a déjà utilisé son emplacement ne peut pas installer. Il faut un repli de lecture web. | MEDIUM |
| Team / Enterprise | Seul un Owner ajoute un connecteur custom pour l'organisation. | Un membre d'une organisation ne peut pas s'auto-installer. Même repli web. | MEDIUM |
| Lien d'installation | `https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=NAME&connectorUrl=ENCODED_URL` préremplit la boîte, avec avertissement « valeurs venues d'un lien externe », confirmation obligatoire. | Le lien magique peut amener l'utilisateur à un clic de l'installation, pas l'installer. | MEDIUM |
| Authentification paresseuse | Des outils publics peuvent tourner avant connexion ; un 401 HTTP déclenche une carte « Connect » dans la conversation puis un nouvel essai automatique. | Possibilité d'un premier usage sans compte préalable côté Claude. | MEDIUM |
| UI dans la conversation (MCP Apps) | Disponible pour tous les utilisateurs sur web, Desktop, iOS, Android. Carte inline : 2 actions max, 4 à 5 données. Un connecteur custom peut être marqué « Unverified » quand son app est ouverte. | La carte « aperçu + Accepter / Refuser » et le sélecteur de contact peuvent exister **sans extension**, y compris sur mobile. | MEDIUM |
| Achats | « Purchases through a connector's in-chat views aren't supported. » Un lien externe ouvert depuis une app custom affiche toujours une fenêtre de confirmation. | Le paiement se fait sur une page web externe. | MEDIUM |
| Préremplissage | `claude://claude.ai/new?q=` documenté pour Desktop (préremplit, n'envoie pas, tronqué vers 14 000 caractères). `https://claude.ai/new?q=` **non documenté** ; un signalement d'octobre 2025 indique que l'envoi automatique ne marche plus. | Le bouton « Ouvrir dans Claude » ne doit pas dépendre du `?q=` web. Prévoir un repli « phrase à copier ». | MEDIUM (Desktop) / LOW (web) |
| Politique d'annuaire | Descriptions d'outil rejetées si elles « promote products and services » ou dictent un comportement à Claude. Logiciel « primarily an advertising or promotional vehicle » refusé. Interdit de collecter des données de conversation superflues, d'interroger la mémoire ou l'historique de Claude. Outils lecture et écriture séparés, annotations `readOnlyHint` / `destructiveHint` obligatoires. Aucune règle ne vise explicitement un message de quota dans un **résultat** d'outil. | Le message « plus de crédits » doit être un état d'erreur factuel, jamais une réclame. Aucune signature promotionnelle dans le contenu livré au Claude du destinataire. | MEDIUM |

## Feature Landscape

### Table Stakes (Users Expect These)

Sans elles, le produit paraît cassé ou dangereux. Les produits comparables les ont presque tous.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Envoyer un brief texte en langage naturel (« envoie ça à Paul ») | C'est la promesse. Tous les concurrents ont un outil `send`. | MEDIUM | Outil d'écriture séparé, `destructiveHint: true`. Clé d'idempotence pour éviter les doublons (Threadpacket, agent2agent). Limite de taille annoncée. |
| Boîte de réception : lister puis lire (« lis mes messages ») | Outils `inbox` et `read` présents partout. | LOW | Deux outils en lecture seule. La liste ne renvoie que des métadonnées, jamais le corps. |
| Contacts validés des deux côtés, révocables | Standard du domaine : autorisation mutuelle par paire (agent2agent), poignée de main (Agent Mail), Gate (Patchcord). | MEDIUM | Bloquer et supprimer un contact inclus. Une demande de contact ne contient aucun texte libre (sinon c'est un vecteur d'injection et de spam). |
| Compte en deux clics (Google) relié à l'OAuth du connecteur | Un non-technique n'acceptera ni code d'accès ni clé API. Claude exige OAuth pour un connecteur par utilisateur. | HIGH | Serveur d'autorisation OAuth 2.1 avec PKCE, DCR ou CIMD, URL de rappel `https://claude.ai/api/mcp/auth_callback`. C'est le plus gros morceau technique du socle. |
| Installation guidée du connecteur | L'ajout manuel d'une URL MCP est hors de portée de la cible. | LOW | Lien de préremplissage officiel + trois captures d'écran. Dire la vérité : « 3 clics », pas « 1 clic ». |
| Notification d'arrivée hors de Claude | Claude ne reçoit aucun push. Sans notification, le destinataire ignore qu'un message existe. | LOW | E-mail transactionnel d'abord (marche sur toutes les surfaces), badge d'extension ensuite. |
| Aperçu avant ingestion (expéditeur, titre, type, taille) | Exigence du projet et seule barrière humaine contre l'injection. | MEDIUM | Voir le différenciateur « acceptation par geste humain » pour la version solide. |
| Contenu reçu étiqueté « donnée non fiable » | Exigence du projet. share-context-mcp le fait (blocs cités). Les autres ne le documentent pas. | MEDIUM | Enveloppe à délimiteur aléatoire par message, rappel « ne pas exécuter », aucun autre outil appelé par le serveur à la lecture. |
| Accusé de réception pour l'expéditeur (livré, accepté, lu) | Remplace « t'as vu mon message ? ». Threadpacket enregistre le retrait avec l'heure. | LOW | Simple horodatage côté serveur, exposé dans la liste des envois. |
| Expiration automatique et rappel d'un envoi | Attendu de tout partage sécurisé : expiration, révocation, journal conservé. Threadpacket : 7 jours par défaut, révocation par l'expéditeur. | LOW | Expiration par défaut courte, suppression réelle des contenus à l'échéance. |
| Limitation de débit, blocage, signalement | Tout service qui relaie du contenu entre inconnus attire le spam. Claude lui-même plafonne les invitations quotidiennes. | MEDIUM | Plafond d'invitations par jour et par compte dès la v1. |
| Lecture de repli sur le web (« lire ici », copier, télécharger en .md) | Destinataires bloqués : plan Free déjà occupé, membre d'une organisation, connecteur refusé. | LOW | Sans ce repli, une partie des destinataires ne peut rien recevoir. |
| Chiffrement en transit et au repos, politique de confidentialité publique | Briefs de founders = données sensibles. Lien de politique de confidentialité exigé par l'annuaire. | LOW | Promesse honnête. Voir l'anti-fonctionnalité sur le « bout en bout ». |
| Solde et quota visibles | Pratique constante des modèles à crédits : compteur visible, date de remise à zéro, alerte avant zéro. | LOW | Renvoyé dans le résultat de chaque envoi (« Envoyé. Il te reste 3 envois ce mois-ci »). C'est une information, pas une promotion. |

### Differentiators (Competitive Advantage)

Là où le produit gagne. Alignés sur la valeur centrale : arriver intact, vite, sans manipulation technique, sans devenir un vecteur d'attaque.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Carnet de contacts inter-organisations | Le natif s'arrête à la frontière de l'organisation (Projets) ou reste en lecture seule (chats). Threadpacket est borné à une « team ». Ici : founder vers freelance, vers agence, vers associé, sans plan commun. | MEDIUM | Résolution par prénom dans la phrase (« Paul »), désambiguïsation si doublon. |
| Lien magique où le premier message **est** la demande de contact | Modèle Calendly : seul l'expéditeur a un compte, le destinataire vit le produit au clic, l'inscription vient après la valeur. Aucun concurrent MCP ne l'a. | HIGH | Le contenu reste retenu tant que le destinataire n'a pas accepté : l'acceptation vaut validation mutuelle. Lien lié à l'adresse invitée, à usage unique, expirant (Claude fait de même : invitation liée à l'e-mail, 30 jours). |
| Acceptation par geste humain hors de portée du modèle | Vraie barrière anti-injection : l'outil de lecture refuse tout message non accepté, et l'acceptation ne peut venir que d'un clic (carte MCP App, extension, page web), jamais d'un appel d'outil décidé par Claude. Aucun concurrent ne le documente. | HIGH | Dépend de la capacité d'une MCP App à modifier l'état serveur sur clic. À confirmer en phase (voir questions ouvertes). |
| Confirmation de sortie (« voici exactement ce qui part, à qui ») | Le risque symétrique : un Claude injecté par un message reçu qui renvoie les données de son propriétaire à l'expéditeur, contact validé. La validation mutuelle limite les destinations, elle ne ferme pas cette boucle. | MEDIUM | (analyse) Carte de confirmation à l'envoi avec destinataire et résumé. Signal serveur : envoi vers X juste après lecture d'un message de X. |
| Brief structuré de passation | Un brief à champs fixes (objectif, contexte, décisions, contraintes, questions ouvertes, pièces) est plus utile qu'une transcription et se relit vite dans l'aperçu. | LOW | Schéma d'arguments de l'outil d'envoi + prompt MCP. Claude accepte les prompts MCP. |
| Fidélité vérifiable du contenu | Promesse « intact ». Un texte long passé en argument d'outil est réécrit par le modèle, donc potentiellement abrégé ou paraphrasé. (analyse) | MEDIUM | Pour le texte long et les fichiers : octets d'origine + empreinte SHA-256, comme Threadpacket. Passe par l'upload, pas par l'argument d'outil. |
| Cartes dans la conversation, mobile compris | Aperçu, acceptation, choix du contact sans extension, sur iOS et Android. Les concurrents sont en texte brut. | MEDIUM | Carte inline : 2 actions, 4 à 5 données. Parfait pour « De Alice · Brief · 3 fichiers · Accepter / Refuser ». |
| Extension Chrome : dépôt de fichiers et de dossiers depuis claude.ai | Comble le trou du connecteur custom. Glisser-déposer, bouton « Envoyer à… », badge de non-lus. | HIGH | Injection d'UI seulement. Aucun appel aux endpoints internes de claude.ai. |
| Lot « dossier » lisible à la demande | Manifeste + lecture fichier par fichier : le Claude du destinataire ne charge que l'utile, dans la limite de 150 000 caractères. Bouton « télécharger le lot » pour l'ajouter soi-même à un Projet. | HIGH | Seule façon honnête d'approcher « envoie le dossier dans son projet ». |
| Invitations gratuites et récompense des deux côtés | Chaque envoi à un non-inscrit est une acquisition : ne jamais le facturer. Récompense en crédits pour les deux à l'activation du destinataire (modèle Dropbox : récompense en produit, pas en argent). | MEDIUM | Récompense déclenchée par un événement réel (premier message lu via le connecteur), un seul parrain par destinataire, auto-parrainage rejeté. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Ingestion automatique à la réception | « Zéro friction ». | Injection de prompt directe dans un Claude qui a peut-être Gmail, Drive ou Stripe. Détruit l'argument de vente. | Aperçu puis acceptation par geste humain. |
| Conversation autonome Claude à Claude | C'est l'image du tweet d'origine. share-context-mcp le fait (agents qui échangent sans validation). | Injection en chaîne, boucle d'exfiltration, crédits brûlés sans humain. Déjà hors périmètre. | Fil de réponses asynchrone, un humain déclenche chaque envoi. |
| Écriture directe dans un Projet du destinataire | « Dans le bon projet de ton gars ». | Impossible par MCP. Exigerait l'API interne de claude.ai : fragile, risque CGU. | Lot téléchargeable + connecteur actif dans une conversation du Projet. Le dire clairement à l'utilisateur. |
| Mention « chiffré de bout en bout » dans le marketing | Argument de confiance fort. | Un connecteur distant doit rendre du clair à Claude : le serveur déchiffre à la lecture. agent2agent le reconnaît (« tamper-evidence, not end-to-end »). Une promesse fausse sur des données de founders est un risque de réputation et juridique. | « Chiffré en transit et au repos, supprimé à l'expiration ». Mode bout en bout réel plus tard, déchiffrement dans l'extension, annoncé comme mode distinct. |
| Message promotionnel dans les réponses d'outil (réclame, « envoyé avec X » dans le contenu livré) | Viralité « à la Loom ». | Texte promotionnel injecté dans le Claude d'un tiers : ressemble à de l'injection, contraire à l'esprit de la politique d'annuaire (« promote products and services », « advertising vehicle »). | Marque sur la page du lien magique et dans l'e-mail. Dans l'outil : état factuel de quota + lien, uniquement quand un envoi demandé échoue. |
| Paiement dans la conversation | Moins de friction. | Non pris en charge dans les vues in-chat. Les connecteurs qui transfèrent de l'argent sont refusés. | Lien vers une page de paiement web. |
| Faire payer ou bloquer la **lecture** | Monétiser les deux côtés. | Contenu pris en otage, boucle virale cassée, destinataire furieux. | Seul l'envoi consomme des crédits. Recevoir est toujours gratuit. |
| Perdre le message quand les crédits sont à zéro | Simplicité d'implémentation. | Le pire moment pour perdre le travail de l'utilisateur. | Mise en attente comme brouillon, départ dès que le solde revient. |
| Urgence artificielle (compte à rebours, remises empilées) | Conversion à court terme. | Cité comme contre-exemple dans les analyses de modèles à crédits. Incompatible avec un produit vendu sur la confiance. | Invite contextuelle, sobre, au moment du besoin. |
| Synchronisation de mémoire ou d'espace partagé permanent | TeamBrain, Contexter, BotSesh le font. | Autre produit (modèle « tirer » d'un espace commun, pas « pousser » à une personne). L'annuaire interdit d'interroger la mémoire de Claude et de bâtir des profils à partir de conversations. | Rester sur la livraison ponctuelle de personne à personne. |
| Annuaire public ou recherche d'utilisateurs | Trouver des contacts facilement. | Spam, énumération de comptes, demandes non sollicitées. agent2agent rend volontairement ses erreurs résistantes à l'énumération. | Ajout par lien d'invitation ou adresse e-mail exacte uniquement. |
| E-mails d'invitation envoyés par le service à des inconnus dès la v1 | Automatiser la boucle virale. | Délivrabilité, abus, responsabilité de spam, infrastructure à construire. | v1 : le service génère le lien, l'expéditeur l'envoie lui-même par WhatsApp ou e-mail. Le lien venant d'une personne connue inspire plus confiance. E-mail envoyé par le service en v1.x. |
| Multi-IA dès la v1 (ChatGPT, Gemini) | Patchcord et Threadpacket le font. | Double la surface de test, dilue le positionnement. Déjà hors périmètre. | Garder le protocole neutre côté serveur, ouvrir plus tard. |
| Autorisation « toujours autoriser » encouragée sur l'outil d'envoi | Fluidité. | Supprime la dernière confirmation avant sortie de données. | Laisser la confirmation par défaut de Claude sur l'outil d'écriture. |

## Feature Dependencies

```
[Compte + serveur OAuth]
    └──requires──> rien (socle)

[Connecteur MCP installé]
    └──requires──> [Compte + serveur OAuth]
    └──requires──> [Installation guidée (lien de préremplissage)]

[Contacts validés]
    └──requires──> [Compte + serveur OAuth]

[Envoi de brief texte]
    └──requires──> [Connecteur MCP installé]
    └──requires──> [Contacts validés]

[Boîte de réception : lister / lire]
    └──requires──> [Connecteur MCP installé]
    └──requires──> [Enveloppe « donnée non fiable »]
    └──requires──> [Aperçu + acceptation]

[Aperçu + acceptation par geste humain]
    └──requires──> [Page web de message]  (repli universel)
    └──enhanced by──> [Carte MCP App]
    └──enhanced by──> [Extension Chrome]

[Lien magique pour non-inscrits]
    └──requires──> [Page web de message]
    └──requires──> [Contacts validés]  (l'acceptation crée le contact)
    └──requires──> [Installation guidée]
    └──requires──> [Limitation de débit des invitations]

[Fichiers (PDF, images, code)]
    └──requires──> [Stockage objet + upload web]
    └──requires──> [Extension Chrome] OU [Page web d'upload]
                       └──later──> [Listing annuaire + hôtes approuvés] ──unlocks──> fichiers via la conversation

[Lot « dossier »]
    └──requires──> [Fichiers]
    └──requires──> [Lecture paginée fichier par fichier]

[Crédits : compteur]
    └──requires──> [Envoi de brief texte]
[Crédits : blocage + achat]
    └──requires──> [Crédits : compteur]
    └──requires──> [Page de paiement web]
[Récompense de parrainage]
    └──requires──> [Lien magique] + [Crédits : compteur]

[Notification e-mail] ──enhances──> [Boîte de réception]
[Accusé de lecture]   ──enhances──> [Envoi de brief texte]
[Brief structuré]     ──enhances──> [Aperçu + acceptation]

[Ingestion automatique]        ──conflicts──> [Aperçu + acceptation]
[Bout en bout réel]            ──conflicts──> [Lecture par connecteur distant sur mobile]
[Promotion dans les résultats] ──conflicts──> [Listing annuaire]
```

### Dependency Notes

- **Tout dépend du serveur OAuth.** Claude n'offre de connexion par utilisateur qu'en OAuth. Pas de contacts, pas d'envoi, pas de lien magique sans lui. C'est la première phase, et la plus risquée techniquement.
- **L'aperçu exige une page web avant toute carte ou extension.** La page de message sert à la fois d'aperçu universel, de repli de lecture et de cible du lien magique. La carte MCP App et l'extension sont des améliorations du même écran.
- **Le lien magique dépend des contacts, pas l'inverse.** Le premier envoi à un non-inscrit crée une invitation avec message retenu. L'acceptation crée le contact mutuel. Cela résout la tension entre « contacts validés des deux côtés » et « envoi à un non-inscrit ».
- **Les fichiers dépendent d'un canal hors connecteur.** Tant que le connecteur est custom, le binaire entre par le navigateur. C'est la vraie raison d'être de l'extension en v1.x, plus que le confort.
- **Le listing dans l'annuaire débloque trois choses d'un coup** : fichiers par la conversation, découverte dans Claude, liens externes sans fenêtre de confirmation. Il impose en retour des descriptions d'outil neutres et un message de quota sobre. Concevoir les outils conformes dès le premier jour évite une réécriture.
- **Le bout en bout réel entre en conflit avec le mobile.** Le déchiffrement côté client suppose l'extension, donc Chrome sur ordinateur. Le proposer en mode optionnel, jamais par défaut.
- **Le blocage par crédits dépend du compteur, et le compteur ne coûte presque rien.** Compter dès la v1, bloquer et vendre après validation de l'usage.

## Viral Loop and Credits Design

### Boucle virale par lien magique (recommandation)

1. L'expéditeur dit « envoie ce brief à paul@exemple.com ». Paul n'a pas de compte : l'outil crée une invitation et renvoie un lien.
2. **L'expéditeur transmet le lien lui-même** (WhatsApp, e-mail). Pas d'envoi par le service en v1.
3. Paul ouvre la page : nom de l'expéditeur, titre, type, taille. Pas de corps tant qu'il n'est pas identifié avec l'adresse invitée.
4. Bouton principal « Recevoir dans mon Claude » : compte Google, puis lien de préremplissage du connecteur, puis connexion OAuth.
5. Retour sur la page : « Ouvre Claude et dis : lis mon message de Alice », avec bouton de copie. Lien `claude://` proposé si Desktop est installé.
6. Lien secondaire « Lire ici sans rien installer ». La valeur d'abord, l'installation ensuite.
7. Après lecture : « Réponds à Alice depuis ton Claude ». Paul devient expéditeur, la boucle repart.

Règles : lien lié à l'adresse invitée, usage unique, expiration, plafond quotidien d'invitations, la boucle fait partie de l'action centrale (envoyer) et non d'un écran « inviter des amis » ajouté après coup. (LOW sur les chiffres de conversion : sources de blogs de croissance uniquement.)

### Freemium à crédits (recommandation)

| Décision | Recommandation | Raison |
|----------|----------------|--------|
| Unité | 1 crédit = 1 envoi. Plafonds de taille par palier plutôt que tarif au poids. | Compréhensible sans explication par un non-technique. |
| Ce qui est gratuit | Recevoir, lire, répondre une fois à un message reçu, inviter un non-inscrit. | Ne jamais faire payer le destinataire ni l'acquisition. |
| Dotation gratuite | Mensuelle avec date de remise à zéro affichée. Volume à calibrer sur l'usage réel de la v1. | Aucune donnée trouvée pour trancher entre mensuel et dotation unique. Le mensuel entretient l'habitude hebdomadaire visée par la métrique v1. |
| Transparence | Solde dans chaque résultat d'envoi, alerte à l'approche de zéro. | Pratique constante des modèles à crédits. |
| À zéro | Message factuel dans la conversation : envoi mis en attente, solde, date de remise à zéro, lien de recharge. Aucun adjectif, aucune remise, aucun compte à rebours. | Compatible avec la politique d'annuaire, cohérent avec un produit de confiance. |
| Achat | Page web externe. Recharge ponctuelle et abonnement. | Achats in-chat non pris en charge. |
| Parrainage | Crédits pour les deux quand l'invité lit son premier message via le connecteur. | Récompense en produit, déclenchée par une activation réelle. |

## MVP Definition

### Launch With (v1)

Objectif : le créateur et 2 à 3 contacts l'utilisent chaque semaine à la place des .md copiés-collés.

- [ ] Compte Google + serveur OAuth du connecteur — socle obligatoire
- [ ] Connecteur MCP distant avec 5 outils : envoyer, lister la boîte, lire, lister les contacts, inviter — le cœur de la promesse
- [ ] Page d'installation avec lien de préremplissage — seule voie réaliste pour un non-technique
- [ ] Contacts mutuels par lien d'invitation, blocage et suppression — exigence de sécurité
- [ ] Envoi et réception de briefs **texte** au format structuré — couvre le cas d'usage qui remplace le .md copié-collé
- [ ] Page web de message : aperçu, accepter ou refuser, lire, copier — barrière humaine et repli universel
- [ ] Enveloppe « donnée non fiable » à délimiteur aléatoire, lecture refusée tant que non accepté — exigence de sécurité
- [ ] Notification e-mail à l'arrivée — sans elle personne ne sait qu'il a un message
- [ ] Accusé livré / accepté / lu, expiration par défaut, rappel d'un envoi — coût faible, ferme la boucle
- [ ] Compteur d'envois visible, sans blocage ni paiement — prépare les crédits et mesure l'usage
- [ ] Plafond de débit sur les envois et les invitations — hygiène minimale

### Add After Validation (v1.x)

- [ ] Lien magique complet pour non-inscrits (message retenu, acceptation = contact) — dès que le cercle initial veut inviter au-delà
- [ ] Carte MCP App d'aperçu et d'acceptation dans la conversation — quand l'aller-retour vers la page web devient la friction principale
- [ ] Extension Chrome : bouton « Envoyer à… », dépôt de fichiers, badge de non-lus — dès que le besoin de fichiers est confirmé
- [ ] Fichiers (.md, PDF, images, code) avec octets d'origine et empreinte — même déclencheur
- [ ] Confirmation de sortie à l'envoi — avant toute ouverture au public
- [ ] Blocage à zéro crédit, page de paiement, parrainage — quand l'usage hebdomadaire est établi
- [ ] Fil de réponses — quand les échanges dépassent l'aller simple
- [ ] E-mail d'invitation envoyé par le service — quand la délivrabilité vaut l'investissement

### Future Consideration (v2+)

- [ ] Lot « dossier » avec manifeste et lecture à la demande — complexité élevée, usage à prouver
- [ ] Listing dans l'annuaire Anthropic (fichiers par la conversation, découverte) — demande revue et hôtes approuvés
- [ ] Transcription fidèle d'une conversation par l'extension — lecture du DOM, fragile aux mises à jour de claude.ai
- [ ] Mode bout en bout réel avec déchiffrement dans l'extension — ordinateur seulement
- [ ] Espaces d'équipe, administration — autre segment
- [ ] Autres IA — hors périmètre v1

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Compte Google + serveur OAuth | HIGH | HIGH | P1 |
| Outils envoyer / lister / lire | HIGH | MEDIUM | P1 |
| Installation guidée par lien de préremplissage | HIGH | LOW | P1 |
| Contacts mutuels, blocage | HIGH | MEDIUM | P1 |
| Page web de message (aperçu, acceptation, lecture) | HIGH | MEDIUM | P1 |
| Enveloppe « donnée non fiable » | HIGH | MEDIUM | P1 |
| Brief structuré | MEDIUM | LOW | P1 |
| Notification e-mail | HIGH | LOW | P1 |
| Accusé de lecture, expiration, rappel | MEDIUM | LOW | P1 |
| Compteur d'envois visible | LOW | LOW | P1 |
| Plafond de débit | MEDIUM | LOW | P1 |
| Lien magique pour non-inscrits | HIGH | HIGH | P2 |
| Carte MCP App d'acceptation | HIGH | MEDIUM | P2 |
| Extension Chrome (envoi, dépôt, badge) | HIGH | HIGH | P2 |
| Fichiers avec empreinte | HIGH | MEDIUM | P2 |
| Confirmation de sortie | MEDIUM | MEDIUM | P2 |
| Blocage, paiement, parrainage | MEDIUM | MEDIUM | P2 |
| Fil de réponses | MEDIUM | LOW | P2 |
| Lot « dossier » | MEDIUM | HIGH | P3 |
| Listing annuaire | MEDIUM | HIGH | P3 |
| Transcription de conversation par l'extension | MEDIUM | HIGH | P3 |
| Bout en bout réel | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | Claude natif | Threadpacket | Patchcord / agent2agent | TeamBrain / Contexter | Our Approach |
|---------|--------------|--------------|-------------------------|-----------------------|--------------|
| Modèle | Partage de chat par e-mail (tous plans) ; Projets partagés (Team/Enterprise) | Paquet brief + fichiers, de personne à personne | Bus de messages entre agents | Espace commun lu par les IA de l'équipe | Livraison ponctuelle de personne à personne |
| Hors organisation | Chats : oui en lecture seule. Projets : non | Limité à une « team » de 10 | Patchcord : `@username` via Gate. agent2agent : paire autorisée | Par invitation à l'espace | Oui, par carnet de contacts |
| Le contenu devient utilisable dans le Claude du destinataire | Non : ni continuer, ni copier, ni télécharger | Oui | Oui | Oui (lecture de l'espace) | Oui |
| Fichiers | Non inclus dans le chat partagé | 25 MiB × 10, par URL d'upload ou page web | Patchcord : pièces jointes par URL présignée | Documents de l'espace | Par extension ou page web, puis par l'annuaire |
| Contact et consentement | Invitation liée à l'e-mail | Nom d'utilisateur + code, lien d'invitation 7 jours | Autorisation mutuelle révocable ; Gate avec file d'attente | Membres de l'espace | Mutuel, révocable, sans texte libre dans la demande |
| Aperçu avant ingestion | Sans objet (lecture humaine) | Non documenté | Non documenté | Non | Oui, acceptation par geste humain |
| Traitement de l'injection | Sans objet | Non documenté | Non documenté (share-context-mcp : blocs cités) | Non documenté | Enveloppe non fiable, lecture refusée avant acceptation, confirmation de sortie |
| Non-inscrits | Compte gratuit exigé avec l'e-mail invité | Lien d'invitation d'équipe | Non | Non | Lien magique, message retenu, lecture web de repli |
| Accusé de lecture | Non | Oui (retrait horodaté) | Non documenté | Non | Oui |
| Expiration et rappel | Arrêt du partage ; invitation 30 jours | 7 jours par défaut, révocation | Patchcord : `recall`, `unsend` | Non | Oui |
| Cible | Tous | Développeurs, multi-agents | Développeurs | Petites équipes | Founders non techniques |
| Prix | Inclus | Gratuit hébergé | Non indiqué | Offre à vie dès 49 USD | Freemium à crédits |
| Traction visible | — | 4 étoiles GitHub | 4 étoiles (Patchcord) | Inconnue | — |

**Lecture.** Aucun analogue direct ne montre de traction (2 à 4 étoiles). MCP Agent Mail en a 2 200, mais pour coordonner les agents de code d'une même personne, un autre besoin. La demande côté développeurs n'est donc pas prouvée, et le segment non technique n'a pas été testé par ces produits. C'est à la fois l'ouverture et le risque : la métrique v1 du projet (usage hebdomadaire réel par 3 ou 4 personnes) est le bon test.

**Risque natif.** ChatGPT propose des projets partagés sur tous les plans, Free compris, depuis fin 2025. Anthropic a déjà l'invitation par e-mail sur les chats. Un bouton « continuer cette conversation dans mon compte » couvrirait une bonne part du cas « brief ». Probabilité jugée élevée à 6-12 mois (analyse, LOW). Ce que le natif couvrirait mal : contacts durables entre organisations, fichiers d'origine, accusés, et plus tard d'autres IA.

## Open Questions

À traiter en recherche de phase, par ordre d'impact.

1. **Une MCP App peut-elle changer l'état serveur sur un clic humain, sans passer par un appel d'outil décidé par le modèle ?** Toute la solidité de « l'acceptation par geste humain » dans la conversation en dépend. Non vérifié. Si non : l'acceptation reste sur la page web et dans l'extension.
2. **La limite « un connecteur custom » du plan Free s'applique-t-elle aux connecteurs de l'annuaire ?** Non précisé dans les pages lues. Change la taille du marché adressable côté destinataires gratuits.
3. **Comportement actuel de `https://claude.ai/new?q=` sur le web.** Non documenté, dernier signalement d'octobre 2025. Test manuel de cinq minutes à faire avant de concevoir « Ouvrir dans Claude ».
4. **Position de la revue d'annuaire sur un connecteur qui relaie du contenu de tiers** et sur un message de quota dans un résultat d'outil. Aucune règle explicite trouvée. À demander à Anthropic avant de soumettre.
5. **Synchronisation des connecteurs vers le mobile et vers Claude Code.** La documentation dit que la même infrastructure sert toutes les surfaces. La présence automatique du connecteur ajouté sur le web dans l'app mobile n'est pas écrite noir sur blanc dans les pages lues.
6. **Fidélité d'un texte long passé en argument d'outil.** Mesurer sur des briefs réels de 5 000 à 20 000 caractères avant de promettre « intact ».
7. **Dotation gratuite et prix.** Aucune donnée de conversion fiable trouvée. À calibrer sur l'usage de la v1.

## Sources

Pages officielles Anthropic consultées le 2026-11-27 (MEDIUM) :

- [Manage project visibility and sharing](https://support.claude.com/en/articles/9519189-manage-project-visibility-and-sharing)
- [Share a chat with specific people](https://support.claude.com/en/articles/16762496-share-a-chat-with-specific-people)
- [Share and unshare chats](https://support.claude.com/en/articles/10593882-share-and-unshare-chats)
- [Get started with custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)
- [Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/remote-mcp)
- [Directory connectors vs custom connectors](https://claude.com/docs/connectors/building/directory-vs-custom) (lien d'installation, fichiers)
- [Build an MCP server for Claude](https://claude.com/docs/connectors/building/index) (limites de taille et de durée, primitives prises en charge)
- [Support file uploads and downloads](https://claude.com/docs/connectors/building/files)
- [Lazy authentication for MCP servers](https://claude.com/docs/connectors/building/lazy-authentication)
- [Connector pre-submission checklist](https://claude.com/docs/connectors/building/review-criteria)
- [Anthropic Software Directory Policy](https://support.claude.com/en/articles/13145358-anthropic-software-directory-policy) (page datée du 27/10/2026)
- [Connector verification](https://claude.com/docs/connectors/verification)
- [Get started with MCP Apps](https://claude.com/docs/connectors/building/mcp-apps/getting-started)
- [MCP Apps design guidelines](https://claude.com/docs/connectors/building/mcp-apps/design-guidelines)
- [Send messages to Claude from your MCP App](https://claude.com/docs/connectors/building/mcp-apps/send-messages)
- [Open external links from MCP Apps](https://claude.com/docs/connectors/building/mcp-apps/external-links)
- [Use interactive connectors in Claude](https://support.claude.com/en/articles/13454812-use-interactive-connectors-in-claude)
- [Open Claude Desktop with a link](https://support.claude.com/en/articles/14729294-open-claude-desktop-with-a-link)
- [Open the Claude mobile app with a link](https://support.claude.com/en/articles/14898120-open-the-claude-mobile-app-with-a-link)
- [Release notes](https://support.claude.com/en/articles/12138966-release-notes)

Produits comparables (pages éditeur, MEDIUM pour l'existence et la liste d'outils, LOW pour le reste) :

- [Threadpacket (evgolsh/handoff)](https://github.com/evgolsh/handoff)
- [Patchcord](https://github.com/ppravdin/patchcord) et sa [skill inbox](https://unpkg.com/patchcord@0.5.92/skills/inbox/SKILL.md)
- [agent2agent](https://glama.ai/mcp/servers/tw00/agent2agent)
- [MCP Agent Mail](https://github.com/Dicklesworthstone/mcp_agent_mail)
- [share-context-mcp](https://github.com/holny/share-context-mcp)
- [TeamBrain : partager un Projet Claude sans plan Team](https://teambrain.co/blog/share-claude-project-without-teams-plan/) (contenu promotionnel)
- Contexter et BotSesh : fiches d'annuaire inaccessibles (403), connus par le seul résumé de recherche (LOW)

Signaux de marché et pratiques (LOW, blogs et éditeurs de facturation) :

- [Signalement : `claude.ai/new?q=` ne soumet plus](https://github.com/anthropics/claude-code/issues/8827) (ouvert le 03/10/2025, fermé sans suite)
- [OpenAI : group chats in ChatGPT](https://openai.com/index/group-chats-in-chatgpt/) et [more ways to work with your team](https://openai.com/index/more-ways-to-work-with-your-team)
- [OpenView : SaaS products with viral loops](https://openviewpartners.com/blog/saas-product-viral-loop/)
- [Andrew Chen : how to design a referral program](https://andrewchen.com/how-to-design-a-referral-program/)
- [Dropbox : règles d'éligibilité du parrainage](https://help.dropbox.com/storage-space/25gb-promotion)
- [impact.com : recipient incentives](https://impact.com/referral/better-referral-rewards-recipient-incentives/)
- [Schematic : AI credits for startups](https://schematichq.com/blog/ai-credits-for-startups)
- [Chargebee : credit-based pricing](https://www.chargebee.com/blog/how-to-make-credit-based-pricing-work-design-sell-and-scale-a-credit-model/)

---
*Feature research for: partage de contexte de Claude à Claude (connecteur MCP distant + extension Chrome)*
*Researched: 2026-11-27*
