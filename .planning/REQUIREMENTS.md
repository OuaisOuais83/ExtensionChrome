# Requirements: Relay (nom provisoire)

**Defined:** 2026-11-27
**Core Value:** Un brief envoyé par le Claude de A arrive, intact et lisible, dans le Claude de B en moins d'une minute et sans aucune manipulation technique — sans jamais devenir un vecteur d'attaque contre le Claude de B.

## v1 Requirements

Jalon 1 : cœur de valeur sur toutes les surfaces (claude.ai web, mobile, Desktop, Claude Code), sans extension. Critère de succès : le porteur et 2-3 contacts l'utilisent chaque semaine à la place des .md copiés-collés.

### Plateforme et décisions irréversibles

- [ ] **PLAT-01**: Un connecteur de test (outil `ping` + OAuth complet) fonctionne sur claude.ai web (compte Free neuf et compte payant), app mobile, Desktop et Claude Code, et les limites réelles sont mesurées et consignées (taille/fidélité d'un argument d'outil, étapes d'installation, `claude.ai/new?q=`, synchronisation mobile, rendu des images Markdown)
- [ ] **PLAT-02**: Le nom commercial (sans « Claude » ni « Anthropic ») et le domaine de production définitif sont fixés avant le premier utilisateur externe
- [ ] **PLAT-03**: Une formulation de confidentialité unique est rédigée et publiée sur une page sécurité, sans le terme « bout en bout »
- [ ] **PLAT-04**: Les données (base, fichiers, journaux) sont hébergées en région Union européenne

### Compte et connecteur

- [ ] **AUTH-01**: L'utilisateur peut créer un compte sans mot de passe (Google ou code / lien par e-mail)
- [ ] **AUTH-02**: L'utilisateur peut ajouter le connecteur à son Claude via une page d'installation guidée (lien de préremplissage officiel, captures d'écran, nombre d'étapes annoncé honnêtement)
- [ ] **AUTH-03**: L'utilisateur autorise le connecteur via un écran de consentement qui nomme son compte en clair
- [ ] **AUTH-04**: La connexion du connecteur persiste au moins 24 heures et à travers les redémarrages sans redemander d'autorisation
- [ ] **AUTH-05**: L'utilisateur peut voir la liste des Claude connectés à son compte et en révoquer un

### Contacts

- [ ] **CONT-01**: L'utilisateur peut inviter un utilisateur existant comme contact via un lien ; le contact n'existe qu'après acceptation des deux côtés
- [ ] **CONT-02**: L'utilisateur peut supprimer ou bloquer un contact
- [ ] **CONT-03**: Quand le destinataire nommé est ambigu, la résolution se fait côté serveur et Claude fait choisir parmi les candidats ; aucun outil ne liste tout le carnet

### Envoi

- [ ] **SEND-01**: L'utilisateur peut dire à son Claude « envoie ce brief à <contact> » depuis n'importe quelle surface et le texte / Markdown est envoyé
- [ ] **SEND-02**: Le brief envoyé suit une structure de passation (objectif, contexte, décisions, contraintes, questions ouvertes)
- [ ] **SEND-03**: L'expéditeur reçoit un reçu (identifiant, taille, empreinte SHA-256) ; un renvoi du même message ne crée pas de doublon (idempotence)
- [ ] **SEND-04**: L'expéditeur voit le statut livré / accepté / lu et peut rappeler un message pas encore lu
- [ ] **SEND-05**: Les envois sont soumis à des plafonds de débit et de diffusion par compte

### Réception

- [ ] **RECV-01**: Le destinataire reçoit une notification e-mail à l'arrivée d'un message
- [ ] **RECV-02**: Le destinataire voit l'aperçu du message sur une page web hors de Claude et peut l'accepter ou le refuser
- [ ] **RECV-03**: Aucun outil du connecteur ne peut accepter un message ; avant acceptation, Claude ne voit que des métadonnées produites par le serveur
- [ ] **RECV-04**: Après acceptation, le destinataire peut dire « lis mes messages » et son Claude liste puis lit le contenu
- [ ] **RECV-05**: Le contenu lu par Claude est enveloppé comme donnée non fiable (délimiteur aléatoire) et assaini (caractères invisibles, images et liens Markdown neutralisés)
- [ ] **RECV-06**: Le destinataire peut lire et copier le message sur le web sans connecteur (repli de lecture)
- [ ] **RECV-07**: Les messages expirent par défaut après un délai fixé

### Sécurité

- [ ] **SEC-01**: Le contenu des messages est chiffré au repos (frontière Vault unique, clé par message, enveloppes de clés multiples) et n'apparaît jamais dans les journaux
- [ ] **SEC-02**: Après lecture d'un contenu tiers, tout envoi sortant exige une confirmation humaine hors du modèle, imposée par le serveur

### Invitation par lien magique

- [ ] **INV-01**: Envoyer à une adresse sans compte crée une invitation avec message retenu ; l'expéditeur transmet lui-même le lien (WhatsApp, e-mail)
- [ ] **INV-02**: L'invité ne voit que des métadonnées vérifiées tant qu'il n'a pas prouvé le contrôle de l'adresse invitée
- [ ] **INV-03**: L'acceptation par l'invité crée son compte et le contact mutuel, puis le guide jusqu'à l'installation du connecteur (avec lecture web si l'installation est impossible)
- [ ] **INV-04**: Les invitations sont soumises à un plafond quotidien par compte

### Crédits

- [ ] **CRED-01**: Chaque envoi est inscrit dans un registre de crédits en ajout seul (sans blocage en v1) et l'utilisateur voit son compteur

## v2 Requirements

Jalon 2. Reconnu mais hors de la roadmap actuelle.

### Extension Chrome

- **EXT-01**: L'utilisateur voit ses contacts et sa boîte de réception dans un side panel sur claude.ai
- **EXT-02**: L'extension affiche un badge à l'arrivée d'un message
- **EXT-03**: Bouton « Ouvrir dans Claude » (préremplissage tenté + copie presse-papiers)
- **EXT-04**: Bouton « Envoyer à… » et glisser-déposer de fichiers
- **EXT-05**: Capture de la conversation affichée sur clic explicite avec aperçu

### Fichiers

- **FILE-01**: Envoi de fichiers et dossiers via un lien de dépôt web (fonctionne sur mobile)
- **FILE-02**: Ingestion (Markdown, code, images, texte extrait des PDF), manifeste et lecture paginée par Claude
- **FILE-03**: Téléchargement de l'original par l'humain

### Monétisation

- **PAY-01**: Dotation mensuelle de crédits, alerte avant zéro, envoi mis en attente à zéro
- **PAY-02**: Erreur factuelle « limite atteinte » avec un lien vers la page de compte
- **PAY-03**: Achat de crédits via Stripe Checkout
- **PAY-04**: Parrainage récompensé à l'activation réelle

### Confidentialité renforcée

- **SEAL-01**: Inbox scellée (chiffrement par grant, déchiffrement lié au jeton actif, signature de l'expéditeur)

### Lancement public

- **PUB-01**: Publication Chrome Web Store
- **PUB-02**: Documents RGPD revus par un juriste, outils anti-abus
- **PUB-03**: Soumission à l'annuaire de connecteurs Anthropic, transferts de fichiers MCP

## Out of Scope

| Feature | Reason |
|---------|--------|
| Écriture directe dans un Projet claude.ai | Impossible via MCP ; exigerait l'API interne non documentée (CGU, fragilité) |
| Mention « chiffré de bout en bout » sur le canal connecteur | Le serveur déchiffre à la lecture ; promesse fausse (précédent Zoom / FTC) |
| Conversation autonome Claude ↔ Claude | Risque d'injection en chaîne, pas d'humain dans la boucle |
| Utilisation des endpoints internes de claude.ai | CGU Anthropic, cassable à chaque mise à jour |
| Lien de paiement ou promotion dans les résultats d'outil | Politique d'annuaire (clause 4.C) et habitue à cliquer sur des liens injectés |
| Ingestion automatique d'un message reçu | Contournerait la porte d'acceptation humaine |
| Outil `export_conversation` | Clause 1.G de la politique d'annuaire ; capture réservée à l'extension sur clic |
| Annuaire public d'utilisateurs | Vie privée et spam |
| Autres IA (ChatGPT, Gemini…) | v1 centrée sur Claude ; piste future |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| PLAT-01 | Phase 1 | Pending |
| PLAT-02 | Phase 1 | Pending |
| PLAT-03 | Phase 1 | Pending |
| PLAT-04 | Phase 1 | Pending |
| AUTH-01 | Phase 2 | Pending |
| AUTH-02 | Phase 2 | Pending |
| AUTH-03 | Phase 2 | Pending |
| AUTH-04 | Phase 2 | Pending |
| AUTH-05 | Phase 2 | Pending |
| CONT-01 | Phase 3 | Pending |
| CONT-02 | Phase 5 | Pending |
| CONT-03 | Phase 3 | Pending |
| SEND-01 | Phase 3 | Pending |
| SEND-02 | Phase 3 | Pending |
| SEND-03 | Phase 3 | Pending |
| SEND-04 | Phase 5 | Pending |
| SEND-05 | Phase 4 | Pending |
| RECV-01 | Phase 3 | Pending |
| RECV-02 | Phase 3 | Pending |
| RECV-03 | Phase 4 | Pending |
| RECV-04 | Phase 4 | Pending |
| RECV-05 | Phase 4 | Pending |
| RECV-06 | Phase 3 | Pending |
| RECV-07 | Phase 5 | Pending |
| SEC-01 | Phase 3 | Pending |
| SEC-02 | Phase 4 | Pending |
| INV-01 | Phase 6 | Pending |
| INV-02 | Phase 6 | Pending |
| INV-03 | Phase 6 | Pending |
| INV-04 | Phase 6 | Pending |
| CRED-01 | Phase 3 | Pending |

**Coverage:**
- v1 requirements: 31 total
- Mapped to phases: 31
- Unmapped: 0

---
*Requirements defined: 2026-11-27*
*Last updated: 2026-11-27 after roadmap creation (traceability filled)*
