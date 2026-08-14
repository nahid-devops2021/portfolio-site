+++
title = "Projects"
slug = "projects"
+++

Here are some of the things I've built and maintain. Most are deployed live on my home server and open-source on GitHub.

---

### 📊 Portfolio Tracker
A DSE (Dhaka Stock Exchange) portfolio dashboard with real-time LTP, transaction history, and sector analytics.

- **Stack:** Python, FastAPI, PostgreSQL, Redis, nginx
- **Features:** JWT auth, live market data, buy/sell transactions, avg-cost recompute, portfolio summary, sector allocation charts
- **Live:** `192.168.0.43:8080` | [GitHub](https://github.com/nahid-devops2021/portfolio-tracker)

---

### 💰 Money Manager
Full-stack personal finance management app with budgeting, expense tracking, and automated reporting via Celery workers.

- **Stack:** FastAPI + PostgreSQL + Redis + RabbitMQ + Celery, Next.js + shadcn + Recharts
- **Features:** JWT with TOTP MFA, rate limiting, async worker pipelines, dark fintech UI
- **Live:** `192.168.0.43:8091` | [GitHub](https://github.com/nahid-devops2021/money-management)

---

### 🧠 Paperclip AI Integration
Agent orchestration platform where Paperclip AI agents run through a Hermes gateway for autonomous task execution.

- **Stack:** Python, Docker, REST APIs
- **Features:** Custom CEO/Reflection Coach/Summarizer agents, hermes_gateway adapter, LAN-hosted

---

### 🌐 Portfolio Site
This very website — a personal portfolio and blog powered by Hugo.

- **Stack:** Hugo, hugo-coder theme, GitHub Actions, nginx Docker
- **Features:** Auto-deploys to GitHub Pages on push; also served from home server
- **Live:** [GitHub Pages](https://nahid-devops2021.github.io/portfolio-site/) | [GitHub](https://github.com/nahid-devops2021/portfolio-site)

---

> 💡 *All projects are continuously evolving. Check the GitHub repos for the latest updates.*
