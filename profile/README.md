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

</div>

---

## Repositórios

| Repositório | Descrição | Stack |
|---|---|---|
| [`atlas-infra`](../atlas-infra) | Orquestração: Docker Compose, Nginx, scripts de deploy e backup | Docker · Nginx |
| [`atlas-auth-service`](../atlas-auth-service) | Autenticação OAuth2 com SUAP, JWT e perfil via gRPC | Django |
| [`atlas-track-service`](../atlas-track-service) | Trilhas de conhecimento, módulos, desafios e progresso | Django · Celery |
| [`atlas-scholarship-service`](../atlas-scholarship-service) | Bolsas, candidaturas, talentos e pontuação | Django |
| [`atlas-ai-service`](../atlas-ai-service) | Análise de repositórios GitHub via LLM local (Ollama) | FastAPI |
| [`atlas-frontend`](../atlas-frontend) | SPA para alunos e professores | React · TypeScript |

---

## Arquitetura

Containers Docker orquestrados pelo [`atlas-infra`](../atlas-infra). O **Nginx** é o único ponto de entrada externo: roteia por prefixo de path. Os serviços Django escutam todos na porta `8000` interna e são distinguidos pelo nome do container na rede Docker.

```
Internet :80/:443
      │
      ▼
   ┌─────────┐   /api/auth/    → auth-service:8000
   │  Nginx  │   /api/tracks/  → tracks-service:8000
   │ gateway │   /api/bolsas/  → scholarship-service:8000
   └─────────┘   /api/ia/      → ai-service:8003
      │           /            → frontend:80
      └── repassa Authorization adiante
```

- **Autenticação** — a validação do token JWT é feita via **gRPC** dentro de cada serviço, consultando `auth-service:50051`. O bloco `auth_request` no Nginx fica pré-configurado e comentado, para uso futuro como barreira na borda.
- **Banco de dados** — PostgreSQL único com três schemas isolados (`auth`, `tracks`, `scholarship`). Cada serviço seleciona seu schema por conexão via `search_path` (variável `PGOPTIONS`). Nenhum serviço acessa o schema de outro diretamente — dados cruzados passam pela API.
- **Mensageria** — RabbitMQ + Celery para tarefas de background (no momento, presente no serviço de trilhas).
- **Cache** — Redis para dados voláteis (validações de token, sessões, rate limit).
- **IA** — `ai-service` consome um LLM local servido pelo container **Ollama**.

---

## Stack

**Backend** — Python · Django REST Framework · FastAPI · httpx · gRPC  
**Frontend** — React · TypeScript · Vite  
**Infra** — Docker · Nginx · PostgreSQL 16 · Redis 7 · RabbitMQ 3 · Ollama  
**Auth** — OAuth2 com SUAP · JWT  
**CI/CD** — GitHub Actions

---

## Como rodar

Todo o ambiente é orquestrado pelo [`atlas-infra`](../atlas-infra). Os serviços são buildados a partir do código-fonte local — clone cada repositório nas pastas esperadas.

### 1. Estrutura de pastas

```
atlas/
├── docker-compose.yml          (do atlas-infra)
├── docker-compose.dev.yml
├── .env
├── nginx/  postgres/  scripts/
├── frontend/                   ← clone de atlas-frontend
└── services/
    ├── auth/                   ← clone de atlas-auth-service
    ├── track/                  ← clone de atlas-track-service
    ├── scholarship/            ← clone de atlas-scholarship-service
    └── ai/                     ← clone de atlas-ai-service
```

### 2. Clonar tudo

```bash
git clone https://github.com/Atlas-IFRN/atlas-infra atlas
cd atlas

git clone https://github.com/Atlas-IFRN/atlas-frontend            frontend
git clone https://github.com/Atlas-IFRN/atlas-auth-service        services/auth
git clone https://github.com/Atlas-IFRN/atlas-track-service       services/track
git clone https://github.com/Atlas-IFRN/atlas-scholarship-service services/scholarship
git clone https://github.com/Atlas-IFRN/atlas-ai-service          services/ai
```

### 3. Configurar variáveis

```bash
cp .env.example .env
# edite .env: senhas do Postgres/RabbitMQ, credenciais do SUAP, secret key
```

### 4. Subir

```bash
docker compose up -d --build
```

> O primeiro `up` demora: o container `ollama-pull` baixa o modelo LLM (~1 GB) e o `ai-service` aguarda o modelo ficar pronto antes de iniciar.

A aplicação fica acessível em `http://localhost` (ou no domínio configurado).

### Desenvolvimento local

Para rodar só a infraestrutura compartilhada (Postgres, Redis, RabbitMQ, Ollama) e executar os serviços manualmente:

```bash
docker compose -f docker-compose.dev.yml up -d
```

Cada serviço Django roda com `python manage.py runserver` e o frontend com `npm run dev`, apontando para essa infra. Consulte o README de cada repositório.

---

## Personas

| Persona | Descrição |
|---|---|
| **Aluno** | Consome trilhas de conhecimento, resolve desafios e se candidata a bolsas |
| **Professor** | Gerencia trilhas, cria vagas e avalia talentos com apoio da IA |

---

<div align="center">
  <sub>IFRN Campus Pau dos Ferros · Projeto Integrador — Sistemas Distribuídos</sub>
</div>
