# dbt Core no Databricks

Neste roteiro você parte de uma tabela de pedidos no Unity Catalog e entrega uma tabela Delta incremental com testes, documentação e execução agendada. O dbt Core lê o projeto, resolve dependências e compila SQL no computador ou runner onde a CLI está instalada. O adaptador `dbt-databricks` envia esse SQL ao warehouse. A ingestão da origem precisa estar pronta antes do primeiro build.

Os exemplos usam Bash, modelos SQL e dbt Core **1.10.x** com o adaptador Databricks **1.10.x**. Para consultar um comando durante o roteiro, abra a [referência da CLI por fase](cli.md).

## 1. Preparar o workspace

No Databricks, abra **SQL Warehouses → seu warehouse → Connection details**. Copie *Server hostname* e *HTTP path* do mesmo warehouse. No Catalog Explorer, identifique o catálogo, o schema e a tabela de origem. O exemplo supõe `raw.vendas.pedidos`, com `pedido_id` único, `valor` numérico e `atualizado_em` e `ingerido_em` como timestamps. Cada linha representa o estado atual de um pedido.

| Recurso | Acesso da identidade que executa o dbt |
| --- | --- |
| SQL Warehouse | `CAN USE` |
| Origem | `USE CATALOG`, `USE SCHEMA` e `SELECT` na tabela |
| Destino | `USE CATALOG`, `USE SCHEMA` e `CREATE TABLE` no schema para criar tabelas e views |
| Objetos de execuções anteriores | Permissões de leitura e escrita para o merge e propriedade ou privilégios adequados para substituir objetos |

Peça um schema de desenvolvimento por pessoa, por exemplo `analytics.dev_ana`. Se os schemas forem provisionados pelo dbt, a identidade também precisará de `CREATE SCHEMA` no catálogo. Confirme as permissões com a identidade que rodará o job. [Privilégios do Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges)

## 2. Instalar, criar e conectar

Na pasta onde pretende criar o projeto, use um ambiente Python isolado. O comando `init` abaixo cria o diretório `vendas`.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install 'dbt-core~=1.10.0' 'dbt-databricks~=1.10.0'
dbt --version
dbt init vendas --skip-profile-setup
cd vendas
python -m pip freeze > requirements.lock.txt
```

Versione o lock para que o runner com a mesma versão de Python possa instalar o ambiente com `python -m pip install -r requirements.lock.txt`. No `dbt_project.yml` gerado, confira `name: vendas` e `profile: vendas`.

Em `~/.dbt/profiles.yml`, acrescente o perfil abaixo, preservando os demais perfis. Para a sessão local, exporte `DATABRICKS_HOST` com o hostname sem `https://`, `DATABRICKS_HTTP_PATH` com o caminho completo e `DBT_SCHEMA` com seu schema de desenvolvimento. Injete o PAT pela variável `DBT_ENV_SECRET_DATABRICKS_TOKEN` usando seu gerenciador de segredos.

```yaml
vendas:
  target: dev
  outputs:
    dev:
      type: databricks
      host: "{{ env_var('DATABRICKS_HOST') }}"
      http_path: "{{ env_var('DATABRICKS_HTTP_PATH') }}"
      token: "{{ env_var('DBT_ENV_SECRET_DATABRICKS_TOKEN') }}"
      catalog: analytics
      schema: "{{ env_var('DBT_SCHEMA') }}"
      threads: 4
```

