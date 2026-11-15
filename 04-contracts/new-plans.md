# Recommandation globale 5.3.7 — relancer les ventes

_UI Color Palette — 7 novembre 2026. Données : Mixpanel (production, hors tests : Bourg-la-Reine, Lexington, Sceaux), Polar (commandes depuis le 5 mai 2026, checkouts depuis avril). Ce document remplace et complète `Recommandations_5.3.6.md`, `Strategie_Blocage_5.3.6.md` et `Modele_Conversion_5.3.7.md`._

## 0. En dix lignes

1. **Le taux d'achat par actif est resté stable sur deux ans (7 à 10 pour 1 000 actifs). Ce qui a baissé, c'est le nombre d'actifs : −55 % entre avril et octobre 2026.** Environ 60 % de la baisse des ventes vient du volume, 40 % du taux.
2. **Depuis mai, la part d'actifs qui voit le pricing a triplé (14 % → 35 %, 46 % en octobre) et la conversion des exposés est passée de 7 % à 2 %.** Cela coïncide avec le passage des boutons « Pro » vers le pricing (je suppose que le changement date de cette période, à confirmer). Les ventes par actif ont peu bougé : l'exposition en plus ressemble à du bruit.
3. **Corrections de ce que j'avais écrit** : la chute « de 12 à 4–5 pour 1 000 depuis la 5.3.0 » n'est pas confirmée par les données mensuelles (voir section 1.3).
4. **Objectif** : 5 % des actifs qui paient, avec ~20 % des exposés. Il faut exposer ~25 % des actifs, mais de façon qualifiée.
5. **Paliers** : conversion ×2 (retour au niveau historique), puis ×4 (mécanisme nouveau), puis 5 %. En parallèle, doubler le volume d'utilisateurs.
6. **Levier 5.3.7 principal** : moins de modals, mieux placés (après la valeur obtenue), directement vers le checkout du plan hebdo.
7. **Crédits** comptés par résultat (variables, styles, couleurs, éléments simulés), pas par action.
8. **Retenir les abonnés** : l'essai hebdo (mai–juillet) a donné 4 essais sur 10 jamais payés et aucun abonné au-delà de 2 paiements ; sans essai, 4 abonnés hebdo sur 9 ont payé 3 semaines ou plus. Une conversion plus forte sur du hebdo seul rapportera peu si la rétention reste faible.
9. **Mesurer avec Polar** (checkouts réussis avec email), plus de `Purchase Enabled`.
10. **Instrumenter avant de décider** : `Units`, plan et prix, début de checkout, valeur choisie.

## 1. Ce que l'historique montre

### 1.1 Mois par mois (Mixpanel, hors développement)

| Mois    | Actifs | Voient le pricing | % des actifs | `Purchase Enabled` | pour 1 000 actifs | par exposé |
| ------- | ------ | ----------------- | ------------ | ------------------ | ----------------- | ---------- |
| 2025-09 | 1 041  | 214               | 21 %         | 7                  | 6,7               | 3,3 %      |
| 2025-10 | 1 544  | 421               | 27 %         | 15                 | 9,7               | 3,6 %      |
| 2025-11 | 1 335  | 247               | 19 %         | 14                 | 10,5              | 5,7 %      |
| 2025-12 | 1 098  | 196               | 18 %         | 10                 | 9,1               | 5,1 %      |
| 2026-01 | 1 178  | 208               | 18 %         | 10                 | 8,5               | 4,8 %      |
| 2026-02 | 1 044  | 174               | 17 %         | 13                 | 12,5              | 7,5 %      |
| 2026-03 | 1 195  | 148               | 12 %         | 14                 | 11,7              | 9,5 %      |
| 2026-04 | 1 125  | 111               | 10 %         | 9                  | 8,0               | 8,1 %      |
| 2026-05 | 798    | 242               | 30 %         | 7                  | 8,8               | 2,9 %      |
| 2026-06 | 665    | 232               | 35 %         | 6                  | 9,0               | 2,6 %      |
| 2026-07 | 627    | 236               | 38 %         | 3                  | 4,8               | 1,3 %      |
| 2026-08 | 522    | 172               | 33 %         | 1                  | 1,9               | 0,6 %      |
| 2026-09 | 508    | 173               | 34 %         | 3                  | 5,9               | 1,7 %      |
| 2026-10 | 501    | 230               | 46 %         | 5                  | 10,0              | 2,2 %      |

