# Diferenças entre Oracle, SQL Server e PostgreSQL

O Protheus roda oficialmente sobre PostgreSQL, SQL Server e Oracle. O schema lógico (aliases, campos, `D_E_L_E_T_`, chaves) é o mesmo nos três -- o que muda é a sintaxe SQL.

## Equivalência de funções

| Operação | ANSI / portável | SQL Server | PostgreSQL | Oracle |
|---|---|---|---|---|
| Coalescência de nulo | `COALESCE(a, b)` | `ISNULL(a, b)` | `COALESCE(a, b)` | `NVL(a, b)` |
| Concatenação de string | `CONCAT(a, b)` | `a + b` | `a \|\| b` | `a \|\| b` |
| Data atual | -- | `GETDATE()` | `CURRENT_DATE` | `SYSDATE` |
| Timestamp atual | -- | `GETUTCDATE()` | `NOW()` | `SYSTIMESTAMP` |
| Limitar linhas | `FETCH FIRST n ROWS ONLY` | `TOP n` | `LIMIT n` | `FETCH FIRST n ROWS ONLY` |
| Paginação | `OFFSET x ROWS FETCH FIRST n ROWS ONLY` | `OFFSET x ROWS FETCH NEXT n ROWS ONLY` | `LIMIT n OFFSET x` | `OFFSET x ROWS FETCH FIRST n ROWS ONLY` |
| Substring | `SUBSTRING(s FROM start FOR len)` | `SUBSTRING(s, start, len)` | `SUBSTRING(s FROM start FOR len)` | `SUBSTR(s, start, len)` |
| Condicional | `CASE WHEN ... THEN ... END` | igual | igual | igual |
| Cast de tipo | `CAST(x AS tipo)` | igual | igual | igual |
| Tabela temporária | -- | `#nome_temp` | `CREATE TEMPORARY TABLE` | `CREATE GLOBAL TEMPORARY TABLE` |
| Leitura sem lock | `WITH (NOLOCK)` (só MSSQL) | `WITH (NOLOCK)` | ignorar (MVCC) | ignorar (MVCC) |

## Paginação -- forma portável

```sql
SELECT A1_COD, A1_NOME
FROM SA1010 SA1
WHERE SA1.D_E_L_E_T_ = ' '
  AND SA1.A1_FILIAL = '01'
ORDER BY SA1.A1_COD
OFFSET 50 ROWS FETCH NEXT 50 ROWS ONLY
```

`OFFSET ... FETCH NEXT ... ROWS ONLY` é ANSI SQL:2008 e funciona em SQL Server (2012+), Oracle (12c+) e, com `LIMIT n OFFSET x`, no PostgreSQL.

## Datas armazenadas como string

Em qualquer um dos três bancos, o Protheus grava data como string `YYYYMMDD` (não como tipo `DATE` nativo) na maioria das tabelas de negócio. Nunca use `EXTRACT`/`YEAR()`/`TO_CHAR` do banco para tratar essas colunas como se fossem tipo data nativo -- compare como string ou converta explicitamente:

```sql
-- Portável nos três bancos, porque compara como string
WHERE SE1.E1_EMISSAO BETWEEN '20260101' AND '20261231'
```

## Anti-padrões cross-database

| Anti-padrão | Problema | Correção |
|---|---|---|
| `GETDATE()` fora de contexto SQL Server | Quebra em PostgreSQL/Oracle | `CURRENT_DATE` (ANSI) ou consultar o dialeto do ambiente antes de escrever |
| `+` para concatenar string em Postgres/Oracle | Erro de sintaxe/tipo | `\|\|` ou `CONCAT()` |
| `WITH (NOLOCK)` fora do SQL Server | Erro de parser em Postgres/Oracle | Omitir -- MVCC já resolve leitura concorrente |
| `ISNULL()` fora do SQL Server | Não existe em Postgres/Oracle | `COALESCE()` |
| `LIMIT n` no SQL Server | Sintaxe inválida | `TOP n` ou `OFFSET/FETCH` |
| Função de data do banco sobre coluna `YYYYMMDD` armazenada como texto | Resultado incorreto em qualquer um dos três | Tratar como string ou converter primeiro |

## Como confirmar o dialeto do ambiente

Se não for informado qual banco está por trás do Protheus do cliente, pergunte ou identifique por sintoma:
- Erro de sintaxe em `LIMIT` → provavelmente SQL Server ou Oracle.
- Erro em `TOP n` → provavelmente PostgreSQL ou Oracle.
- Presença de `dbo.` no nome qualificado de tabela → SQL Server.
- Presença de schema em maiúsculas por padrão e `VARCHAR2`/`NUMBER` nos metadados → Oracle.
