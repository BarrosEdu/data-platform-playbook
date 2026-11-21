# Data Platform Playbook

Playbook pessoal de comandos, configurações e boas práticas para as principais
ferramentas de linha de comando usadas na criação e operação de plataformas de
dados.

## Objetivo

Centralizar exemplos reproduzíveis de CLI para tarefas como:

- provisionamento de infraestrutura;
- execução e orquestração de workloads;
- administração de plataformas de dados;
- observabilidade e troubleshooting;
- automação de ambientes locais e cloud.

## Playbooks

- [dbt Core no Databricks](dbt/README.md) — configure o SQL Warehouse e o Unity Catalog, construa um modelo Delta e leve o projeto para produção.
- [Catálogo completo da CLI do dbt Core 1.10.x](dbt/cli.md#catálogo-completo-de-comandos) — veja o que cada comando faz e consulte exemplos por fase do projeto.
- [Comandos de testes do dbt](dbt/cli.md#comandos-de-testes) — rode a suíte, selecione um teste de coluna e investigue falhas no Databricks.

## Segurança

Não publique credenciais, tokens, chaves privadas ou arquivos de configuração
com dados sensíveis. Use variáveis de ambiente e arquivos locais ignorados pelo
Git.

