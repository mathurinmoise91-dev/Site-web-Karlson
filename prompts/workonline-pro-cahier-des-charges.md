# WorkOnline Pro – Cahier des charges complet (prompt Lovable / Bolt)

> Version courte pour démarrer : voir `workonline-pro-prototype.md`.

Crée une **application web complète** appelée **WorkOnline Pro** (React + Tailwind CSS + Supabase ou équivalent) qui combine :

1. une **marketplace d'emplois en ligne de longue durée** (CDI, CDD long, contrats pluriannuels) entre **entreprises** et **travailleurs**,
2. un **service de création légale d'entreprise** en ligne avec **abonnement à 30 $ par mois**,
3. un **système de paiement sécurisé avec escrow et validation du travail**,
4. une **sécurité renforcée** contre les pirates,
5. un **tableau de bord admin** pour suivre tous les flux d'argent, commissions et pourcentages,
6. un **système de suspension en cas d'impayé**,
7. un **espace partenariats** pour les entreprises qui veulent travailler avec WorkOnline Pro,
8. un **espace « Partenariats internationaux B2B »** où des entreprises peuvent trouver des partenaires à l'étranger,
9. un **marché de contrats** où les entreprises peuvent **acheter et vendre des contrats** avec **10 % de commission** pour WorkOnline Pro,
10. une **vérification d'identité obligatoire (KYC)** à l'inscription avec **photo de pièce d'identité + selfie + document légal de l'entreprise** pour éviter les arnaques.

L'application doit être **responsive** (desktop + mobile), sécurisée (HTTPS, rôles, chiffrement des données sensibles) et prête pour un déploiement réel.

---

## 1. Rôles et permissions

- Rôles : `user` (candidat), `company` (entreprise), `entrepreneur`, `admin`.
- Règles :
  - Seul `admin` peut voir les pièces d'identité et documents entreprise en clair.
  - Aucun utilisateur ne peut publier d'offre, acheter/vendre un contrat, ou recevoir un paiement tant que `statut_kyc != "validé"`.
  - Toutes les routes sensibles (paiements, documents, admin) sont protégées et journalisées.

---

## 2. Tables de données principales

Noms et champs indicatifs, à adapter à Supabase :

- `users` : `id`, `email`, `password_hash`, `role` (user/company/entrepreneur/admin), `first_name`, `last_name`, `country`, `phone`, `statut_kyc` (en_attente/validé/rejeté), `created_at`
- `identity_documents` : `id`, `user_id`, `type_piece` (carte_identite, passeport, permis, etc.), `url_recto`, `url_verso`, `url_selfie`, `statut` (en_attente/validé/rejeté), `admin_comment`, `created_at`
- `company_documents` : `id`, `user_id`, `type_document` (kbis, registre_commerce, statuts, autre), `url_document`, `statut` (en_attente/validé/rejeté), `admin_comment`, `created_at`
- `companies` (profil entreprise) : `id`, `user_id`, `company_name`, `registration_number`, `address`, `website`, `sector`, `size`, `description`, `logo_url`, `created_at`
- `entrepreneurs` (profil entrepreneur) : `id`, `user_id`, `company_name`, `registration_number`, `address`, `sector`, `statut_abonnement` (actif/resilie), `created_at`
- `job_offers` : `id`, `company_id`, `title`, `description`, `contract_type` (CDI/CDD/autre), `duration_months`, `salary_min`, `salary_max`, `currency`, `country`, `remote` (oui/non/hybride), `skills`, `status` (active/fermee/suspendue), `created_at`
- `applications` : `id`, `job_id`, `candidate_id`, `message`, `expected_salary`, `status` (envoyee/entretien/embauche/rejetee), `created_at`
- `employment_contracts` : `id`, `company_id`, `candidate_id`, `job_id`, `contract_type`, `duration_months`, `salary`, `currency`, `start_date`, `payment_status` (depot_requis/bloque/valide/payé), `created_at`
- `payments` : `id`, `payer_id`, `receiver_id`, `amount`, `currency`, `type` (salaire/commission/abonnement/partenariat/contrat), `status` (en_attente/bloque/valide/payé/refund), `metadata` (JSON), `created_at`
- `subscriptions` : `id`, `user_id`, `type` (creation_entreprise), `monthly_price` (30), `status` (actif/resilie), `last_payment_date`, `next_payment_date`, `created_at`
- `partnerships` : `id`, `company_id`, `type` (recrutement_volume, integration_rh, marque_blanche, autre), `description`, `conditions`, `status` (actif/inactif), `created_at`
- `international_partnerships` : `id`, `company_id`, `partner_type` (distributeur, revendeur, fournisseur, coentreprise, agent, autre), `target_countries`, `description`, `pack` (base/premium/commission), `status` (ouverte/en_negociation/conclue/fermee), `created_at`
- `contracts_market` : `id`, `seller_id`, `type_contrat` (prestation, distribution, service, partenariat_commercial, autre), `description`, `price`, `currency`, `sector`, `country`, `status` (en_vente/vendu/suspendu), `created_at`
- `contract_sales` : `id`, `contract_id`, `buyer_id`, `total_price`, `commission` (10 %), `net_seller`, `payment_status` (en_attente/bloque/valide/payé), `created_at`
- `admin_logs` : `id`, `admin_id`, `action` (validation_kyc, suspension, deblocage, rejet, etc.), `target_user_id`, `reason`, `created_at`

