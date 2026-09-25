# ETL de Dados Cervejeiros — Punk API / DIY Dog

Pipeline de ETL (Extract, Transform, Load) e análise de dados construído em Python + SQL, usando dados reais e abertos de receitas de cerveja. Projeto pessoal de portfólio, com foco em modelagem relacional, qualidade de dado e SQL avançado.

## Fonte dos dados

Os dados vêm do **DIY Dog**, o livro de receitas open-source da cervejaria BrewDog, acessados via um espelho comunitário da Punk API (a API oficial foi descontinuada em 2023). Licença: uso livre para replicar e compartilhar, **não-comercial** — este projeto é de portfólio pessoal, dentro dos termos de uso.

Foram extraídas **415 receitas**, iterando por ID com checkpoint de progresso (retomada segura em caso de interrupção), respeitando um intervalo entre requisições para não sobrecarregar o servidor do espelho.

## Arquitetura do banco

Dados normalizados em 4 tabelas relacionadas por chave estrangeira (`cerveja_id`):

| Tabela | Conteúdo |
|---|---|
| `tab_cervejas` | Dados principais: nome, ABV, IBU, temperatura de fermentação, etc. (tabela de dimensão) |
| `tab_maltes` | Maltes usados por receita (um-para-muitos) |
| `tab_lupulos` | Lúpulos usados, com quantidade e estágio de adição (um-para-muitos) |
| `tab_harmonizacao` | Pratos recomendados para cada cerveja (um-para-muitos) |

Decisão de design: normalizar em vez de manter tudo em uma tabela larga, evitando redundância e garantindo uma fonte única de verdade para os dados de cada cerveja (ao custo de exigir `JOIN` nas consultas que cruzam informação).

## Qualidade de dados

Durante a análise, dois tipos de valor atípico foram encontrados e tratados de forma diferente, após investigação:

- **IBU acima de 120 (alguns acima de 1000):** não é erro de importação — é uma característica conhecida da metodologia de cálculo teórico de IBU (fórmulas como Tinseth/Rager), que pode superar o teto de percepção humana (~100-120) em receitas muito carregadas de lúpulo. Em vez de apagar, foi criada uma coluna derivada `ibu_percebido` (limitada a 120), preservando o valor original calculado para não perder informação legítima.
- **Temperatura de fermentação de 99°C (1 registro):** fisicamente impossível — leveduras de cerveja não sobrevivem a essa temperatura. Testada a hipótese de erro de conversão Fahrenheit→Celsius (não se confirmou). Sem explicação sistemática encontrada, o valor foi corrigido para `NULL` (desconhecido), com o registro documentado aqui em vez de inventar um valor plausível.

## Técnicas de SQL demonstradas

- **Modelagem relacional** com chaves estrangeiras e tabelas de dimensão/fato
- **Window functions** (`ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`) para ranking de cervejas mais amargas por faixa de temperatura de fermentação
- **Índices e plano de execução** (`EXPLAIN QUERY PLAN`): comparação documentada de `SCAN` (varredura completa) vs. `SEARCH...USING INDEX` (busca direta) antes e depois de criar índice
- **Transações com SAVEPOINT**: inserção em lote com checkpoint intermediário. Se um lote falha (ex: violação de chave primária duplicada), apenas aquele lote é revertido, sem perder lotes anteriores já confirmados; registros perdidos são automaticamente reprocessados individualmente
- **Função customizada** registrada no banco (`conn.create_function`), equivalente conceitual a uma função `PL/pgSQL` no Postgres
- **Auditoria de sessão**: log automático (via `sqlite3.trace_callback`) de todo comando SQL executado na sessão, com timestamp e sem necessidade de instrumentar cada célula manualmente

## Como rodar

1. Instale as dependências: `pip install -r requirements.txt`
2. Abra `ETL_dados_cervejeiros.ipynb` no Google Colab ou Jupyter local
3. Rode as células em ordem (Kernel → Restart & Run All para reprodução limpa) — os dados brutos (`cervejas_raw.jsonl`) são baixados automaticamente pelo próprio notebook

## Limitações conhecidas
- Este notebook foi projetado para execução única e sequencial, do início ao fim
  (Kernel → Restart & Run All). Não é idempotente: re-executar isoladamente certas
  células (ex: inserção de dados de teste) pode falhar por depender de estado
  criado anteriormente na mesma sessão, como registros já inseridos.

## Próximos passos

- **Migração para PostgreSQL** (hospedagem gratuita via Neon): resolve limitação de concorrência do SQLite para acesso multiusuário, alinhando o projeto com o banco de dados mais pedido em vagas de dados no mercado
- **API de serving** com FastAPI, expondo consultas do banco (ex: busca por estilo, ranking de amargor) como endpoint público
- Camada de visualização/BI sobre os dados já modelados

## Autor

Denis Willian da Silva — [LinkedIn](https://www.linkedin.com/in/denis-w-silva/) · [GitHub](https://github.com/da-silva-d-w)
