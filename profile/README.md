<div align="center">

# Atlas

Plataforma acadêmica de trilhas de conhecimento e bolsas com avaliação automática de código via IA.  
Desenvolvida para o **IFRN Campus Pau dos Ferros** como Projeto Integrador de Sistemas Distribuídos.

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-092E20?style=flat&logo=django&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=flat&logo=typescript&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)

</div>

---

## Repositórios

| Serviço | Descrição | Stack |
|---|---|---|
| [`atlas-auth-service`](../atlas-auth-service) | Autenticação OAuth2 com SUAP, geração e validação de JWT | Django |
| [`atlas-tracks-service`](../atlas-tracks-service) | Trilhas de conhecimento, módulos, desafios e progresso | Django |
| [`atlas-scholarship-service`](../atlas-scholarship-service) | Bolsas, candidaturas, talentos e pontuação | Django |
| [`atlas-ai-service`](../atlas-ai-service) | Análise de repositórios GitHub via LLM | FastAPI |
| [`atlas-frontend-service`](../atlas-frontend-service) | SPA para alunos e professores | React + TypeScript |

---

## Arquitetura

10 containers Docker orquestrados via `docker-compose`. O Nginx é o único ponto de entrada externo — valida tokens via `auth_request` antes de repassar qualquer requisição.

```
[Cliente] → Nginx (80/443)
               ├── auth_request → auth-service:8000
               ├── tracks-service:8001
               ├── scholarship-service:8002
               └── ia-service:8003
```

- **Banco de dados** — PostgreSQL único com schemas separados (`auth`, `tracks`, `scholarship`). Dados cruzados trafegam exclusivamente via API HTTP interna.
- **Mensageria** — RabbitMQ + Celery para análise de IA, notificações e tarefas de background.
- **Cache** — Redis armazena validações de token por 60s, sessões e rate limit.

---

## Stack

**Backend** — Python · Django REST Framework · Gunicorn · httpx  
**Frontend** — React · TypeScript  
**Infra** — Docker · Nginx · PostgreSQL 16 · Redis 7 · RabbitMQ 3  
**Auth** — OAuth2 com SUAP · JWT  
**CI/CD** — GitHub Actions

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
