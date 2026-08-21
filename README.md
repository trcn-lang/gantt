# Outil de planification RH — Grist

Outil de planification et de suivi d'activité pour un bureau RH (35 agents / 4 équipes), réservé aux chefs d'équipe, chef de bureau et adjoints (~8 utilisateurs). Charte graphique DSFR.

Composé de **3 widgets custom** branchés sur les mêmes tables Grist :

| Widget | Fichier | Rôle |
|---|---|---|
| **Rétroplanning** | `ganttv8` | Planification multi-mois : Gantt, dépendances, cascade |
| **Kanban par Porteur** | `kanbanv7` | Suivi quotidien de l'avancement, par personne |
| **Récap — Vue synthétique A4** | `Gantt` (Récap) | Reporting imprimable, lecture seule |

---

## 1. Modèle de données (tables Grist)

| Table | Rôle | Colonnes clés |
|---|---|---|
| **Projets** | Un projet RH (ex: "Mobilité SVT S1 2026") | `Projet` (nom), `Equipe`, `Type_Projet`, `Description` |
| **Taches** | Tâches du planning | `Tache`, `Projet`, `Equipe`, `Date_debut`, `Date_fin`, `Duree_jours`, `Statut`, `Porteur`, `Predecesseurs` (liste réf.), `Jalon_declencheur` (liste réf.), `Incompressible` (bool), `Commentaires` |
| **Jalons** | Étapes/échéances du projet | `Nom`, `Projet`, `Date`, `Type` (Livrable / Pour information), `Important` (bool — voir §5.2), `Taches_requises` (liste réf.) |
| **ACTIONS** | Sous-actions d'une tâche | `Tache` (réf.), `Nom`, `Statut`, `Porteur`, `Commentaires` |
| **Porteurs** | Référentiel des agents | `Nom` (inclut un porteur système `Non attribué`) |
| **PARAMETRES** | Listes de référence | `Statut`, `Equipe`, `Type_Projet` |

> ⚠️ Le champ `Important` sur Jalons a été renommé **fonctionnellement** en **"Fixe"** dans l'interface (voir §5.2) ; le nom de colonne Grist reste `Important` en base.

---

## 2. Rétroplanning (`ganttv8`)

### 2.1 Vue Gantt

