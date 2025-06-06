
# 📈 Multi-Service Stocks App

A container-first micro‑services system for tracking stock positions, computing capital gains, and exposing a REST API — all runnable with a single **`docker compose up`**.

> **Stack**: Flask • MongoDB • Nginx • Docker Compose  
> **Services**: `stocks-service` × 3 (instances) • `capital-gains-service` • MongoDB • Nginx LB/edge  

---

## ✨ Features
* **CRUD stocks** – add, update, and delete ticker holdings.
* **Per‑ticker value** and **full portfolio value** endpoints.
* **Capital‑gains calculator** – aggregates data from two logical portfolios.
* **Horizontal scaling** – two weighted instances (`stocks1-a`, `stocks1-b`) behind Nginx plus an independent `stocks2`.
* **Graceful restart demo** – hit `/kill` to force a container exit and watch Docker restart it.
* **Stateless containers** – all persistence lives in the `mongodb` volume.

---

## 🗺️ Architecture

```text
                        ┌──────────────┐
                        │  Nginx 80    │
                        │ reverse‑proxy│
                        └───┬─────┬────┘
            /stocks1...     │     │      /stocks2...
      (weighted RR)         │     │
 ┌──────────────┐      ┌────▼────┐│
 │ stocks1‑a    │      │stocks1‑b││
 │ Flask @8000  │◀────►│Flask @8000│
 └──────────────┘      └──────────┘│
                                   │
                                   │
                              ┌────▼──────┐
                              │ stocks2   │
                              │ Flask @8000│
                              └────┬──────┘
                                   │
          REST calls               │
                                   ▼
                         ┌──────────────────┐
                         │ capital‑gains    │
                         │ Flask @8080      │
                         └────────┬─────────┘
                                  │
                            Mongo queries
                                  ▼
                         ┌──────────────────┐
                         │    MongoDB       │
                         │  replica vol     │
                         └──────────────────┘
```

* **Load‑balancing** – `stocks1-a` weight 3, `stocks1-b` weight 1 (defined in `nginx/nginx.conf`).  
* **Isolation** – each logical portfolio (`stocks1`, `stocks2`) uses its own MongoDB database.  
* **Network** – single user‑defined bridge `app-network`.

---

## 🏗️ Quick start

```bash
git clone https://github.com/EylonIyov/Stocks-App.git
cd Stocks-App/assignment2          # repo root that holds docker-compose.yml
docker compose up --build          # starts Mongo, three Flask services & Nginx
```

First run pulls the `mongo` and `nginx` images, builds the two Python services, and seeds nothing — feel free to POST your own positions straight away.

### Health‑check

```bash
curl http://localhost/stocks1/stocks          # through Nginx to stocks1 backend
curl http://localhost/stocks2/stocks          # through Nginx to stocks2
curl http://localhost/capital-gains?years=1&inflation=0.03   # CG endpoint
```

---

## ⚙️ Environment variables

| Service | Variable | Default | Purpose |
|---------|----------|---------|---------|
| **stocks-service** | `MONGODB_URI` | `mongodb://mongodb:27017/stocks1` | Points to the correct database (`stocks1` or `stocks2`) |
|  | `SERVICE_NAME` | `stocks1` | Label returned by `/stocks` for clarity |
|  | `PORT` | `8000` | Container port (mapped to 5001 / 5002 on host) |
| **capital-gains-service** | `STOCKS1_A_URL` | `http://stocks1-a:8000` | Internal URL to first backend |
|  | `STOCKS1_B_URL` | `http://stocks1-b:8000` | (not called directly — Nginx handles RR) |
|  | `STOCKS2_URL` | `http://stocks2:8000` | Internal URL to second portfolio svc |
| **docker volume** | `mongodb_data` | — | Persists the DB between restarts |

All of these are already defined inline in `docker-compose.yml`, so **no extra setup** is required for local development.

---

## 🛣️ API Reference

### `stocks-service` (endpoints exposed *per* instance)

| Method | Path | Description |
|--------|------|-------------|
| **GET** | `/stocks` | List all stored tickers |
| **POST** | `/stocks` | Add a new stock `{"ticker": "...", "amount": 15, "buy_price": 123.45}` |
| **GET** | `/stocks/<id>` | Retrieve one stock by Mongo `_id` |
| **PUT** | `/stocks/<id>` | Update amount / price |
| **DELETE** | `/stocks/<id>` | Remove from portfolio |
| **GET** | `/stock-value/<id>` | Current market value via Alpha Vantage |
| **GET** | `/portfolio-value` | Total portfolio valuation |
| **GET** | `/kill` | `exit(1)` – Docker will auto‑restart |

### `capital-gains-service`

| Method | Path | Query params | Description |
|--------|------|--------------|-------------|
| **GET** | `/capital-gains` | `years` (int), `inflation` (float, optional) | Returns net capital gain across both portfolios, adjusted for inflation. |

_Note: restricted paths (`/stock-value`, `/portfolio-value`) are **blocked** at the proxy level (403) to enforce separation of concerns._

---

## 🧑‍💻 Development

```bash
# run only one backend for iterative work
docker compose up --build stocks1-a

# install deps in a venv instead of Docker
pip install -r stocks-service/requirements.txt
export MONGODB_URI=mongodb://localhost:27017/devdb
python stocks-service/app.py
```

* Lint / type‑check locally with **ruff** and **mypy** (coming soon).  
* Unit tests can be added with **pytest**; mount a volume into the container or run them on the host.

---

## ➡️ Roadmap

- [ ] WebSocket stream for live tick data  
- [ ] CI pipeline (GitHub Actions)  
- [ ] Autoscaling demo with Docker Swarm / Kubernetes  
- [ ] Front‑end dashboard (React + Recharts)

---

## 🤝 Contributing

1. Fork → create a feature branch → commit → open PR.  
2. Please follow PEP‑8 and include tests for any new behaviour.

---

## 📝 License

This repository is released under the **MIT License** – see [`LICENSE`](LICENSE) for details.

---

> _Built with ❤️ by **Eylon Iyov** – Reichman University, 2025_
