# Projet 2 — Prétraitement de CIC-DDoS2019

**Auteur :** Tjeck Tjeck Thomas Noel — Maîtrise en informatique, UQO
**Notebook :** `Projet2_Pretraitement_CICDDoS2019_TjeckTjeckThomasNoel.ipynb`

Prétraitement complet de CIC-DDoS2019, des 18 fichiers CSV bruts jusqu'à des jeux train/test prêts pour l'apprentissage, sans fuite de données. Le notebook est structuré en parties **A à F** (questions 1 à 16) et s'exécute de bout en bout, dans l'ordre.

---

## 1. Contenu du dossier

| Fichier | Rôle |
|---|---|
| `Projet2_Pretraitement_CICDDoS2019_TjeckTjeckThomasNoel.ipynb` | Notebook complet (code commenté, sorties, figures, interprétations) |
| `requirements.txt` | Versions exactes des librairies |
| `README.md` | Ce fichier |

## 2. Création de l'environnement

Version de Python utilisée : **3.13.16**. Python 3.11 ou plus récent est recommandé.

**Option A : venv**

```bash
python -m venv .venv
# Linux / macOS
source .venv/bin/activate
# Windows
.venv\Scripts\activate
pip install -r requirements.txt
```

**Option B : conda**

```bash
conda create -n cicddos python=3.13 -y
conda activate cicddos
pip install -r requirements.txt
```

Lancer ensuite le notebook :

```bash
jupyter notebook Projet2_Pretraitement_CICDDoS2019_TjeckTjeckThomasNoel.ipynb
```

**Option C : Google Colab** (environnement utilisé pour les résultats du rapport). Il suffit d'ouvrir le notebook dans Colab, sans installation : toutes les librairies sont déjà présentes, sauf éventuellement `imbalanced-learn`, que la cellule Q14 installe automatiquement si nécessaire.

## 3. Données : emplacement attendu des CSV

Le notebook utilise la version Kaggle des CSV officiels du Canadian Institute for Cybersecurity :
**« CIC-DDoS2019 30GB Full Dataset CSV Files »**, https://www.kaggle.com/datasets/rodrigorosasilva/cic-ddos2019-30gb-full-dataset-csv-files

**Téléchargement automatique (par défaut).** La cellule de la Q1 appelle :

```python
path = kagglehub.dataset_download("rodrigorosasilva/cic-ddos2019-30gb-full-dataset-csv-files")
```

L'archive fait environ 3,1 Go, et environ 31 Go une fois décompressée. Les CSV sont placés dans le cache `kagglehub` (`~/.cache/kagglehub/...` ; `/root/.cache/kagglehub/...` sur Colab). Aucun jeton Kaggle n'est en général demandé pour ce jeu public.

**CSV déjà téléchargés en local.** Il faut respecter cette arborescence, puis remplacer `path` par le dossier `data/` dans la cellule Q1 :

```
data/
├── 01-12/   DrDoS_DNS.csv, DrDoS_LDAP.csv, DrDoS_MSSQL.csv, DrDoS_NTP.csv, DrDoS_NetBIOS.csv,
│            DrDoS_SNMP.csv, DrDoS_SSDP.csv, DrDoS_UDP.csv, Syn.csv, TFTP.csv, UDPLag.csv
└── 03-11/   LDAP.csv, MSSQL.csv, NetBIOS.csv, Portmap.csv, Syn.csv, UDP.csv, UDPLag.csv
```

```python
path = "data"      # au lieu de kagglehub.dataset_download(...)
```

Les fichiers officiels peuvent aussi être obtenus sur la page du CIC (https://www.unb.ca/cic/datasets/ddos-2019.html), après un formulaire d'inscription.

## 4. Ordre d'exécution et durée

| Étape | Durée approximative |
|---|---|
| Q1 : téléchargement Kaggle (3,1 Go) | ≈ 1 min (instantané si déjà en cache) |
| Q2 : comptage des labels sur les 70,4 M de flux | 5 à 7 min |
| Q3 : lecture par morceaux et échantillonnage | 15 à 20 min |
| Q4 à Q16 | moins de 2 min au total |

**Cache Parquet (Google Colab).** Après la Q3, une cellule enregistre l'échantillon et les effectifs de la Q2 dans Google Drive (`MyDrive/CICDDoS2019_cache/`). Après un redémarrage de session, il suffit d'exécuter la cellule d'imports, puis la cellule **« ⚡ RECHARGEMENT RAPIDE »** (environ 5 s), et de reprendre à la Q4. En local, on peut remplacer `/content/drive/MyDrive/CICDDoS2019_cache` par n'importe quel dossier et ignorer `drive.mount(...)`.

**Sorties (Q16)**, dans `CICDDoS2019_cache/export/` :

- `pipeline_pretraitement.joblib` : pipeline scikit-learn ajusté sur le train (log1p + RobustScaler, One-Hot de `protocol`) ;
- `train.parquet` (217 743 × 51) et `test.parquet` (93 319 × 51) : 46 features transformées + `label`, `label_brut`, `attack_family`, `y_bin`, `y_multi`.

Pour recharger le pipeline dans une autre session, il faut définir la fonction avant `joblib.load` et utiliser **scikit-learn 1.6.1** :

```python
import numpy as np, joblib
def log1p_signe(a):
    return np.sign(a) * np.log1p(np.abs(a))
pipeline = joblib.load("pipeline_pretraitement.joblib")
```

## 5. Reproductibilité

- **Graines :** `random_state = 42` (découpage, sous-échantillonnage, SMOTE) et `np.random.seed(42)` / `np.random.default_rng(42)` (échantillonnage).
- **Échantillonnage :** lecture par morceaux de 500 000 lignes. Taux de tirage par classe = `min(1, 50 000 / effectif réel)`. BENIGN, UDPLag et WebDDoS sont conservés en entier. On obtient 916 769 flux sur 70 427 637 (≈ 1,3 %).
- **Découpage :** 70/30 stratifié sur `label` (14 classes harmonisées), `random_state = 42`.
- **Versions** (affichées par la dernière cellule du notebook) : Python 3.13.16, pandas 2.2.3, NumPy 2.1.3, scikit-learn 1.6.1, imbalanced-learn 0.14.2, matplotlib 3.10.0, seaborn 0.13.2, joblib 1.6.0, pyarrow 23.0.1, kagglehub 1.0.2.

## 6. Matériel conseillé

Au moins 12 Go de RAM et environ 40 Go d'espace disque libre (CSV décompressés et cache). Les résultats ont été obtenus sur Google Colab Pro.
