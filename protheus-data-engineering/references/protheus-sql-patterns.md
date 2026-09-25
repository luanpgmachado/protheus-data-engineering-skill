# Padrões de SQL para o Protheus

O Protheus embute SQL em ADVPL através de macros (`%table%`, `%xfilial%`, `%notDel%`...) que são traduzidas automaticamente para SQL puro em tempo de execução. Como Engenheiro de Dados você não usa essas macros -- mas **precisa reproduzir manualmente o que elas fazem**, porque o Power Query/SQL direto não tem esse pré-processador.

## Tradução das macros ADVPL para SQL puro

Se você encontrar SQL embutido em relatório/rotina ADVPL legado (ex. ao auditar de onde vem um dado), esta tabela traduz o que cada macro significa:

| Macro ADVPL | O que expande | Equivalente em SQL puro que você escreve |
|---|---|---|
| `%table:XXX%` | Nome físico da tabela | `SA1010` (descubra via `data-dictionary.md`) |
| `%xfilial:XXX%` | Valor da filial atual | Literal da filial, ex. `'01'`, ou omitido se a tabela for compartilhada |
| `%notDel%` | Filtro de exclusão lógica | `D_E_L_E_T_ <> '*'` (equivalente a `D_E_L_E_T_ = ' '`) |
| `%exp:variavel%` | Valor de variável/parâmetro, já escapado | Bind parameter (`?`, `@param`) ou literal escapado |
| `%Order:XXX%` | Colunas da chave primária | Lista de colunas do índice 1, conforme SIX |
| `%nolock%` | Hint de leitura sem lock (só SQL Server) | `WITH (NOLOCK)` no SQL Server; omitir em Oracle/PostgreSQL |

## Regra 1 — `D_E_L_E_T_` é obrigatório em toda tabela

O Protheus **nunca** faz `DELETE` físico em tabela de negócio -- todo registro "excluído" recebe `D_E_L_E_T_ = '*'` e continua na tabela. Isso vale para toda tabela do FROM **e** de cada JOIN, sem exceção:

```sql
SELECT SA1.A1_COD, SA1.A1_NOME
FROM SA1010 SA1
INNER JOIN SE1010 SE1
    ON SE1.E1_CLIENTE = SA1.A1_COD
   AND SE1.E1_LOJA    = SA1.A1_LOJA
   AND SE1.D_E_L_E_T_ = ' '        -- obrigatório também na tabela do JOIN
WHERE SA1.D_E_L_E_T_ = ' '          -- obrigatório na tabela principal
```

Esquecer este filtro numa tabela do JOIN é o erro mais comum e mais silencioso: o resultado parece plausível, mas inclui registros logicamente excluídos.

## Regra 2 — Filial é escopo, não filtro automático

Não existe uma regra única de "sempre filtrar uma filial". A regra correta:

1. Verifique o modo de compartilhamento da tabela (`X2_MODO` -- ver `data-dictionary.md`).
2. Se a tabela é exclusiva por filial (`E`, a maioria das transacionais) e o pedido do usuário é sobre uma filial específica, filtre por ela.
3. Se o pedido é "consolidado" ou "todas as filiais", **não filtre** -- e se o relatório precisa identificar de qual filial veio cada linha, inclua a coluna de filial no SELECT/GROUP BY em vez de restringi-la no WHERE.
4. Se a tabela é compartilhada (`C`), o conceito de filial não se aplica -- não adicione o filtro por hábito.

## Regra 3 — Datas são strings `YYYYMMDD`

```sql
-- Errado: aplica função de data do banco numa coluna que é texto
WHERE YEAR(SE1.E1_EMISSAO) = 2026

-- Certo: compara como string, ou faz o range em string
WHERE SE1.E1_EMISSAO BETWEEN '20260101' AND '20261231'
```

Ao construir uma data a partir de parâmetros de Power Query/BI, formate para `YYYYMMDD` antes de enviar ao banco.

## Regra 4 — Nunca `SELECT *`

Tabelas Protheus costumam ter 100+ colunas, muitas delas de controle interno (`R_E_C_N_O_`, campos de trigger, campos legados não usados). Liste sempre as colunas necessárias -- isso também documenta, para quem ler a consulta depois, quais campos realmente importam.

## Regra 5 — JOIN explícito, nunca vírgula no FROM

```sql
-- Evitar
FROM SC5010 SC5, SA1010 SA1 WHERE SC5.C5_CLIENTE = SA1.A1_COD ...

-- Preferir
FROM SC5010 SC5
INNER JOIN SA1010 SA1 ON SC5.C5_CLIENTE = SA1.A1_COD AND ...
```

## Regra 6 — Parametrizar valores dinâmicos, nunca concatenar

Se a ferramenta de BI/ETL permitir SQL parametrizado (bind variables) para valores que vêm de filtro de usuário/parâmetro de dataflow, use-o em vez de montar a string por concatenação. O princípio é o mesmo que gera injeção de SQL em ADVPL (`%exp:` existe exatamente para isso) -- concatenar valor externo direto na string de query é sempre o padrão a evitar, mesmo fora do ADVPL.

## Regra 7 — Nome físico de tabela não é confiável entre ambientes

O alias lógico (`SA1`) é estável entre clientes/ambientes; o nome físico (`SA1010`) muda conforme a empresa configurada. Ao documentar uma consulta para reuso, deixe claro que o nome físico foi obtido de um ambiente específico e pode precisar de ajuste em outro -- ou parametrize o sufixo de empresa quando a ferramenta permitir.
