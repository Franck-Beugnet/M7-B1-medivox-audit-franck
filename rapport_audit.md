# Rapport d'audit — Prédicteur de séjour prolongé v1 (MediVox)

> **Destinataires** : 👩‍💻 Hélène Tournier (Directrice Technique) · ⚖️ Marc Lebourg (DPO)  
> **Date de restitution** : Mardi 22 septembre 2026  
> **Statut du document** : Audit d'état des lieux (Observation, Mesures chiffrées & Hiérarchisation des risques — Sans modification de code)  
> **Code audité** : `legacy/train.py`, `legacy/predict.py`, `legacy/dms_predictor_v1.joblib`, `data/dms_dataset.csv`

---

## 1. Synthèse exécutive (Lecture 5 minutes — Le plus grave d'abord)

Le « prédicteur DMS v1 » actuellement déployé en production chez MediVox Cliniques présente des **vulnérabilités critiques qui engagent directement la responsabilité juridique, clinique et opérationnelle du groupe**. Bien que le modèle s'exécute techniquement sans erreur fatale immédiate, il place MediVox en situation de risque majeur :

### ⚖️ Pour Marc Lebourg (DPO) : Urgence éthique et conformité réglementaire
1. **Discrimination systémique et perte de chance massive** :
   * La durée de séjour réelle constatée est strictement identique entre hommes et femmes (**5,62 jours** vs **5,59 jours**).
   * Pourtant, le modèle sélectionne **48,6 % des hommes contre seulement 14,1 % des femmes**, soit un **Disparate Impact effondré à 0,291** (seuil d'alerte à 0,80).
   * **Conséquence clinique** : Si le score pilote l'accès anticipé aux Soins de Suite et Réadaptation (SSR), **73,7 % des femmes ayant un séjour long effectif sont ignorées par le modèle (Faux Négatifs)**, subissant une **perte de chance médicale directe**.
2. **Non-conformité RGPD & Risque de sanction AI Act** :
   * **Art. 5(1)(c) RGPD (Minimisation)** : La variable `sexe` est injectée directement dans le modèle sans aucune justification médicale clinique documentée.
   * **Art. 22 RGPD (Décision automatisée)** : Le script renvoie un verdict binaire brut sans trace de réévaluation humaine effective (*jurisprudence CJUE SCHUFA*).
   * **AI Act (Art. 6 & Annexe III point 5.d)** : Si le score est sollicité lors de l'accueil pour les admissions d'urgence, le système bascule **de plein droit en HAUT RISQUE**, avec l'obligation immédiate de gouvernance des données, journalisation et supervision humaine sous peine de sanctions lourdes (jusqu'à 35 M€ ou 7 % du CA).

