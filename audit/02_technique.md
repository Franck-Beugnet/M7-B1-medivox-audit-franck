# Volet 2 — Audit Technique, Architecture & Sécurité (MediVox)

> **Lectorats cibles** : 👩‍💻 Hélène Tournier (Directrice Technique) en priorité · ⚖️ Marc Lebourg (DPO).  
> **Composants audités** : [legacy/train.py](legacy/train.py), [legacy/predict.py](legacy/predict.py), [legacy/dms_predictor_v1.joblib](legacy/dms_predictor_v1.joblib), flux de données et processus de déploiement.

---

## 1. Architecture Logicielle & Dette Technique

### 1.1 Non-modularité & Couplage fort
* **Absence de pipeline de traitement** : Le script [legacy/train.py](legacy/train.py#L14-L18) réalise des transformations manuelles et ad-hoc (`df["sexe"] == "M"`). Aucun `Pipeline` ou `ColumnTransformer` scikit-learn n'est utilisé. Le prétraitement est dissocié du modèle sérialisé.
* **Duplication et divergence du prétraitement** : Dans [legacy/predict.py](legacy/predict.py#L13-L17), l'encodage du sexe n'est plus recalculé : le script attend directement un entier `0` ou `1` via `sys.argv[4]`. L'appelant doit donc connaître et implémenter la logique interne du modèle.
* **Absence de validation des entrées (Schema Validation)** :
  * [legacy/predict.py](legacy/predict.py#L11-L14) caste directement les arguments sans contrôle : `int(sys.argv[1])`, `float(sys.argv[3])`.
  * Aucune vérification des plages de valeurs (ex : un âge négatif ou de 250 ans, un IMC de 0 ou 100, un nombre de comorbidités négatif ou un code sexe différent de 0/1 ne lèvent aucune exception avant l'inférence). Une entrée invalide provoque un crash non géré (`ValueError: invalid literal for int()`) ou produit une prédiction absurde sans alerte.
* **Traitement incomplet des données métier** : Les variables `departement`, `service` et `type_admission` présentes dans [data/dms_dataset.csv](data/dms_dataset.csv) sont purement éliminées sans justification technique ni test de pouvoir prédictif.

---

## 2. Validation Méthodologique & Surapprentissage (Overfitting)

### 2.1 Absence de split d'entraînement et d'évaluation
* **Évaluation sur données d'entraînement (Data Leakage total)** : Dans [legacy/train.py](legacy/train.py#L25), le score est calculé directement sur les données d'apprentissage :
  ```python
  print("modele entraine et sauve. accuracy train =", model.score(X, y))
  ```
  Le prestataire a affiché une précision d'entraînement de **77,2 %** ($AUC = 0{,}866$, $F_1 = 0{,}683$), masquant un surapprentissage sévère.
* **Chiffres d'audit en validation croisée réelle (5-Fold Stratifié)** :
  * **Accuracy réelle hors échantillon** : **67,5 %** (chute de **-9,7 points** par rapport au train).
  * **ROC AUC réelle hors échantillon** : **0,722** (chute de **-14,4 points** par rapport au train).
  * **Régression Logistique simple (baseline)** : Atteint **69,5 % d'accuracy** et **0,740 d'AUC** en validation croisée, surpassant le Random Forest hérité tout en étant 5 000 fois plus légère.
* **Absence de protocole de validation temporelle** : Pour des séjours hospitaliers soumis à des effets de saisonnalité ou des vagues épidémiques, aucun découpage temporel n'a été prévu.

---

## 3. Sécurité des Systèmes d'Information & Vulnérabilités

### 3.1 Fuite de secret d'infrastructure en clair (CWE-798)
* Dans [legacy/train.py](legacy/train.py#L9) :
  ```python
  DB_PASSWORD = "medivox_prod_2024"
  ```
  Le mot de passe de la base de données de production est écrit en dur dans le code source stocké dans le dépôt git. Tout intervenant ayant accès au code dispose des droits d'accès à la base de production.
* Absence totale de gestion par variables d'environnement (`os.environ`) ou coffre de secrets (Vault/KMS).

### 3.2 Exposition de l'inférence et transport non sécurisé
* **Déploiement manuel par copie de fichier (`scp`)** : Le modèle est poussé à la main sur un serveur sans signature cryptographique, sans somme de contrôle (checksum SHA-256) ni contrôle d'intégrité.
* **Exécution en production par appel de commande SSH distante** : L'application cliente ou le SI hospitalier exécute [legacy/predict.py](legacy/predict.py) en injectant des arguments dans un shell distant via SSH. Cette pratique expose directement l'infrastructure à des risques d'injection de commandes et empêche tout mécanisme d'équilibrage de charge (load balancing).

---

## 4. Points Uniques de Défaillance (SPOF) & Robustesse

### 4.1 SPOF d'infrastructure (Single Point of Failure)
* Le modèle dépend d'un fichier binaire unique déposé sur le disque local d'un seul serveur (`legacy/dms_predictor_v1.joblib`). Si ce serveur tombe, ou si le fichier est corrompu/supprimé, l'ensemble du service de prédiction des cliniques MediVox est indisponible.
* Aucun mécanisme de bascule (failover), aucune réplication, aucun conteneur (Docker), aucun orchestrateur de conteneurs.

### 4.2 Robustesse logicielle inexistante
* Si un paramètre d'entrée est manquant ou altéré, le script plante (`IndexError: list index out of range` ou `ValueError`).
* Aucun code d'erreur standardisé (pas d'API REST / JSON). Le résultat est écrit sur la sortie standard `stdout` sous forme de texte brut (`print(...)`), rendant le parsing applicatif fragile à la moindre modification d'affichage.

---

## 5. Gouvernance MLOps : Versioning, Traçabilité & Monitoring

### 5.1 Absence de Versioning et de Reproductibilité
* **Sérialisation opaque non versionnée** : Le fichier `.joblib` ne contient aucun artefact de métadonnées (date d'entraînement, hash du commit git du code d'entraînement, version de `scikit-learn`, `numpy` ou `python`).
* **Risque de désérialisation incompatible ou malveillante** : `joblib.load()` exécute du code pickle Python non restreint. Charger un fichier non signé expose à l'exécution de code arbitraire si les permissions du système de fichiers sont compromises.

### 5.2 Absence totale d'observabilité et de monitoring
* **Zéro journalisation (logs)** : Aucune trace des requêtes d'inférence, des arguments fournis, des probabilités calculées ni des timestamps d'exécution.
* **Incapacité à détecter les dérives (Drift)** :
  * *Data Drift* : Impossible de savoir si la population de patients accueillie en 2026 a changé d'âge moyen, d'IMC ou de sévérité par rapport aux données d'il y a 2 ans.
  * *Concept Drift* : Impossible de détecter l'évolution des pratiques cliniques internes (réduction générale des DMS cibles dans les cliniques).
* **Absence de boucle de feedback** : Aucune confrontation en continu entre les prédictions émises et la durée réelle de séjour constatée à la sortie du patient.

---

## 6. Synthèse des Risques Techniques

| Réf | Vulnérabilité / Risque technique | Gravité | Conséquence opérationnelle pour MediVox |
|---|---|---|---|
| **TECH-01** | Mot de passe de production en dur dans le code (`DB_PASSWORD`) | 🔴 Critique | Risque d'intrusion et de compromission immédiate de la base de données de santé en production. |
| **TECH-02** | SPOF absolu : serveur unique, modèle local sans redondance, déploiement scp | 🔴 Critique | Interruption totale du service clinique en cas de panne matérielle ou système. |
| **TECH-03** | Surapprentissage masqué par absence de split / validation croisée | 🟠 Élevé | Performances réelles en chute libre (-10 % d'accuracy, -14 points d'AUC), modèle sous-optimal par rapport à une simple régression logistique. |
| **TECH-04** | Inférence via CLI SSH sans API, sans schéma de validation ni gestion d'erreurs | 🟠 Élevé | Crashs réguliers sur entrées invalides, risque d'injection shell, absence d'interopérabilité SI. |
| **TECH-05** | Absence complète de versioning, logs et monitoring de drift | 🟠 Élevé | Aveuglement opérationnel total, incapacité d'audit post-incident, non-conformité aux exigences de traçabilité. |

### ❓ Questions ouvertes pour Hélène Tournier (Directrice Technique) :
1. *Quelle est la disponibilité cible (SLA) exigée par les équipes médicales pour ce service de prédiction (24/7 vs heures ouvrées) ?*
2. *Quel est le mode d'intégration cible souhaité dans le SI hospitalier (API REST JSON conteneurisée, messagerie HL7 / FHIR, intégration directe dans le DPI) ?*
3. *Existe-t-il un coffre-fort de secrets d'entreprise et un registre de conteneurs dans l'infrastructure MediVox ?*
