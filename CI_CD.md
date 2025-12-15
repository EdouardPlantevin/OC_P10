# Audit Qualité & Documentation CI/CD

Ce document formalise le processus d'intégration et de déploiement continus (CI/CD) mis en place pour le projet BobApp, présente les indicateurs de performance (KPIs) choisis, et dresse un bilan de la qualité actuelle du code suite à la première analyse automatisée.

---

## 1. Description du Workflow CI/CD

Nous avons automatisé le cycle de vie de l'application via **GitHub Actions**. Le pipeline se déclenche à chaque *push* ou *pull request* sur la branche principale (`main`) et se compose de 4 étapes séquentielles.

### 🔹 Étape 1 : Validation Backend (`backend-test`)
* **Objectif :** Garantir que le code Java compile et que les tests passent avant toute intégration.
* **Détails techniques :**
    * Environnement : Java 11 (Distribution Temurin).
    * Action : Compilation via Maven (`mvn clean verify`).
    * Résultat : Génération du rapport de couverture de code **JaCoCo**.

### 🔹 Étape 2 : Validation Frontend (`frontend-test`)
* **Objectif :** Vérifier la stabilité de l'interface Angular sans régression.
* **Détails techniques :**
    * Environnement : Node.js 16.
    * Action : Installation des dépendances (`npm ci`) et exécution des tests via **Karma** avec un navigateur "Headless" (ChromeHeadless).
    * Résultat : Génération du rapport de couverture **LCOV**.

### 🔹 Étape 3 : Analyse Qualité (`sonar`)
* **Objectif :** Centraliser les métriques et bloquer le code non conforme.
* **Condition :** Ne s'exécute que si les tests Back et Front sont validés.
* **Détails techniques :**
    * Analyse statique du code via **SonarCloud**.
    * Importation des rapports de couverture (JaCoCo & LCOV) générés aux étapes précédentes.
    * Vérification des critères de qualité (Quality Gate).

### 🔹 Étape 4 : Livraison (`docker`)
* **Objectif :** Mettre à disposition les nouvelles versions de l'application.
* **Condition :** Ne s'exécute que si la Quality Gate Sonar est verte.
* **Détails techniques :**
    * Construction des images Docker (Backend & Frontend).
    * Publication (Push) sur le registre **Docker Hub**.

---

## 2. KPIs Proposés (Quality Gate)

Afin de maîtriser la dette technique, nous avons défini des seuils stricts (KPIs) sur **SonarCloud**. Nous appliquons la stratégie du *"Clean as You Code"* : ces règles s'imposent obligatoirement sur tout **nouveau code** ajouté.

| KPI (Indicateur) | Seuil (New Code) | Justification |
| :--- | :--- | :--- |
| **Couverture de Code** | **> 80%** | **(Obligatoire)** Assure que toute nouvelle fonctionnalité est testée pour éviter les régressions futures. |
| **Fiabilité** | **Note A (0 Bug)** | Aucun bug critique n'est toléré en production. |
| **Sécurité** | **Note A (0 Vulnérabilité)** | Protection des données utilisateurs et de l'intégrité du système. |
| **Maintenabilité** | **Note A** | Le code doit rester lisible et respecter les standards pour faciliter le travail de l'équipe. |

---

## 3. Analyse des Métriques et Retours Utilisateurs

Suite à la première exécution complète du pipeline, voici l'état des lieux de l'application existante.

### 📊 Bilan des Métriques (SonarCloud)

| Métrique | Résultat Actuel | Analyse |
| :--- | :--- | :--- |
| **Fiabilité** | **Note D (1 Bug)** | 🔴 **Critique.** Un défaut majeur de logique a été détecté. |
| **Sécurité** | **Note A** | 🟢 Code sain, aucune faille détectée. |
| **Maintenabilité** | **Note A (11 Code Smells)** | 🟡 Globalement propre, quelques nettoyages mineurs à prévoir. |
| **Couverture** | **16.7%** | 🔴 **Insuffisant.** Loin du seuil de 80%. Le Backend manque cruellement de tests unitaires. |
| **Duplications** | **0.0%** | 🟢 Excellent, pas de code dupliqué. |

### 🔍 Corrélation avec les Retours Utilisateurs

L'analyse technique explique parfaitement les dysfonctionnements signalés par les utilisateurs :

> **Retour Utilisateur :** *"Je tombe toujours sur la même blague !"*

* **Cause Technique identifiée :** SonarCloud a détecté une erreur critique dans `JokeService.java` ("Save and re-use this Random").
* **Explication :** L'objet `Random` est instancié *à l'intérieur* de la méthode. Lors de pics de trafic, plusieurs utilisateurs appellent la méthode à la même milliseconde, générant la même "graine" aléatoire et donc la même blague.

> **Retour Utilisateur :** *"L'application est parfois lente."*

* **Cause Technique identifiée :** La ré-instanciation inutile d'objets lourds (comme le `Random` ou le `ObjectMapper` dans `JsonReader`) à chaque requête surcharge la mémoire et le processeur (Garbage Collection).

---

## 4. Recommandations et Plan d'Action

Pour stabiliser BobApp, nous recommandons le plan d'action suivant, classé par priorité :

### 🥇 Priorité 1 : Hotfix Immédiat (Fiabilité)
**Problème :** Bug du `Random` dans `JokeService.java`.
**Action :** Refactoriser la classe pour déclarer l'objet `Random` en tant que constante de classe (`static final`).
**Gain :** Résolution immédiate du problème de redondance des blagues et amélioration des performances.

### 🥈 Priorité 2 : Sécurisation (Couverture)
**Problème :** Couverture de 16.7% trop faible.
**Action :** Créer des tests unitaires JUnit sur le `JokeService` (actuellement non testé).
**Objectif :** Valider que le correctif du Hotfix fonctionne et augmenter le coverage vers les 80%.

### 🥉 Priorité 3 : Nettoyage (Maintenabilité)
**Problème :** 11 "Code Smells" identifiés.
**Action :**
1.  **Frontend :** Ajouter le modificateur `readonly` sur les injections de dépendances (ex: `jokes.service.ts`).
2.  **Backend :** Renommer le champ `joke` dans le modèle `Joke.java` (confus) en `content` ou `text`.
**Gain :** Base de code saine et professionnelle ("Clean Code").