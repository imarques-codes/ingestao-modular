# ingestao-modular
Framework modular para pipelines de dados em Python: conectores, validadores e loaders (JSON/PostgreSQL) plugáveis e independentes. A arquitetura permite a substituição de componentes sem impacto no código principal, utilizando Pydantic para assegurar a consistência técnica e a flexibilidade total na ingestão de dados.

<img width="1417" height="739" alt="FluxoPipeline(2)" src="https://github.com/user-attachments/assets/1a4b939f-5b42-4452-a5bb-8a60d00d6314" />

## Motivação
Pipelines de dados frequentemente sofrem com a redundância de funções básicas como logging, retentativas e tratamento de exceções. Este framework elimina o retrabalho ao fornecer um core de orquestração padronizado, permitindo que o desenvolvedor foque exclusivamente na lógica de negócio de cada fonte.
A arquitetura foi inspirada em padrões de engenharia de dados de alta performace, viando o desacoplamento de responsabilidade e a extensibilidade total via componentes plugáveis, sem necessidade de alterar o código principal.

---

## Instalação e Setup
O projeto utiliza o /poetry para garantir um ambiente isolado e gestão eficiente de dependências.

```bash
# Clone o repositório e acesse a pasta
git clone https://github.com/imarques-codes/ingestao-modular
cd modular-ingestao-modular

# Instale as dependências base
poetry install

# Para suporte ao PostgreSQL (opcional)
poetry install --with postgres
```
## Ambiente de Desenvolvimento
Para facilitar os testes e a homologação, disponibilizo um ambiente via ##Docker com dados pré-carregados(Selic, IPCA, Municípios):

```bash
docker compose up -d
```

## Setup

> **Nota:** O arquivo `init.sql` configura automaticamente o banco `pipeline_dev`  
> **Usuário:** `pipeline`  
> **Senha:** `pipeline123`  

**Requisitos:**
- Python 3.11+
- Poetry

---

## Uso Rápido

### Pipeline Simples (JSON)

```python
from pydantic import BaseModel
from core.pipeline import Pipeline
from connectors.rest_connector import RESTConnector
from validators.pydantic_validator import PydanticValidator
from loaders.json_loader import JsonLoader

class Municipio(BaseModel):
    id: int
    nome: str

pipeline = Pipeline(
    name="municipios_ibge",
    connector=RESTConnector(
        name="ibge_api",
        url="https://servicodados.ibge.gov.br/api/v1/localidades/estados/PI/municipios",
    ),
    validator=PydanticValidator(model=Municipio, unique_by="id"),
    loader=JsonLoader(path="output/municipios.json"),
)

metrics = pipeline.run()
print(metrics)
```

---

## Com transformação usando transformers:

```python
from transformers import IBGETransformer, TransformPipeline, FilterTransformer, EnrichTransformer
# Usando transformer individual
pipeline = Pipeline(
    name="municipios_ibge",
    connector=RESTConnector(...),
    validator=PydanticValidator(...),
    loader=JsonLoader(...),
    transform=IBGETransformer(),  # Transformer específico para IBGE
)

# Ou compondo múltiplos transformers
transform_pipeline = TransformPipeline([
    IBGETransformer(),
    FilterTransformer(condition=lambda r: r["id"] > 1000),
    EnrichTransformer(fields={"fonte": "API_IBGE"}, add_timestamp=True),
])

pipeline = Pipeline(
    name="municipios_ibge",
    connector=RESTConnector(...),
    validator=PydanticValidator(...),
    loader=JsonLoader(...),
    transform=transform_pipeline,  # Pipeline de transformers
)
```

## Função customizada (suportada):

```python
def transform_municipios(data: list[dict]) -> list[dict]:
    """Extrai apenas campos relevantes."""
    return [{"id": m["id"], "nome": m["nome"]} for m in data]

pipeline = Pipeline(
    name="municipios_ibge",
    connector=RESTConnector(...),
    validator=PydanticValidator(...),
    loader=JsonLoader(...),
    transform=transform_municipios,  # Função callable também funciona
)
```

---

## Exemplos

### Exemplo 1 — API do IBGE

```bash
# Com Poetry (recomendado)
poetry run python examples/pipeline_ibge.py

# Diretamente (após poetry install)
python examples/pipeline_ibge.py
```

Busca os **224 municípios do Piauí** via API pública do IBGE, valida schema e tipos usando `PydanticValidator`, aplica transformação para extrair campos relevantes e salva em JSON local.

> **Nota:** O exemplo usa `Municipio_IBGE` de `core.models.py` e uma função `transform_municipios` para estruturar os dados.  
> Alternativamente, pode-se usar `IBGETransformer()` para o mesmo fim.

