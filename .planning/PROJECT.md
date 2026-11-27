# Relay (nom provisoire)

## What This Is

Un service qui permet à une personne de dire à son Claude « envoie ce brief / ce dossier à Paul », et au Claude de Paul de le recevoir directement, sans passer par un .md copié-collé, un mail ou un Drive. Chaque utilisateur ajoute des « Claudes » en contacts et leur envoie du contexte (brief, fichiers, conversations, dossiers entiers). Cible initiale : les founders SaaS / e-commerce non techniques (pas de GitHub, pas de terminal), puis le grand public.

Inspiration : tweet du 21/11/2026 (@andrealbriziom) — « le jour où on pourra dire à notre claude de parler au claude d'une autre personne pour le briefer... juste t'ajoutes des claudes en contact et tu send les dossiers dans le bon projet de ton gars ».

## Core Value

Un brief envoyé par le Claude de A arrive, intact et lisible, dans le Claude de B en moins d'une minute et sans aucune manipulation technique — sans jamais devenir un vecteur d'attaque contre le Claude de B.

## Business Context

- **Customer**: founders SaaS / e-commerce non techniques, puis grand public
- **Revenue model**: freemium à crédits ; croissance virale par invitation (chaque envoi à un non-utilisateur = acquisition) ; quand les crédits sont épuisés, le connecteur MCP l'annonce dans la conversation avec un lien d'achat
- **Success metric v1**: le créateur et 2-3 contacts l'utilisent réellement chaque semaine à la place des .md copiés-collés
- **Strategy notes**: produit public à terme (Chrome Web Store, comptes, paiement)

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Un utilisateur peut ajouter un autre utilisateur comme contact (contact validé des deux côtés)
- [ ] Depuis n'importe quel Claude (web, desktop, mobile, Claude Code), l'utilisateur peut dire en langage naturel « envoie X à <contact> » et le contenu part
- [ ] Contenu transportable : brief texte, fichiers (.md, PDF, code, images), conversation ou extrait, dossier / projet entier
- [ ] Le destinataire peut dire « lis mes messages » à son Claude et obtenir la liste puis le contenu
- [ ] Le destinataire voit un aperçu et accepte avant que son Claude ingère le contenu
- [ ] Le contenu reçu est présenté à Claude comme donnée non fiable, jamais comme instruction ; aucune action automatique à la réception
- [ ] Un destinataire sans compte reçoit un lien magique (mail / WhatsApp) : aperçu, puis un clic installe le connecteur et ouvre le contenu dans son Claude
- [ ] Extension Chrome sur claude.ai : bouton « Envoyer à… », carnet de contacts, badge de notification, bouton « Ouvrir dans Claude »
- [ ] Chiffrement de bout en bout visé (voir décision ouverte ci-dessous)
- [ ] Système de crédits freemium ; message « plus de crédits + lien d'achat » renvoyé par le MCP dans la conversation
- [ ] Deux niveaux d'usage : novices (extension + clics) et avancés (connecteur seul, langage naturel)

### Out of Scope

- Utiliser l'API interne non documentée de claude.ai (écriture directe dans les Projets, automatisation du site) — fragile et risque CGU Anthropic ; on passe par le connecteur MCP officiel
- Conversation live autonome Claude↔Claude sans humain dans la boucle — risque sécurité (prompt injection en chaîne) et complexité ; à reconsidérer après v1
- App mobile native / extension Safari iOS — le mobile est couvert par le connecteur MCP (connecteurs claude.ai synchronisés sur l'app)
- Autres IA (ChatGPT, Gemini…) — v1 centrée sur Claude ; piste de différenciation future

## Context

- Écosystème : les connecteurs MCP distants (custom connectors) ajoutés sur claude.ai sont disponibles sur web, desktop et mobile ; Claude Code consomme aussi des serveurs MCP. C'est le seul canal officiel pour donner des outils à Claude côté consommateur.
- Une extension Chrome peut enrichir l'UI de claude.ai (boutons, panneau, notifications) mais ne doit pas piloter le site via ses endpoints internes.
- Risque majeur : prompt injection. Injecter le contenu d'un tiers dans un Claude qui a peut-être accès au Gmail/Drive/Stripe du destinataire. La sécurité est un argument de vente, pas un bonus.
- Risque produit : Anthropic peut sortir une fonction native de partage. Différenciation : carnet de contacts inter-organisations, simplicité pour non-techniques, multi-surfaces, plus tard multi-IA.
- À vérifier en recherche : nombre de custom connectors autorisés sur le plan Free de claude.ai ; existence de fonctionnalités natives concurrentes (partage de projets, etc.) ; règles de l'annuaire de connecteurs Anthropic sur le contenu promotionnel dans les réponses d'outils ; prefill d'URL claude.ai (ex. `claude.ai/new?q=`).
- Utilisateur porteur du projet : francophone, TDAH — cycles courts, livrables visibles rapidement.

## Constraints

- **Plateforme**: canal officiel uniquement (MCP distant + extension Chrome non intrusive) — CGU Anthropic et robustesse aux mises à jour de claude.ai
- **Sécurité**: contacts validés uniquement, aperçu avant lecture, contenu étiqueté non fiable, zéro action automatique — protection contre la prompt injection
- **Confidentialité**: E2E visé ; tension connue : un connecteur MCP distant doit renvoyer du clair à Claude, donc le déchiffrement côté serveur au moment de la lecture est inévitable sauf déchiffrement côté client (extension, MCP local). Compromis à trancher en recherche
- **UX**: utilisable par un non-technique sans terminal ni GitHub ; installation en quelques clics
- **Nom**: éviter « Claude » dans le nom commercial (marque Anthropic)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Cœur = serveur MCP distant, surcouche = extension Chrome | Couvre web, desktop, mobile et Claude Code ; canal officiel ; l'extension apporte le confort novice | — Pending |
| Lien magique pour destinataires sans compte | Moteur viral du freemium ; zéro friction à la première réception | — Pending |
| Freemium à crédits, upsell via message MCP | Monétisation alignée sur l'usage ; point de contact directement dans la conversation | — Pending |
| Chiffrement de bout en bout visé | Briefs de founders = données sensibles ; argument de confiance | — Pending (compromis technique à trancher) |
| Pas d'API interne claude.ai | Fragilité + CGU | — Pending |
| Tous les scénarios (founder→freelance, associé↔associé, founder→agence) traités de façon générique | Pas de scénario prioritaire exprimé | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-11-27 after initialization*
