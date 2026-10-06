# WorkOnline Pro – Prompt « premier prototype » (Lovable / Bolt)

Version courte du cahier des charges (`workonline-pro-cahier-des-charges.md`).
Objectif : lancer vite une V1 qui fonctionne, puis l'enrichir étape par étape
avec les prompts de la section « Étapes suivantes ».

---

## Prompt V1 (à coller tel quel)

```markdown
Crée le premier prototype de **WorkOnline Pro**, une plateforme web (React + Tailwind + Supabase) qui met en relation **entreprises** et **travailleurs** pour des **emplois en ligne de longue durée**, avec **vérification d'identité obligatoire** et un **marché de contrats** (commission 10 %). Interface en français, responsive (mobile + desktop), design professionnel et sobre (bleu marine + blanc).

**Paiements en mode démo uniquement** : aucun vrai paiement, montants et statuts simulés, badge « Mode démo » visible.

### Rôles et sécurité
- Rôles : `candidat`, `entreprise`, `admin`, stockés dans une table séparée `user_roles` (jamais dans le profil) et vérifiés par une fonction SQL `has_role()`.
- Un utilisateur ne peut jamais se donner le rôle admin ni modifier lui-même son statut KYC.
- RLS activé sur toutes les tables : chacun ne voit et ne modifie que ses propres données (les offres et contrats publiés sont lisibles par tous).

### Tables
- `profiles` : id (= auth.users.id), prenom, nom, pays, telephone, type_compte (candidat/entreprise), nom_entreprise, statut_kyc (en_attente/valide/rejete, défaut en_attente), motif_rejet
- `user_roles` : user_id, role
- `kyc_documents` : user_id, type (identite_recto/identite_verso/selfie/document_entreprise), chemin_fichier, created_at
- `job_offers` : entreprise_id, titre, description, type_contrat (CDI/CDD long/autre), duree_mois, salaire_min, salaire_max, devise, pays, teletravail (oui/non/hybride), statut (active/fermee)
- `applications` : offre_id, candidat_id, message, statut (envoyee/entretien/embauche/rejetee)
- `contracts_market` : vendeur_id, titre, type_contrat, description, prix, devise, secteur, pays, statut (en_vente/vendu)
- `contract_sales` : contrat_id, acheteur_id, prix, commission, net_vendeur, statut (bloque/valide)
- `admin_logs` : admin_id, action, cible_user_id, motif, created_at

### Écrans
1. **Accueil** : hero « Emplois en ligne de longue durée, recrutement vérifié et marché de contrats sécurisé », 3 boutons (Je cherche un emploi / Je recrute / Marché de contrats), dernières offres, encart « Création d'entreprise – 30 $/mois (bientôt) ».
2. **Inscription / Connexion** (email + mot de passe) : choix Candidat ou Entreprise, puis redirection vers la vérification.
3. **Vérification d'identité (KYC)** : pièce d'identité recto/verso + selfie (caméra ou fichier) ; pour les entreprises, en plus un document légal (Kbis / registre du commerce). Fichiers dans un bucket Storage **privé** `kyc` : chaque utilisateur n'accède qu'à ses fichiers, l'admin les voit via URL signée. Message : « Votre compte est en cours de vérification. Vous recevrez un email lorsque votre compte sera validé. » Si rejeté : afficher le motif + bouton pour renvoyer un document.
4. **Offres d'emploi** : liste avec filtres (type de contrat, pays, télétravail, salaire), page détail, bouton « Postuler ». Formulaire « Publier une offre » pour les entreprises.
5. **Marché de contrats** : liste, détail, formulaire « Vendre un contrat ». Bouton « Acheter » → récapitulatif : « Vous achetez ce contrat pour [prix]. WorkOnline Pro prélève une commission de 10 % ([montant]). Le vendeur reçoit [90 %]. » → vente créée au statut « bloqué » (escrow simulé) → l'acheteur clique « Valider la réception » → statut « validé », contrat « vendu ».
6. **Tableau de bord** selon le rôle :
   - Candidat : mes candidatures et leur statut.
   - Entreprise : mes offres, candidatures reçues (boutons Entretien / Embaucher / Rejeter), mes contrats vendus et achetés.
7. **Admin** :
   - Vérifications KYC : comptes en attente, aperçu des documents, boutons Valider / Rejeter (motif obligatoire).
   - Finances : total des ventes de contrats, total des commissions 10 %, montant bloqué en escrow, tableau des transactions.
   - Journal des actions admin.

### Règles appliquées dans la base (pas seulement dans l'interface)
- Tant que `statut_kyc != valide` : impossible de publier une offre, de postuler, de vendre ou d'acheter un contrat (bandeau d'avertissement + boutons désactivés, et blocage par RLS).
- `commission` (10 %) et `net_vendeur` (90 %) calculés par un trigger SQL, jamais envoyés par le navigateur.
- Seul un admin peut changer `statut_kyc` ; chaque validation ou rejet crée une ligne dans `admin_logs`.

Ajoute des données de démonstration (5 offres d'emploi, 4 contrats à vendre) pour que l'app ne soit pas vide.
```