---

### Exemplo 2 — Usando Transformers

```bash
poetry run python examples/pipeline_with_transformers.py
```

Demonstra o uso de transformers individuais e `TransformPipeline` para compor múltiplas transformações em sequência.

---

### Exemplo 3 — API BCB (Selic)

```bash
poetry run python examples/pipeline_selic.py
```

Busca dados da taxa Selic via API do Banco Central, valida com `PydanticValidator`, aplica `SelicTransformer` (formata datas, calcula taxa anualizada) e salva em JSON local.

---

##Componentes
---
<img width="1205" height="773" alt="ArquiteturaFramework" src="https://github.com/user-attachments/assets/c0ab506b-b222-4016-a4f0-23a470e6c2a5" />

## Connectors

| Classe        | Descrição |
|--------------|----------|
| RESTConnector | APIs REST (GET/POST) com suporte a Bearer token, API Key, headers customizados, params e timeout configurável |
| FileConnector | Arquivos locais: CSV, JSON, JSONL, XLSX (via pandas/openpyxl) |

---

## Validators

| Classe            | Descrição |
|------------------|----------|
| PydanticValidator | Validação completa usando Pydantic: schema, tipos, constraints, duplicatas e validações customizadas |

---

## Transformers

| Classe             | Descrição |
|-------------------|----------|
| FieldMapper        | Mapeia/renomeia campos, suporta campos aninhados via notação `a.b.c` |
| EnrichTransformer  | Adiciona campos fixos e timestamp opcional aos registros |
| FilterTransformer  | Filtra registros baseado em função de condição |
| TransformPipeline  | Compõe múltiplos transformers em sequência |
| IBGETransformer    | Transformer específico para dados da API do IBGE |
| SelicTransformer   | Transformer específico para dados da API BCB (Selic) |

---

## Loaders

| Classe           | Descrição |
|-----------------|----------|
| JsonLoader       | Salva em arquivo JSON local (overwrite) com indentação configurável |
| PostgresLoader   | INSERT ou UPSERT em PostgreSQL via `execute_batch` (psycopg2). Suporta `conflict_on` para definir campos de conflito no UPSERT |

---

## Estendendo o framework
---
<img width="1392" height="829" alt="abstracao-Baseconnector (2)" src="https://github.com/user-attachments/assets/3fbf287b-88ac-44e4-bc1a-8bc4fd766ee7" />

### Criar um novo Connector

```python
from core.base_connector import BaseConnector

class MyAPIConnector(BaseConnector):
    def fetch(self) -> list[dict]:
        # sua lógica aqui
        return data
```

O mesmo padrão vale para **validators, transformers e loaders**.  
O `Pipeline` aceita qualquer implementação das interfaces base.

---

### Criar um novo Transformer

```python
from transformers.base_transformer import BaseTransformer

class MyTransformer(BaseTransformer):
    def transform(self, data: Any) -> Any:
        # sua lógica de transformação aqui
        return transformed_data
```

Transformers podem ser usados diretamente no `Pipeline` ou compostos em um `TransformPipeline`.

---

## Métricas e Logging

Cada execução retorna um `PipelineMetrics` (modelo Pydantic com validações):

```python
class PipelineMetrics(BaseModel):
    pipeline: str
    success: bool = False
    records_fetched: int = Field(default=0, ge=0)
    records_loaded: int = Field(default=0, ge=0)
    records_failed: int = Field(default=0, ge=0)
    duration_seconds: float = Field(default=0.0, ge=0)
    started_at: datetime
    finished_at: datetime | None = None
    stage_failed: str | None = None
    error_message: str | None = None
```

### Exemplo de saída

```python
metrics = pipeline.run()
print(metrics)

# PipelineMetrics(✓ municipios_ibge | fetched=224 loaded=224 duration=0.48s)
```

---

### Logging estruturado

Compatível com **Datadog, CloudWatch e ELK**:

- Terminal (TTY): JSON colorido para facilitar leitura  
- Arquivo/redirecionamento: JSON puro para observabilidade  

```json
{
  "timestamp": "2025-03-09T14:22:31.123456+00:00",
  "level": "INFO",
  "message": "pipeline_finished",
  "pipeline": "municipios_ibge",
  "status": "success",
  "duration": 0.48,
  "records_fetched": 224,
  "records_loaded": 224
}
```

O logger detecta automaticamente se está em um terminal e aplica cores.  
Use `setup_logging(use_colors=False)` para forçar JSON puro.

---

## Testes

```bash
# Executar todos os testes
poetry run pytest -v

# Com cobertura de código
poetry run pytest --cov=core --cov=validators --cov=connectors --cov=loaders --cov=transformers --cov=audit --cov-report=term-missing
```

