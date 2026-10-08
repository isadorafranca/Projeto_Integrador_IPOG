# Ficha técnica — Pipeline Multiagente Agno para Superstore USA

**Fonte:** `Projeto_integrador_Isadora_França.ipynb`
**Ambiente original:** Google Colab / Python 3
**Dataset:** `SampleSuperstore.csv`
**Data do registro:** 2026-09-28

## Objetivo

Executar uma análise executiva, comercial e operacional do dataset Superstore USA por meio de um pipeline multiagente construído com **Agno** e modelos **Gemini**, usando ferramentas analíticas determinísticas em pandas e geração de relatórios em Markdown.

## Dependências

```bash
pip install -q agno google-genai tabulate "google-auth==2.49.0"
```

Bibliotecas principais:

- `os`

- `time`

- `pandas`

- `IPython.display`

- `agno`

- `google-genai`

- `tabulate`

- `google-auth==2.49.0`

## Configuração e autenticação

1. Cria a pasta `outputs/`.

1. Tenta obter a chave por `google.colab.userdata`:
  - primeiro `GEMINI_API_KEY`;
  - depois `GOOGLE_API_KEY`.

1. Fora do Colab, solicita a chave com `getpass`.

1. Define as variáveis de ambiente `GOOGLE_API_KEY` e `GEMINI_API_KEY`.

> As chaves não fazem parte deste registro.

## Carregamento e preparação dos dados

O pipeline procura o CSV nestes caminhos, nesta ordem:

1. `SampleSuperstore.csv`

1. `Sample - Superstore.csv`

1. `data/SampleSuperstore.csv`

1. `data/Sample - Superstore.csv`

Leitura com `utf-8` e fallback para `latin1` em caso de erro de decodificação.

Transformações realizadas:

- conversão de `Order Date` para datetime;

- conversão de `Ship Date` para datetime;

- criação de `Shipping Days` como diferença em dias entre envio e pedido;

- criação de `Profit Margin %` como `Profit / Sales * 100`.

Resultado observado na execução registrada: **9.994 linhas e 14 colunas** após o carregamento/preparação.

## Ferramentas analíticas determinísticas

### `get_executive_kpis()`

Calcula:

- Faturamento Total (USD);

- Lucro Total (USD);

- Margem Líquida Geral (%);

- Desconto Médio Concedido (%);

- Volume de Pedidos Únicos;

- Base de Clientes Ativos.

### `get_performance_by_dimension(dimension)`

Agrupa vendas e lucro por uma dimensão, normalmente:

- `Region`;

- `Segment`;

- `Category`.

Retorna:

- vendas totais;

- lucro total;

- quantidade de pedidos;

- margem de lucro;

- participação nas vendas;

- tabela ordenada por vendas decrescentes.

### `get_product_and_shipping_diagnostics()`

Produz dois diagnósticos:

1. **Produtos:** identifica as cinco subcategorias com maior prejuízo financeiro, incluindo vendas, lucro, desconto médio e margem.

1. **Logística:** compara os modais de frete por vendas, pedidos e, quando disponível, média de dias de entrega.

## Agentes

Todos são criados pela função `build_agent(role, model_id)` usando `agno.agent.Agent`, `agno.models.google.Gemini`, Markdown habilitado e instruções específicas.

### 1. Analyst Agent — `analyst`

Nome: `Senior Data Analyst`

Ferramentas:

- `get_executive_kpis`;

- `get_performance_by_dimension`;

- `get_product_and_shipping_diagnostics`.

Responsabilidade: obter números exatos, consolidar os dados e retornar tabelas sem arredondamento indevido ou omissão.

### 2. CEO Agent — `ceo`

Nome: `CEO Strategic Reporter`

Responsabilidade: redigir relatório executivo conciso, focado em margens, riscos e direcionamento corporativo, estruturado em:

1. Resumo Executivo;

1. KPIs Globais;

1. Riscos Críticos;

1. Três Ações Prioritárias.

### 3. Sales Agent — `sales`

Nome: `Commercial Sales Reporter`

Responsabilidade: analisar Região e Segmento de Clientes (`Consumer`, `Corporate`, `Home Office`), apresentar tabelas e recomendar ações para elevar rentabilidade e conter descontos excessivos.

### 4. Operations Agent — `ops`

Nome: `Supply Chain & Merchandising Reporter`

Responsabilidade: analisar subcategorias críticas, itens deficitários, modais de envio, estoque, controle de descontos e prazos de entrega.

## Modelos e resiliência

Modelos configurados em ordem de custo/capacidade:

```python
MODEL_IDS = [
    "gemini-3.5-flash-lite",
    "gemini-3.5-flash",
    "gemini-3.6-flash",
    "gemini-3.7-flash",
]
```

Mecanismo de execução:

- `run_agent_with_retry()` faz até 3 tentativas;

- erros `429` e `503` acionam espera progressiva de `(tentativa + 1) * 4` segundos;

- erros `404` ou `NOT_FOUND` geram `ModelUnavailable` e não são repetidos;

- `run_step()` troca para o próximo modelo quando uma etapa falha;

- `_modelo_idx` é global: depois de uma troca, as etapas seguintes começam no modelo seguinte;

- etapas concluídas não são refeitas.

## Orquestração

A execução ocorre sequencialmente:

1. **Analyst Agent:** gera `dossie_dados` com KPIs, análises por dimensão e diagnósticos.

1. **CEO Agent:** recebe o dossiê e gera o relatório executivo.

1. **Sales Agent:** recebe o mesmo dossiê e gera o relatório tático de vendas.

1. **Operations Agent:** recebe o mesmo dossiê e gera o relatório operacional de produtos e logística.

Há `time.sleep(2)` entre as etapas de redação para reduzir pressão sobre a API.

## Arquivos de saída

Os relatórios são salvos em `outputs/`:

- `relatorio_ceo.md`;

- `relatorio_vendas.md`;

- `relatorio_produtos_logistica.md`;

- `dossie_dados.md` — cópia auditável dos dados consolidados usados pelos demais agentes.

O notebook também exibe o relatório do CEO ao final.

## Resultado da execução registrada

As quatro etapas foram concluídas com êxito usando `gemini-3.5-flash-lite`:

- coleta do dossiê de dados: concluída;

- relatório do CEO: concluído;

- relatório de vendas: concluído;

- relatório de produtos e logística: concluído.

## Observações para reutilização

- Fazer upload do `SampleSuperstore.csv` antes de executar no Colab.

- Configurar uma chave válida em `GEMINI_API_KEY` ou `GOOGLE_API_KEY`.

- Confirmar se os IDs dos modelos ainda estão disponíveis na API no momento da execução.

- O código imprime um aviso quando `GOOGLE_API_KEY` e `GEMINI_API_KEY` estão simultaneamente configuradas; a biblioteca informa que usa `GOOGLE_API_KEY`.

- O pipeline depende das colunas esperadas do Superstore, especialmente `Sales`, `Profit`, `Discount`, `Order ID`, `Customer ID`, `Order Date`, `Ship Date`, `Region`, `Segment`, `Category`, `Sub-Category` e `Ship Mode`.
