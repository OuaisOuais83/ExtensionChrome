# Roadmap: Relay (nom provisoire)

## Overview

Le jalon 1 livre le cœur de valeur sur toutes les surfaces Claude (claude.ai web, app mobile, Desktop, Claude Code), sans extension Chrome. On commence par mesurer la plateforme sur de vrais comptes et figer les choix irréversibles (nom, domaine, région, promesse de confidentialité). On branche ensuite le compte et le connecteur. La boucle centrale arrive en trois tranches verticales : un brief part du Claude de A et se lit sur le web, puis il se lit dans le Claude de B avec toutes les défenses contre l'injection de prompt, puis chacun suit et maîtrise ses messages. Le lien magique ouvre le service au-delà du cercle de confiance : il vient en dernier et peut glisser sans bloquer le critère de succès (le porteur et 2-3 contacts l'utilisent chaque semaine à la place des .md copiés-collés). Le jalon 2 (extension, fichiers, paiement, inbox scellée, annuaire) correspond aux exigences v2 de REQUIREMENTS.md et n'est pas phasé ici.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Spike de plateforme et choix fondateurs** - Connecteur de test sur toutes les surfaces, limites réelles mesurées ; nom, domaine, région UE et promesse de confidentialité figés
- [ ] **Phase 2: Compte et connecteur** - Compte sans mot de passe, installation guidée, consentement nominatif, connexion durable et révocable
- [ ] **Phase 3: Envoi de brief entre contacts** - Contacts mutuels, brief structuré envoyé depuis Claude avec reçu, e-mail, acceptation et lecture sur le web, contenu chiffré
- [ ] **Phase 4: Lecture dans Claude sans danger** - « Lis mes messages » dans Claude, porte d'acceptation hors modèle, enveloppe non fiable, confirmation de sortie, plafonds d'envoi
- [ ] **Phase 5: Suivi, rappel et blocage** - Statuts livré / accepté / lu, rappel d'un message non lu, expiration, suppression et blocage de contact
- [ ] **Phase 6: Invitation par lien magique** - Envoi à un non-inscrit, preuve d'adresse, création du compte et du contact, lecture web de repli, plafond quotidien

## Phase Details

### Phase 1: Spike de plateforme et choix fondateurs
**Goal:** En tant que porteur du projet, je veux tester un connecteur sur chaque surface Claude et figer les choix irréversibles, afin de bâtir sur du vérifié.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: PLAT-01, PLAT-02, PLAT-03, PLAT-04
**Success Criteria** (what must be TRUE):
  1. Le porteur ajoute le connecteur de test sur claude.ai web (compte Free neuf et compte payant), sur l'app mobile, sur Desktop et dans Claude Code, se connecte via OAuth et obtient une réponse de l'outil `ping` sur chacune de ces surfaces.
  2. Un relevé consigne, chiffres à l'appui, la taille et la fidélité d'un argument d'outil (5 000, 20 000 et 50 000 caractères), le nombre réel d'étapes d'installation, le comportement de `claude.ai/new?q=`, la synchronisation du connecteur vers le mobile et le rendu des images Markdown dans un résultat d'outil.
  3. Le nom commercial (sans « Claude » ni « Anthropic ») est choisi, le domaine de production définitif est enregistré et l'URL définitive du connecteur est fixée.
  4. Une page sécurité publique, sur ce domaine, affiche la formulation unique de confidentialité, sans le terme « bout en bout ».
  5. La base, le stockage de fichiers et les journaux sont créés en juridiction Union européenne, ce qui se vérifie dans la configuration de l'hébergeur.
**Plans**: TBD
**UI hint**: yes
**Research flag**: oui, la phase est elle-même la recherche (limites non documentées mesurées sur de vrais comptes). Écrire à `mcp-review@anthropic.com` dès cette phase pour obtenir une lecture écrite des clauses 1.G et 4.C de la politique d'annuaire.

