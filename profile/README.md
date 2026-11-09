# IBMEC-CDIA

A organização de **Ciência de Dados e IA** (CDIA) do IBMEC reúne projetos de
ensino e pesquisa em aprendizado de máquina aplicado a dados financeiros. Os
projetos servem de material de estudo em disciplinas do IBMEC, com código,
especificações e fluxos de entrega reproduzíveis.

## Projetos

### financial_market_simulator

Projeto que usa **autoencoders** para detectar movimentos incomuns no preço de
ativos financeiros. Para cada ticker (por exemplo `AAPL` ou `PETR4.SA`), o
modelo aprende o comportamento normal dos log-retornos diários. Uma janela de
retornos que o modelo reconstrói mal recebe um escore alto e é marcada como
anômala, ou seja, o mercado se comportou de forma incomum naquele período.

O projeto é de ensino e pesquisa. Os modelos descrevem comportamentos
incomuns em preços passados. Eles não preveem preços e não são recomendação de
investimento.

**Repositório:** https://github.com/IBMEC-CDIA/financial_market_simulator

#### Como o projeto funciona

1. **Dados.** Os preços de fechamento diários são baixados pelo yfinance e
   validados com o Great Expectations antes de qualquer uso.
2. **Retornos e divisão.** Os preços viram log-retornos diários. A série é
   dividida em ordem de tempo: 70% para treino, 15% para validação e 15% para
   teste. O conjunto de teste é avaliado uma única vez, depois do treino.
3. **Treino.** Um autoencoder denso é treinado com janelas deslizantes de
   log-retornos. A configuração padrão usa janelas de 10 retornos, camada
   oculta de 16, camada latente de 4, lote de 32, 300 épocas, taxa de
   aprendizado de 0,001 e semente 42. O limiar de anomalia é definido pela
   contaminação esperada, 5% por padrão, e é salvo junto com os pesos.
4. **Avaliação.** O modelo precisa superar uma linha de base ingênua no
   conjunto de teste, e o erro de validação não pode ser muito maior que o
   erro de treino. Esse critério evita o overfitting.
5. **Registro.** Cada modelo treinado gera um arquivo `.pt` com os pesos e
   um relatório `*-overfitting.json`. O treino também registra o experimento
   e o modelo no Weights & Biases.
6. **Serving.** Uma API FastAPI carrega os modelos e responde com o escore de
   anomalia de cada janela, roteando cada pedido para o modelo do ticker
   pedido. Uma rota adicional sugere compra, venda ou manutenção de um ativo.
7. **Entrega.** Workflows do GitHub Actions testam o código, treinam os
   modelos, constroem a imagem Docker com os modelos e a publicam no Docker
   Hub. Outro workflow substitui o container no Azure.

#### Arquitetura

```
 Dados               Modelo               Treino e avaliação
 ───────────────     ───────────────      ─────────────────────────────
 yfinance       ->   autoencoder    ->   treino de um ticker
 validação (GE)      trainer              treino de vários tickers
 log-retornos        overfitting          TickerModelManager
 split 70/15/15      config               │
 janelas             │                    ├─ Weights & Biases
                     │                    └─ artefatos: .pt + relatório
                     ▼
 Entrega             Serving                    Registro
 ─────────────       ─────────────────────      ─────────────────────────────
 GitHub Actions  ->  FastAPI              <-    pasta models/ (local)
 Docker image        ModelService               Weights & Biases (latest)
 Docker Hub          Predictor, sinais
 Azure ACI           /predictions, /signals
```

O código fica em `src/generative_models/`, dividido em:

- `data/`: coleta, validação, normalização, datasets e janelas.
- `models/`: pacote `base_autoencoder` (modelo de referência), pacote
  `regression` e os pacotes de autoencoders convolucional, GRU e LSTM, que
  ainda são placeholders.
- `tracking/`: integração com o Weights & Biases.
- `registry/` e `serving/`: registro de modelos, carregamento e API.

#### Conexões externas

| Serviço | Para que é usado |
|---|---|
| yfinance | Preços de fechamento diários dos tickers. |
| Weights & Biases | Registro dos experimentos e dos modelos treinados. |
| GitHub Actions | CI, treino, build da imagem e deploy. |
| Docker Hub | Armazenamento das imagens da API. |
| Azure Container Instances | Execução da API em produção. |

#### API

A API expõe, entre outras, estas rotas:

| Método | Rota | Função |
|---|---|---|
| GET | `/health` | Verifica se o serviço está no ar. |
| GET | `/models/loaded` | Lista os modelos em memória. |
| GET | `/models/available` | Lista os modelos disponíveis no wandb. |
| GET | `/models/default` | Mostra o modelo padrão. |
| POST | `/models/reload` | Baixa novamente um modelo. |
| POST | `/predictions/log-returns` | Detecta anomalias em log-retornos. |
| POST | `/predictions/prices` | Detecta anomalias em preços de fechamento. |
| POST | `/predictions/market` | Detecta anomalias em dados de mercado baixados na hora. |
| GET | `/signals` | Sugestão de compra, venda ou manutenção de um ativo. |

A documentação interativa fica em `/docs`, quando a API está rodando.

#### Workflows de CI/CD

| Workflow | Quando roda | O que faz |
|---|---|---|
| `ci.yml` | Push e pull request | Verifica a estrutura do projeto, roda o pylint, os testes com cobertura mínima de 90%, pip-audit, Bandit e Gitleaks. |
| `train.yml` | Manual | Confere o CI do commit, valida o modelo e os tickers, roda os testes de ponta a ponta, coleta os preços, treina e avalia o overfitting. |
| `build.yml` | Manual | Baixa os modelos (de um run de treino ou o `latest` do wandb), constrói a imagem, testa a API, escaneia vulnerabilidades com Trivy e publica no Docker Hub. |
| `deploy.yml` | Manual | Substitui o container no Azure a partir de uma imagem do Docker Hub e espera a API responder com um modelo carregado. |

Os workflows de treino, build e deploy só são iniciados à mão, pela aba
Actions ou pela linha de comando com `gh workflow run`.

#### Ambiente

- Python 3.12, com o gerenciador de pacotes e ambientes [uv](https://docs.astral.sh/uv/).
- PyTorch para o modelo, FastAPI e Uvicorn para a API, Pandas e NumPy para os dados.
- Docker para construir a imagem da API.
- Variáveis de ambiente lidas de um arquivo `.env` local, que nunca é versionado.
  O modelo de configuração está em [`.env.example`](https://github.com/IBMEC-CDIA/financial_market_simulator/blob/main/.env.example).

#### Como começar

```bash
uv venv --python 3.12.10
uv sync --extra dev
uv run pytest
```

O README do repositório traz o passo a passo completo, incluindo como treinar
um modelo pela linha de comando, como servir a API localmente e como construir
a imagem Docker.

#### Especificações e material de estudo

A pasta `specs/` tem as especificações de cada módulo, escritas para os
alunos. Cada spec descreve o comportamento esperado, os critérios de aceite e
os testes correspondentes. Os notebooks em `notebooks/` incluem exemplos de
regressão e de detecção de outliers.

#### Limites

- Os resultados dependem do período e do ticker escolhidos.
- Os modelos não foram validados para negociação.
- A regra de sinais da API é uma proposta que a equipe ainda precisa confirmar.
- O deploy não tem rollback. Se a criação falhar depois de remover o container
  antigo, o serviço fica fora do ar até um novo deploy.