- Une ligne par projet (regroupement), une ligne par tâche à l'intérieur.
- Barres de tâches colorées selon le statut (À faire / En cours / Terminé).
- Losanges pour les jalons, positionnés à leur date, colorés selon leur type (§5.1‑5.2).
- Flèches SVG entre tâches liées (prédécesseurs) et entre tâches et jalons (tâches requises / jalon déclencheur).
- **Zoom** : boutons zoom avant/arrière, réinitialisation (100 %), et "Vue d'ensemble" (ajuste l'échelle pour voir tout le projet).
- **Repli/dépli** d'un projet (masque ses tâches et jalons sans les supprimer de la plage de dates affichée).
- Impression / lecture desktop uniquement (pas de mode mobile dédié).

### 2.2 Gestion des Projets

- Création / édition / suppression d'un projet (Nom, Équipe, Type de projet, Description).
- Boutons rapides sur chaque ligne projet pour ajouter directement une Tâche ou un Jalon.

### 2.3 Gestion des Jalons

- Carte de création/édition : Nom, Date, Type (**Livrable** / **Pour information**).
- Si Type = Livrable :
  - Case **☐ Fixe** (ancrage dur — voir §5.2).
  - Sélecteur multi **Tâches requises** (tâches dont dépend l'atteinte du jalon).
- Détection anti-cycle : impossible de sélectionner une tâche requise qui dépendrait elle-même (directement ou indirectement) de ce jalon.

### 2.4 Gestion des Tâches

- Carte de création/édition : Nom, Date début, Date fin / Durée, case **☐ Incompressible**, Porteur, Statut, Prédécesseurs, Jalon(s) déclencheur(s), Commentaires, sous-Actions.
- **Calcul de date automatique en temps réel**, sans mode à sélectionner (§5.3) :
  - Si vous modifiez la **Durée**, la Date fin se recalcule.
  - Si vous modifiez la **Date fin**, la Durée se recalcule.
  - Les jours ouvrés (samedi/dimanche + jours fériés français) sont exclus du calcul.
- Détection anti-cycle sur les prédécesseurs et les jalons déclencheurs.
- Sous-actions : liste de tâches secondaires attachées à la tâche (nom, statut, porteur, commentaires), gérées dans la même carte.

### 2.5 Cascade automatique ("décalage en cascade")

Déclenchée **automatiquement à l'enregistrement** d'une tâche ou d'un jalon, si sa date de fin (ou sa date, pour un jalon) a changé. Une **popup non bloquante** s'affiche avant toute écriture définitive, avec deux sections (voir §5.4) :

- **⚠️ Conflits détectés** : jalons/tâches dont la contrainte de date est désormais violée (purement informatif, aucune action automatique).
- **Propagation** : ce qui va être décalé (ou absorbé) en aval, avec possibilité d'**Annuler** (la tâche/jalon reste sauvegardé, seule la cascade est abandonnée) ou de **Confirmer**.

### 2.6 Retard structurel (§5.5)

Icône ⚠️ ("Met en retard : …") affichée sur le nom de chaque tâche/jalon en amont d'un blocage. En aval, l'élément bloqué affiche un **contour rouge directement sur son marqueur** (barre de tâche ou diamant de jalon) plutôt qu'une icône séparée, pour ne jamais créer de second point visuel ambigu sur la timeline — infobulle détaillée au survol.

---

## 3. Kanban par Porteur (`kanbanv7`)

- Grille **Porteur (ligne) × Statut (colonne)** : À faire / En cours / Terminé.
- **Glisser-déposer** d'une carte tâche pour changer son statut et/ou son porteur en un geste.
- Filtres : par Équipe, par Projet.
- Carte tâche affiche :
  - Badge projet.
  - **Badge d'échéance** : "⏰ En retard depuis X j" (jours ouvrés, date du jour dépassée) ou "Fin dans X j".
  - Ligne 🔴 "Bloqué par : …" si un prédécesseur n'est pas encore terminé.
  - Ligne 🔶 "En attente de : …" si un jalon déclencheur n'est pas encore atteint.
  - Ligne 🔶 "Conditionne : …" si la tâche est elle-même requise par un jalon non atteint.
  - Ligne 🟡 "Dépend de : …" (liste des prédécesseurs).
  - **Ligne 🔴 "En retard (bloquée par …)"** et **⚠️ "Met en retard : …"** — retard structurel (§5.5).
  - Commentaire (tronqué), indicateurs de sous-actions (☐ à faire / ☑ terminées).
- Même carte de création/édition de tâche que le Rétroplanning (Mode Durée/Date fixe temps réel, Incompressible, cascade automatique avec popup identique).
- Gestion des sous-actions directement depuis la carte tâche.

---

## 4. Récap — Vue synthétique A4 (`Gantt`)

Widget de **reporting imprimable**, en lecture seule (aucune création/édition — clic = simple affichage).

- Bouton **🖨️ Imprimer** (`window.print()`), mise en page pensée pour l'A4.
- Filtres : Projet, Plage temporelle (tous les jalons / 12 / 26 / 52 semaines), Type de jalon affiché (Livrable / Pour information), case "Mettre en avant les tâches en retard".
- Timeline segmentée par plage temporelle, avec jalons positionnés et marqueur "Aujourd'hui".
- Marqueurs rouges (losange bordé) sur les tâches en retard si l'option est cochée (§5.5 réutilise `getStatutTache`).
- **Tableau "Tâches à suivre"** : liste triée par date de fin, avec statut calculé (`getStatutTache` : À faire / En cours / Terminé / **En retard**) et annotation des retards structurels (⚠️ / 🔴) en texte à côté du nom de la tâche.
- Légende complète des couleurs et symboles utilisés.

> ⚠️ **Point connu** : le filtre "Jalon majeur" affiché dans la barre d'outils est un reliquat de l'ancien modèle (type de jalon supprimé lors de la fusion Majeur/Livrable). Il n'a plus aucun effet puisqu'aucun jalon ne porte plus ce type — à retirer de l'interface.

---

## 5. Mécanismes transverses (communs aux 3 widgets)

### 5.1 Jalons : Livrable vs Pour information

- **Pour information** : simple repère chronologique, aucune logique de calcul.
- **Livrable** : porte une liste de Tâches requises ; son statut se déduit automatiquement de l'avancement de ces tâches (Terminé / En cours / À faire / En retard) via `getStatutJalon` (Récap) ou l'équivalent dans le Rétroplanning.

### 5.2 Jalon "Fixe" (ancrage dur)

- Case à cocher, visible uniquement pour un jalon de type Livrable.
- **Fixe** = losange **rouge** (reprend le code couleur de l'ancien "Jalon majeur", supprimé du modèle).
- **Non-fixe** = losange **orange** (couleur standard Livrable).
- Un jalon Fixe **peut être déplacé manuellement** par un utilisateur (traduction métier : la hiérarchie a décalé l'échéance) — mais **ne bouge jamais automatiquement** via la cascade (§5.4) : c'est le point d'arrêt de toute propagation de retard.

### 5.3 Tâches : calcul de date sans mode à choisir

Aucun toggle "Durée" / "Date fixe" : le dernier champ modifié par l'utilisateur pilote le calcul (Date fin ↔ Durée), en temps réel (à chaque frappe), en jours ouvrés.

### 5.4 Cascade automatique — règles de propagation

| Élément en aval | Comportement |
|---|---|
| Tâche **compressible** (case Incompressible décochée) | Absorbe le retard : sa Date fin ne bouge pas, sa durée est réduite (minimum 1 jour). Point d'arrêt de la cascade sur cette branche. |
| Tâche **incompressible** | Décalée intégralement (durée conservée). La cascade continue vers ses propres successeurs. |
| Jalon **non-fixe** | Décalé à la date de fin de la tâche qui le déclenche + 1 jour ouvré. |
| Jalon **Fixe** | Ne bouge jamais. Signalé comme point de conflit. |

En parallèle, la popup détecte aussi les **conflits amont** : une tâche repoussée qui dépasse désormais la date d'un jalon dont elle est requise, ou un jalon avancé avant la fin d'une de ses tâches requises.

### 5.5 Retard structurel

Remplace l'ancienne notion de **chemin critique / tâche critique** (PERT), jugée trop théorique pour un usage de pilotage hebdomadaire (voir §6).

**Règle** : une tâche/jalon T2 est en retard structurel si l'un de ses prédécesseurs T1 (via `Predecesseurs` ou `Taches_requises`) a une `Date_fin` postérieure à la date de début de T2 (ou à la date du jalon).

- **T1** affiche : icône ⚠️ *"Met en retard : T2"* à côté de son nom.
- **T2** affiche un **contour rouge sur son propre marqueur** (barre de tâche dans le Gantt, ou anneau rouge sur le diamant dans le cas d'un jalon) — infobulle *"En retard (bloquée par T1)"* au survol.

> ⚠️ **Principe de design important** : le retard structurel ne doit **jamais** créer de second point visuel distinct du marqueur de date existant (barre / diamant). Une première version utilisait un émoji 🔴 préfixé au nom du jalon, positionné à côté du diamant — ça créait un point ambigu qui semblait marquer une autre date. Corrigé en appliquant le signal directement **sur** le marqueur réel (contour), pour garder un seul point d'ancrage par date, sans exception.

Calcul purement dérivé (aucune écriture en base), recalculé à chaque rendu (`construire()`), donc toujours à jour. Le calcul est fait paire par paire sur tout le graphe de dépendances : les chaînes de retard se révèlent naturellement, sans logique de propagation dédiée.

Ce badge structurel est **distinct** du badge d'échéance classique du Kanban ("En retard depuis X j"), qui lui compare simplement la date de fin à la date du jour.

---

## 6. Fonctionnalités retirées

### Calcul PERT (chemin critique)

Entièrement supprimé du code (les 3 widgets) : `calculatePERT()`, `Est_critique`, `Marge_totale`, `Chemin_critique`, `Date_au_plus_tôt`, `Date_au_plus_tard`.

**Raison** : concept théorique (marge, chemin critique) peu actionnable pour un pilotage hebdomadaire par des chefs d'équipe RH. Remplacé par la notion plus concrète de **retard structurel** (§5.5), directement lisible ("qui bloque qui") sans notion de théorie des graphes.

**Conséquence** : si ces colonnes existent encore dans les tables Grist, elles ne sont plus alimentées ni lues par aucun des 3 widgets — elles peuvent être supprimées du schéma sans casser l'outil.

### Fusion des types de Jalons

L'ancien type **"Jalon majeur"** a été fusionné dans **Livrable** + le flag **Important/Fixe** (§5.2). Les jalons créés avant cette migration doivent avoir été convertis manuellement (Type = Livrable, Important = Oui) — voir le point connu du §4 concernant le filtre résiduel du Récap.

### Mode "Durée / Date fixe" par toggle

Une première version avec un bouton bascule explicite dans la carte tâche a été remplacée par la détection automatique du §5.3 (aucune action requise de l'utilisateur).

### Bouton "Répercuter le retard"

Un bouton manuel dans la carte tâche a été remplacé par le déclenchement **automatique** de la cascade à chaque enregistrement (§5.4), jugé plus fiable qu'une action à ne pas oublier.

---

## 7. Colonnes Grist à vérifier / ajouter

| Table | Colonne | Type | Statut |
|---|---|---|---|
| Taches | `Incompressible` | Toggle (bool) | À ajouter si absente |
| Jalons | `Important` | Toggle (bool) | Doit déjà exister (ex-migration "Jalon majeur") |
| Taches | `Est_critique`, `Marge_totale`, `Chemin_critique`, `Date_au_plus_tot`, `Date_au_plus_tard` | — | **Obsolètes**, supprimables du schéma (§6) |

---

## 8. Bugs / améliorations connues (backlog UX mineur)

### Résolus
- ✅ **Bouton "📍 Aujourd'hui"** (Rétroplanning) : centre la vue sur la date du jour. A nécessité un correctif en 2 temps : (1) le bouton lui-même, réutilisant le mécanisme `ZOOM_CENTER_DAY` existant, puis (2) un vrai bug — la plage de dates affichée (`minI`/`maxI`) était calculée uniquement à partir des tâches/jalons existants, donc si "aujourd'hui" tombait hors de cette plage (projet déjà terminé, par ex.), le bouton n'avait littéralement aucun jour valide où se centrer. Corrigé en étendant systématiquement `minI`/`maxI` pour toujours inclure aujourd'hui (± 3 jours de marge).
- ✅ **Alignement checkbox/label** dans les listes multi-sélection (Prédécesseurs, Jalons déclencheurs, Tâches requises) : plusieurs couches de bug en cascade — spécificité CSS (`​.form-group label{display:block}` qui écrasait le flex des items), texte injecté en nœud brut au lieu d'un `<span>` dédié, et surtout une piste de grille CSS parente (`.modal-columns-wrapper` / `.modal-tache-body`) sans `min-width:0`, qui empêchait la colonne de se réduire correctement et cassait tout le calcul de largeur en aval. Résolu en réécrivant l'item en **CSS Grid** (`16px 1fr`, plus déterministe que flex) + `min-width:0`/`minmax(0,1fr)` sur les conteneurs parents.
- ✅ **Flèches SVG de dépendance orphelines** dans le Gantt : une flèche pouvait apparaître avec des coordonnées `NaN` (position introuvable) après une cascade, créant un trait déconnecté à l'écran. Corrigé par des guards (`isNaN(...)`) avant tout tracé de chemin SVG.

### Ouverts
- **Filtre "Jalon majeur"** résiduel dans le Récap A4 (§4) : à retirer de l'interface, sans effet depuis la migration des jalons.

### Point de vigilance pour la suite
Le bug du bouton "Aujourd'hui" illustre un piège récurrent dans ce code : **toute plage de dates calculée à partir des seules données métier (tâches/jalons) peut exclure "aujourd'hui" silencieusement**. À vérifier systématiquement pour toute nouvelle fonctionnalité qui dépend d'un repère "date du jour" (vue Calendrier notamment, cf. backlog §9).

---

## 9. Backlog fonctionnel restant (non codé)

Par ordre de priorité décroissante, discuté mais non implémenté à ce stade :

> 🎯 **Prochain chantier prévu** : point 5, **Duplication / Templates de projet récurrent** — voir le prompt de reprise associé à ce README pour le cadrage détaillé.

1. **Traçabilité des reports** — historiser les changements de date (qui, quand, motif) via une table `Historique` dédiée. Point le plus attendu, notamment pour tracer la causalité des cascades.
2. **Charge de travail par porteur** — tableau de bord détectant la surcharge (nombre de tâches "En cours" simultanées par agent).
3. **Vue Portefeuille** — page d'accueil résumant tous les projets en une ligne (statut global, % avancement, prochains jalons).
4. **Vue Calendrier** — représentation calendaire (hebdo/mensuelle) des jalons et tâches. ⚠️ Attention au piège décrit en §8 ("point de vigilance").
5. **Duplication / Templates de projet récurrent** — duplication d'un projet existant avec décalage global des dates (ex. "Mobilité S1" → "Mobilité S2"), remappage propre des références (prédécesseurs, jalons déclencheurs, tâches requises).
6. **Confort** — tags libres, recherche texte, export PDF natif (au-delà de `window.print()`), responsive mobile.
7. **Versions de livrables (V1/V2/V3)** — le modèle actuel (jalon = date figée) ne capture pas les livrables versionnés ; nécessite une réflexion de modélisation dédiée avant tout code.

---

*Document généré à partir de l'inspection du code source des 3 widgets (`ganttv8`, `kanbanv7`, `Gantt`).*