---

## 3. Écrans et flux utilisateurs

### Inscription / Connexion / Vérification

- **Inscription**
  - Choix du type de compte : Candidat / Entreprise / Entrepreneur.
  - Champs : nom, prénom, email, mot de passe, pays, téléphone.
  - Après soumission → redirection vers **Vérification d'identité**.
- **Connexion**
  - Email + mot de passe + lien « Mot de passe oublié ».
  - Option 2FA (code par email/SMS/app).
- **Vérification d'identité (KYC)**
  - Pour tous : upload pièce d'identité (recto/verso) et capture selfie (webcam / téléphone).
  - Pour Entreprise / Entrepreneur : upload document légal de l'entreprise (Kbis, registre du commerce, statuts, etc.).
  - Message après upload : « Votre compte est en cours de vérification. Vous recevrez un email lorsque votre compte sera validé. »
  - Tant que `statut_kyc != "validé"` : accès limité (pas de publication d'offre, pas d'achat/vente de contrats, pas de paiements).
- **Profil utilisateur**
  - Affiche : informations personnelles, statut KYC, documents uploadés.
  - Si statut = « Rejeté » : affiche le motif et bouton « Télécharger un nouveau document ».

### Accueil

- Hero : « Trouvez un emploi en ligne de longue durée, recrutez des talents, créez votre entreprise légalement, développez votre activité à l'international, ou achetez/vendez des contrats avec WorkOnline Pro ».
- 5 CTA : « Je cherche un emploi », « Je recrute », « Je crée mon entreprise », « Je me développe à l'international », « Marché de contrats ».
- Sections : offres d'emploi récentes, entreprises qui recrutent, création d'entreprise (encart « 30 $ / mois »), partenariats internationaux (exemples d'offres), contrats à vendre (exemples).
- Footer : « Comment ça marche », « Tarifs & commissions », « Sécurité & Confiance », « Vérification & Sécurité », « Partenariats », « Partenariats internationaux », « Marché de contrats », « Blog », « Contact », CGU, Mentions légales, Politique de confidentialité.

### Emploi

- **Liste des offres** : filtres (secteur, type de contrat, durée, salaire, pays, télétravail) ; carte ou liste avec titre, entreprise, lieu, type de contrat, salaire.
- **Détail d'une offre** : description complète, compétences, entreprise, bouton « Postuler » (si candidat vérifié).
- **Publier une offre** (entreprise vérifiée) : titre, description, type de contrat, durée, salaire, pays, télétravail, compétences.
- **Tableau de bord Candidat** : onglets « Candidatures envoyées », « Entretiens », « Contrats signés » ; pour chaque candidature : statut, entreprise, poste, date.
- **Tableau de bord Entreprise** : onglets « Offres actives », « Candidatures reçues », « Entretiens », « Embauches » ; pour chaque offre : nombre de candidatures, statut.

### Création d'entreprise

- **Parcours de création** :
  1. Choix du statut (micro-entreprise, SARL, SAS, etc.) avec explications.
  2. Saisie des informations (nom, adresse, activité, associés, capital).
  3. Génération des statuts (texte à remplir/valider).
  4. Dépôt du capital (si nécessaire) via partenaire.
  5. Envoi du dossier d'immatriculation.
- **Page Tarifs** : « Créez votre entreprise légalement avec WorkOnline Pro – 30 $ / mois » + liste des services inclus.
- **Tableau de bord Entrepreneur** : infos entreprise, statut abonnement, factures, documents juridiques.

### Paiements & Escrow (emplois)

- **Flux de paiement pour un contrat de travail** :
  1. L'entreprise dépose le salaire sur la plateforme avant la date de début.
  2. L'argent est bloqué en escrow.
  3. Le candidat travaille.
  4. L'entreprise clique sur « Valider le travail ».
  5. L'argent est versé au candidat.
- **Tableau de bord Entreprise** : « Dépôts », « Salaires bloqués », « Validations », « Litiges ».
- **Tableau de bord Candidat** : « Salaires en attente », « Salaires reçus », « Historique ».
- **Gestion des impayés** : si le dépôt n'est pas fait à la date prévue → statut du contrat = « suspendu », avec le message : « Votre contrat est suspendu car le dépôt du salaire n'a pas été effectué. Veuillez régulariser pour reprendre le travail. »

### Partenariats internationaux B2B

- **Page « Partenariats internationaux »** : titre « Développez votre entreprise à l'international avec WorkOnline Pro », explication du service, 2 CTA : « Je cherche un partenaire à l'étranger » / « Je propose un partenariat international ».
- **Publier une offre de partenariat international** : type de partenaire, pays cibles, description, pack (base/premium/commission).
- **Liste des offres** : filtres (pays, secteur, type de partenaire), badge « Premium » pour les offres sponsorisées.
- **Tableau de bord Entreprise** : « Mes offres », « Mises en relation », « Deals ».

### Marché de contrats

- **Page « Marché de contrats »** : titre « Achetez et vendez des contrats en toute sécurité avec WorkOnline Pro », explication + 2 CTA : « Je veux vendre un contrat » / « Je veux acheter un contrat ».
- **Publier un contrat à vendre** : type de contrat, description, prix, secteur, pays, documents.
- **Liste des contrats à vendre** : filtres (type, secteur, pays, prix) ; carte/liste avec titre, prix, vendeur, secteur.
- **Détail d'un contrat** : description complète, prix, conditions, aperçu des documents, bouton « Acheter ce contrat ».
- **Processus d'achat** :
  - Récapitulatif : prix du contrat, commission 10 % WorkOnline Pro, total.
  - Paiement sécurisé → argent bloqué en escrow.
  - Le vendeur télécharge les documents.
  - L'acheteur valide la réception → 90 % au vendeur, 10 % à WorkOnline Pro.
- **Tableau de bord Entreprise** : « Mes contrats vendus » (revenus 90 %, commissions 10 %), « Mes contrats achetés ».

### Admin

- **Dashboard financier** : graphiques des revenus par source (embauches, abonnements 30 $, partenariats, contrats 10 %), évolution mensuelle ; totaux : argent entrant, sortant, solde, escrow.
- **Page Transactions** : tableau (date, payeur, bénéficiaire, montant, type, statut) ; filtres (mois, type, utilisateur).
- **Page Vérifications (KYC)** : liste des utilisateurs en attente ; visualisation des documents (pièce d'identité, selfie, document entreprise) ; boutons « Valider », « Demander un document complémentaire », « Rejeter » (avec motif) ; statistiques (en attente, validées, rejetées).
- **Page Impayés & Suspensions** : liste des comptes en retard ; boutons « Suspendre », « Débloquer ».
- **Pages Partenariats, Partenariats internationaux, Marché de contrats** : listes, détails, stats, commissions.

---

## 4. Règles métier à implémenter

- Aucun utilisateur ne peut publier d'offre, acheter/vendre un contrat, ou recevoir un paiement tant que `statut_kyc != "validé"`.
- Pour chaque contrat de travail :
  - l'entreprise doit déposer le salaire avant la date de début,
  - si le dépôt n'est pas fait à J-1 → statut = « suspendu »,
  - après validation du travail → paiement au candidat.
- Pour chaque achat de contrat :
  - commission automatique de 10 % pour la plateforme,
  - 90 % au vendeur,
  - paiement via escrow avec validation de réception.
- Abonnement création d'entreprise : 30 $ / mois, prélèvement automatique, suspension si impayé.
- Toutes les actions admin (validation KYC, suspension, rejet) sont journalisées dans `admin_logs`.

---

## 5. Messages utilisateurs types

- Compte en attente : « Votre compte est en cours de vérification. Vous recevrez un email lorsque votre compte sera validé. »
- Document rejeté : « Votre document a été refusé : [motif]. Veuillez télécharger un nouveau document. »
- Contrat suspendu (impayé) : « Votre contrat est suspendu car le dépôt du salaire n'a pas été effectué. Veuillez régulariser pour reprendre le travail. »
- Achat de contrat : « Vous achetez ce contrat pour [prix]. WorkOnline Pro prélève une commission de 10 % ([montant]). Total à payer : [total]. »

---

## 6. Sécurité et confidentialité

- HTTPS obligatoire sur tout le site.
- Chiffrement des données sensibles (pièces d'identité, documents entreprise, données bancaires).
- Seuls les admins peuvent voir les documents KYC en clair.
- 2FA recommandée pour tous les comptes.
- Journalisation des actions sensibles (paiements, validations KYC, suspensions).
