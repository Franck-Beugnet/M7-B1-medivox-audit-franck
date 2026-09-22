# Volet 4 — Tableau Consolidé des Risques & Hiérarchisation (MediVox)

> **Lectorats cibles** : 👩‍💻 Hélène Tournier (Directrice Technique) & ⚖️ Marc Lebourg (DPO).  
> **Méthode de cotation** :  
> 🔴 **Critique / Élevé** : Bloquant réglementaire ou opérationnel immédiat, risque juridique majeur, faille de sécurité critique.  
> 🟠 **Moyen** : Dette technique sévère, surapprentissage, instabilité, risque de non-conformité conditionnel.  
> 🟡 **Faible / Modéré** : Inefficience de ressources, amélioration de qualité, bonne pratique MLOps recommandée.

---

## 1. Tableau Consolidé des Risques (15 indicateurs clés)

| N° | Domaine | Indicateur audité | Constat chiffré / Factuel | Sévérité | Conséquence client (MediVox) |
|---|---|---|---|:---:|---|
| **1** | **Sécurité** | Secret d'infrastructure en dur dans le code | Variable `DB_PASSWORD = "medivox_prod_2024"` présente en clair dans `legacy/train.py`. | 🔴 Critique | Risque d'accès non autorisé direct à la base de données de santé par toute personne ayant accès au code source. |
| **2** | **Infrastructure** | Point unique de défaillance (SPOF) | Modèle unique hébergé sur une seule machine, sans réplication, déploiement manuel par commande `scp`. | 🔴 Critique | Arrêt total du service prédictif en cas d'incident matériel ou système sur le serveur unique. |
| **3** | **Éthique & Biais** | Disparate Impact de genre violent | $\text{DI} = \mathbf{0{,}291}$ sur les prédictions ($14{,}1\,\%$ F vs $48{,}6\,\%$ M) alors que les durées réelles sont identiques ($5{,}62$ j vs $5{,}59$ j). | 🔴 Critique | Risque de discrimination indirecte systémique et perte de chance massive de prise en charge pour les patientes. |
| **4** | **Clinique & Éthique** | Taux de Faux Négatifs critique chez les femmes | $\text{FNR} = \mathbf{73{,}7\,\%}$ chez les femmes vs référence clinique ($1\,860$ séjours longs réels non détectés). | 🔴 Critique | Sous hypothèse d'allocation de lits SSR, préjudice direct de santé pour près de 3 patientes sur 4 en séjour long. |
| **5** | **RGPD** | Violation du principe de minimisation | Variable `sexe` injectée directement en feature ($9{,}5\,\%$ d'importance) sans justification médicale documentée. | 🔴 Critique | Manquement direct à l'art. 5(1)(c) du RGPD, non défendable auprès de la CNIL lors d'un contrôle. |
| **6** | **RGPD** | Décision automatisée sans traçabilité humaine | Verdict binaire `RISQUE_SEJOUR_PROLONGE` sans log d'override humain (jurisprudence CJUE *SCHUFA*). | 🔴 Critique | Risque de nullité des décisions d'admission ou de transfert et contentieux patient au titre de l'art. 22 du RGPD. |
| **7** | **AI Act** | Qualification Haut Risque potentielle non anticipée | Score potentiellement utilisé lors de l'accueil pour les urgences (`type_admission == 'urgence'`). | 🔴 Critique / 🟠 Élevé | Bascule de plein droit sous l'Annexe III point 5.d (triage d'urgences) avec sanctions pouvant atteindre 35 M€ ou 7 % du CA. |
| **8** | **Technique** | Surapprentissage massif (Overfitting) | Évaluation sur le train ($Acc = 77{,}2\,\%$), chute à **$67{,}5\,\%$** en validation croisée réelle ($AUC = 0{,}722$). | 🟠 Élevé | Le modèle en production est moins performant qu'une simple régression logistique ($Acc = 69{,}5\,\%$, $AUC = 0{,}740$). |
| **9** | **Architecture** | Inférence fragile via CLI SSH sans API | Script `predict.py` exécuté par commande shell distante, parsing sur `stdout`, aucun schéma JSON. | 🟠 Élevé | Crash de l'appelant en cas de modification de format, risque d'injection shell, absence d'interopérabilité SI. |
| **10** | **Robustesse** | Absence de validation d'entrées (Schema) | Casts bruts `int(sys.argv[1])`, aucune validation des bornes (âge, IMC négatif ou aberrant acceptés). | 🟠 Élevé | Crash imprévu ou génération silencieuse de prédictions aberrantes pour les équipes soignantes. |
| **11** | **MLOps** | Absence totale de logs et de traçabilité | Aucune trace des requêtes d'inférence, des entrées ni des probabilités retournées en base de données. | 🟠 Élevé | Impossibilité légale d'auditer les décisions a posteriori et de prouver la supervision humaine requise. |
| **12** | **MLOps** | Absence de monitoring de dérive (Drift) | Aucun suivi de data drift (profil patient) ni de concept drift (politique de DMS interne). | 🟠 Élevé | Dégradation silencieuse et progressive de la qualité prédictive du modèle au fil des années. |
| **13** | **Data Management** | Données patients non pseudonymisées | Fichier `dms_dataset.csv` stocké avec `patient_id` en clair, non chiffré au repos, sans politique de purge. | 🟠 Élevé | Non-conformité aux exigences de sécurité de l'art. 32 du RGPD pour les données de santé. |
| **14** | **Sobriété** | Surdimensionnement algorithmique | Modèle Random Forest de $4{,}96$ Mo pour 4 features, soit $5\,000$ fois plus lourd qu'une LogReg ($1$ Ko). | 🟡 Modéré | Inefficience logicielle et temps de démarrage froid (cold start) inutile à chaque appel. |
| **15** | **Ressources** | Latence d'inférence sous-optimale | Inférence 10k séjours en $52$ ms (RF) vs $1{,}06$ ms (LogReg), soit un facteur 50 d'inefficience. | 🟡 Modéré | Pénalité de latence sur les traitements batch importants de fin de journée. |

---

## 2. Synthèse de la Hiérarchisation pour la Décision

* **Niveau 🔴 (6 risques critiques)** : Concentrés sur la **sécurité d'accès** (mot de passe en clair), la **continuité d'activité** (SPOF machine unique) et la **responsabilité juridique/éthique** (disparate impact $0{,}291$, perte de chance féminine de $73{,}7\,\%$, minimisation RGPD violée). Ces risques imposent des mesures conservatoires immédiates.
* **Niveau 🟠 (7 risques élevés)** : Concentrés sur la **robustesse logicielle** (overfitting masqué, absence de validation d'entrées, transport SSH fragile) et la **gouvernance MLOps** (absence de logs, monitoring inexistant, sécurité des données).
* **Niveau 🟡 (2 risques modérés)** : Concentrés sur la **sobriété et le dimensionnement** (poids du modèle, inefficience relative). Ils confortent le choix d'une alternative sobre lors de la future refonte.
