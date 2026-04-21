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
Para facilitar os testes e a homologação, disponibilizo um ambiente via #Docker com dados pré-carregados(Selic, IPCA, Municípios):

```bash
docker compose up -d
```



