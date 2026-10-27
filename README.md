# QuantGuard

Modèle de trading quantitatif à capital garanti sur **taux souverains** (US, zone euro) et **crypto-actifs** (BTC, ETH).
Hypothèses économiques, méthodes et préconisations : [`docs/RESEARCH_MEMO.md`](docs/RESEARCH_MEMO.md).

## Mise en ligne

- **Vercel** : édition statique. Le direct (prix, positions, alarmes, surface de volatilité) est calculé dans le navigateur et les calculs du soir tournent sur GitHub Actions. Guide : [`VERCEL.md`](VERCEL.md).
- **Serveur complet** (Render, VPS avec Docker, Railway, Fly.io) : tout tourne côté serveur, y compris le signal crypto à la seconde. Guide : [`DEPLOIEMENT.md`](DEPLOIEMENT.md).

Dans les deux cas, le mot de passe se règle avec `QG_PASSWORD`.

## Accès local en une commande

Prérequis : Python 3.11 ou plus récent ([python.org](https://www.python.org/downloads/)) et une connexion internet normale.

| Système | Lancement |
|---|---|
| macOS / Linux | dans un terminal, depuis le dossier : `sh start.sh` (ou `python3 run.py`) |
| Windows | double-clic sur `start.bat` (ou `py run.py`) |

Le navigateur s'ouvre sur **http://localhost:8050**, page **Direct**. Au premier lancement :

1. l'environnement `.venv` est créé et les dépendances installées (1 à 2 min) ;
2. le flux temps réel crypto démarre aussitôt : cours, graphiques à la seconde, signal recalculé chaque seconde ;
3. le premier calcul complet tourne en fond, environ 10 min : collecte sur 10 ans, entraînement, validation hors-échantillon, démarrage du forward test. La page Direct affiche son avancement dans « Traitements de fond », puis les autres pages se remplissent.

Ensuite, tant que le serveur tourne : recalcul temps réel chaque seconde, vérification des publications officielles toutes les 15 min (signaux et forward test mis à jour à chaque nouvelle donnée), recherche complète chaque nuit de semaine à 22:15 UTC. Le serveur écoute uniquement sur `127.0.0.1`. Ctrl+C pour l'arrêter.

### Fréquence de mise à jour des données

| Donnée | Fréquence | Source |
|---|---|---|
| BTC, ETH : prix, bid/ask, transactions | **à chaque transaction**, échantillonné et analysé chaque seconde | WebSocket Coinbase Exchange ; Kraken en contrôle croisé ; repli REST à la seconde |
| Marquage des positions crypto, stops matériels, kill switch, VL intraday | chaque seconde | moteur temps réel |
| Signal crypto (variables + modèle + intervalle conforme) | chaque seconde (clôture provisoire = prix courant) | dernier modèle entraîné |
| Taux US et zone euro, macro | **quotidienne** : publication officielle vérifiée toutes les 15 min | Fed H.15, BCE, Trésor US, BLS |
| Réentraînement des modèles | à chaque nouvelle clôture officielle ; validation complète chaque nuit | — |

Il n'existe pas de source officielle gratuite de taux à la seconde. Pour des taux intraday, il faut un flux de marché sous licence (futures CME/Eurex via un broker, par exemple l'API d'Interactive Brokers) branché sur `TickStore.update` dans `quantguard/live/feed.py`.

## Commandes détaillées

```bash
python -m quantguard.pipeline check      # accès aux 5 hôtes officiels
python -m quantguard.pipeline all        # collecte → recherche/validation → forward test → var/state.json
python -m quantguard.pipeline daily      # mise à jour légère (signaux + forward test, validation réutilisée)
python -m quantguard.pipeline serve --scheduler   # serveur seul (ce que lance run.py)
python -m pytest -q tests                # tests unitaires
```

## Architecture

```
config/settings.toml          tous les paramètres (garantie, bornes, risque, univers, validation, déploiement)
quantguard/
  data/        sources.py     registre des séries officielles et délais de publication
               collectors.py  FRED, BCE (SDMX), Trésor US, Coinbase, Kraken, avec retries
               store.py       SQLite, fenêtre de 10 ans, contrôles qualité, panel point-in-time
  features/    engineering.py rendements de futures synthétiques, variables comportementales, a priori économique
  models/      regime.py      HMM gaussien (probabilités filtrées)
               ml.py          ensemble boosting + ridge walk-forward, purge et embargo, combinaison avec l'a priori
               conformal.py   inférence conforme adaptative
               changepoint.py BOCPD incrémental
  strategy/    protection.py  CPPI à plancher actualisé + cliquet
               rules.py       règles d'entrée E1–E5 et de sortie S1–S7, bornes [−30 %, +30 %]
  risk/        limits.py      sizing en 8 couches, perte journalière, kill switch
               metrics.py     Sharpe, Sortino, VaR/ES, EVT, PSR, DSR, PBO (CSCV)
  backtest/    engine.py      moteur unique pour le backtest et le live
  validation/  walkforward.py signaux → backtest → portes hors-échantillon, grille de 27 configurations
  forward/     paper.py       paper trading persistant, test KS live vs backtest
  deploy/      stages.py      paper → pilote 2 % → 10 % → 35 % → 100 %
  live/        feed.py        WebSocket Coinbase + Kraken, repli REST 1 s, barres de 1 s
               engine.py      moteur 1 Hz : marquage, stops en séance, kill switch, nowcast du signal
  dashboard/   server.py      serveur local : flux SSE /api/stream (1 msg/s), /api/state, /api/job, ordonnanceur
               static/        page unique, 8 vues dont « Direct » et « Gestion des risques »
  context.py                  contexte macro daté et sourcé, surcouche de positionnement
tests/         test_core.py   19 tests unitaires ; synthetic.py = données simulées pour les tests d'intégration
```

## Pages du dashboard

| Ancre | Contenu |
|---|---|
| `#direct` | cours à la seconde, graphiques 15 min avec stop/objectif, VL et P&L intraday, kill switch temps réel, signal recalculé chaque seconde, journal |
| `#vol` | surfaces de volatilité : implicite BTC/ETH en temps réel (Deribit, SVI, deltas, maturités constantes, arbitrage, prime de variance) ; prévue GARCH pour les taux US et euro |
| `#apercu` | VL contre plancher garanti, KPI, développement vs hors-échantillon vs forward, signaux du jour |
| `#positions` | positions ouvertes marquées en direct, distribution des trades par rapport aux bornes, journal des trades |
| `#marches` | cours et taux avec sparklines, signaux avec intervalles conformes, IC du modèle, variables influentes |
| `#validation` | étapes de déploiement et critères, portes hors-échantillon (DSR, PBO), forward test, grille de robustesse |
| `#risques` | **Gestion des risques** : 1 risque par trade, 2 risque journalier, 3 position sizing dynamique, 4 stop loss matériel, 5 kill switch |
| `#contexte` | faits macro et géopolitiques sourcés, calendrier, préconisations |
| `#donnees` | couverture et qualité de chaque série |

## Avertissement

Outil de recherche, pas un conseil en investissement. La « garantie » du capital n'est contractuelle qu'avec un garant (voir mémo §4).
