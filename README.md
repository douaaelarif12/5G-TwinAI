# 5G TwinAI — Jumeau numérique intelligent des réseaux 5G

5G TwinAI est une plateforme logicielle qui simule un réseau 5G urbain et expose son état en temps réel dans un tableau de bord web. Le projet relie une simulation radio, un moteur de KPI, des modèles de Machine Learning, un agent de renforcement et une API FastAPI avec diffusion WebSocket.

Le projet fonctionne sans matériel radio : les stations de base, les utilisateurs mobiles et le canal radio sont simulés en Python.

## Fonctionnalités

- Simulation discrète d’un réseau composé de 3 stations gNB et de 20 équipements mobiles.
- Mobilité des utilisateurs selon un modèle Random Waypoint.
- Calcul de perte de propagation, SINR, débit, allocation de PRB, charge cellulaire et handovers.
- Génération et stockage de KPI dans un fichier CSV et une base SQLite.
- Prédiction de congestion avec une combinaison LSTM + XGBoost.
- Détection d’anomalies avec Isolation Forest.
- Explication des variables importantes à partir du gain XGBoost.
- Agent PPO pour proposer des actions de rééquilibrage de charge.
- Comparaison en ligne entre l’agent PPO et une stratégie basée sur des règles.
- API REST, endpoint Prometheus et WebSocket pour les mises à jour temps réel.
- Dashboard React avec carte réseau, KPI, prédictions, anomalies, explications, A/B testing, replay et suivi de l’agent RL.

## Architecture

```text
simulation/       Modèle radio, gNB, UE, mobilité et moteur SimPy
kpi/              Calcul, génération et stockage des indicateurs
ml/               Prétraitement, LSTM, XGBoost, ensemble et anomalies
optimizer/        Environnement RL, PPO et stratégie de référence
api/              API FastAPI, boucle temps réel et WebSocket
dashboard/        Interface React/Vite, graphiques et panneaux métier
models/           Modèles entraînés et scalers livrés avec le projet
data/             Données KPI et fichiers générés
reports/          Résultats d’évaluation et visualisations
tests/            Tests unitaires de la simulation et des composants
```

### Flux de données

```text
Simulation SimPy
      ↓
État radio et mobilité par tick
      ↓
Calcul des KPI
      ↓
LSTM + XGBoost · Isolation Forest
      ↓
PPO + stratégie basée sur des règles
      ↓
FastAPI / WebSocket / Prometheus
      ↓
Dashboard React
```

## Technologies

### Backend et simulation

- Python 3.11
- SimPy, NumPy, Pandas, SciPy
- FastAPI, Uvicorn, WebSockets
- Pydantic et Prometheus Client

### Machine Learning

- PyTorch pour le LSTM
- XGBoost pour le modèle tabulaire
- Scikit-learn pour le prétraitement et Isolation Forest
- SHAP pour les composants d’explication présents dans le projet
- Stable-Baselines3 et Gymnasium pour le PPO

### Frontend et déploiement

- React 19 avec Vite
- Recharts, D3.js et Tailwind CSS
- Docker Compose et Nginx

## Installation locale

Python 3.11 est recommandé, car les dépendances Machine Learning sont compilées et testées pour cette version.

```powershell
git clone https://github.com/douaaelarif12/5G-TwinAI.git
cd 5G-TwinAI

py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Installer les dépendances du dashboard :

```powershell
cd dashboard
npm ci
npm run build
cd ..
```

Démarrer l’API :

Pour éviter le conflit avec VoiceScope, le backend 5G utilise le port `8001` :

```powershell
python -m uvicorn api.main:app --host 127.0.0.1 --port 8001
```

La documentation interactive FastAPI est accessible sur [http://127.0.0.1:8001/docs](http://127.0.0.1:8001/docs).

Démarrer le dashboard en développement dans un second terminal :

```powershell
cd dashboard
npm run dev
```

Le dashboard de développement est accessible sur [http://127.0.0.1:5173](http://127.0.0.1:5173). Il utilise automatiquement l’API 5G sur le port `8001`.

## Lancement avec Docker

```bash
docker compose up --build
```

Le dashboard est servi par Nginx sur le port `80`. L’API écoute sur le port `8000` à l’intérieur du réseau Docker et est exposée sur le port hôte `8001`. Les routes `/api`, `/ws`, `/metrics` et `/health` sont relayées par Nginx.

- Interface Docker : [http://127.0.0.1](http://127.0.0.1)
- API Docker : [http://127.0.0.1:8001/health](http://127.0.0.1:8001/health)

Arrêter les services :

```bash
docker compose down
```

## API principale

| Méthode | Route | Rôle |
|---|---|---|
| WebSocket | `/ws/simulation` | Flux des ticks et historique récent |
| GET | `/health` | État de santé et modèles chargés |
| GET | `/api/status` | État de la simulation et nombre de clients |
| POST | `/api/simulation/start` | Démarrer la boucle de simulation |
| POST | `/api/simulation/stop` | Arrêter la boucle de simulation |
| GET | `/api/history` | Lire l’historique des ticks |
| GET | `/api/kpis/summary` | Résumé des KPI récents |
| POST | `/api/predict` | Prédire la congestion à partir d’un instant KPI |
| POST | `/api/agent/action` | Interroger l’agent PPO et la stratégie de référence |
| GET | `/api/anomaly/current` | Lire l’anomalie courante |
| GET | `/api/anomaly/history` | Lire l’historique des anomalies |
| GET | `/api/ab/history` | Comparer PPO et stratégie basée sur des règles |
| GET | `/api/shap/explanation` | Lire l’importance des variables XGBoost |
| GET | `/metrics` | Exposition Prometheus |
| POST | `/api/replay/record/start` | Commencer l’enregistrement d’un replay |
| POST | `/api/replay/record/stop` | Arrêter l’enregistrement |
| POST | `/api/replay/play` | Rejouer les ticks enregistrés |

## Paramètres de simulation

Les valeurs principales sont centralisées dans `config.py` :

- grille de simulation : `1000 × 1000 m` ;
- fréquence centrale : `3.5 GHz` ;
- puissance gNB : `43 dBm` ;
- 3 gNB et 20 UE ;
- 100 PRB maximum par cellule ;
- un tick simulé par seconde ;
- fenêtre ML de 10 ticks et horizon de prédiction de 30 ticks.

Ces paramètres représentent une simulation logicielle. Ils ne constituent pas une mesure d’un réseau radio réel.

## Tests et qualité

```powershell
python -m pytest -q
cd dashboard
npm run lint
npm run build
```

Les tests couvrent notamment la perte de propagation, le SINR, le débit, la mobilité des UE, l’allocation des PRB et l’exécution de la simulation.

## Limites connues

- Les résultats dépendent des données, des modèles présents dans `models/` et de leur environnement Python.
- Le projet simule le réseau ; il ne se connecte pas à une infrastructure 5G réelle.
- Les probabilités de congestion sont des sorties de modèles et doivent être évaluées sur un jeu indépendant avant un usage opérationnel.
- L’endpoint d’explication conserve le chemin `/api/shap/explanation`, mais l’implémentation actuelle utilise les importances par gain XGBoost.
- Le mode mock de l’API peut être utilisé si un composant de simulation ou un modèle ne peut pas être chargé ; l’état retourné indique alors le mode actif.

## Licence

Le projet est distribué sous licence MIT. Voir [LICENSE](LICENSE).

## Auteur

**Douaa EL ARIF** — Étudiante-ingénieure en Systèmes d’Information et de Communication.

- GitHub : [douaaelarif12](https://github.com/douaaelarif12)

