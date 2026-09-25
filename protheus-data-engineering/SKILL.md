---
name: protheus-data-engineering
description: Use quando a demanda for Engenharia de Dados/BI sobre o banco de dados do TOTVS Protheus -- Power BI, Power Query/M, SQL direto (Oracle, SQL Server ou PostgreSQL), auditoria de query, ou entendimento do modelo de dados. Cobre identificação de tabelas por área de negócio (SA1/SA2/SB1/SC5/SC6/SD1/SD2/SE1/SE2/SF1/SF2/CT1/CT2 etc.), dicionário de dados SX2/SX3/SIX/SX5/SX6/SX9, filial e compartilhamento de tabela, D_E_L_E_T_, chaves e relacionamentos, cabeçalho vs item, performance de JOIN/índice, e diferenças entre Oracle/SQL Server/PostgreSQL. NÃO cobre desenvolvimento ADVPL/TLPP, MVC, REST Protheus, Pontos de Entrada, TIR, compilação, Workarea/DbSeek/RecLock, ou alteração do dicionário SX -- para isso, este repositório não mantém skill dedicada (fora do escopo de Engenharia de Dados). Dispara em português com frases como "query de faturamento por cliente", "que tabela do Protheus tem X", "join entre SC5 e SF2", "por que minha consulta duplicou linha", "SA1 é o quê", "otimizar essa consulta no banco do Protheus".
---

# Protheus Data Engineering

Porta de entrada para qualquer demanda de **Engenharia de Dados/BI sobre o banco de dados do TOTVS Protheus**: Power BI → Power Query/M → SQL direto no banco (Oracle, SQL Server ou PostgreSQL) → modelagem → dashboard. O uso é **predominantemente read-only**: o objetivo é ler e entender dados corretamente, não desenvolver rotinas dentro do ERP.

Esta skill deliberadamente **não** ensina ADVPL, TLPP, MVC, REST Protheus, Pontos de Entrada, Workarea (`DbSelectArea`/`DbSeek`/`RecLock`), compilação ou alteração do dicionário SX. Se a conversa migrar para esses temas, isso é sinal de que a demanda saiu do escopo de Engenharia de Dados -- confirme com o usuário antes de seguir por esse caminho.

## Fluxo obrigatório para gerar/revisar uma consulta

Antes de escrever qualquer SQL contra o Protheus, siga esta sequência -- ela existe porque pular etapas é a causa mais comum de dado errado (linha duplicada, filial vazando, registro excluído aparecendo):

1. **Identificar as tabelas conceituais envolvidas.** Ex.: "faturamento por cliente" → SF2/SD2 (nota de saída), SC5/SC6 (pedido), SA1 (cliente), SB1 (produto). Consulte [`references/standard-tables.md`](references/standard-tables.md).
2. **Identificar os campos necessários.** Se o campo não estiver documentado, consulte o dicionário real (passo 8) antes de assumir o nome.
3. **Validar as chaves de junção.** Não invente FK -- confirme em [`references/relationships.md`](references/relationships.md) ou no SX9/índices do ambiente real.
4. **Avaliar a estratégia de filial da tabela** (compartilhada vs exclusiva). Ver [`references/data-dictionary.md`](references/data-dictionary.md#compartilhamento-e-filial). **Não assuma que o filtro de filial deve restringir a uma única filial** -- identifique o modo de compartilhamento e aplique o filtro conforme o escopo pedido pelo usuário (uma filial, um grupo, ou todas).
5. **Aplicar `D_E_L_E_T_`** em toda tabela referenciada no FROM e em cada JOIN -- sem exceção. Ver [`references/protheus-sql-patterns.md`](references/protheus-sql-patterns.md).
6. **Consultar o dicionário (SX3/SIX/SX9) quando houver dúvida** sobre campo, índice ou relacionamento -- não adivinhe.
7. **Avaliar a cardinalidade do JOIN** (1:1 cabeçalho↔item vs 1:N) para não duplicar linha ao somar valor. Ver [`references/relationships.md`](references/relationships.md#armadilhas-de-cardinalidade).
8. **Se houver acesso ao banco real (MCP SQL ou equivalente), valide o dicionário do cliente** em vez de assumir apenas campos padrão -- campo customizado e alteração de cliente são sempre possíveis. Rode a consulta de descoberta descrita em [`references/data-dictionary.md`](references/data-dictionary.md#consultando-o-dicionário-via-sql).
9. **Só então gerar o SQL**, já alinhado a índice (`references/sql-performance.md`) e no dialeto certo (`references/database-specific.md`).

## Arquivos de referência

| Arquivo | Ler quando |
|---|---|
| [`references/data-dictionary.md`](references/data-dictionary.md) | Precisar entender/consultar SX2 (tabelas), SX3 (campos), SIX (índices), SX5/SX6 (domínios/parâmetros), SX9 (relacionamentos), ou descobrir estrutura de tabela customizada |
| [`references/standard-tables.md`](references/standard-tables.md) | Precisar identificar qual tabela padrão atende a uma necessidade de negócio, e seus campos-chave |
| [`references/relationships.md`](references/relationships.md) | Precisar montar JOIN entre tabelas, entender sequência de documento (pedido→NF→título→lançamento), ou diagnosticar duplicidade de linha |
| [`references/protheus-sql-patterns.md`](references/protheus-sql-patterns.md) | Escrever/revisar o WHERE de qualquer query -- regras de `D_E_L_E_T_`, filial, data como string, injeção de SQL |
| [`references/sql-performance.md`](references/sql-performance.md) | Consulta lenta, dúvida de índice, revisão de performance, ou anti-padrão de SQL |
| [`references/database-specific.md`](references/database-specific.md) | Precisar traduzir sintaxe entre Oracle / SQL Server / PostgreSQL |

## Regras que nunca mudam

- **Toda tabela transacional do Protheus usa exclusão lógica.** Sempre filtrar `D_E_L_E_T_ <> '*'` (equivalente a `D_E_L_E_T_ = ' '`) em toda tabela do FROM e de cada JOIN. Nunca existe `DELETE` físico em tabela padrão.
- **Filial não é um filtro fixo, é uma decisão de escopo.** Identifique se a tabela é compartilhada (`X2_MODO = C`) ou exclusiva por filial (`X2_MODO = E`) antes de decidir como filtrar -- ver `data-dictionary.md`.
- **Datas ficam armazenadas como string `YYYYMMDD`.** Nunca usar função de data do banco para extrair ano/mês/dia direto da coluna -- comparar como string ou converter primeiro.
- **Nome físico de tabela não é confiável entre ambientes.** O alias lógico (`SA1`, `SE1`...) é estável; o nome físico (`SA1010`) muda por empresa/ambiente. Ao gerar SQL para reuso, prefira parametrizar o nome físico ou documentar que ele foi obtido do dicionário daquele ambiente específico.
- **Campo/tabela custom é sempre possível.** Antes de afirmar que um campo não existe ou assumir apenas o padrão TOTVS, considere que o cliente pode ter customizado -- valide no dicionário real quando houver acesso ao banco.
- **Cabeçalho e item nem sempre são tabelas separadas.** Em vários módulos (SC1, SC2, STJ) cabeçalho e item estão na mesma tabela; confirme a granularidade antes de fazer GROUP BY ou assumir 1:N.
