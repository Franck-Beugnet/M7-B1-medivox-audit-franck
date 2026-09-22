# Procédure d'audit IA — template 7 sections (MediVox)

> Procédure **fournie** : remplissez chaque section. Un audit **outillé**, pas
> improvisé. Périmètre = observer/documenter/hiérarchiser (≠ corriger, ≠ AIPD).

## 1. Périmètre et hors-périmètre
_Ce qui est audité (modèle, code, dataset) ; ce qui est exclu (pen-test, AIPD,
refonte). Les 2 lectorats du rapport (technique / DPO)._

### 1.1 Réponses aux attentes des 2 lectorats (Hélène & Marc)
* **👩‍💻 Attentes d'Hélène Tournier (Directrice Technique)** :
  * *Question 1 : « Où en est-on ? »* → **INCLUS dans l'audit** (diagnostic exhaustif, chiffré et hiérarchisé de la dette technique, des risques, des performances et des faiblesses MLOps).
  * *Questions 2 & 3 : « Quoi changer ? » et « À quel prix ? »* → **HORS-PÉRIMÈTRE de l'audit M7-B1** (la conception de l'architecture cible, la sélection des solutions et le chiffrage budgétaire/charge sont l'objet du brief M7-B2). L'audit formule en revanche les *questions ouvertes et prérequis indispensables* pour qu'Hélène puisse arbitrer et chiffrer ces changements.
* **⚖️ Attentes de Marc Lebourg (DPO)** :
  * Qualification des risques légaux (RGPD art. 9 et 22, AI Act art. 6).
  * L'audit n'est **pas une AIPD formelle**, mais fournit à Marc tous les éléments techniques et factuels nécessaires pour qu'il instruise l'AIPD.

### 1.2 Grille de cadrage : Périmètre audité vs Hors-périmètre (Exclu)

