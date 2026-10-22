# FinTrack — Financial Transfers API

[![Python](https://img.shields.io/badge/Python-3.11-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-009688.svg)](https://fastapi.tiangolo.com/)
[![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0+-red.svg)](https://www.sqlalchemy.org/)
[![Architecture](https://img.shields.io/badge/Architecture-Clean%20Architecture-brightgreen.svg)]()
[![Code style: ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://github.com/astral-sh/ruff)
[![Type Checked: mypy](https://img.shields.io/badge/mypy-strict-blue.svg)](http://mypy-lang.org/)

API REST de alta confiabilidade para transferências financeiras entre contas bancárias, projetada seguindo **Clean Architecture**, **Domain-Driven Design (DDD)** tático e padrões de engenharia de software para sistemas financeiros de missão crítica.

---

## 1. Visão Geral e Funcionalidades

- **Cadastro de Contas:** Criação de contas correntes com titular e saldo inicial opcional (validado como não-negativo).
- **Consulta de Saldo e Detalhes:** Recuperação atômica do estado atual da conta.
- **Transferência entre Contas:**
  - Validação estrita de regras de negócio (valor estritamente positivo, saldo suficiente, contas distintas).
  - **Atomicidade Financeira:** Débito e crédito executados em uma única transação de banco de dados via *Unit of Work*.
  - **Controle de Concorrência e Prevenção de Deadlock:** Bloqueio pessimista (`SELECT ... FOR UPDATE`) com ordenação determinística de identificadores (`sorted([source_id, dest_id])`).
  - **Idempotência Segura:** Chave de idempotência única por transação. Repetições idênticas retornam a transação original sem reprocessar débito/crédito; reutilização de chave com parâmetros conflitantes resulta em `HTTP 409 Conflict`.
  - **Precisão Monetária:** *Value Object* `Money` encapsulando `Decimal` com arredondamento bancário (*half-even*) e duas casas decimais, eliminando erros de ponto flutuante IEEE 754.

---

## 2. Arquitetura e Estrutura de Diretórios

O projeto implementa rigorosamente a **Clean Architecture (Onion/Hexagonal)** com fluxo de dependências direcionado exclusivamente para o centro (Domínio):

```
                    ┌─────────────────────────┐
                    │   Presentation (API)    │
                    │   FastAPI, Schemas      │
                    └───────────┬─────────────┘
                                │ depends on
                    ┌───────────▼─────────────┐
                    │       Application       │
                    │   Use Cases, DTOs, UoW  │
                    └───────────┬─────────────┘
                                │ depends on
                    ┌───────────▼─────────────┐
                    │         Domain          │
                    │ Entities, VOs, Rules    │
                    └─────────────────────────┘
                                ▲
                                │ implements
                    ┌───────────┴─────────────┐
                    │     Infrastructure      │
                    │   SQLAlchemy, Models    │
                    └─────────────────────────┘
```

### Estrutura de Pastas

```
.
├── .dockerignore
├── .env.example
├── .python-version
├── cosmic-ray.toml          # Configuração dos testes de mutação
├── docker-compose.yaml      # Orquestração da API e PostgreSQL com healthcheck
├── Dockerfile               # Multi-stage build com usuário não-root
├── pyproject.toml           # Dependências Poetry e configurações das ferramentas
├── README.md
├── src/
│   └── fintrack/
│       ├── api/             # Camada de Apresentação (FastAPI)
│       │   ├── dependencies.py      # Injeção de dependências (IoC)
│       │   ├── exception_handlers.py# Mapeamento de erros de domínio para HTTP
│       │   ├── main.py              # Aplicação FastAPI e lifespans
│       │   ├── routers/
│       │   │   ├── accounts.py      # Endpoints de contas
│       │   │   └── transfers.py     # Endpoints de transferências
│       │   └── schemas.py           # Contratos Pydantic de entrada/saída
│       ├── application/     # Camada de Aplicação (Casos de Uso)
│       │   ├── dtos.py              # Data Transfer Objects desacoplados
│       │   ├── interfaces.py        # Portas (UnitOfWork Protocol)
│       │   └── use_cases/
│       │       ├── create_account.py
│       │       ├── get_account.py
│       │       └── transfer_funds.py# Orquestração com idempotência e locks
│       ├── domain/          # Camada de Domínio (Regras puras)
│       │   ├── entities.py          # Account, Transfer
│       │   ├── exceptions.py        # Hierarquia de DomainError
│       │   ├── repositories.py      # Interfaces de repositório (Protocols)
│       │   └── value_objects.py     # Money (Decimal, half-even)
│       └── infrastructure/  # Camada de Infraestrutura (Persistência)
│           ├── config.py            # Pydantic Settings
│           ├── database.py          # SQLAlchemy Engine e Base
│           ├── models.py            # Mapeamento ORM separado das entidades
│           ├── repositories.py      # Implementações com SQLAlchemy
│           └── unit_of_work.py      # Implementação transacional do UoW
└── tests/
    ├── conftest.py          # Fixtures compartilhadas
    ├── api/                 # Testes unitários de schemas e exception handlers
    ├── application/         # Testes dos casos de uso com Fakes/In-Memory UoW
    └── domain/              # Testes do modelo de domínio e value objects
```

---

## 3. Decisões Técnicas Principais

1. **Separação entre Entidades de Domínio e Modelos ORM:**
   - As entidades (`Account`, `Transfer`) são dataclasses puras do Python, livres de anotações ou dependências do SQLAlchemy.
   - Os modelos ORM (`AccountModel`, `TransferModel`) residem estritamente na infraestrutura.
   - O repositório realiza a conversão bidirecional, garantindo que o domínio possa ser testado e mantido sem acoplamento com o banco de dados.

2. **Prevenção de Deadlocks em Concorrência:**
   - Ao transferir entre Contas A e B, se uma transação bloquear A depois B e outra transação simultânea bloquear B depois A, ocorreria um *deadlock*.
   - O caso de uso ordena deterministicamente os UUIDs das contas antes de adquirir o lock pessimista:
     ```python
     ids_ordered = sorted([input_dto.source_account_id, input_dto.destination_account_id])
     for account_id in ids_ordered:
         uow.accounts.get_by_id_for_update(account_id)
     ```
   - Isso elimina a possibilidade de espera circular nos locks do PostgreSQL.

3. **Garantia de Idempotência e Detecção de Conflitos:**
   - Cada transferência armazena uma `idempotency_key` com restrição de unicidade no banco de dados.
   - Se uma requisição com a mesma chave for reenviada:
     - Com **parâmetros idênticos**: retorna o registro da transferência já efetuada com sucesso (resposta idempotente, sem débito duplicado).
     - Com **parâmetros divergentes**: rejeita com `HTTP 409 Conflict` (`ConflictingTransferError`), impedindo fraude ou reuso acidental de chaves.

4. **Inversão de Dependência via `typing.Protocol`:**
   - As interfaces de repositório e Unit of Work utilizam *structural subtyping* (`Protocol`), eliminando herança artificial entre camadas.

5. **Testabilidade Pura (Unitária):**
   - Os testes de aplicação utilizam implementações `FakeAccountRepository`, `FakeTransferRepository` e `FakeUnitOfWork` em memória.
   - Nenhum teste depende de banco real, rede ou `TestClient`, garantindo execução em milissegundos e 100% determinística.

---

## 4. Instalação e Execução

### Opção 1: Via Docker Compose (Recomendado)

O Docker Compose orquestra o PostgreSQL 16 com volume persistente e a API FastAPI com usuário não-root e verificação de saúde:

```bash
# Iniciar a infraestrutura e a API
docker compose up --build -d

# Visualizar logs da API
docker compose logs -f api

# Parar serviços
docker compose down
```

A documentação interativa Swagger estará acessível em: [http://localhost:8000/docs](http://localhost:8000/docs).

### Opção 2: Execução Local com Poetry

**Pré-requisitos:** Python 3.11+ e PostgreSQL rodando localmente (ou via container isolado).

```bash
# 1. Instalar dependências via Poetry
poetry install

# 2. Configurar variáveis de ambiente
cp .env.example .env

# 3. Iniciar o servidor de desenvolvimento
poetry run uvicorn fintrack.api.main:app --reload --port 8000
```

---

## 5. Qualidade de Código e Testes

### 5.1 Testes Unitários

Executar todos os testes unitários com relatório de cobertura:

```bash
poetry run pytest -v --cov=src/fintrack --cov-report=term-missing
```

### 5.2 Lint e Formatação com Ruff

O Ruff realiza verificação estática de boas práticas, PEP 8, imports e complexidade:

```bash
# Verificar problemas
poetry run ruff check .

# Formatar automaticamente
poetry run ruff format .
```

### 5.3 Análise Estática de Tipos com Mypy

Configurado em modo `strict`:

```bash
poetry run mypy src
```

### 5.4 Testes de Mutação com Cosmic Ray

O Cosmic Ray injeta mutações (modificações no AST) no código de domínio para verificar se a suíte de testes unitários detecta e elimina os mutantes (*kills mutants*).

```bash
# 1. Inicializar sessão de mutação sobre o módulo de domínio
poetry run cosmic-ray init cosmic-ray.toml fintrack_mutation.sqlite

# 2. Executar os testes de mutação
poetry run cosmic-ray exec cosmic-ray.toml fintrack_mutation.sqlite

# 3. Gerar relatório de sobrevivência de mutantes
poetry run cr-report fintrack_mutation.sqlite --show-diff
```

**Como interpretar os resultados:**
- **Mutantes mortos (Killed):** Os testes falharam quando o código foi alterado, confirmando que a asserção é relevante e eficaz.
- **Mutantes sobreviventes (Surviving):** O código foi alterado mas os testes continuaram passando. Indica casos de borda não verificados ou asserções superficiais que requerem enriquecimento da suíte de testes.

---

## 6. Exemplos de Utilização da API (cURL)

### 6.1 Criar Conta com Saldo Inicial

```bash
curl -X POST http://localhost:8000/accounts \
  -H "Content-Type: application/json" \
  -d '{
    "owner_name": "Alice Silva",
    "initial_balance": "1000.00"
  }'
```
**Resposta (HTTP 201 Created):**
```json
{
  "id": "e0e224e7-4b47-4959-9945-88f5713c7bc1",
  "owner_name": "Alice Silva",
  "balance": "1000.00",
  "created_at": "2026-10-22T22:30:00Z"
}
```

### 6.2 Criar Segunda Conta

```bash
curl -X POST http://localhost:8000/accounts \
  -H "Content-Type: application/json" \
  -d '{
    "owner_name": "Bob Santos",
    "initial_balance": "250.00"
  }'
```
**Resposta (HTTP 201 Created):**
```json
{
  "id": "b8a1c93a-8631-4ce7-897b-cb14771cfd42",
  "owner_name": "Bob Santos",
  "balance": "250.00",
  "created_at": "2026-10-22T22:31:00Z"
}
```

### 6.3 Consultar Saldo

```bash
curl -X GET http://localhost:8000/accounts/e0e224e7-4b47-4959-9945-88f5713c7bc1
```
**Resposta (HTTP 200 OK):**
```json
{
  "id": "e0e224e7-4b47-4959-9945-88f5713c7bc1",
  "owner_name": "Alice Silva",
  "balance": "1000.00",
  "created_at": "2026-10-22T22:30:00Z"
}
```

### 6.4 Efetuar Transferência com Sucesso

```bash
curl -X POST http://localhost:8000/transfers \
  -H "Content-Type: application/json" \
  -d '{
    "idempotency_key": "transfer-req-uuid-001",
    "source_account_id": "e0e224e7-4b47-4959-9945-88f5713c7bc1",
    "destination_account_id": "b8a1c93a-8631-4ce7-897b-cb14771cfd42",
    "amount": "300.00"
  }'
```
**Resposta (HTTP 201 Created):**
```json
{
  "id": "92f75a7b-3b32-4467-9bf4-1100346c4f03",
  "idempotency_key": "transfer-req-uuid-001",
  "source_account_id": "e0e224e7-4b47-4959-9945-88f5713c7bc1",
  "destination_account_id": "b8a1c93a-8631-4ce7-897b-cb14771cfd42",
  "amount": "300.00",
  "created_at": "2026-10-22T22:32:00Z"
}
```

### 6.5 Reenvio Idempotente (Mesma chave e mesmos parâmetros)

Reenviando o mesmo payload do item 6.4:
**Resposta (HTTP 201 Created):**
Retorna a transferência original `92f75a7b-3b32-4467-9bf4-1100346c4f03` sem deduzir saldo adicional de Alice.

### 6.6 Conflito de Idempotência (Mesma chave, parâmetros diferentes)

```bash
curl -X POST http://localhost:8000/transfers \
  -H "Content-Type: application/json" \
  -d '{
    "idempotency_key": "transfer-req-uuid-001",
    "source_account_id": "e0e224e7-4b47-4959-9945-88f5713c7bc1",
    "destination_account_id": "b8a1c93a-8631-4ce7-897b-cb14771cfd42",
    "amount": "999.00"
  }'
```
**Resposta (HTTP 409 Conflict):**
```json
{
  "code": "IDEMPOTENCY_CONFLICT",
  "message": "Idempotency key 'transfer-req-uuid-001' already used with different parameters."
}
```

### 6.7 Rejeição por Saldo Insuficiente

```bash
curl -X POST http://localhost:8000/transfers \
  -H "Content-Type: application/json" \
  -d '{
    "idempotency_key": "transfer-req-uuid-002",
    "source_account_id": "e0e224e7-4b47-4959-9945-88f5713c7bc1",
    "destination_account_id": "b8a1c93a-8631-4ce7-897b-cb14771cfd42",
    "amount": "99999.00"
  }'
```
**Resposta (HTTP 422 Unprocessable Entity):**
```json
{
  "code": "INSUFFICIENT_FUNDS",
  "message": "Insufficient funds: balance=700.00, requested=99999.00"
}
```

---

## 7. Limitações Conhecidas e Próximos Passos

1. **Alembic para Migrações de Esquema:** Em um ambiente de produção contínuo, as tabelas devem ser gerenciadas via scripts versionados de migração com Alembic em vez de `Base.metadata.create_all`.
2. **Double-Entry Ledger:** Evoluir a tabela `transfers` para um livro-razão (*ledger*) de partidas dobradas com lançamentos de crédito e débito explícitos por movimentação, atendendo a normas contábeis formais.
3. **Outbox Pattern:** Para publicação de eventos assíncronos (e.g., Kafka / RabbitMQ) após confirmação de transferência sem comprometer a transação atômica do banco de dados.
4. **Suporte a Múltiplas Moedas:** Expandir o value object `Money` para incluir código ISO 4217 e taxas de câmbio históricas.
