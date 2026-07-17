# MVP — Sistema Financeiro (Django REST + Streamlit)

## 1. Visão Geral

Sistema financeiro simples para controle de lançamentos (receitas/despesas e cobranças), com:

- **Backend**: Django REST Framework (API)
- **Frontend**: Streamlit (dashboard e formulários)
- **Banco de dados**: PostgreSQL
- **Notificações**: envio automático de e-mail de lembrete de cobrança
- **Futuro**: módulo de IA para cadastro automático via upload de CSV

---

## 2. Objetivo do MVP

Permitir que o usuário:

1. Cadastre lançamentos financeiros (receitas, despesas, cobranças) em uma tela.
2. Visualize gráficos e indicadores em outra aba do dashboard.
3. Receba (e o sistema envie) lembretes de cobrança por e-mail antes do vencimento.
4. No futuro, suba um CSV e a IA cadastre os lançamentos automaticamente.

---

## 3. Banco de Dados: PostgreSQL

O sistema usa **PostgreSQL** como banco de dados único, pelos seguintes motivos:

- **Integração nativa com o Django ORM** — sem necessidade de bibliotecas extras (`djongo`, `mongoengine` etc.).
- **Dados financeiros são naturalmente relacionais** — usuário → lançamentos → categorias, com chaves estrangeiras e integridade referencial.
- **Relatórios e agregações** (soma por categoria, evolução mensal, saldo) são simples e eficientes com `SUM`, `GROUP BY` e as ferramentas de agregação do próprio ORM (`aggregate`, `annotate`).
- **Menor curva de aprendizado** para o MVP, com ampla documentação e suporte no ecossistema Django.

---

## 4. Arquitetura

```
┌─────────────────┐        HTTP/REST (JSON)        ┌──────────────────────┐
│    Streamlit     │ ──────────────────────────────▶ │   Django REST API     │
│  (Frontend/UI)    │ ◀────────────────────────────── │  (Backend/regra neg.) │
└─────────────────┘                                  └──────────┬───────────┘
                                                                  │
                                                        ┌─────────▼─────────┐
                                                        │   PostgreSQL       │
                                                        └────────────────────┘
                                                                  │
                                                        ┌─────────▼─────────┐
                                                        │ Celery + Redis     │
                                                        │ (agendamento de     │
                                                        │  e-mails de cobrança)│
                                                        └────────────────────┘
```

- **Streamlit** consome a API via `requests`.
- **Django REST** cuida de autenticação, regras de negócio e persistência no PostgreSQL.
- **Celery + Redis** (ou `django-crontab`/`APScheduler` para simplificar no MVP) cuidam do envio agendado de e-mails.

---

## 5. Modelagem de Dados (MVP)

### 5.1 Entidades principais

**Usuário** (pode usar o `User` padrão do Django)
- id, nome, email, senha

**Categoria**
- id, nome, tipo (`receita` / `despesa`)

**Lançamento**
- id
- usuário (FK)
- descrição
- valor (decimal)
- tipo (`receita` / `despesa` / `cobrança`)
- categoria (FK)
- data_lancamento
- data_vencimento (nullable — usado para cobranças)
- status (`pendente` / `pago` / `atrasado`)
- email_cliente (nullable — para quem vai a cobrança)
- lembrete_enviado (boolean, default False)

**LembreteConfig** (opcional no MVP)
- dias_antes_vencimento (ex: 3 dias antes)
- ativo (boolean)

### 5.2 Exemplo simplificado (Django models.py)

```python
from django.db import models
from django.contrib.auth.models import User

class Categoria(models.Model):
    TIPO_CHOICES = [("receita", "Receita"), ("despesa", "Despesa")]
    nome = models.CharField(max_length=100)
    tipo = models.CharField(max_length=10, choices=TIPO_CHOICES)

class Lancamento(models.Model):
    TIPO_CHOICES = [
        ("receita", "Receita"),
        ("despesa", "Despesa"),
        ("cobranca", "Cobrança"),
    ]
    STATUS_CHOICES = [
        ("pendente", "Pendente"),
        ("pago", "Pago"),
        ("atrasado", "Atrasado"),
    ]
    usuario = models.ForeignKey(User, on_delete=models.CASCADE)
    descricao = models.CharField(max_length=255)
    valor = models.DecimalField(max_digits=10, decimal_places=2)
    tipo = models.CharField(max_length=10, choices=TIPO_CHOICES)
    categoria = models.ForeignKey(Categoria, on_delete=models.SET_NULL, null=True)
    data_lancamento = models.DateField(auto_now_add=True)
    data_vencimento = models.DateField(null=True, blank=True)
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default="pendente")
    email_cliente = models.EmailField(null=True, blank=True)
    lembrete_enviado = models.BooleanField(default=False)

    def __str__(self):
        return f"{self.descricao} - R$ {self.valor}"
```