| Domaine / Volet | Périmètre audité (Inclus) | Hors-périmètre (Exclu) |
|---|---|---|
| **Usage réel du score & Qualification du préjudice** | • **Investigation de l'usage opérationnel réel du score** : qui consomme la prédiction (médecin, soignant, régulateur, direction financière) et quelle action concrète est déclenchée (allocation de lit/SSR vs refus/dépriorisation médico-économique)<br>• **Tranchage du préjudice** : levée de l'alternative « Faux Négatifs lésés » (perte de chance de suivi) vs « Faux Positifs lésés » (stigmatisation/tri financier) dès que l'usage est clarifié<br>• Vérification de l'existence d'une supervision humaine effective vs validation de pure forme (critères arrêt *SCHUFA* / art. 22 RGPD) | • Arbitrage ou modification unilatérale des processus d'admission et de prise en charge des cliniques MediVox |
| **Données & Variables (dissonances dataset / train / predict)** | • Dataset historique `data/dms_dataset.csv` (10 000 séjours)<br>• **Audit des écarts de variables** : variables collectées mais ignorées (`departement`, `service`, `type_admission`), utilisation directe d'une variable sensible (`sexe`), et abandon de la durée réelle continue (`dms_jours`) au profit d'un seuillage binaire opaque (`sejour_prolonge`)<br>• **Divergences d'interfaces** entre `train.py` (calcul de `sexe_bin` sur `sexe`) et `predict.py` (attente d'un entier `sexe_bin` en argument CLI sans validation de format ni schéma d'entrée) | • Nettoyage, ré-étiquetage ou restructuration du jeu de données source<br>• Collecte de nouvelles données cliniques ou patients |
| **Cycle de vie du modèle, Versioning & Monitoring** | • **Audit du versioning** : persistance brute `legacy/dms_predictor_v1.joblib` sans métadonnées (pas de hash, pas de traçabilité de version scikit-learn, pas d'horodatage ni de lien avec le commit git ou le commit de données)<br>• **Audit du monitoring & traçabilité en production** : absence totale de métriques de drift (data drift, concept drift), absence de journalisation (logs) des prédictions, des probabilités, et des décisions humaines réelles en aval | • Conception et déploiement d'un Model Registry (MLflow, DVC) ou d'une plateforme de monitoring (Evidently, Prometheus, Grafana) — reporté à M7-B2 |
| **Code & Architecture technique** | • Scripts existants `legacy/train.py` et `legacy/predict.py` (lecture seule)<br>• Absence de modularité, absence de pipeline sklearn, absence de split train/test ou de validation croisée<br>• Vulnérabilités de sécurité évidentes (`DB_PASSWORD` en dur, exécution directe via SSH/CLI sans API sécurisée, modèle local sur disque = SPOF) | • **Correction ou refactorisation** du code legacy<br>• Tests d'intrusion offensifs (pentest)<br>• Mise en place de pipelines CI/CD ou sécurisation du serveur |
| **Volet Éthique & Biais** | • Calcul et investigation du *Disparate Impact* (taux de sélection, FNR, FPR par sous-groupe F/M)<br>• Évaluation de l'amplification du biais par le modèle par rapport aux étiquettes et à `dms_jours` | • **Mitigation active des biais** (repondération, modification des seuils ou débiaisement) |
| **Volet Réglementaire** | • Conformité RGPD art. 9 (données sensibles de santé, base légale, minimisation)<br>• Examen des conditions de l'art. 22 RGPD (décision automatisée à effet juridique/significatif)<br>• Qualification raisonnée du niveau de risque AI Act (art. 6 & annexes) selon l'usage opérationnel | • **Rédaction d'une AIPD complète** (rôle de Marc Lebourg)<br>• Conseils juridiques engageants |
| **Volet Ressources & Sobriété** | • Mesures outillées avec `psutil` (temps d'entraînement/inférence, mémoire RSS, taille disque)<br>• Comparaison objective à 1 ou 2 alternatives plus sobres (ex. Régression Logistique, Arbre de décision simple) | • Optimisation poussée du code ou déploiement sur cluster |
| **Livrables de l'audit** | • Tableau hiérarchisé des risques (🔴/🟠/🟡) avec impact client<br>• Rapport d'audit à 2 grilles de lecture (👩‍💻 Hélène / ⚖️ Marc)<br>• Questions ouvertes ciblées pour préparer les arbitrages de M7-B2 | • Devis financier ou engagement de calendrier de refonte (« à quel prix ») |

## 2. Audit éthique
_Variables sensibles (directes/indirectes) ; **disparate impact chiffré** sur ≥ 1
variable **puis investigué** (préjudice défini, erreurs par groupe, étiquette vs
réalité) ; RGPD santé (art. 9, minimisation, conservation) ; **usage réel** du score ;
AI Act (**qualification raisonnée** art. 6 → obligations si haut risque) ; art. 22
(2 conditions examinées)._

> 📄 **Document détaillé associé** : L'analyse complète, les mesures outillées et les argumentations juridiques sont détaillées dans [audit/01_ethique.md](audit/01_ethique.md).

### Synthèse du volet éthique & points clés audités :
* **Variables sensibles auditées** : Utilisation directe de `sexe_bin` (9,53 % d'importance, +16,8 points de probabilité de risque à profil clinique identique pour 85,5 % des patients) et surveillance des proxies indirects (`departement`, `service`).
* **Disparate Impact mesuré & investigué** :
  * Durée réelle de séjour (`dms_jours`) identique : **5,62 j** (F) vs **5,59 j** (M).
  * Biais de l'étiquetage historique : $32{,}07\,\%$ vs $49{,}15\,\%$ ($\text{DI} = \mathbf{0{,}653}$).
  * Amplification par le modèle : sélection de **$14{,}15\,\%$** des femmes vs **$48{,}57\,\%$** des hommes ($\text{DI} = \mathbf{0{,}291} \ll 0{,}80$).
* **Investigation des erreurs et préjudice** : FNR Femmes à **$73{,}7\,\%$** (1 860 séjours réels longs ignorés) vs FPR Hommes à **$22{,}3\,\%$** (565 fausses alertes). Le préjudice est tranché selon l'usage opérationnel : perte de chance de soins d'aval (Femmes) vs risque d'éviction/stigmatisation (Hommes).
* **Conformité RGPD** : Violation de la minimisation (art. 5(1)(c)), risque de requalification en décision automatisée non traçable (art. 22 / jurisprudence CJUE *SCHUFA*).
* **AI Act** : Qualification raisonnée (art. 6) — bascule en Haut Risque si utilisé en régulation/triage aux urgences (Annexe III 5.d) ; exception de risque limité fermée (art. 6(3)) en raison du profilage de santé.

$\rightarrow$ *Consulter l'intégralité des tableaux de données, investigations et questions ouvertes dans [audit/01_ethique.md](audit/01_ethique.md).*

## 3. Audit technique
_Architecture (modularité, couplage) ; sécurité (secrets, validation, transport) ;
scalabilité ; **points de rupture** (SPOF)._

## 4. Audit ressources
_Mesures **psutil** (temps train/inférence, RSS, taille modèle) ; comparaison à
**≤ 2 alternatives** ; lecture sobriété (chiffrée, honnête)._

## 5. Tableau d'indicateurs consolidé
_12-18 lignes : indicateur / sévérité (🔴🟠🟡) / conséquence client. Hiérarchisé,
pas tout au même niveau._

## 6. Synthèse exécutive
_½ page lisible en 5 min par un décideur non-ML (le « plus grave » d'abord)._

## 7. Questions ouvertes
_Ce qu'il faut clarifier avec le client avant toute évolution._