### 👩‍💻 Pour Hélène Tournier (Directrice Technique) : Dette technique et sécurité
1. **Secret d'infrastructure en clair (CWE-798)** :
   * Le mot de passe de production de la base de données (`DB_PASSWORD = "medivox_prod_2024"`) figure en clair dans le code source git [legacy/train.py](legacy/train.py#L9).
2. **Point unique de défaillance (SPOF)** :
   * Le service repose sur un modèle unique localisé sur un serveur physique unique, sans redondance, déployé manuellement par simple copie `scp` et invoqué par commande SSH distante, sans conteneurisation ni API REST.
3. **Surapprentissage sévère & Inefficience** :
   * La précision affichée de 77 % à l'entraînement est fictive : en validation croisée réelle, la précision chute à **67,5 %** ($AUC = 0{,}722$).
   * Une simple régression logistique surpasse ce modèle en généralisation ($AUC = 0{,}740$) tout en étant **5 000 fois plus légère sur disque** (1 Ko vs 5 Mo) et **50 fois plus rapide** à l'inférence.

---

## 2. Contexte et Périmètre de l'Audit

### 2.1 Contexte de la mission
Le prestataire historique de MediVox a développé un modèle qualifié de « prédicteur DMS ». En pratique, il s'agit d'un classifieur binaire signalant les séjours susceptibles de dépasser une durée critique. Avant d'engager toute refonte ou évolution, la direction générale, la direction technique et le DPO ont mandaté cet audit indépendant pour établir un état des lieux contradictoire et outillé.

### 2.2 Périmètre audité vs Hors-périmètre (Grille de cadrage)

| Périmètre audité (Inclus) | Hors-périmètre (Exclu de l'audit M7-B1) |
|---|---|
| • Analyse statique et dynamique des scripts `legacy/train.py` et `legacy/predict.py`.<br>• Évaluation du modèle binaire `legacy/dms_predictor_v1.joblib`.<br>• Audit du jeu de données historique `data/dms_dataset.csv` (10 000 séjours).<br>• Mesures outillées d'équité (Disparate Impact, FNR, FPR par genre).<br>• Mesures de ressources système via `psutil` (RAM, CPU, latence, disque).<br>• Qualification juridique raisonnée (RGPD art. 9 & 22, AI Act art. 6).<br>• Hiérarchisation des risques et formulation des questions ouvertes. | • **Correction ou modification du code hérité** (principe d'intégrité de l'audit).<br>• Conception de la nouvelle architecture cible ou chiffrage budgétaire (prévus en M7-B2).<br>• **Rédaction d'une AIPD formelle** (responsabilité exclusive du DPO).<br>• Audit de sécurité offensif (tests d'intrusion / pen-test).<br>• Mitigation active des biais algorithmiques.<br>• Prise de décision unilatérale sur l'arrêt de la production. |

---

## 3. Volet Éthique, Biais & Réglementaire ⚖️ *(Pour Marc Lebourg)*

### 3.1 Disparate Impact chiffré et amplification du biais
L'audit a comparé les taux d'événements entre les femmes ($N=5\,011$) et les hommes ($N=4\,989$) sur le dataset et le modèle :

$$\text{DI} = \frac{P(\hat{Y} = 1 \mid \text{Sexe} = \text{F})}{P(\hat{Y} = 1 \mid \text{Sexe} = \text{M})}$$

| Échelon mesuré | Taux Femmes | Taux Hommes | Disparate Impact (F/M) | Constat & Diagnostic |
|---|---|---|---|---|
| **Durée réelle observée (`dms_jours`)** | Moyenne : **5,62 j** (médiane 5,6) | Moyenne : **5,59 j** (médiane 5,5) | — | **Aucune différence clinique réelle** entre hommes et femmes. |
| **Étiquettes historiques (`sejour_prolonge`)** | 32,07 % ($1\,607$ séjours) | 49,15 % ($2\,452$ séjours) | **0,653** (< 0,80) | **Biais d'étiquetage historique** : 917 séjours féminins réels longs non étiquetés. |
| **Prédictions du modèle ($\hat{Y}$ seuil 0,5)** | **14,15 %** ($709$ séjours) | **48,57 %** ($2\,423$ séjours) | **0,291** (<<< 0,80) | **Amplification violente** : le modèle divise encore par deux la sélection féminine. |
| **Probabilité moyenne prédite** | **32,08 %** | **49,07 %** | — | Écart injustifié de **+17 points de probabilité** en faveur des hommes. |

### 3.2 Taux d'erreur par groupe et tranchage du préjudice
Confronté à une référence clinique objective fondée sur la durée réelle ($y_{\text{ref}} = \text{dms\_jours} \ge 5{,}6$ jours) :
* **Femmes** : **FNR = 73,7 %** ($1\,860$ patientes ignorées) | **FPR = 1,8 %** ($45$ fausses alertes).
* **Hommes** : **FNR = 24,2 %** ($594$ patients ignorés) | **FPR = 22,3 %** ($565$ fausses alertes).

**Comment trancher le préjudice selon l'usage opérationnel :**
* **Usage A (Soins / Prévention)** : Si le score déclenche une réservation de lit en Soins de Suite ou un staff de coordination, **les patientes sont les victimes majeures** ($\text{FNR} = 73{,}7\,\%$) d'une **perte de chance médicale**.
* **Usage B (Gestion médico-économique)** : Si le score est utilisé pour filtrer les admissions programmées ou surtaxer les séjours, **les patients hommes sont pénalisés** ($\text{FPR} = 22{,}3\,\%$) par un **risque d'éviction ou de stigmatisation**.

### 3.3 Utilisation directe de la variable sensible `sexe`
La variable `sexe_bin` pèse **9,53 % de l'importance des variables** du Random Forest. À âge, IMC et comorbidités rigoureusement identiques, basculer le sexe de F à M augmente la probabilité de risque de **+16,8 points en moyenne** pour **85,5 % des patients**, sans aucune justification clinique.

### 3.4 Conformité RGPD & Qualification AI Act
* **RGPD Art. 9 & Art. 5** : Manquement direct à la minimisation des données par l'usage du sexe. Absence de chiffrement et de purge sur [data/dms_dataset.csv](data/dms_dataset.csv) (Art. 32).
* **RGPD Art. 22 (Décision automatisée)** : Le script émet un verdict binaire brut. Sans preuve journalisée d'un contrôle médical effectif, le traitement tombe sous le coup de l'interdiction de l'article 22 (*jurisprudence SCHUFA*).
* **AI Act Art. 6 & Annexe III** : Si le score est appelé à l'accueil pour les urgences (`type_admission == 'urgence'`), le système est qualifié **HAUT RISQUE** (Annexe III point 5.d). L'exception de l'art. 6(3) est inapplicable en raison du profilage de santé.

---

## 4. Volet Technique, Architecture & Sécurité 👩‍💻 *(Pour Hélène Tournier)*

### 4.1 Architecture & Dette Technique
* **Absence de Pipeline** : Aucune encapsulation scikit-learn. Le script `predict.py` attend un entier brut `0/1` en 4e argument CLI sans appliquer la règle de transcodage de `train.py`.
* **Absence de Schéma de validation** : Les paramètres d'entrée sont castés sans contrôle (`int(sys.argv[1])`). Aucun contrôle de bornes (âge, IMC négatifs acceptés sans erreur).
* **Abandon de variables métier** : `service`, `departement` et `type_admission` sont écartées sans analyse de corrélation ni justification technique.

### 4.2 Évaluation méthodologique & Surapprentissage
* **Surapprentissage masqué** : Le modèle affichait une précision d'entraînement de 77,2 % sans aucun split train/test. En validation croisée stratifiée 5-folds :
  * Accuracy réelle hors échantillon : **67,5 %** (chute de près de 10 points).
  * ROC AUC réelle : **0,722** (contre 0,866 sur le train).

### 4.3 Vulnérabilités de sécurité et Points de rupture (SPOF)
* **Secret en clair (CWE-798)** : Mot de passe `DB_PASSWORD = "medivox_prod_2024"` présent dans le script d'entraînement.
* **Exposition SSH & Inférence CLI** : Inférence exécutée par appel shell distant avec parsing sur `stdout`, créant des risques de sécurité et empêchant la mise en place d'un load balancer.
* **SPOF d'infrastructure** : Serveur unique, modèle unique non redondé, déploiement artisanal par `scp`.

### 4.4 MLOps, Traçabilité & Monitoring
* **Absence de versioning** : Binaire `dms_predictor_v1.joblib` sans métadonnées, sans hash git, sans tag de dépendances.
* **Zéro journalisation (logs)** : Aucune trace des prédictions, des probabilités, ni des entrées patients.
* **Absence de détection de dérive (Drift)** : Aucune métrique de suivi du data drift ou du concept drift.

---

## 5. Volet Ressources & Sobriété Numérique 👩‍💻 *(Pour Hélène Tournier)*

L'audit outillé via `psutil` a confronté le modèle hérité à deux alternatives standards ré-entraînées sur les mêmes données :

| Indicateur de ressource / Métrique | Modèle Hérité (Random Forest) | Alternative 1 : Régression Logistique | Alternative 2 : HistGradientBoosting | Gain / Ratio vs Legacy |
|---|---|---|---|---|
| **Taille disque du modèle** | **4,96 Mo** | **0,001 Mo** (1 Ko) | **0,186 Mo** (186 Ko) | **$\times 5\,000$ plus léger** avec LogReg |
| **Mémoire vive process (RSS)** | **167,3 Mo** | **~85 Mo** | **~92 Mo** | Réduction de près de 50 % de la RAM |
| **Temps d'entraînement (10k lignes)** | **548,1 ms** | **45,2 ms** | **844,4 ms** | **12 fois plus rapide** avec LogReg |
| **Latence d'inférence (10k séjours)** | **52,01 ms** | **1,06 ms** | **32,10 ms** | **50 fois plus rapide** avec LogReg |
| **Performance réelle (CV-5 AUC / Acc)** | $AUC = \mathbf{0{,}722}$ / $Acc = \mathbf{67{,}5\,\%}$ | $AUC = \mathbf{0{,}740}$ / $Acc = \mathbf{69{,}5\,\%}$ | $AUC = \mathbf{0{,}738}$ / $Acc = \mathbf{68{,}9\,\%}$ | **La Régression Logistique surpasse le modèle legacy !** |

**Analyse de sobriété** : Le coût énergétique brut d'inférence CPU est modeste, mais le coût d'exploitation est disproportionné (170 Mo de RAM et cold start I/O à chaque appel CLI). L'usage d'un Random Forest de 5 Mo est une complexité inutile qui dégrade les performances par rapport à un modèle linéaire sobre et explicable.

---

## 6. Tableau Consolidé des Risques (Hiérarchisé 🔴 / 🟠 / 🟡)

| N° | Domaine | Vulnérabilité / Indicateur | Constat chiffré | Sévérité | Conséquence pour MediVox |
|---|---|---|---|:---:|---|
| **1** | **Sécurité** | Secret d'infrastructure en dur | `DB_PASSWORD = "medivox_prod_2024"` dans le code git. | 🔴 Critique | Compromission possible de la base de données de production. |
| **2** | **Infrastructure** | Point unique de défaillance (SPOF) | Modèle unique hébergé sur une seule machine, déploiement scp. | 🔴 Critique | Interruption totale du service clinique en cas d'avarie serveur. |
| **3** | **Éthique & Biais** | Disparate Impact de genre violent | $\text{DI} = \mathbf{0{,}291}$ sur les prédictions (durées réelles identiques à $5{,}6$ j). | 🔴 Critique | Risque de contentieux pour discrimination indirecte systémique. |
| **4** | **Clinique & Éthique** | Faux Négatifs massifs chez les femmes | $\text{FNR} = \mathbf{73{,}7\,\%}$ vs référence clinique ($1\,860$ patientes oubliées). | 🔴 Critique | Perte de chance médicale majeure sur l'anticipation des soins d'aval. |
| **5** | **RGPD** | Violation de la minimisation | Variable `sexe` en entrée sans justification médicale clinique. | 🔴 Critique | Manquement à l'art. 5(1)(c) RGPD, sanctionnable par la CNIL. |
| **6** | **RGPD** | Décision automatisée sans traçabilité | Verdict binaire sans trace de réévaluation humaine (*SCHUFA*). | 🔴 Critique | Non-conformité à l'art. 22 du RGPD. |
| **7** | **AI Act** | Risque de qualification Haut Risque | Score potentiellement utilisé en triage d'urgences (Annexe III 5.d). | 🔴 Critique / 🟠 Élevé | Sanctions financières pouvant atteindre 35 M€ ou 7 % du CA. |
| **8** | **Technique** | Surapprentissage masqué (Overfitting) | Accuracy réelle de **$67{,}5\,\%$** (contre $77{,}2\,\%$ affichés). | 🟠 Élevé | Modèle legacy battu par une simple régression logistique ($69{,}5\,\%$). |
| **9** | **Architecture** | Inférence CLI via SSH sans API | Appel shell direct, aucun contrat d'interface, parsing sur `stdout`. | 🟠 Élevé | Crash de l'appelant, vulnérabilité aux injections shell, zéro résilience. |
| **10** | **Robustesse** | Absence de validation de schéma | Casts bruts, absence de contrôle des plages de valeurs (âge/IMC). | 🟠 Élevé | Plantages non gérés ou production silencieuse d'absurdités. |
| **11** | **MLOps** | Absence totale de logs et traçabilité | Zéro enregistrement des prédictions ni des probabilités calculées. | 🟠 Élevé | Impossibilité légale d'auditer les décisions et de piloter le modèle. |
| **12** | **MLOps** | Absence de monitoring de drift | Aucun suivi de data drift ni de concept drift. | 🟠 Élevé | Obsolescence et dérive silencieuse des performances dans le temps. |
| **13** | **Data Management** | Données de santé non pseudonymisées | Dataset stocké avec `patient_id` en clair, non chiffré au repos. | 🟠 Élevé | Manquement aux obligations de sécurité de l'art. 32 du RGPD. |
| **14** | **Sobriété** | Surdimensionnement algorithmique | Modèle de 5 Mo pour 4 variables, soit 5 000 fois plus lourd qu'une LogReg. | 🟡 Modéré | Gaspillage de mémoire et latence de démarrage froid inutile. |
| **15** | **Ressources** | Inefficience de calcul à l'inférence | Inférence 50 fois plus lente que l'alternative linéaire. | 🟡 Modéré | Pénalité d'exécution sur les batchs quotidiens. |

---

## 7. Questions Ouvertes pour le Client (Préparation de M7-B2)

Ces questions doivent être tranchées avec les équipes de MediVox avant d'engager la moindre évolution technique ou juridique :

1. **Sur l'usage clinique et soignant du score** :
   * *À quel protocole opérationnel précis l'étiquette `RISQUE_SEJOUR_PROLONGE` est-elle raccordée dans les cliniques ? Déclenche-t-elle la réservation de lits d'aval (SSR) ou un contrôle administratif médico-économique ?*
2. **Sur le contrôle humain effectif (RGPD art. 22)** :
   * *Le médecin ou soignant peut-il débrayer la prédiction ? Cette décision contradictoire est-elle consignée dans le Dossier Patient Informatisé (DPI) ?*
3. **Sur la justification médicale du sexe** :
   * *La direction médicale dispose-t-elle d'une étude clinique justifiant le maintien du sexe biologique comme facteur déterminant de la durée de séjour ?*
4. **Sur le parcours des urgences (AI Act art. 6)** :
   * *Le score est-il généré lors du triage à l'arrivée aux urgences ? Si oui, le système bascule sous les obligations strictes de l'Annexe III point 5.d.*
5. **Sur les choix d'architecture et de budget (Attentes d'Hélène Tournier)** :
   * *Quelle est la disponibilité cible (SLA) attendue ? Quels sont les standards internes en matière d'hébergement HDS, de conteneurisation et de gestion des secrets ?*