---

## Étapes suivantes (un prompt à la fois, quand la V1 fonctionne)

### Étape 2 – Contrats de travail et escrow des salaires

```markdown
Ajoute les contrats de travail. Quand une entreprise clique « Embaucher », crée un `employment_contract` (entreprise, candidat, offre, salaire, devise, date_debut, statut_paiement : depot_requis/bloque/valide/paye/suspendu). L'entreprise doit déposer le salaire avant la date de début (dépôt simulé → statut « bloqué »). Si le dépôt n'est pas fait à J-1, le contrat passe en « suspendu » avec le message : « Votre contrat est suspendu car le dépôt du salaire n'a pas été effectué. Veuillez régulariser pour reprendre le travail. » L'entreprise clique « Valider le travail » → statut « payé » au candidat. Tableaux de bord : entreprise (Dépôts, Salaires bloqués, Validations, Litiges) et candidat (Salaires en attente, Salaires reçus, Historique). Page admin « Impayés & Suspensions » avec boutons Suspendre / Débloquer, journalisés dans `admin_logs`.
```

### Étape 3 – Création d'entreprise (30 $/mois)

```markdown
Ajoute le rôle `entrepreneur` et un parcours de création d'entreprise en 5 étapes : choix du statut (micro-entreprise, SARL, SAS… avec explications), informations (nom, adresse, activité, associés, capital), génération d'un brouillon de statuts à valider, dépôt du capital via partenaire (étape informative), envoi du dossier. Page Tarifs « Créez votre entreprise légalement avec WorkOnline Pro – 30 $ / mois ». Table `subscriptions` (statut actif/resilie/suspendu, prochaine échéance). Tableau de bord entrepreneur : infos entreprise, statut abonnement, factures, documents. Abonnement suspendu automatiquement si impayé.
```

### Étape 4 – Partenariats et partenariats internationaux B2B

```markdown
Ajoute deux espaces réservés aux entreprises vérifiées : (1) « Partenariats » avec WorkOnline Pro (recrutement en volume, intégration RH, marque blanche) ; (2) « Partenariats internationaux B2B » : titre « Développez votre entreprise à l'international avec WorkOnline Pro », boutons « Je cherche un partenaire à l'étranger » / « Je propose un partenariat international », formulaire (type de partenaire, pays cibles, secteur, description, pack base/premium/commission), liste filtrable (pays, secteur, type) avec badge « Premium », et tableau de bord « Mes offres / Mises en relation / Deals ». Pages admin correspondantes avec statistiques.
```

### Étape 5 – Vrais paiements et sécurité renforcée

```markdown
Remplace les paiements simulés par Stripe Connect : paiement de l'acheteur ou de l'entreprise, fonds conservés jusqu'à validation, puis versement au bénéficiaire et commission de 10 % pour WorkOnline Pro via application fee. Abonnement 30 $/mois via Stripe Billing avec webhook qui suspend le compte si le paiement échoue. Ajoute la 2FA (application d'authentification), les emails de notification (compte validé, document rejeté, contrat suspendu, paiement reçu) et, dans le dashboard admin, des graphiques de revenus par source (embauches, abonnements, partenariats, contrats) et l'évolution mensuelle.
```
