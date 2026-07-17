# 💰 Sistema Financeiro

Sistema simples de controle financeiro com cadastro de lançamentos, dashboard com gráficos e lembretes automáticos de cobrança por e-mail.

> 📄 Documento completo de planejamento (arquitetura, modelagem de dados, roadmap): [`docs/mvp.md`](docs/mvp.md)

---

## ✨ Funcionalidades

- ✅ Cadastro de receitas, despesas e cobranças
- ✅ Dashboard com gráficos (totais, evolução mensal, despesas por categoria)
- ✅ Envio automático de e-mail de lembrete antes do vencimento de cobranças
- 🚧 *(futuro)* Importação de CSV com cadastro automático via IA

---

## 🛠️ Stack Técnica

| Camada | Tecnologia |
|---|---|
| Backend | Python 3.11+, Django, Django REST Framework |
| Autenticação | djangorestframework-simplejwt |
| Banco de dados | PostgreSQL |
| Frontend | Streamlit |
| Gráficos | Plotly |
| Envio de e-mail | Django `send_mail` (SMTP) |
| Agendamento | django-crontab (evolução futura: Celery + Redis) |
| Containers | Docker + Docker Compose |

---

## 📁 Estrutura do Projeto

```
sistema-financeiro/
├── backend/
│   ├── core/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── management/commands/enviar_lembretes.py
│   ├── config/          # settings.py, urls.py
│   └── manage.py
├── frontend/
│   └── app.py            # Streamlit
├── docs/
│   └── mvp.md             # documento de planejamento
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 🚀 Como Rodar o Projeto

### Pré-requisitos
- [Docker](https://www.docker.com/) e Docker Compose instalados
- (Alternativa sem Docker) Python 3.11+, PostgreSQL instalado localmente

### 1. Clonar o repositório

```bash
git clone https://github.com/seu-usuario/sistema-financeiro.git
cd sistema-financeiro
```

### 2. Configurar variáveis de ambiente

Copie o arquivo de exemplo e ajuste os valores:

```bash
cp .env.example .env
```

```env
# .env.example
DEBUG=True
SECRET_KEY=troque-esta-chave
DATABASE_URL=postgres://postgres:postgres@db:5432/financeiro

EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=seu-email@gmail.com
EMAIL_HOST_PASSWORD=sua-senha-de-app
```

### 3. Subir os containers

```bash
docker-compose up --build
```

Isso vai subir:
- `db` → PostgreSQL na porta `5432`
- `backend` → API Django na porta `8000`
- `frontend` → Streamlit na porta `8501`

### 4. Rodar as migrations (primeira vez)

```bash
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py createsuperuser
```

### 5. Acessar

- API: [http://localhost:8000/api/](http://localhost:8000/api/)
- Admin Django: [http://localhost:8000/admin/](http://localhost:8000/admin/)
- Dashboard Streamlit: [http://localhost:8501](http://localhost:8501)

---

## 🐳 docker-compose.yml (referência)

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: financeiro
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  backend:
    build: ./backend
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db

  frontend:
    build: ./frontend
    command: streamlit run app.py --server.port 8501 --server.address 0.0.0.0
    volumes:
      - ./frontend:/app
    ports:
      - "8501:8501"
    depends_on:
      - backend

volumes:
  pgdata:
```

---

## ⏰ Lembretes de Cobrança

O envio de lembretes por e-mail roda através de um comando de management:

```bash
docker-compose exec backend python manage.py enviar_lembretes
```

Para automatizar, agende essa execução via cron (ex: todo dia às 8h):

```
0 8 * * * docker-compose exec backend python manage.py enviar_lembretes
```

---

## 🗺️ Roadmap

- [x] Cadastro de lançamentos (receitas, despesas, cobranças)
- [x] Dashboard com gráficos
- [x] Envio de lembrete por e-mail
- [ ] Configuração de regras de lembrete pelo usuário
- [ ] Importação de CSV com cadastro automático via IA
- [ ] Relatórios exportáveis (PDF/Excel)
- [ ] Multiusuário com permissões avançadas

---

## 📄 Licença

Este projeto está sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.
