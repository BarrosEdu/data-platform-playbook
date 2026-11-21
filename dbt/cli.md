# CLI do dbt Core por fase do projeto

Referência para **dbt Core 1.10.x** com `dbt-databricks`. Os exemplos usam o projeto de pedidos do [guia Databricks](README.md). Execute os comandos no terminal com o ambiente Python ativo. A partir da fase 2, trabalhe na pasta que contém `dbt_project.yml`. Em um runner, use `--project-dir` para indicar essa pasta e `--profiles-dir` para indicar a pasta do perfil. [Implementação da CLI 1.10](https://github.com/dbt-labs/dbt-core/blob/1.10.latest/core/dbt/cli/main.py)

## Catálogo completo de comandos

O dbt Core 1.10.x tem **19 comandos executáveis**, organizados abaixo por fase. A coluna “Comando” usa o nome canônico. Há também dois grupos (`dbt docs` e `dbt source`) e dois aliases. Para consultar as opções disponíveis na versão instalada, acrescente `--help` ao comando, por exemplo `dbt source freshness --help`. As seções seguintes mostram como aplicá-los no Databricks.

| Fase | Comando | O que faz |
| --- | --- | --- |
| Criação | `dbt init` | Cria a estrutura inicial do projeto e oferece configurar o perfil de conexão. |
| Conexão | `dbt debug` | Verifica o ambiente, os arquivos de configuração e a conexão com o destino selecionado. |
| Dependências | `dbt deps` | Resolve e instala os pacotes dbt declarados pelo projeto, usando o arquivo de lock quando aplicável. |
| Inspeção | `dbt parse` | Lê o projeto, identifica recursos e dependências e grava o manifesto. |
| Inspeção | `dbt list` | Lista os recursos que correspondem à seleção. O alias `dbt ls` produz o mesmo resultado. |
| Desenvolvimento | `dbt compile` | Renderiza Jinja, macros e referências em SQL que você pode inspecionar antes da execução dos modelos. |
| Desenvolvimento | `dbt show` | Executa a consulta de um recurso ou uma consulta inline e exibe uma amostra do resultado no terminal. |
| Ingestão | `dbt source freshness` | Mede a idade da carga das origens configuradas e compara com seus limites de aviso e erro. O alias legado é `dbt source snapshot-freshness`. |
| Carga | `dbt seed` | Carrega os CSVs de referência do projeto em tabelas no destino. |
| Transformação | `dbt run` | Executa os modelos selecionados em ordem de dependência, aplicando suas materializações. |
| Histórico | `dbt snapshot` | Compara as linhas atuais com o snapshot anterior e registra versões históricas conforme a estratégia configurada. |
| Qualidade | `dbt test` | Executa testes de dados e testes unitários selecionados e informa quais passaram ou falharam. |
| Integração | `dbt build` | Executa os recursos selecionados, como seeds, snapshots, modelos e testes, respeitando o grafo e as dependências entre eles. |
| Documentação | `dbt docs generate` | Gera os arquivos do site de documentação com descrições, linhagem e metadados do catálogo. |
| Documentação | `dbt docs serve` | Inicia um servidor HTTP local para navegar na documentação já gerada. |
| CI | `dbt clone` | Cria no target relações correspondentes às do ambiente descrito pelo manifesto passado em `--state`. A implementação depende do adaptador. |
| Operação | `dbt run-operation` | Invoca pelo nome uma macro do projeto ou de um pacote, com argumentos opcionais. Seu efeito depende da macro. |
| Recuperação | `dbt retry` | Usa os resultados da execução anterior para retomar os nós pendentes após uma falha em um comando compatível. |
| Limpeza | `dbt clean` | Remove os diretórios locais listados em `clean-targets`, por exemplo o diretório de artefatos do projeto. |

Os grupos organizam os subcomandos. Use `dbt docs --help` para ver as ações de documentação e `dbt source --help` para ver a ação de verificação das origens. Na raiz, `dbt --help` lista os comandos públicos e `dbt --version` mostra as versões do Core e dos adaptadores instalados. [Registro oficial dos comandos e aliases da série 1.10](https://github.com/dbt-labs/dbt-core/blob/1.10.latest/core/dbt/cli/main.py)

## 1. Criar o projeto

| Comando | Onde e quando usar |
| --- | --- |
| `dbt --version` | Em qualquer pasta, confirme a versão do Core e a presença do plugin Databricks. |
| `dbt --help` e `dbt build --help` | Consulte os comandos instalados e as opções aceitas por cada um. |
| `dbt init vendas --skip-profile-setup` | Na pasta pai, inicialize o projeto uma vez. Omita a flag para usar o assistente de perfil. |
| `dbt debug --config-dir` | Localize a pasta de configuração do perfil. |
| `dbt debug --target dev` | Dentro do projeto, valide arquivos, adaptador e conexão com o warehouse de desenvolvimento. |

## 2. Resolver dependências e inspecionar o grafo

| Comando | Quando usar e o que observar |
| --- | --- |
| `dbt deps` | Após clonar ou alterar um projeto com pacotes declarados, instale-os e versione o `package-lock.yml` gerado. Use `dbt deps --upgrade` para atualizar dentro das restrições e revise o diff. |
| `dbt parse` | Valide Jinja, YAML, referências e configurações antes da execução. O grafo fica em `target/manifest.json`. Erros de sintaxe do SQL Databricks ainda podem aparecer no warehouse. |
| `dbt list --resource-type model` | Liste os modelos conhecidos. `dbt ls` é o alias. |
| `dbt ls --select +fct_pedidos --output name` | Confira a seleção antes de um build ou reprocessamento. |
| `dbt compile --select fct_pedidos` | Inspecione o SQL renderizado em `target/compiled/`. Macros de introspecção podem consultar o warehouse durante a compilação. |
| `dbt show --select stg_pedidos --limit 5` | Execute a consulta do modelo e veja uma amostra no terminal. As relações físicas referenciadas precisam existir. |

**Como selecionar.** `+fct_pedidos` inclui ancestrais e `stg_pedidos+` inclui descendentes. Um espaço entre seletores representa união, e uma vírgula representa interseção. Por exemplo, `dbt ls --select 'path:models/marts,tag:diario'` lista modelos nessa pasta com a tag indicada. Use `--exclude` para remover recursos e `--selector NOME` para reutilizar uma seleção definida em `selectors.yml`. [Sintaxe de seleção](https://docs.getdbt.com/reference/node-selection/syntax)

## 3. Carregar, construir e testar

| Comando | Quando usar e efeito no Databricks |
| --- | --- |
| `dbt source freshness --select source:vendas_brutas.pedidos` | Após a ingestão, confira os limites de atraso configurados na source. O resultado padrão fica em `target/sources.json`. Rode essa etapa explicitamente no job. |
| `dbt seed` | Carregue CSVs pequenos e versionados de `seeds/` como tabelas de referência, por exemplo uma lista de códigos. |
| `dbt run --select +fct_pedidos` | Materialize os modelos selecionados em ordem de dependência. Útil ao iterar no SQL. |
| `dbt test --select fct_pedidos` | Execute os testes selecionados. Para testes de dados, a relação precisa existir. Um teste singular em `tests/` retorna as linhas que violam a regra. |
| `dbt test --select test_type:unit` | Valide lógica com fixtures quando o projeto declarar testes unitários. Use-os para cobrir casos de borda antes de processar toda a origem. |
| `dbt snapshot` | Execute snapshots declarados em `snapshots/` para manter histórico de versões das linhas. Agende conforme a frequência das mudanças que precisa capturar. |
| `dbt build --select +fct_pedidos` | Construa e teste o recorte em ordem de dependência. Um teste bloqueante com erro pode fazer descendentes serem pulados. Sem seleção, o build inclui os recursos habilitados compatíveis, inclusive seeds e snapshots. |
| `dbt build --full-refresh --select fct_pedidos --target dev` | Reconstrua a tabela incremental do exemplo após alterar uma regra histórica. Requer ler a origem completa e substitui a relação de destino. Confira o target e se a origem ainda contém o histórico necessário. |

Para parametrizar uma regra implementada com `var('dias')`, passe `--vars '{"dias": 3}'`. Para limitar concorrência durante um diagnóstico, use `--threads 1`. Para investigar testes de dados, `dbt test --store-failures` grava registros que violaram a regra no destino de auditoria e requer privilégios de escrita nesse schema.

### Comandos de testes

Rode os comandos dentro de `vendas/`, com o target `dev` do guia. Em um schema novo, comece com `dbt build --select +fct_pedidos` para criar as relações e executar os testes na ordem do grafo. Depois de ajustar apenas uma regra de qualidade, use `dbt test` para verificá-la sobre os dados já materializados.

| Quero… | Comando |
| --- | --- |
| Ver quais testes de dados dependem do modelo, antes de executá-los | `dbt ls --select fct_pedidos --resource-type test --output name` |
| Rodar toda a suíte configurada no projeto | `dbt test` |
| Rodar os testes associados ao modelo final | `dbt test --select fct_pedidos` |
| Verificar só a unicidade da coluna `pedido_id` do modelo final | `dbt test --select unique_fct_pedidos_pedido_id` |
| Rodar todas as verificações de campos obrigatórios | `dbt test --select test_name:not_null` |
| Rodar os testes genéricos declarados em YAML | `dbt test --select test_type:generic` |
| Rodar as regras escritas como consultas SQL em `tests/` | `dbt test --select test_type:singular` |
| Validar a lógica dos modelos com entradas simuladas, quando houver fixtures em `unit_tests:` | `dbt test --select test_type:unit` |

O nome `unique_fct_pedidos_pedido_id` corresponde à declaração `unique` do guia. Se personalizar o nome no YAML, copie o nome retornado por `dbt ls`. Confira o número de testes executados no resumo: uma seleção vazia pode terminar sem erro. [Seleção de testes no dbt](https://docs.getdbt.com/reference/node-selection/test-selection-examples)

#### Criar, executar e investigar uma regra SQL

No exemplo de pedidos, toda chave presente no staging deve chegar à tabela final. Salve a consulta abaixo em `tests/pedidos_ausentes_no_mart.sql` no projeto dbt. Cada linha retornada identifica uma chave ausente.

```sql
select distinct origem.pedido_id
from {{ ref('stg_pedidos') }} as origem
left anti join {{ ref('fct_pedidos') }} as destino
    on origem.pedido_id = destino.pedido_id
where origem.pedido_id is not null
```

```bash
dbt test --select pedidos_ausentes_no_mart
# Se falhar, persista as chaves para investigação no SQL Warehouse.
dbt test --select pedidos_ausentes_no_mart --store-failures
```

Com a configuração padrão, zero linhas resulta em `PASS`. Em caso de `FAIL`, abra no SQL Editor a relação de auditoria indicada pelo dbt e consulte os IDs. Verifique se a ingestão terminou antes do build e se as linhas ficaram fora da janela incremental. Após corrigir a causa, rode `dbt build --select +fct_pedidos` para reconstruir o recorte e repetir as verificações.

## 4. Documentar e revisar

| Comando | Quando usar |
| --- | --- |
| `dbt docs generate` | Após o build, combine descrições do projeto e metadados do warehouse. Procure `index.html`, `manifest.json` e `catalog.json` em `target/`. |
| `dbt docs serve --port 8080` | Abra a documentação gerada para revisão local. Use Ctrl+C para encerrar o servidor. Para compartilhar, publique os artefatos em hospedagem com acesso adequado aos metadados. |

## 5. CI e operação

Para CI seletiva, defina um output chamado `ci` no perfil com schema exclusivo por PR. Disponibilize o manifesto de produção em `artifacts/prod/manifest.json` e preserve esse diretório durante a execução. O exemplo pressupõe um runner com permissão de leitura das relações de produção.

```bash
dbt ls --target ci --select 'state:modified+' --state artifacts/prod
dbt build --target ci --select 'state:modified+' --defer --state artifacts/prod
```

`state:modified+` seleciona recursos alterados e seus descendentes. O defer permite resolver referências não selecionadas usando as relações do manifesto anterior quando elas faltam no destino de CI. No primeiro deploy, sem manifesto de referência, rode `dbt build --target ci` completo. [Seleção por estado e defer](https://docs.getdbt.com/reference/node-selection/defer)

| Comando | Quando usar e pré-requisitos |
| --- | --- |
| `dbt clone --target ci --state artifacts/prod --select fct_pedidos` | Prepare a relação no schema de CI a partir do estado anterior, por exemplo para exercitar o caminho incremental. Requer acesso à origem e ao destino. A materialização do clone depende do tipo de relação e do adaptador. |
| `dbt retry` | Após corrigir uma falha transitória, retome recursos que falharam ou foram pulados na última execução compatível. Preserve o mesmo projeto, target e `target/run_results.json`. |
| `dbt run-operation minha_macro --args '{"dias": 3}'` | Execute uma macro administrativa já definida pelo projeto ou pacote. Substitua o nome e argumentos pelos da macro e confira o target antes de executar DDL. |
| `dbt clean` | Na máquina de desenvolvimento ou no fim do job, remova as pastas locais configuradas em `clean-targets`. Arquive resultados antes e rode `dbt deps` novamente se a pasta de pacotes tiver sido removida. |

No pipeline, trate o código de saída **0** como sucesso, **1** como erro tratado e **2** como erro não tratado. Para investigar, correlacione o nó com falha em `target/run_results.json`, seu SQL compilado, `logs/dbt.log` e a consulta no Query History do Databricks. Para mais detalhes em uma reprodução local, execute `dbt --debug build --select fct_pedidos`. [Códigos de saída](https://docs.getdbt.com/reference/exit-codes)