### Phase 2: Compte et connecteur
**Goal:** En tant que founder non technique, je veux créer mon compte et brancher le connecteur à mon Claude en quelques étapes guidées, afin qu'il me reconnaisse durablement.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
**Success Criteria** (what must be TRUE):
  1. Un nouvel utilisateur crée son compte sans mot de passe, avec Google ou avec un code / lien reçu par e-mail.
  2. Depuis la page d'installation (lien de préremplissage officiel, captures d'écran, nombre d'étapes annoncé honnêtement), un utilisateur non technique ajoute le connecteur à son Claude, voit un écran de consentement qui nomme son compte en clair, puis son Claude lui confirme à quel compte il est relié.
  3. 24 heures plus tard, et après redémarrage du navigateur ou de l'app, le connecteur répond toujours sans redemander d'autorisation.
  4. L'utilisateur voit sur le web la liste des Claude connectés à son compte et en révoque un ; ce Claude perd l'accès et doit se reconnecter.
**Plans**: TBD
**UI hint**: yes
**Research flag**: ciblée : détails OAuth propres à Claude (CIMD avec DCR en repli, découverte, rotation des refresh tokens, taille de jeton, délai de 10 secondes sur `/token`), câblage `workers-oauth-provider` + Better Auth selon le résultat du spike de la phase 1.

### Phase 3: Envoi de brief entre contacts
**Goal:** En tant qu'utilisateur, je veux dire à mon Claude « envoie ce brief à Paul » et que Paul le lise sur le web, afin d'en finir avec les .md copiés-collés.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: CONT-01, CONT-03, SEND-01, SEND-02, SEND-03, RECV-01, RECV-02, RECV-06, SEC-01, CRED-01
**Success Criteria** (what must be TRUE):
  1. Un utilisateur invite un autre utilisateur existant par un lien ; le contact n'apparaît dans les deux carnets qu'après acceptation des deux côtés.
  2. Depuis n'importe quelle surface Claude, « envoie ce brief à Paul » envoie un brief texte / Markdown structuré (objectif, contexte, décisions, contraintes, questions ouvertes) ; si deux contacts s'appellent Paul, le serveur renvoie les candidats et Claude fait choisir, sans qu'aucun outil ne liste tout le carnet.
  3. L'expéditeur reçoit un reçu (identifiant, taille, empreinte SHA-256) ; renvoyer le même message ne crée ni doublon ni seconde ligne au registre, et le compteur d'envois visible sur le web augmente d'une seule unité.
  4. Le destinataire reçoit un e-mail à l'arrivée, ouvre la page web du message, voit l'aperçu, l'accepte ou le refuse ; après acceptation, il lit et copie le texte intact (même empreinte SHA-256) sans connecteur.
  5. Le contenu est stocké chiffré (clé par message) ; une chaîne canari envoyée dans un message n'apparaît dans aucun journal ni en clair dans la base.
**Plans**: TBD
**UI hint**: yes
**Research flag**: ciblée : surface d'acceptation hors modèle sur chaque appareil (page web ouverte depuis le mobile, Desktop, Claude Code), filtrage des journaux au niveau du transport, enveloppes de clés multiples du Vault (type `kms` seul au départ).

### Phase 4: Lecture dans Claude sans danger
**Goal:** En tant que destinataire, je veux dire « lis mes messages » à mon Claude, afin de reprendre un brief accepté sans que son contenu puisse piloter mon Claude.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: RECV-03, RECV-04, RECV-05, SEC-02, SEND-05
**Success Criteria** (what must be TRUE):
  1. Le destinataire dit « lis mes messages » sur n'importe quelle surface, mobile compris : son Claude liste ses messages puis lit le contenu de ceux qu'il a acceptés sur le web.
  2. Même avec « toujours autoriser » activé, aucun outil ne peut accepter un message, et un message non accepté n'expose à Claude que des métadonnées produites par le serveur, jamais un texte écrit par l'expéditeur.
  3. Le contenu lu arrive dans une enveloppe « donnée non fiable » à délimiteur aléatoire, débarrassé des caractères invisibles, images et liens Markdown neutralisés ; un corpus d'injection rejoué contre un compte relié à Gmail et Drive ne provoque ni envoi sortant non confirmé ni chargement d'image externe.
  4. Après lecture d'un contenu tiers, toute demande d'envoi sortant reste en attente jusqu'à ce que l'humain la confirme hors du modèle, sur une page web qui montre exactement ce qui part et à qui.
  5. Au-delà des plafonds de débit et de diffusion du compte, l'envoi est refusé avec un message factuel et rien ne part.
