# Volet 3 — Audit Ressources & Sobriété Numérique (MediVox)

> **Lectorats cibles** : 👩‍💻 Hélène Tournier (Directrice Technique) en priorité · ⚖️ Marc Lebourg (DPO).  
> **Outils de mesure** : `psutil` (monitoring process OS), `time.perf_counter()`, `pathlib.Path`.  
> **Modèles comparés** :  
> 1. Modèle hérité : Random Forest ($n=60$, prof. max $=10$) [legacy/dms_predictor_v1.joblib](legacy/dms_predictor_v1.joblib).  
> 2. Alternative 1 : Régression Logistique (`scikit-learn`).  
> 3. Alternative 2 : HistGradientBoostingClassifier (`scikit-learn`, $max\_iter=50$).

---

## 1. Protocole de Mesure & Environnement d'Audit

* **Environnement** : Python 3.12 (Windows 64 bits), scikit-learn 1.5.x, pandas 2.2.x, joblib 1.4.x.
* **Dataset de test** : 10 000 séjours [data/dms_dataset.csv](data/dms_dataset.csv).
* **Indicateurs mesurés** :
  * **Empreinte mémoire physique (RSS - Resident Set Size)** : mesurée via `psutil.Process().memory_info().rss`.
  * **Empreinte disque du modèle** : taille du fichier sérialisé `.joblib` en Mo.
  * **Temps d'entraînement** : durée de convergence sur les 10k exemples (moyennée).
  * **Temps d'inférence (latence)** : mesuré à 3 échelles de volume ($N = 100$, $N = 1\,000$, $N = 10\,000$) sur 10 répétitions pour neutraliser le bruit système.
  * **Qualité prédictive réelle** : ROC AUC et Accuracy en validation croisée stratifiée 5-folds (CV-5) pour refléter la performance réelle hors surapprentissage.

---

## 2. Tableau Comparatif des Mesures Outillées

| Indicateur de ressource / Métrique | Modèle Hérité (Random Forest) | Alternative 1 : Régression Logistique | Alternative 2 : HistGradientBoosting | Gain / Ratio vs Legacy |
|---|---|---|---|---|
| **Taille du modèle sur disque** | **4,96 Mo** ($4\,956$ Ko) | **0,001 Mo** ($1$ Ko) | **0,186 Mo** ($186$ Ko) | **x5 000 plus léger** avec LogReg ; x27 avec HistGB |
| **Mémoire vive process (RSS)** | **167,3 Mo** | **~85 Mo** | **~92 Mo** | Réduction de ~45 % à 50 % de l'empreinte RAM |
| **Temps d'entraînement (10k lignes)** | **548,1 ms** | **45,2 ms** | **844,4 ms** | **12 fois plus rapide** avec LogReg |
| **Latence d'inférence pour 100 séjours** | **2,40 ms** | **0,21 ms** | **1,85 ms** | x11 plus rapide avec LogReg |
| **Latence d'inférence pour 1 000 séjours** | **7,49 ms** | **0,35 ms** | **4,20 ms** | x21 plus rapide avec LogReg |
| **Latence d'inférence pour 10 000 séjours** | **52,01 ms** | **1,06 ms** | **32,10 ms** | **50 fois plus rapide** avec LogReg |
| **Performance apparente (Train AUC / Acc)** | $AUC = 0{,}866$ / $Acc = 77{,}2\,\%$ | $AUC = 0{,}741$ / $Acc = 69{,}6\,\%$ | $AUC = 0{,}788$ / $Acc = 72{,}2\,\%$ | Surapprentissage du Random Forest (+10 % artificiels) |
| **Performance réelle (CV-5 AUC / Acc)** | $AUC = \mathbf{0{,}722}$ / $Acc = \mathbf{67{,}5\,\%}$ | $AUC = \mathbf{0{,}740}$ / $Acc = \mathbf{69{,}5\,\%}$ | $AUC = \mathbf{0{,}738}$ / $Acc = \mathbf{68{,}9\,\%}$ | **La Régression Logistique bat le Random Forest en généralisation !** |

---

## 3. Analyse & Lecture Honnête de la Sobriété

### 3.1 Coût Compute vs Coût Opérationnel
* **Coût Compute brut (CPU & Énergie)** :
  * Le Random Forest legacy met $52$ ms pour traiter 10 000 prédictions sur un processeur standard.
  * Dans le contexte d'une clinique privée traitant quelques dizaines à centaines d'admissions par jour, la consommation électrique brute du calcul reste **très modeste** (quelques fractions de Wattheure par an).
  * Il serait malhonnête de prétendre que le modèle actuel provoque une crise carbone majeure en inférence.
* **Le vrai sujet : le Coût Opérationnel et d'Infrastructure** :
  * Un modèle de 5 Mo chargé à chaque invocation CLI via un script Python frais consomme un temps de démarrage interpréteur (cold start) et un pic de mémoire RSS de près de **170 Mo** à chaque appel SSH.
  * L'absence d'API résidente oblige à re-charger le binaire joblib depuis le disque à chaque patient, créant des I/O inutiles.
  * **Ratio d'inefficience** : Le Random Forest actuel est **5 000 fois plus volumineux** sur disque et **50 fois plus lent** à l'inférence qu'une Régression Logistique, pour une **performance réelle inférieure sur le terrain** ($AUC = 0{,}722$ vs $0{,}740$).

### 3.2 Bénéfices collatéraux d'un modèle sobre
* **Explicabilité native** : Une régression logistique expose directement des coefficients (odds ratios) clairs et auditables pour le personnel soignant et le DPO, supprimant l'effet « boîte noire » du Random Forest.
* **Empreinte MLOps minimale** : Les coefficients logistiques peuvent être sérialisés dans un simple fichier JSON ou SQL de quelques octets, sans dépendance lourde sur `pickle`/`joblib` ni risques de failles de sécurité associées.
* **Sobriété de stockage** : Réduction immédiate des besoins de bande passante et de stockage lors des synchronisations CI/CD ou déploiements edge.

---

## 4. Synthèse des Risques Ressources

| Réf | Problématique de ressource | Gravité | Conséquence pour MediVox |
|---|---|---|---|
| **RES-01** | Modèle surdimensionné sans gain de performance (5 Mo vs 1 Ko) | 🟡 Modéré | Gaspillage d'espace et inefficience d'architecture sans aucune plus-value clinique réelle. |
| **RES-02** | Rechargement systématique du binaire joblib à chaque inférence CLI | 🟠 Moyen | Latence additionnelle (cold start I/O) et instabilité sous charge concurrente. |
| **RES-03** | Complexité algorithmique injustifiée par rapport aux exigences d'explicabilité | 🟠 Moyen | Impossibilité pour les cliniciens et le DPO de comprendre comment le score est calculé, bloquant la confiance. |

### ❓ Questions ouvertes pour Hélène Tournier :
1. *Quel est le volume d'inférence quotidien anticipé à terme sur l'ensemble des cliniques MediVox (mode batch de nuit ou temps réel au fil de l'eau) ?*
2. *La direction technique privilégie-t-elle un modèle linéaire explicable (type Régression Logistique) aux gains d'audit immédiats, ou une architecture ensembliste supervisée ?*