`Pricing Consulted` n'existe que depuis septembre 2025. Avant, il y avait 9 à 15 `Purchase Enabled` par mois pour 826 à 1 048 actifs (2025 entier : 125 pour ~12 300 actifs-mois, soit 10 pour 1 000).

Repères de versions : 5.3.0 début avril 2026, 5.3.1 début mai, 5.3.2 mi-mai, 5.3.3 fin mai.

### 1.2 Les trois enseignements

| Période                | Actifs/mois (moy.) | Exposés | Acheteurs/mois | Pour 1 000 actifs | Par exposé |
| ---------------------- | ------------------ | ------- | -------------- | ----------------- | ---------- |
| Sept. 2025 – avr. 2026 | ~1 195             | 18 %    | ~11,5          | 9,6               | 5,4 %      |
| Janv. – avr. 2026      | ~1 135             | 14 %    | ~11,5          | 10,1              | 7,2 %      |
| Mai – oct. 2026        | ~603               | 35 %    | ~4,2           | 6,9               | 1,9 %      |

1. **Le volume est la cause principale.** Entre janvier–avril et mai–octobre, les actifs ont perdu 47 % (1 135 → 603 en moyenne), les nouveaux utilisateurs (`Consent Proof Sent`) ont baissé de 1 690 en janvier à 797 en octobre. Sur la baisse des ventes (de ~11,5 à ~4,2 par mois), environ 60 % vient des actifs et 40 % du taux par actif.
2. **L'exposition a triplé sans gain.** Le saut de mai (111 → 242 exposés alors que les actifs baissaient) coïncide avec le changement où les boutons Pro redirigent vers le pricing (date à confirmer). Les ventes par actif n'ont perdu que ~30 % : le surplus d'exposition ne produit pas de ventes.
3. **Le taux par actif varie par saison.** En 2025, août–septembre étaient à ~6,5 pour 1 000 contre ~10 en octobre–décembre. Les creux de juillet–septembre 2026 (2 à 6 pour 1 000) sont peut-être en partie saisonniers ; octobre 2026 revient à 10 (Mixpanel) ou 6 (Polar).

### 1.3 Corrections de ce que j'avais écrit plus tôt

- « Depuis la 5.3.0, les acheteurs pour 1 000 actifs sont passés de 12 à 4–5 » : sur les mois complets, avril (5.3.0) est à 8,0, mai à 8,8, juin à 9,0 ; la baisse est surtout en juillet–septembre, avec un retour à 10 en octobre. La moyenne mai–octobre est à 6,9 pour 1 000. **Ne pas attribuer la baisse à la 5.3.0 seule.**
- « Les 4–5 ‰ d'aujourd'hui » : c'est la moyenne basse de certains mois ; utiliser environ 7 ‰ (6,9 Mixpanel, 7,2 Polar, 6,1 en retirant les 4 essais jamais payés) comme niveau de référence mai–octobre.
- `Purchase Enabled` suit assez bien Polar sur mai–octobre (25 contre 26 nouveaux acheteurs). Mon inquiétude d'un décalage fort de mesure est donc moins grande ; `Purchase Enabled` reste une activation de licence, et Polar reste la référence.

### 1.4 Polar (depuis le 5 mai 2026)