---

## 6. API (endpoints principais)

| Método | Endpoint | Descrição |
|---|---|---|
| POST | `/api/auth/login/` | Login (token/JWT) |
| GET | `/api/lancamentos/` | Lista lançamentos (com filtros: tipo, status, data) |
| POST | `/api/lancamentos/` | Cria lançamento |
| PUT/PATCH | `/api/lancamentos/{id}/` | Edita lançamento |
| DELETE | `/api/lancamentos/{id}/` | Remove lançamento |
| GET | `/api/categorias/` | Lista categorias |
| GET | `/api/dashboard/resumo/` | Retorna totais (receitas, despesas, saldo) para os gráficos |
| GET | `/api/dashboard/por-categoria/` | Retorna soma agrupada por categoria |
| GET | `/api/dashboard/evolucao-mensal/` | Retorna série temporal (mês a mês) |
| POST | `/api/cobrancas/enviar-lembretes/` | Dispara manualmente o envio de lembretes (uso interno/cron) |
| POST | `/api/importar-csv/` *(futuro/IA)* | Upload de CSV para cadastro automático |

Recomenda-se usar **Django REST Framework `ViewSets` + `Router`** para reduzir boilerplate, e **JWT (djangorestframework-simplejwt)** para autenticação.

---

## 7. Streamlit — Telas do MVP

Usar `st.sidebar` ou `st.tabs` para navegação simples entre abas.

### Aba 1 — Cadastro
- Formulário (`st.form`) com: descrição, valor, tipo, categoria, data de vencimento, e-mail do cliente (se cobrança).
- Botão "Salvar" → `POST /api/lancamentos/`
- Tabela (`st.dataframe`) com os lançamentos existentes, permitindo editar/excluir.

### Aba 2 — Dashboard / Gráficos
- Cards com totais: Receitas, Despesas, Saldo (`st.metric`)
- Gráfico de pizza: despesas por categoria (`plotly` ou `st.bar_chart`)
- Gráfico de linha: evolução mensal de receitas x despesas
- Tabela de cobranças pendentes/atrasadas, com destaque visual

### Aba 3 — Cobranças / Lembretes
- Lista de cobranças com status e data de vencimento
- Botão para forçar envio manual de lembrete
- (Futuro) configuração de quantos dias antes enviar o lembrete

### Aba 4 — Importar CSV *(futuro, com IA)*
- Upload de arquivo `.csv`
- Preview dos dados
- Botão "Cadastrar automaticamente" → IA interpreta colunas e cria lançamentos

**Exemplo simplificado de chamada da API no Streamlit:**

```python
import streamlit as st
import requests

API_URL = "http://localhost:8000/api"

st.set_page_config(page_title="Sistema Financeiro", layout="wide")

tab1, tab2, tab3 = st.tabs(["📝 Cadastro", "📊 Dashboard", "📧 Cobranças"])

with tab1:
    with st.form("novo_lancamento"):
        descricao = st.text_input("Descrição")
        valor = st.number_input("Valor", min_value=0.0, step=0.01)
        tipo = st.selectbox("Tipo", ["receita", "despesa", "cobranca"])
        enviar = st.form_submit_button("Salvar")
        if enviar:
            resp = requests.post(f"{API_URL}/lancamentos/", json={
                "descricao": descricao, "valor": valor, "tipo": tipo
            })
            st.success("Lançamento salvo!") if resp.ok else st.error("Erro ao salvar")

with tab2:
    resumo = requests.get(f"{API_URL}/dashboard/resumo/").json()
    col1, col2, col3 = st.columns(3)
    col1.metric("Receitas", f"R$ {resumo['receitas']:.2f}")
    col2.metric("Despesas", f"R$ {resumo['despesas']:.2f}")
    col3.metric("Saldo", f"R$ {resumo['saldo']:.2f}")
```

---

## 8. Envio de E-mail de Lembrete de Cobrança

### Opção simples para o MVP (sem Celery)
Usar **`django-crontab`** ou um comando de management (`manage.py`) agendado via **cron do sistema operacional** (ou tarefa agendada do Windows), rodando 1x por dia.

