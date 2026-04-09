# n8n-ai-workflows
![n8n](https://img.shields.io/badge/n8n-orange)
![Python](https://img.shields.io/badge/Python-3.10+-blue)
![Docker](https://img.shields.io/badge/Docker-enabled-blue)
![GPT-5](https://img.shields.io/badge/GPT--5_mini-green)
![Ollama](https://img.shields.io/badge/Ollama-local_LLM-purple)

# NBA Betting Analysis — Self-Hosted n8n Workflows

Automated NBA value bet detection running on a self-hosted homelab server. Two n8n workflows analyze odds, team stats, and injury data to identify statistically motivated betting opportunities, delivered via Telegram.

> **Model philosophy:** This is not a free prediction model. It works as a **market correction** — small, data-driven adjustments to the bookmakers' own implied probabilities.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Homelab Server                       │
│                                                         │
│  n8n (Docker)  ──►  Stats API (Flask :5001)             │
│                          │                              │
│                     SQLite (nba.db)                     │
│                     player_data.json                    │
│                                                         │
│  Ollama (local LLM, qwen/llama)                         │
└─────────────────────────────────────────────────────────┘
        │                          │
        ▼                          ▼
  The-Odds-API              ESPN Injury Feed
  (Odds + Markets)          (Injury Status)
        │
        ▼
   Telegram Bot
   (Alerts & Picks)
```

| Component | Details |
|---|---|
| n8n | Docker container, orchestrates all workflows |
| Stats API | Flask, Port 5001 — serves team stats, H2H, injury state |
| SQLite DB | `nba.db`, ~4700 rows, seasons 2024-25 + 2025-26, updated daily |
| Ollama | Local LLM (qwen3.5:9B / llama3.1:8B) for injury comment analysis |
| OpenAI | GPT-5 Mini — final pick validation and reasoning |

---

## Workflows

### 1. NBA Analyse v3

**File:** `NBA Analyse v3 - github.json`  
**Schedule:** Daily at 23:00 (Europe/Berlin)  
**Purpose:** Full analysis pipeline — odds → stats → injuries → feature model → AI validation → Telegram

#### Pipeline

```
Schedule Trigger
  ├── Get NBA Odds          (The-Odds-API: h2h, spreads, totals)
  └── Get NBA Injuries      (ESPN)
        │
        ├── Extract Teams         [No-vig probabilities, spreads, totals per game]
        ├── Filter Target Date    [Today + tomorrow Berlin time]
        ├── Normalize Team Names  [ESPN ↔ NBA-API name mapping]
        │
        ├── Get Team Stats        [POST /stats → last-10 record, B2B, rest days, PPG, etc.]
        └── Normalize Injuries    [ESPN → structured injury_data per team]
              │
              ├── Merge Odds + Stats
              ├── Merge Injuries
              └── Attach Injuries    [B2B multiplier + age factor applied]
                    │
                    ├── Build Features / Pre-Model   [edges, market correction, totals]
                    ├── Refine Value / Confidence    [guardrails, confidence, totals pick]
                    └── If                           [value_side ≠ SKIP or totals signal]
                          │
                          ├── AI Agent (GPT)         [validate + justify, NO recalculation]
                          ├── Format Telegram Message
                          └── Send a text message
```

#### Market Correction Model

The model calculates:

```
model_prob = market_novig_prob + clamped_edge
```

Where `clamped_edge` is the sum of individual factor edges, hard-clamped to prevent overconfidence:

| Scenario | Max deviation |
|---|---|
| Normal | ±6% |
| Strong factor present | ±12% |

**Strong factors** (unlock ±12%): Injury gap > 8pts between teams, Back-to-Back, H2H edge > 2%, Net-rating gap > 5 PPG.

#### Confidence Classification

| Level | EV | Odds | Prob | Divergence | Prob-Gap | Action |
|---|---|---|---|---|---|---|
| High | ≥0.18 | ≤3.5 | ≥62% | ≤18% | ≥6% | Forward to AI |
| Medium | ≥0.10 | ≤5.0 | ≥50% | ≤25% | ≥5% | Forward to AI |
| Low | else | — | — | — | — | SKIP |

#### Hard-SKIP Guardrails

A pick is always forced to SKIP if:

- Odds ≥ 8.0
- Odds ≥ 5.0 AND EV < 0.15
- model_prob < 0.38
- Divergence > 15% AND EV < 0.15
- B2B team selected AND EV < 0.12

#### Totals Analysis

Separate Over/Under signal based on:
- Shooting absences → Under
- Rim protector absences → Over (easier scoring)
- B2B fatigue → Under
- Short rest → Under
- Playmaker absences → Under

Only forwarded if signal strength ≥ 3.5 pts from market line.

---

### 2. NBA Injury Monitor

**File:** `NBA Injury Monitor - github.json`  
**Schedule:** Every 5 minutes during game windows (`*/5 16-23,0-3 * * *`)  
**Purpose:** Real-time injury tracking — detects changes in team injury impact scores and sends immediate Telegram alerts

#### Pipeline

```
Cron (*/5 16-23,0-3 * * *)
  └── ESPN Injuries
        └── Extrahiere Spieler       [1 item per player, base impact pre-calculated]
              ├── Smart-Filter KI?   [If: needs_ai = true]
              │     ├── YES → Ollama Kommentar-Analyse (local LLM)
              │     │           └── Parse Ollama JSON
              │     │                 └── Wende AI-Faktor an   [ai_adjustment 0.5–1.5]
              │     └── NO  → Fast-Lane Faktor 1.0             [standard status-based]
              │
              └── Zusammenfuehren (Merge)
                    └── Aggregiere zu Teams    [final_impact_score per team]
                          └── Diff-Check (POST /injury_state)
                                └── Änderungen vorhanden?
                                      └── Formatiere Alert → Telegram Alert
```

#### Smart Routing (AI vs Fast-Lane)

Not every injury note needs LLM analysis. A player is routed to Ollama only if:
- Their base impact score > 5 (star/starter level), **OR**
- The `shortComment` contains strong clinical keywords (`surgery`, `torn`, `fracture`, `crutches`, `limping`, `cleared`, `injection`, ...)

Standard status-based injuries (e.g. plain "Questionable") bypass Ollama entirely → lower latency, fewer tokens.

#### AI Adjustment Factor

Ollama outputs a `ai_adjustment` float (0.5–1.5) applied to the base impact:

| Adjustment | Meaning |
|---|---|
| 1.3–1.5 | Worse than status indicates (surgery, crutches, indefinite) |
| 1.1–1.2 | Negative trend (limping, swelling, did not practice) |
| 1.0 | Neutral / standard |
| 0.7–0.9 | Positive trend (cleared, intends to play, progressing) |

#### Diff-Check

Each run POSTs all current team impact scores to the Stats API (`/injury_state`). The API compares against the last stored state and returns only teams with changed scores. An alert is only sent when something actually changed — no duplicate noise.

---

## Data Sources

| Source | Data | Notes |
|---|---|---|
| [The-Odds-API](https://the-odds-api.com) | Moneyline, spreads, totals | EU region, decimal format |
| [ESPN API](https://site.api.espn.com/apis/site/v2/sports/basketball/nba/injuries) | Injury reports | Public, no key required |
| Local Stats API | Team stats, H2H, injury state | Flask app on `YOUR_SERVER_IP:5001` |
| `player_data.json` | Player ratings, positions, birth year | Manually maintained, ~70 players |

---

## Setup

### Prerequisites

- n8n (self-hosted, Docker recommended)
- Python 3.10+ with Flask for the Stats API
- Ollama with `llama3.1:8b` or `qwen3.5:9b`
- OpenAI API key (GPT-5 Mini or similar)
- Telegram Bot token + chat ID
- The-Odds-API key

### Import Workflows

1. In n8n: **Workflows → Import from file**
2. Import `NBA Analyse v3 - github.json`
3. Import `NBA Injury Monitor - github.json`
4. Set up credentials in n8n:
   - **OpenAI** — add your API key
   - **Telegram** — add your bot token
   - **Ollama** — point to your Ollama host
5. Update HTTP Request nodes that reference `YOUR_SERVER_IP` with your actual Stats API host
6. In the **Get NBA Odds** node, replace `YOUR_ODDS_API_KEY` with your The-Odds-API key
7. In the Telegram nodes, replace `YOUR_TELEGRAM_CHAT_ID` with your chat ID

### Stats API

The Stats API (`stats_api.py`) exposes:

| Endpoint | Method | Purpose |
|---|---|---|
| `/stats` | POST | Team stats, H2H for a given game |
| `/player_data` | GET | Player ratings and positions |
| `/injury_state` | POST | Diff-check against last stored injury state |

Start it with:
```bash
cd nba-agent
python3 -m venv venv && source venv/bin/activate
pip install flask
nohup venv/bin/python stats_api.py &
```

Keep the database up to date:
```bash
python3 update_db.py
```

---

## Files

| File | Purpose |
|---|---|
| `NBA Analyse v3 - github.json` | Main analysis workflow (n8n export) |
| `NBA Injury Monitor - github.json` | Injury monitoring workflow (n8n export) |
| `stats_api.py` | Flask REST API — serves team stats and injury state |
| `update_db.py` | Incremental DB updater (both seasons) |
| `player_data.json` | Static player data: name, birth_year, net_rating, position |
| `nba.db` | SQLite database (not included in repo) |

---

## Telegram Output Example

```
🏀 NBA VALUE BETS – 09.04.2026
━━━━━━━━━━━━━━━━

🏟 SPIEL: Oklahoma City Thunder vs Denver Nuggets
✅ TIPP: Oklahoma City Thunder
💰 QUOTE: 1.85
📊 KONFIDENZ: 🟢 Hoch
📝 BEGRÜNDUNG: OKC zeigt +6.2 Net-Δ in den letzten 10 Spielen bei vollem Kader;
   Denver spielt Back-to-Back mit Jokic questionable (Kniereizung).
🎯 EMPFEHLUNG: JA SETZEN

📈 Form (letzte 10 | letzte 2):
  Oklahoma City Thunder: 8-2 🔥W5 ⬆️ | Rest: 2T
  Denver Nuggets: 5-5 ❄️L2 ⬇️ ⚠️B2B | Rest: 1T
🤝 H2H: OKC 3-1 Denver | Ø +4.5 OKC | H2H Total: 218.3
📊 Net-Δ: Oklahoma City Thunder +6.2 | Denver Nuggets +0.8
🏥 Verletzungen:
  Denver Nuggets: 12.4 [Off 8.1 | PG 6.5] | Nikola Jokic
📐 Markt-Korrektur: 8.3% | max ±12%

📉 Totals: Under 221.5 | Erwartet: 213.2 | B2B -5.0, PG out -1.3 | 🟡 Mittel
```

---

## Important Notes

- The AI Agent (GPT) is **only allowed to validate and justify** — it must never recalculate probabilities, EV, or the value_side. All math is done deterministically in the workflow nodes before the AI sees it.
- Local server URLs (`YOUR_SERVER_IP`) must be reachable from within your n8n Docker container. If using Docker, the host gateway (e.g. `172.18.0.1`) is often needed instead of `localhost`.
- `player_data.json` needs manual maintenance as rosters change throughout the season.