### Cobertura atual (82%)

- ✅ **Pipeline (88%)** — sucesso, falhas, erros de conector, transformações  
- ✅ **PydanticValidator (89%)** — schema, tipos, constraints, duplicatas  
- ✅ **RESTConnector (90%)** — GET/POST, auth, headers, params  
- ✅ **FileConnector (71%)** — leitura e tratamento de erros  
- ✅ **JsonLoader (100%)** — escrita, diretórios, encoding  
- ✅ **PostgresLoader (93%)** — INSERT/UPSERT, SQL, erros  
- ✅ **Transformers (86–100%)** — todos os transformers  
- ✅ **PipelineMetrics (100%)** — validações e representação  
- ✅ **Exceptions (100%)** — erros customizados  

---

## Estrutura de Testes

```plaintext
tests/test_core/           # Pipeline, exceptions
tests/test_validators/     # PydanticValidator
tests/test_connectors/     # RESTConnector, FileConnector
tests/test_loaders/        # JsonLoader, PostgresLoader
tests/test_transformers/   # Todos os transformers
tests/test_audit/          # PipelineMetrics
```
---
## Estrutura

```plaintext
modular-ingestion-framework/
├── core/
│   ├── __init__.py
│   ├── base_connector.py      # ABC — interface para conectores
│   ├── base_validator.py      # ABC — interface para validadores
│   ├── base_loader.py         # ABC — interface para loaders
│   ├── pipeline.py            # Orquestrador: fetch → validate → transform → load
│   ├── exceptions.py          # PipelineError, ValidationError
│   └── models.py              # Modelos Pydantic de exemplo
├── connectors/
│   ├── __init__.py
│   ├── rest_connector.py      # REST GET/POST com auth, headers, params
│   └── file_connector.py      # CSV, JSON, JSONL, XLSX (pandas)
├── validators/
│   ├── __init__.py
│   └── pydantic_validator.py  # Validação com Pydantic + unique_by
├── loaders/
│   ├── __init__.py
│   ├── json_loader.py         # JSON local
│   └── postgres_loader.py     # INSERT / UPSERT PostgreSQL
├── transformers/
│   ├── __init__.py
│   ├── base_transformer.py
│   ├── field_mapper.py
│   ├── enrich_transformer.py
│   ├── filter_transformer.py
│   ├── transform_pipeline.py
│   ├── ibge_transformer.py
│   └── selic_transformer.py
├── audit/
│   ├── logger.py              # Logging estruturado em JSON
│   └── metrics.py             # PipelineMetrics (Pydantic)
├── examples/
│   ├── pipeline_ibge.py
│   ├── pipeline_selic.py
│   └── pipeline_with_transformers.py
├── docs/
│   └── imgs/
│       ├── diagrama_pipeline_flow.png
│       ├── diagrama_arquitetura.png
│       └── diagrama_abstracao.png
├── input/
│   ├── diagrama*.excalidraw
│   ├── municipios_pi.csv
│   └── municipios_pi.xlsx
├── scripts/
│   ├── pipeline_ibge.py
│   ├── pipeline_example.py
│   ├── explorando_*.py
│   └── README.md
├── tests/
│   ├── test_core/
│   ├── test_validators/
│   ├── test_connectors/
│   ├── test_loaders/
│   ├── test_transformers/
│   └── test_audit/
├── docker-compose.yml
├── init.sql
├── pyproject.toml
└── poetry.lock
```

---

## Decisões de Design

- **Interfaces com ABC + `@abstractmethod`** → contrato explícito entre componentes (*Liskov Substitution Principle*)  
- **Validação antes da transformação** → falha rápida antes de qualquer escrita  
- **Strategy Pattern** → componentes intercambiáveis sem alterar o pipeline  
- **Template Method (Pipeline)** → fluxo fixo: `fetch → validate → transform → load`  
- **Transformers modulares** → reutilização de lógica comum  
- **Composição de transformers** → `TransformPipeline` encadeia transformações  
- **Logging estruturado em JSON** → integração com Datadog, CloudWatch, ELK  
- **Métricas com Pydantic** → validação automática de tipos e constraints  
- **Logger adaptativo** → cores em TTY, JSON puro em produção  
- **Tratamento de erros** → exceções customizadas (`PipelineError`, `ValidationError`)  

---

## Roadmap

- [ ] S3Connector — leitura de arquivos no AWS S3 (boto3)  
- [ ] KafkaLoader — publicação em tópicos Kafka  
- [ ] GitHub Actions — CI com pytest a cada push  
- [ ] AsyncPipeline — versão assíncrona com `asyncio`  