```python
# management/commands/enviar_lembretes.py
from django.core.management.base import BaseCommand
from django.core.mail import send_mail
from django.utils import timezone
from datetime import timedelta
from core.models import Lancamento

class Command(BaseCommand):
    def handle(self, *args, **kwargs):
        limite = timezone.now().date() + timedelta(days=3)
        cobrancas = Lancamento.objects.filter(
            tipo="cobranca", status="pendente",
            data_vencimento__lte=limite,
            lembrete_enviado=False
        )
        for c in cobrancas:
            send_mail(
                subject="Lembrete de Cobrança",
                message=f"Olá, sua cobrança '{c.descricao}' de R$ {c.valor} vence em {c.data_vencimento}.",
                from_email="financeiro@empresa.com",
                recipient_list=[c.email_cliente],
            )
            c.lembrete_enviado = True
            c.save()
```

Executar via cron: `0 8 * * * python manage.py enviar_lembretes`

### Opção robusta (recomendada para evoluir)
**Celery + Redis + Celery Beat** — permite agendamento mais flexível, reenvio, filas, retries em caso de falha de envio.

---

## 9. Roadmap Futuro — Módulo de IA (pós-MVP)

**Objetivo**: usuário sobe um CSV e a IA identifica e cadastra os lançamentos automaticamente.

Etapas sugeridas:

1. **Parsing do CSV** (`pandas`) para leitura bruta.
2. **Mapeamento inteligente de colunas** — usar modelo de linguagem (ex: API da Anthropic/Claude) para identificar quais colunas correspondem a `descrição`, `valor`, `data`, `categoria`, mesmo que o CSV tenha nomes de colunas diferentes ou fora de ordem.
3. **Classificação automática de categoria** — a IA sugere a categoria com base na descrição (ex: "Uber" → Transporte).
4. **Tela de revisão** — antes de gravar, mostrar preview editável ao usuário (evita erros da IA irem direto pro banco).
5. **Endpoint dedicado**: `POST /api/importar-csv/` recebe o arquivo, chama o serviço de IA, retorna JSON estruturado para revisão no Streamlit.

Essa etapa é isolada do MVP para não travar a entrega inicial — o cadastro manual continua funcionando normalmente.

---

## 10. Stack Técnica Resumida

| Camada | Tecnologia |
|---|---|
| Backend | Python 3.11+, Django, Django REST Framework |
| Autenticação | djangorestframework-simplejwt |
| Banco de dados | PostgreSQL |
| Frontend | Streamlit |
| Gráficos | Plotly / st.bar_chart / st.line_chart |
| Agendamento de e-mail | django-crontab (MVP) → Celery + Redis (evolução) |
| Envio de e-mail | Django `send_mail` (SMTP — Gmail, SendGrid, etc.) |
| IA (futuro) | API Claude (Anthropic) + pandas |
| Deploy sugerido | Docker Compose (django + streamlit + postgres) |

---

## 11. Estrutura de Pastas Sugerida

```
sistema-financeiro/
├── backend/
│   ├── core/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── management/commands/enviar_lembretes.py
│   ├── config/ (settings.py, urls.py)
│   └── manage.py
├── frontend/
│   └── app.py  (Streamlit)
├── docker-compose.yml
└── README.md
```

---

## 12. Escopo do MVP (o que entra x o que fica pra depois)

**Entra no MVP:**
- Cadastro/edição/exclusão de lançamentos
- Dashboard com gráficos básicos (pizza + linha do tempo + totais)
- Envio de e-mail de lembrete (via cron simples)
- Autenticação básica

**Fica para depois (v2):**
- Celery/Redis para agendamento robusto
- Configuração de regras de lembrete pelo usuário (dias antes, múltiplos lembretes)
- Módulo de IA para importação de CSV
- Multiempresa / multiusuário com permissões avançadas
- Relatórios exportáveis (PDF/Excel)

---

## 13. Próximos Passos

1. Criar projeto Django + app `core` com os models acima.
2. Configurar PostgreSQL e rodar migrations.
3. Criar serializers e viewsets básicos.
4. Criar projeto Streamlit com as 3 abas (Cadastro, Dashboard, Cobranças).
5. Configurar envio de e-mail (SMTP) e testar comando `enviar_lembretes`.
6. Rodar cron/agendador local.
7. Validar fluxo ponta a ponta com dados de teste.
8. Só depois, iniciar o módulo de IA para CSV.