**Plans**: TBD
**UI hint**: yes
**Research flag**: ciblée : corpus d'injection en régression, règle de sortie déterministe côté serveur, limites de l'approbation par appel (« toujours autoriser », mode Research), chargement des images Markdown dans les résultats d'outil de claude.ai.

### Phase 5: Suivi, rappel et blocage
**Goal:** En tant qu'utilisateur, je veux suivre mes envois, rappeler un message et écarter un contact, afin de garder la main sur ce que je partage et reçois.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: CONT-02, SEND-04, RECV-07
**Success Criteria** (what must be TRUE):
  1. L'expéditeur voit, dans son Claude et sur sa page d'envois, si chaque message est livré, accepté ou lu.
  2. L'expéditeur rappelle un message pas encore lu ; le destinataire ne peut plus l'ouvrir, ni dans Claude ni sur le web.
  3. Un message expire après le délai par défaut et n'est plus lisible nulle part ; l'expéditeur voit qu'il a expiré.
  4. L'utilisateur supprime ou bloque un contact ; ce contact ne peut plus lui envoyer de message, et un contact bloqué ne peut pas non plus lui renvoyer d'invitation.
**Plans**: TBD
**UI hint**: yes
**Research flag**: non : schémas standards (machines à états, tâche planifiée d'expiration).

### Phase 6: Invitation par lien magique
**Goal:** En tant qu'utilisateur, je veux envoyer un brief à quelqu'un qui n'a pas de compte, afin qu'il le lise et rejoigne le service en quelques clics.
**Mode:** mvp
**Depends on**: Phase 5
**Requirements**: INV-01, INV-02, INV-03, INV-04
**Success Criteria** (what must be TRUE):
  1. Envoyer à une adresse sans compte crée une invitation avec message retenu et rend à l'expéditeur un lien qu'il transmet lui-même (WhatsApp, e-mail) ; la plateforme n'écrit pas à l'invité.
  2. Sur la page du lien, l'invité ne voit que des métadonnées vérifiées tant qu'il n'a pas prouvé qu'il contrôle l'adresse invitée ; l'ouverture du lien par un scanner de messagerie ne consomme pas l'invitation.
  3. Après cette preuve, l'invité accepte : son compte et le contact mutuel sont créés, puis il est guidé jusqu'à l'installation du connecteur et lit le brief dans son Claude.
  4. Un invité qui ne peut pas installer le connecteur (emplacement Free déjà pris, membre non Owner d'une organisation) lit et copie le brief sur le web sans rien installer.
  5. Au-delà du plafond quotidien d'invitations, l'expéditeur ne peut plus en créer ce jour-là et en est informé par un message factuel.
**Plans**: TBD
**UI hint**: yes
**Research flag**: légère : anti-abus des invitations, base légale RGPD de l'adresse invitée, liens consommés par les scanners de messagerie.

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Spike de plateforme et choix fondateurs | 0/0 | Not started | - |
| 2. Compte et connecteur | 0/0 | Not started | - |
| 3. Envoi de brief entre contacts | 0/0 | Not started | - |
| 4. Lecture dans Claude sans danger | 0/0 | Not started | - |
| 5. Suivi, rappel et blocage | 0/0 | Not started | - |
| 6. Invitation par lien magique | 0/0 | Not started | - |
