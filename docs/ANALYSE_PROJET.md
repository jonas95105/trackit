# TrackIT — Analyse du projet

Document produit avant le développement (étape 2). Il recense ce qui a été trouvé
dans les livrables du dossier, les écarts entre ces livrables et les décisions
d'implémentation qui en découlent.

## 1. Documents analysés

| Livrable | Fichier | Contenu exploité |
|---|---|---|
| UML — cas d'utilisation | `Uml/Diagrammes de Cas d'Utilisation/*` | Global + UC-01 à UC-06 : acteurs et permissions |
| UML — classes | `Uml/Diagramme de classe` | Entités et cardinalités |
| UML — état-transition | `Uml/Diagramme d'État-Transition` | Cycle de vie d'une anomalie |
| UML — séquence, communication, activité | `Uml/…` | Scénario « Déclarer une anomalie » |
| UML — déploiement | `Uml/Diagramme de Déploiement` | Architecture technique (HTML/CSS/JS, Spring Boot, MySQL, Stripe, SMTP) |
| UML — navigation, contexte | `Uml/…` | Écrans et enchaînements |
| 08 — Schéma BDD | `Schéma de base de données - TrackIT/*` | 15 tables, types SQL, relations |
| 09 — Dictionnaire de données | `09_Dictionnaire_de_donnees_TrackIT.pages` | Types, longueurs, contraintes, règles |
| 10 — Rapport de conception graphique | `10_Rapport_de_conception_graphique_TrackIT_FINAL.docx` | Charte, navigation, comportements attendus |
| 11 — Prototype + guide de navigation | `11_Prototype_navigable_TrackIT.pptx`, `11_Guide_navigation_TrackIT.docx` | 17 écrans, messages, règles de validation |
| Maquettes | `maquettes_finales/*.png` | Accueil, liste, fiche, connexion, tableau de bord, profil |

**Le cahier des charges n'est pas présent dans le dossier.** Les règles « priorité au
cahier des charges » n'ont donc pas pu s'appliquer ; la référence retenue est, dans
l'ordre : consignes du commanditaire (prompt) → dictionnaire de données + schéma BDD →
diagrammes UML → rapport de conception / prototype → maquettes.

## 2. Fonctionnalités identifiées

| Module | Fonctionnalité | Source |
|---|---|---|
| Authentification | Connexion e-mail / mot de passe, JWT, déconnexion | UC-06, déploiement, maquette 04 |
| Authentification | Inscription d'une organisation (« S'inscrire », « Commencer maintenant ») | Maquettes 01 et 04 |
| Anomalies | Consulter, rechercher, filtrer, paginer | UC-01, maquette 02, prototype 11 |
| Anomalies | Déclarer (5 champs obligatoires + pièce jointe facultative) | UC-01, activité, prototype 6-9 |
| Anomalies | Modifier, classifier, changer le statut | UC-01, prototype 14-15 |
| Anomalies | Commentaires, pièces jointes, historique | Maquette 03, prototype 16 |
| Supervision | Superviser, prioriser, réaffecter, contrôler les délais | UC-02 |
| Analyse | Statistiques, incidents récurrents, rapports, export PDF / Excel | UC-03 |
| Administration | CRUD utilisateurs, attribution de rôle, référentiels (types, priorités, statuts), systèmes IT | UC-04 |
| Premium | Choisir une offre, payer via Stripe, activer les fonctionnalités Premium | UC-05 |
| API REST | CRUD anomalies authentifié par JWT | UC-06 |
| Notifications | Notification du responsable technique à la déclaration, du déclarant au changement de statut | Séquence, activité, prototype 9 et 15 |
| Tableau de bord | 4 KPI, évolution, répartition par type, anomalies récentes, sélecteur de période | Maquette 05 |
| Profil | Informations personnelles, mot de passe, préférences, notifications | Maquette 06 |

## 3. Rôles et matrice de permissions retenue

| Action | Utilisateur | Opérateur | Resp. technique | Analyste | Administrateur |
|---|:-:|:-:|:-:|:-:|:-:|
| Consulter / rechercher les anomalies de son organisation | ✔ | ✔ | ✔ | ✔ | ✔ |
| Déclarer une anomalie | | ✔ | | | |
| Modifier / classifier (titre, description, système, type) | | ✔ ¹ | ✔ | | |
| Changer le statut | | ✔ ¹ | ✔ | | |
| Fermer une anomalie (Résolue → Fermée) | | | ✔ | | ✔ |
| Prioriser (changer la priorité) | | ✔ ¹ | ✔ | | |
| Prendre en charge (s'affecter une anomalie non affectée) | | ✔ | | | |
| Affecter / réaffecter à un opérateur | | | ✔ | | |
| Contrôler les délais | | | ✔ | | |
| Commenter, joindre un fichier | | ✔ | ✔ | | |
| Statistiques, rapports, exports | | | ✔ (lecture stats) | ✔ | |
| Gérer les utilisateurs, rôles, référentiels, systèmes | | | | | ✔ |
| Souscrire Premium / paiements | | | | | ✔ |
| Supprimer une anomalie (UC-06) | | | | | ✔ |

¹ Uniquement sur les anomalies qu'il a déclarées ou qui lui sont affectées.

Tous les contrôles sont appliqués côté serveur (Spring Security + vérifications dans les
services) ; le frontend masque seulement les actions non autorisées.

**Périmètre** : TrackIT est un SaaS multi-organisation (table `ORGANISATION`). Chaque
utilisateur ne voit que les anomalies, systèmes et utilisateurs de son organisation. Le
« périmètre » du responsable technique est son organisation.

## 4. Architecture

```
Navigateur (HTML / CSS / JavaScript — conforme au diagramme de déploiement)
        │  HTTPS / JSON + JWT (Authorization: Bearer)
        ▼
Spring Boot 3 (Java 17)
  Controller → Service → Repository (Spring Data JPA / Hibernate) → MySQL 8
  Spring Security + JWT · Bean Validation · OpenAPI (springdoc)
  Flyway (migrations) · OpenPDF / Apache POI (exports) · Stripe Java (Checkout)
        │
        ├── MySQL 8 (JDBC)
        └── Stripe API (HTTPS, mode test)
```

Le frontend n'est défini que par le diagramme de déploiement : « Application Web
HTML / CSS / JavaScript ». Cette technologie est conservée (JavaScript natif, modules ES,
sans framework ni étape de build).

## 5. Modèle de données

Les 15 tables du schéma 08 sont toutes conservées, avec les noms de colonnes du
dictionnaire 09 (en minuscules : `role`, `organisation`, `utilisateur`, `systeme_it`,
`anomalie`, `type_anomalie`, `priorite`, `statut`, `commentaire`, `historique`,
`piece_jointe`, `notification`, `rapport`, `abonnement`, `paiement`).

Contraintes ajoutées conformément au dictionnaire : `UNIQUE` (role.nom, organisation.email,
utilisateur.email, type_anomalie.nom, priorite.nom, statut.nom, paiement.reference_stripe),
`CHECK` (priorite.niveau entre 1 et 4, paiement.montant > 0, abonnement.date_fin ≥ date_debut),
`DEFAULT` (actif = TRUE, lu = FALSE, dates = CURRENT_TIMESTAMP), index sur toutes les clés
étrangères et sur les colonnes de filtrage (`anomalie.date_creation`).

Référentiels retenus :

| Table | Valeurs | Source |
|---|---|---|
| ROLE | Utilisateur, Opérateur, Responsable technique, Analyste, Administrateur | UC global |
| STATUT | Nouvelle, En cours, Résolue, Fermée | Dictionnaire, rapport 10, prototype |
| PRIORITE | Basse (1), Moyenne (2), Haute (3), Critique (4) | Dictionnaire (niveau 1-4), maquettes |
| TYPE_ANOMALIE | Réseau, Application, Matériel, Accès, Base de données, Performance, Fonctionnel, Autre | Maquettes 02 et 05 |

## 6. Incohérences détectées et décisions

| # | Incohérence | Documents | Décision |
|---|---|---|---|
| 1 | Le diagramme d'état-transition contient les états **Assignée** et **Suspendue** | État-transition vs dictionnaire, rapport 10, prototype | Seuls les 4 statuts officiels sont implémentés. « Assignée » devient une donnée (`id_operateur`), pas un statut. « Suspendue » n'est pas implémenté. La transition « Incident réouvert » (Résolue → En cours) est conservée car elle ne crée pas de statut. |
| 2 | `ABONNEMENT` relié à **Utilisateur** (0..1) | Classes vs schéma BDD + dictionnaire | Rattaché à **ORGANISATION** (schéma + dictionnaire, plus détaillés et cohérents avec le SaaS). |
| 3 | `ORGANISATION`, `PIECE_JOINTE`, `NOTIFICATION` absents | Classes vs schéma BDD | Implémentés (schéma + dictionnaire). |
| 4 | `HISTORIQUE.action` : VARCHAR(100) vs VARCHAR(255) | Dictionnaire vs schéma | VARCHAR(255) (compatible avec les deux). |
| 5 | `Priorite.niveau : String`, `Statut.etat`, `TypeAnomalie.libelle` | Classes vs dictionnaire | Noms et types du dictionnaire (`nom`, `niveau INT`). |
| 6 | L'affectation est faite par un **Opérateur** dans le prototype (écran 12) alors que l'UC-02 la réserve au **Responsable technique** | Prototype vs UC-02 / état-transition | L'opérateur peut **se** prendre en charge une anomalie non affectée ; seule l'affectation/réaffectation à un autre opérateur est réservée au responsable technique. |
| 7 | La maquette de profil propose un champ **Rôle** modifiable | Maquette 06 vs UC-04 (« Attribuer rôle » = administrateur) | Le rôle est affiché en lecture seule dans le profil ; seul l'administrateur l'attribue. |
| 8 | « Changer la photo » : aucune colonne photo | Maquette 06 vs dictionnaire | Non implémenté ; un avatar à initiales est affiché. |
| 9 | « Continuer avec Google » | Maquette 04 vs tous les autres documents | Aucune authentification Google n'est prévue ailleurs : bouton non implémenté (absent de l'écran). |
| 10 | « Mot de passe oublié ? » : pas de table de jetons de réinitialisation | Maquette 04 vs dictionnaire | Le lien informe l'utilisateur ; la réinitialisation est faite par l'administrateur (UC-04 « Modifier utilisateur »). |
| 11 | Historique d'affectation : pas de colonnes ancien/nouvel opérateur | Prototype 16 vs dictionnaire | L'affectation est décrite dans `action` (« Affectation : Younes ERRAHMANI »). |
| 12 | Commentaire lors d'un changement de statut / d'une affectation | Prototype 12, 14 vs dictionnaire | Enregistré comme `COMMENTAIRE` lié à l'anomalie. |
| 13 | Le contenu exact des « fonctionnalités Premium » n'est défini nulle part | UC-05 | Hypothèse : les **exports PDF / Excel** des rapports sont Premium. Les statistiques restent accessibles à tous les analystes. |
| 14 | Délais (« Contrôler délais ») : aucune donnée de délai cible | UC-02 vs dictionnaire | Délais cibles par niveau de priorité définis en configuration (`trackit.sla.*`), sans nouvelle table. |
| 15 | Serveur SMTP dans le déploiement, aucun envoi d'e-mail spécifié | Déploiement | Notifications internes (table NOTIFICATION) ; l'envoi d'e-mail n'est pas implémenté. |
| 16 | Les référentiels (types, priorités, statuts) n'ont pas d'organisation | Schéma BDD | Référentiels globaux, administrés par les administrateurs. Les 4 statuts ne peuvent être ni créés ni supprimés (seule la description est modifiable). |

## 7. Fonctionnalités manquantes au départ

Aucun code n'existait dans le dossier : backend, frontend, base de données, dump, tests et
README sont à créer intégralement.

Éléments volontairement non implémentés (non spécifiés ou contradictoires, voir section 6) :
connexion Google, envoi d'e-mails SMTP, photo de profil, réinitialisation autonome du mot de
passe, statuts « Assignée » et « Suspendue ». Le sélecteur de langue de l'accueil n'affiche
que « FR » (aucune autre langue n'est prévue).

Les données de démonstration des rapports (table `RAPPORT`) n'ont pas de fichier associé
(`chemin_fichier` à NULL, autorisé par le dictionnaire) : l'interface les signale comme
« Fichier non conservé » ; les rapports générés depuis l'application sont téléchargeables.

## 8. Plan d'implémentation

1. Migrations Flyway : schéma (V1), référentiels (V2), données de démonstration (V3, désactivables).
2. Backend : entités JPA, repositories, DTO, services, contrôleurs, gestion globale des erreurs.
3. Sécurité : BCrypt, JWT, filtres, `@PreAuthorize`, contrôle de périmètre organisation.
4. Métier : machine à états des statuts, historique, notifications, affectation, pièces jointes.
5. Recherche : `Specification` JPA + pagination/tri en base.
6. Statistiques calculées en SQL, rapports PDF (OpenPDF) et Excel (Apache POI).
7. Stripe Checkout (mode test) + webhook signé + mode simulation sans clé.
8. Frontend HTML/CSS/JS reproduisant les 6 maquettes + écrans du prototype.
9. Générateur de données réalistes (≥ 100 lignes par table métier) puis `mysqldump` → `database/14_DUMP_TrackIT.sql`.
10. Swagger, tests (authentification, autorisations, création, statut, affectation, recherche).
11. README, puis exécution réelle et corrections.
