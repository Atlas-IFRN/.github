<div align="center">

# Atlas

Plataforma acadêmica de trilhas de conhecimento e bolsas com avaliação automática de código via IA.  
Desenvolvida para o **IFRN Campus Pau dos Ferros** como Projeto Integrador de Sistemas Distribuídos.

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-092E20?style=flat&logo=django&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=flat&logo=typescript&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?style=flat&logo=rabbitmq&logoColor=white)

</div>

## Repositórios

| Repositório                                                                              | Descrição                                                                                   | Stack                | Documentação                                                          |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------- | --------------------------------------------------------------------- |
| [`atlas-infra`](https://github.com/Atlas-IFRN/atlas-infra)                               | Orquestração: Docker Compose, Nginx (gateway), PostgreSQL, Redis, RabbitMQ, deploy e backup | Docker · Nginx       | [Wiki](https://github.com/Atlas-IFRN/atlas-infra/wiki)                |
| [`atlas-auth-service`](https://github.com/Atlas-IFRN/atlas-auth-service)                 | Autenticação OAuth2 com SUAP, emissão e validação de JWT e perfis de usuário                | Django               | [Wiki](https://github.com/Atlas-IFRN/atlas-auth-service/wiki)         |
| [`atlas-track-service`](https://github.com/Atlas-IFRN/atlas-track-service)               | Trilhas, módulos, conteúdos, progresso e submissão de desafios                              | Django · Celery      | [Wiki](https://github.com/Atlas-IFRN/atlas-track-service/wiki)        |
| [`atlas-scholarship-service`](https://github.com/Atlas-IFRN/atlas-scholarship-service)   | Bolsas, candidaturas, banco de talentos e notas                                             | Django · Celery      | [Wiki](https://github.com/Atlas-IFRN/atlas-scholarship-service/wiki)  |
| [`atlas-feed-service`](https://github.com/Atlas-IFRN/atlas-feed-service)                 | Feed institucional: posts, comentários, curtidas e banners                                  | Django               | [Wiki](https://github.com/Atlas-IFRN/atlas-feed-service/wiki)         |
| [`atlas-notification-service`](https://github.com/Atlas-IFRN/atlas-notification-service) | Notificações — consumidor central via RabbitMQ                                              | Django · Celery      | [Wiki](https://github.com/Atlas-IFRN/atlas-notification-service/wiki) |
| [`atlas-ai-service`](https://github.com/Atlas-IFRN/atlas-ai-service)                     | Avaliação de repositórios GitHub via LLM local com Ollama                                   | FastAPI              | [Wiki](https://github.com/Atlas-IFRN/atlas-ai-service/wiki)           |
| [`atlas-frontend`](https://github.com/Atlas-IFRN/atlas-frontend)                         | SPA para alunos e professores                                                               | React · TypeScript   | [Wiki](https://github.com/Atlas-IFRN/atlas-frontend/wiki)             |
| [`atlas-observability`](https://github.com/Atlas-IFRN/atlas-observability)               | Coleta e visualização das métricas dos serviços                                             | Prometheus · Grafana | [Wiki](https://github.com/Atlas-IFRN/atlas-observability/wiki)        |


---

## Visão geral

O Atlas é uma plataforma de microsserviços onde **alunos** percorrem trilhas de conhecimento, resolvem desafios de código avaliados por IA e se candidatam a bolsas, enquanto **professores** gerenciam trilhas, publicam vagas e avaliam talentos. Tudo é orquestrado em containers Docker, com o **Nginx** como único ponto de entrada externo.

---

## Arquitetura

Containers Docker orquestrados pelo [`atlas-infra`](../atlas-infra). O **Nginx** é o único ponto de entrada externo e roteia por prefixo de path. Os serviços Django escutam na porta `8000` interna, distinguidos pelo nome do container na rede Docker.

```
Internet :80/:443
      │
      ▼
   ┌─────────┐   /api/auth/          → auth-service:8000
   │  Nginx  │   /api/track/         → tracks-service:8000
   │ gateway │   /api/scholarship/   → scholarship-service:8000
   │ (auth_  │   /api/feed/          → feed-service:8000
   │ request)│   /api/notifications/ → notification-service:8000
   └─────────┘   /api/ai/            → ai-service:8003
      │           /                  → frontend:80
      ▼
  valida o JWT na borda (→ auth-service/internal/validate/)
  e injeta X-User-Id / X-User-Role
```

- **Autenticação** — o Nginx valida o JWT **na borda** via `auth_request` (chamando `auth-service/api/auth/internal/validate/`) e injeta `X-User-Id` / `X-User-Role`. Cada serviço **também** valida o token localmente de forma *stateless* (SimpleJWT), lendo os claims (`role`, `ira`, `is_staff`, ...) sem novas chamadas de rede. A chave de assinatura (`DJANGO_SECRET_KEY`) é compartilhada entre os serviços.
- **Banco de dados** — PostgreSQL 16 único com cinco schemas isolados (`auth`, `tracks`, `scholarship`, `notification`, `feed`). Cada serviço seleciona seu schema por conexão via `search_path` (`PGOPTIONS`). Nenhum serviço acessa o schema de outro — dados cruzados passam pela API HTTP interna.
- **Mensageria** — RabbitMQ + Celery para trabalho assíncrono. Auth, feed, tracks e scholarship **publicam** eventos (`notifications.create`); o **notification-service é o consumidor central**. O tracks também usa uma fila dedicada para a avaliação de desafios.
- **IA** — o `ai-service` (FastAPI) avalia repositórios com um LLM local servido pelo container **Ollama**; é acionado de forma assíncrona pelo tracks quando um aluno submete um desafio.
- **Cache** — Redis para dados voláteis (sessões, rate limit, cache).
- **Observabilidade & auditoria** — cada serviço expõe `/metrics` (coletado pelo [`atlas-observability`](../atlas-observability)) e mantém um `AuditLog` com registro automático de operações. Backups do Postgres via `pg_dump` agendado no `atlas-infra`.

---

## Stack

**Backend** — Python · Django REST Framework · FastAPI · httpx / requests · Celery  
**Frontend** — React · TypeScript · Vite · Material UI · TanStack Query  
**Infra** — Docker · Nginx · PostgreSQL 16 · Redis 7 · RabbitMQ 3 · Ollama  
**Auth** — OAuth2 com SUAP · JWT (SimpleJWT)  
**Observabilidade** — Prometheus · Grafana · pg_stat_statements  
**CI/CD** — GitHub Actions

---

## Como rodar

Todo o ambiente é orquestrado pelo [`atlas-infra`](../atlas-infra). Os serviços são buildados a partir do código-fonte local — clone cada repositório nas pastas esperadas.

### 1. Estrutura de pastas

```
atlas/                              (clone de atlas-infra)
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env
├── nginx/  postgres/  scripts/
├── frontend/                       ← clone de atlas-frontend
└── services/
    ├── auth/                       ← clone de atlas-auth-service
    ├── track/                      ← clone de atlas-track-service
    ├── scholarship/                ← clone de atlas-scholarship-service
    ├── feed/                       ← clone de atlas-feed-service
    ├── notification/               ← clone de atlas-notification-service
    └── ai/                         ← clone de atlas-ai-service
```

### 2. Clonar tudo

```bash
git clone https://github.com/Atlas-IFRN/atlas-infra atlas
cd atlas

git clone https://github.com/Atlas-IFRN/atlas-frontend             frontend
git clone https://github.com/Atlas-IFRN/atlas-auth-service         services/auth
git clone https://github.com/Atlas-IFRN/atlas-track-service        services/track
git clone https://github.com/Atlas-IFRN/atlas-scholarship-service  services/scholarship
git clone https://github.com/Atlas-IFRN/atlas-feed-service         services/feed
git clone https://github.com/Atlas-IFRN/atlas-notification-service services/notification
git clone https://github.com/Atlas-IFRN/atlas-ai-service           services/ai
```

> O `scripts/deploy.sh` do `atlas-infra` automatiza esse clone/atualização no servidor.

### 3. Configurar variáveis

```bash
cp .env.example .env
# edite .env: senhas do Postgres/RabbitMQ, credenciais do SUAP, DJANGO_SECRET_KEY compartilhada
```

### 4. Subir

```bash
docker compose up -d --build
```

> O primeiro `up` demora: o container `ollama-pull` baixa o modelo LLM e o `ai-service` aguarda o modelo ficar pronto antes de iniciar.

A aplicação fica acessível em `http://localhost` (ou no domínio configurado).

### Desenvolvimento local

Para rodar só a infraestrutura compartilhada (Postgres, Redis, RabbitMQ, Ollama) e executar os serviços manualmente:

```bash
docker compose -f docker-compose.dev.yml up -d
```

Cada serviço Django roda com `python manage.py runserver`, o `ai-service` com `uvicorn` e o frontend com `npm run dev`. Consulte o README de cada repositório.

---

## Personas

| Persona | Descrição |
|---|---|
| **Aluno** | Consome trilhas de conhecimento, resolve desafios avaliados por IA e se candidata a bolsas |
| **Professor** | Gerencia trilhas, cria vagas e avalia talentos com apoio da IA |

---

<div align="center">
  <sub>IFRN Campus Pau dos Ferros · Projeto Integrador — Sistemas Distribuídos</sub>
</div>