Dentro de `vendas/`, rode `dbt debug --target dev`. Prossiga quando o teste de conexão passar. Isso confirma a conexão, mas só o build exercita leitura da origem e criação no destino. Para automação externa, use uma identidade de serviço com OAuth M2M. No output correspondente, configure `auth_type: oauth`, `client_id` e `client_secret` por variáveis de ambiente e remova o campo `token`. [Configuração do adaptador](https://github.com/databricks/dbt-databricks)

## 3. Declarar a origem e o modelo de staging

Crie `models/staging/fontes.yml` no projeto gerado. Em Unity Catalog, o campo `database` de uma source corresponde ao catálogo. A coluna de ingestão mede o atraso da carga mesmo quando o pedido foi atualizado dias antes.

```yaml
version: 2
sources:
  - name: vendas_brutas
    database: raw
    schema: vendas
    tables:
      - name: pedidos
        config:
          loaded_at_field: ingerido_em
          freshness:
            warn_after: {count: 2, period: hour}
            error_after: {count: 6, period: hour}
```

Em `models/staging/stg_pedidos.sql`, crie uma view que padroniza tipos. A chamada a `source()` registra a dependência da tabela física no grafo do projeto.

```sql
{{ config(materialized='view') }}

select
    pedido_id,
    cast(valor as decimal(18, 2)) as valor,
    atualizado_em,
    ingerido_em
from {{ source('vendas_brutas', 'pedidos') }}
```

## 4. Materializar em Delta e verificar o resultado

Em `models/marts/fct_pedidos.sql`, use `ref()` para que o dbt resolva o nome da view no ambiente atual e a construa primeiro. O primeiro build lê toda a origem. Nos seguintes, o filtro relê três dias de ingestão a partir da maior marca já processada e o merge atualiza ou insere pela chave.

```sql
{{ config(materialized='incremental', file_format='delta',
          incremental_strategy='merge', unique_key='pedido_id') }}

select pedido_id, valor, atualizado_em, ingerido_em
from {{ ref('stg_pedidos') }}
{% if is_incremental() %}
where ingerido_em >= (
    select coalesce(max(ingerido_em), timestamp '1970-01-01') - interval 3 days
    from {{ this }}
)
{% endif %}
```

O merge pressupõe uma linha por chave na origem, inclusive na primeira carga. Para uma origem de eventos, deduplique no staging com uma ordem determinística de versão antes do merge. Dimensione a janela pelo atraso observado e planeje reprocessamentos para dados fora dela. Exclusões físicas na origem exigem tratamento explícito. [Materializações e estratégias no Databricks](https://docs.getdbt.com/reference/resource-configs/databricks-configs)

Em `models/pedidos.yml`, registre a granularidade e teste a chave antes e depois da transformação.

```yaml
version: 2
models:
  - name: stg_pedidos
    columns:
      - name: pedido_id
        data_tests: [not_null, unique]
      - name: ingerido_em
        data_tests: [not_null]
  - name: fct_pedidos
    description: "Uma linha por pedido, com o valor e a última atualização recebida."
    columns:
      - name: pedido_id
        description: "Identificador estável do pedido."
        data_tests: [not_null, unique]
```

```bash
dbt source freshness --select source:vendas_brutas.pedidos
dbt build --select +fct_pedidos
dbt show --inline "select count(*) as pedidos from {{ ref('fct_pedidos') }}"
dbt docs generate
dbt docs serve
```

Confira no Catalog Explorer a tabela `analytics.<seu_schema>.fct_pedidos`. Depois de uma nova ingestão, repita o build e confirme que um pedido atualizado mantém uma única linha. Use `docs serve` localmente para explorar descrições e linhagem.

Para testar uma coluna específica ou investigar os pedidos que falharam, consulte os [comandos de testes](cli.md#comandos-de-testes).

## 5. Integrar com CI e Lakeflow Jobs

Em pull requests, instale as versões travadas, use um schema por PR e valide o grafo alterado. Para comparar com produção, baixe o `manifest.json` do último deploy bem-sucedido para um diretório separado de `target/`. A identidade de CI precisa poder ler as relações de produção usadas pelo defer. Veja o [fluxo de CI e operação](cli.md#5-ci-e-operação).

Para agendar no Databricks, crie uma tarefa **dbt** em Lakeflow Jobs. Selecione o repositório Git e o diretório que contém `dbt_project.yml`. Configure o compute da CLI, a versão do adaptador e o SQL Warehouse que executará as consultas. Com o perfil gerado pela tarefa, selecione catálogo e schema de produção na interface. Configure o *Run as* com o service principal de produção e valide um disparo manual. [Tarefa dbt em Lakeflow Jobs](https://docs.databricks.com/aws/en/jobs/tasks/dbt)

Encadeie a tarefa após a ingestão. Execute `dbt deps` se houver pacotes, depois `dbt source freshness` e `dbt build`. Use o código de saída para barrar a etapa seguinte em caso de erro. Defina alertas, limite execuções concorrentes sobre o mesmo destino e guarde os artefatos por execução antes que outro comando os sobrescreva.

## Diagnóstico rápido

| Sintoma | Onde verificar |
| --- | --- |
| Conexão falha no `dbt debug` | Host, HTTP path, validade da credencial, rede e `CAN USE` no warehouse selecionado. |
| Conecta, mas o build recebe acesso negado | Identidade efetiva, catálogo/schema da relação que falhou e privilégios sobre esse objeto. |
| A tabela apareceu em outro schema | Confira o target e a configuração `schema` do modelo. Por padrão, schemas personalizados são concatenados ao schema do target. |
| Merge falha por múltiplas correspondências | Agrupe a origem por `pedido_id` e inspecione chaves com mais de uma linha. Corrija a granularidade no staging. |
| Build lento ou caro | Localize o SQL no Query History. Compare bytes lidos, duração e tempo de fila antes de ajustar filtro incremental, tamanho do warehouse ou `threads`. |