- 88 commandes, 23 clients : 25 abonnements démarrés (19 hebdo, 5 mensuel, 1 annuel) et 1 achat lifetime.
- Nouveaux acheteurs par mois : mai 6, juin 9, juillet 2, août 4, septembre 2, octobre 3. Ces nombres comptent les démarrages d'essai hebdo : 4 essais n'ont jamais payé, donc 22 acheteurs ont payé au moins une fois sur 26 (environ 6 pour 1 000 actifs sur mai–octobre, au lieu de 7).
- **Essai hebdo** (confirmé par toi : il y a eu un essai sur l'hebdo). Dans Polar, les abonnements hebdo avec une première commande à 0 sont les essais : 10 démarrages entre le 5 mai et le 16 juillet, aucun depuis. Résultat de ces 10 essais : **4 n'ont jamais payé, 3 ont payé une semaine, 3 en ont payé deux, aucun au-delà, aucun encore actif.** Soit 6 sur 10 passés au payant, puis un départ rapide.
- **Hebdo démarré directement** (9 abonnements, juin–octobre) : 0 jamais payé, 1 a payé une semaine, 4 en ont payé deux, 4 en ont payé trois ou plus (dont 3 encore actifs : 23, 11 et 4 paiements à ce jour). La rétention est nettement meilleure sans essai.
- Volume de payants hebdo : 8 abonnements ayant payé au moins une fois pendant les 10 semaines d'essai (5 mai – 16 juillet), 7 depuis, en 15 semaines (environ 0,8 contre 0,5 par semaine). Les actifs ont aussi baissé de ~25 % entre les deux périodes, donc je ne peux pas séparer l'effet de l'arrêt de l'essai de celui du volume.
- Mensuel : sur 5 abonnements, 3 ont payé 2 mois, 1 cinq, 1 six.
- Pays des commandes : États-Unis 47 %, Canada 13 %.
- Checkouts créés depuis avril (plugin et web app) : hebdo 113 (19 réussis), mensuel 25 (5), lifetime 28 (1), annuel 10 (1). Le site crée les quatre plans à chaque affichage de la page pricing : 51 checkouts depuis le 10 octobre, aucun réussi.
- Avant mai 2026, les licences passaient par d'autres canaux (Lemon Squeezy, in-app Figma) non accessibles ici. Les chiffres 2025 reposent sur `Purchase Enabled`.

### 1.5 Leviers d'achat mesurés plus tôt (avec `Purchase Enabled`, à confirmer avec Polar quand l'historique le permettra)

- **Le clic sur un plan est le meilleur signal d'intention** : 23 % (`GO_TO_CHECKOUT`, 2025) et 20 % (`GO_TO_PRO_WEEK`, 2026), contre ~3 % pour un affichage du pricing. L'ouverture volontaire du pricing donne 22 % en 5.3.6.
- **Première session** : 9 acheteurs sur 14 des versions 5.3.3 à 5.3.5 ont acheté le jour de leur première activité.
- **Remises d'accueil et Black Friday (5.1.0)** : conversion passée de 8 à 13 pour 1 000.
- **Les crédits (5.1.2, 5.1.6, 5.2.2) n'avaient pas fait baisser la conversion.**
- **Avant l'achat**, les acheteurs voient le pricing en médiane 3,5 fois.
- **Exports** : bon levier en 2025 (5,7 %), aucun achat en 2026 : à re-vérifier.
- **Fonctions les plus liées à l'achat** (sur 30 jours, référence 0,54 %) : génération (1,4–2,9 %), accessibilité APCA/WCAG (1,7–3,1 %), espaces colorimétriques (3,4 %), simulation de vision (3,1 %), presets design systems (2,5 %), échelles custom, easing, mode contraste (2,0–2,4 %). Ce sont des utilisateurs engagés : bons candidats à un blocage partiel, sans preuve que le blocage fait acheter.
- **Création de palette, copie d'un hex, sync styles/variables** : pas des leviers (0,4–1,0 %).

## 2. Objectifs et paliers

Objectif : **5 % des actifs qui paient, avec ~20 % de conversion chez les exposés**, donc exposer ~25 % des actifs (aujourd'hui 35 à 46 %).

Rappels : le meilleur niveau mensuel est ~12 pour 1 000 (1,2 %) ; 5 % est quatre fois plus. Le meilleur taux par exposé était 9,5 % (mars 2026, à 12 % d'exposition). Atteindre 20 % demande de multiplier par plus de deux ce meilleur taux, avec deux fois plus d'exposition.

| Scénario (par mois)       | Actifs   | Conversion des actifs | Acheteurs |
| ------------------------- | -------- | --------------------- | --------- |
| Aujourd'hui               | ~500–600 | ~0,7 %                | ~4        |
| Conversion ×2 (~1,4 %)    | ~600     | 1,4 %                 | ~8        |
| Volume ×2 seul            | ~1 200   | 0,7 %                 | ~8        |
| Volume ×2 + conversion ×2 | ~1 200   | 1,4 %                 | ~17       |
| Volume ×2 + conversion ×4 | ~1 200   | 2,7 %                 | ~32       |
| Volume ×2 + 5 %           | ~1 200   | 5 %                   | ~60       |

Paliers :

1. **Conversion ×2** (~1,4 %) : revenir au niveau historique en remplaçant le bruit par des blocages qualifiés. Peu risqué.
2. **Conversion ×4** (~2,7 %) : demande un mécanisme nouveau (offre d'accueil, checkout allégé ; un essai a déjà été testé sur l'hebdo, voir 1.4 : il a mal retenu les abonnés).
3. **5 %** : horizon.

Volume : doubler les actifs ramène à peu près au niveau de janvier. Les nouveaux utilisateurs pèsent plus que les actifs existants (achats dès la première session).

## 3. Spécification 5.3.7

### 3.1 Politique de paywall

- **Trois types de moments** :
  - _Qualifié_ (après une valeur obtenue : génération réalisée, crédits d'essai épuisés, export demandé) : modal avec le plan hebdo en premier, ouvrant directement le checkout, lien « voir tous les plans ».
  - _Volontaire_ (bouton Pricing) : page complète.
  - _Précoce_ (simulation, palettes locales, étapes, avant tout essai) : remplacer le modal par un essai par crédits, ou un aperçu en ligne avec badge Pro.
- **Fréquence** : au plus un modal par jour et par utilisateur ; les occurrences suivantes sont un badge discret. En 5.3.6, `SIMULATE_PALETTE` et `LOCAL_PALETTES` produisent 63 % des affichages pour 3 à 5 % de clics.
- **Cible d'exposition 5.3.7** : sous 40 % des actifs.
- `Origin` toujours renseigné.

### 3.2 Matrice libre / bloqué

| Famille                          | Gratuit                                                  | Bloqué (plan hebdo)                                             |
| -------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------- |
| Palettes                         | 2 palettes (limite actuelle), édition libre              | au-delà — **ne pas durcir**                                     |
| Couleurs / étapes                | à décider ; étapes relevées à 8 envisagé                 | au-delà                                                         |
| Génération (`GENERATE_*`)        | essai par crédits                                        | au-delà des crédits                                             |
| Accessibilité                    | contraste de base                                        | APCA / WCAG, intervalles, rapport de contraste                  |
| Presets design systems           | Tailwind, Material 3                                     | les autres ; presets custom inclus dans le plan                 |
| Espaces colorimétriques          | 1–2 standards                                            | espaces avancés                                                 |
| Simulation de vision             | 1–2 modes courants ; essais par crédits                  | autres modes ; sélection en crédits à coût plafonné             |
| Échelles                         | 1–10 / 10–100                                            | 100–1000, courbes chroma/hue (scale et couleurs), easing avancé |
| Couleurs sources                 | `SHIFT_CHROMA`, `SHIFT_HUE`, `UPDATE_HEX`, `ADD_COLOR`   | `UPDATE_LCH`, courbes sur couleur, `LOCK_SOURCE_COLORS`         |
| Copie de valeurs                 | copie d'un hex unique                                    | copie groupée (palette entière)                                 |
| Exports                          | export Tailwind limité à la palette gratuite (à décider) | autres formats (à re-vérifier)                                  |
| Sync styles / variables / tokens | crédits (3.3)                                            | au-delà des crédits                                             |
| Imports, harmonies               | libre                                                    | —                                                               |

### 3.3 Crédits comptés par résultat

- **Coût** : 1 crédit par variable ou style créé ou modifié, par couleur générée dans un document, par élément modifié en simulation de palette. Ne facturer que ce qui est créé ou modifié (pas les réécritures identiques).
- **Simulation sur sélection** : coût plafonné ou forfaitaire par action (la taille de la sélection n'est pas maîtrisée).
- **Dotation** : gratuit, quelques dizaines à 200 crédits (5 à 10 essais) ; plan payant : 1 000 crédits par mois (prorata pour l'hebdo, par exemple 250 par semaine). À 1 000 crédits gratuits, les quelques dizaines d'utilisateurs qui génèrent ne rencontreraient jamais la limite.
- **Ordre de grandeur** : une palette moyenne = 3,64 couleurs × 6 étapes ≈ 22 éléments (29 avec 8 étapes) → 1 000 crédits ≈ 34 à 45 syncs ; le plus gros utilisateur de sync du dernier mois en a fait 20.
- **Visibilité** : afficher le coût estimé avant l'action et le solde restant.
- **IA (Mistral)** : tokens convertis en crédits, dotation gratuite plafonnée par compte pour borner le coût. Sa conversion en achat est faible (0,5 %) : un essai limité plutôt qu'un levier de blocage. Même mécanisme plus tard pour la structuration de systèmes.

### 3.4 Plans et checkout

- Plan hebdo en entrée (4 clics sur 12 en 5.3.6 ; 17 % des checkouts aboutissent).
- Sortir le lifetime de l'écran principal (1 réussi sur 28).
- Vérifier le texte et la position du bouton Ultimate : il est le premier plan cliqué depuis les blocages (6 sur 12 en 5.3.6).
- Hypothèses à tester : permettre dans le plugin l'email au checkout puis la création de compte (comme sur le site) ; deux plans visibles (hebdo, annuel ancré) et les autres derrière un lien.
- L'essai hebdo (mai–juillet) a mal retenu : 4 essais sur 10 jamais payés, aucun payant au-delà de 2 semaines. Ne pas le réactiver tel quel ; si un essai revient (5.3.8), le coupler à des crédits et à un rappel avant la fin, et suivre le nombre de semaines payées, pas le démarrage.

### 3.5 Offre d'accueil

- Offre limitée dans le temps (24 à 48 h) déclenchée après la première valeur obtenue, pas à l'ouverture. Durée et prix annoncés exacts, sans fausse urgence.
- Le test à 2,99 $ par semaine se lit avec Polar, sans le mélanger avec d'autres changements.

### 3.6 Rétention des abonnés

- Avec l'essai, aucun abonné hebdo n'a dépassé 2 paiements ; sans essai, 4 sur 9 ont payé 3 semaines ou plus (3 encore actifs). Pistes à tester : proposer le passage au mensuel ou à l'annuel (remise) à la 2ᵉ semaine ; dotation de crédits qui se renouvelle pour justifier la poursuite ; récapitulatif de ce que l'abonné a produit.
- Suivre le revenu et la rétention à 4 semaines, pas seulement le nombre d'achats.

### 3.7 Relance

- Au retour dans le plugin : rappeler ce qui a été fait (crédits utilisés, palettes créées) et proposer le plan, plutôt que répéter le même mur (3,5 vues avant achat en médiane).
- Email Resend aux comptes sans achat (jours 0, 2, 5), consentement acquis. Relance des checkouts abandonnés avec email : à vérifier, la plupart des checkouts expirés du site n'ont pas d'email.

### 3.8 Site

- Créer le checkout au clic sur un plan, pas à l'affichage de la page (aujourd'hui quatre checkouts par affichage), et envoyer un événement Mixpanel au clic.

### 3.9 Instrumentation

1. `Units` (éléments produits) sur `SYNC_*`, `GENERATE_*`, `SIMULATE_*`.
2. Plan et prix sur `Pricing Consulted` (la propriété `Plan` existe : vérifier qu'elle est renseignée).
3. Événement de début de checkout avec plan et `Origin`.
4. `external_customer_id` renseigné dans Polar avec l'id du compte, pour relier achat et utilisateur du plugin.
5. Valeur choisie pour l'espace colorimétrique et le mode de simulation.
6. Tableau de bord Polar : checkouts par origine (plugin, web app, site), plan, statut, avec email.

## 4. Après la 5.3.7

- **5.3.8** : offre d'accueil, checkout allégé (un essai seulement avec les garde-fous de 3.4), plans réordonnés, relance email ; exposition vers 25–30 %.
- **Plus tard** : crédits Mistral, structuration de systèmes.

## 5. Mesure et règles de décision

- **KPI principaux** : acheteurs réels (Polar, checkouts réussis avec email) pour 1 000 actifs par mois ; part d'actifs exposés au pricing ; conversion des exposés ; rétention à 4 semaines des abonnés.
- **Références de départ** : 7 pour 1 000 actifs (mai–octobre), 35 % d'exposition (46 % en octobre), ~2 % de conversion des exposés, 4 essais hebdo sur 10 jamais payés (mai–juillet), 4 abonnés hebdo directs sur 9 à 3 paiements ou plus.
- **Lecture** : avec ~50 nouveaux exposés par semaine, un A/B ne détecte que de gros effets (1,2 % → 5 % demande ~325 exposés par bras, ~13 semaines). Changer par étapes et lire d'abord les taux d'ouverture par origine (centaines d'affichages), puis les achats en tendance sur 4 semaines. Un effet de 1,2 % à 20 % se verrait vite, mais seulement s'il existe.
- **Règle de décision à 4 semaines de 5.3.7** : si l'exposition est passée sous 40 % et que la conversion des exposés a au moins doublé, poursuivre vers la 5.3.8. Si l'exposition baisse mais que les ventes baissent aussi, desserrer le blocage le plus large en premier.
- **Saison** : comparer octobre–décembre à octobre–décembre 2025 (~10 pour 1 000), pas à l'été.

## 6. Réserves

- Peu d'achats par mois (1 à 15) : toutes les comparaisons mensuelles sont bruitées.
- Mixpanel ne compte que les utilisateurs qui ont accepté le suivi ; Polar compte tous les achats.
- Les taux par fonction (section 1.5) viennent de `Purchase Enabled` et d'utilisateurs engagés ; ce sont des indices, pas des preuves de causalité.
- L'effet d'une fonction bloquée ne se mesure pas par son usage : un clic sur une fonction bloquée ne déclenche pas d'événement d'usage.
- Pas de données sur le revenu (devises mélangées) ni sur la source du trafic.
