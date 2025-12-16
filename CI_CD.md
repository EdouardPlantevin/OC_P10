# 📄 Audit de la CI/CD et de la Qualité - BobApp

Ce document détaille l'infrastructure d'Intégration et de Déploiement Continus (CI/CD) mise en place pour le projet **BobApp**, les outils sélectionnés, les indicateurs de performance (KPIs) définis, ainsi qu'une analyse de la qualité actuelle du projet.

---

## 1. Stack Technique et Outils Utilisés

Pour industrialiser le cycle de vie de l'application, nous avons sélectionné la suite d'outils suivante. Chaque outil répond à un besoin précis du processus DevOps :

| Outil | Type | Utilité dans le projet |
| :--- | :--- | :--- |
| **GitHub Actions** | Orchestrateur CI/CD | Automatise le lancement des tâches (tests, analyse, build) à chaque modification du code. Il remplace les actions manuelles et garantit la répétabilité. |
| **Maven** | Build Automation (Back) | Gère les dépendances Java, compile le code Backend et exécute les tests unitaires (`mvn verify`). |
| **npm / Angular CLI** | Build Automation (Front) | Gère les dépendances JavaScript, compile l'application Frontend et lance les tests via Karma/ChromeHeadless. |
| **SonarCloud** | Analyseur de Qualité | Scanne le code statique pour détecter les bugs, failles de sécurité et "Code Smells". Il centralise les rapports de couverture de code. |
| **Docker** | Conteneurisation | Encapsule l'application (Back et Front) dans des images légères et portables, prêtes pour la production. |
| **Docker Hub** | Registre d'images | Stocke de manière sécurisée les images Docker versionnées, accessibles pour le déploiement serveur. |

---

## 2. Description du Workflow CI/CD

Le pipeline est défini dans le fichier `.github/workflows/main.yml`. Il est entièrement automatisé mais permet aussi un déclenchement manuel en cas d'incident.

### Déclencheurs (Triggers)
Le workflow se lance automatiquement dans 3 cas :
1.  **Push** sur la branche `main` (Intégration Continue).
2.  **Pull Request** vers la branche `main` (Vérification avant fusion).
3.  **Création d'un Tag** (ex: `v1.0.0`) (Déclenchement de la Livraison/Release).

### Les Étapes du Pipeline (Jobs)

Le processus suit une logique séquentielle stricte : **Tester ➔ Analyser ➔ Livrer.**

#### 🔹 Étape 1 : Tests Automatisés (Parallélisés)
* **Job Backend :** Installation de Java 11, compilation du projet Spring Boot et exécution des tests unitaires. Génération du rapport de couverture **JaCoCo**.
* **Job Frontend :** Installation de Node.js 16, installation des modules et exécution des tests Angular dans un navigateur virtuel (ChromeHeadless). Génération du rapport **LCOV**.

#### 🔹 Étape 2 : Contrôle Qualité (`sonar`)
* **Condition :** Ne démarre que si les tests Back et Front sont validés (Verts).
* **Action :** GitHub envoie le code et les rapports de couverture à SonarCloud.
* **Quality Gate :** SonarCloud vérifie si le code respecte les KPIs définis. Si la qualité est insuffisante (Bug critique ou couverture trop faible), le pipeline s'arrête ici (Échec).

#### 🔹 Étape 3 : Livraison Continue (`docker`)
* **Condition :** Ne s'exécute que si la Quality Gate est validée **ET** que l'événement est un **Tag** (version release).
* **Action :** Construction des images Docker `bobapp-back` et `bobapp-front`. Les images sont taguées avec le numéro de version (ex: `v1.0.0`) et poussées sur Docker Hub.

---

## 3. KPIs et Quality Gate

Pour garantir la maintenabilité future de l'application et stopper l'introduction de dette technique, nous avons configuré des seuils stricts dans SonarCloud.

Ces KPIs s'appliquent obligatoirement sur le **Nouveau Code** (Clean as You Code) :

1.  **Couverture de Code (Code Coverage) :** **Min. 80%**
    * *Objectif :* Ce seuil (supérieur aux 70% requis) assure que toute nouvelle fonctionnalité est testée unitairement pour éviter les régressions.
2.  **Fiabilité (Reliability) :** **Note A (0 Nouveau Bug)**
    * *Objectif :* Bloquer le déploiement de tout code contenant des bugs logiques ou crashs potentiels.

---

## 4. Analyse des Métriques et de la Qualité

Suite à la première exécution du pipeline sur le code existant, voici l'audit technique de BobApp :

| Métrique SonarCloud | Résultat Actuel | Analyse |
| :--- | :--- | :--- |
| **Fiabilité** | **Note D (1 Bug)** | 🔴 **Critique.** Un bug majeur de logique a été détecté dans le service des blagues. |
| **Couverture** | **16.7%** | 🔴 **Insuffisant.** Très loin du seuil de 80%. Le backend manque de tests unitaires sur la couche service. |
| **Maintenabilité** | **Note A (11 Code Smells)** | 🟡 Correct, mais quelques nettoyages de code sont nécessaires (variables inutilisées, nommage). |
| **Sécurité** | **Note A** | 🟢 Excellent. Aucune faille de sécurité détectée. |

---

## 5. Analyse des Retours Utilisateurs et Recommandations

Nous avons croisé les métriques techniques avec les retours des utilisateurs ("Notes et avis") pour identifier les priorités.

### Problème n°1 : "Je tombe toujours sur la même blague !"
* **Analyse Technique :** SonarCloud a levé une alerte rouge sur `JokeService.java` : *"Save and re-use this Random"*. L'objet générateur de nombres aléatoires est récréé à chaque appel. Lors de pics de trafic, plusieurs utilisateurs génèrent la même "graine" temporelle et reçoivent donc la même blague.
* **Action Requise (Priorité Haute) :** Correctif (Hotfix) immédiat en passant l'objet `Random` en variable statique de classe.

### Problème n°2 : "L'application est parfois lente"
* **Analyse Technique :** La réinstanciation systématique d'objets lourds (détectée par l'analyse statique) surcharge le Garbage Collector de Java et ralentit le serveur.
* **Action Requise :** Le correctif du problème n°1 améliorera également les performances.

### Problème n°3 : Risque de régression
* **Analyse Technique :** Avec seulement **16.7%** de couverture, toute modification du code risque de casser une fonctionnalité existante sans qu'on s'en aperçoive.
* **Action Requise :** Mise en place d'une campagne de tests unitaires (JUnit) pour remonter progressivement la couverture vers les 80% exigés par le KPI.

#modif