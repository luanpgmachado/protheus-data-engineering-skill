# Performance e Índices

Todo índice do Protheus está registrado na tabela de dicionário `SIX` (ver `data-dictionary.md` para a consulta de descoberta real). A tabela abaixo é o layout padrão mais comum -- **ordens podem variar por ambiente/versão**; confirme via `SELECT * FROM SIX010 WHERE INDICE = '<alias>'` antes de otimizar uma consulta em produção.

## Índices comuns por tabela

| Tabela | Ordem | Expressão da chave | Uso |
|---|---|---|---|
| SA1 | 1 | `A1_FILIAL+A1_COD+A1_LOJA` | Busca por código do cliente |
| SA1 | 3 | `A1_FILIAL+A1_CGC` | Busca por CNPJ/CPF |
| SA2 | 1 | `A2_FILIAL+A2_COD+A2_LOJA` | Busca por código do fornecedor |
| SA2 | 3 | `A2_FILIAL+A2_CGC` | Busca por CNPJ/CPF |
| SB1 | 1 | `B1_FILIAL+B1_COD` | Busca por produto |
| SB2 | 1 | `B2_FILIAL+B2_COD+B2_LOCAL` | Saldo por produto+armazém |
| SC5 | 1 | `C5_FILIAL+C5_NUM` | Pedido por número |
| SC6 | 1 | `C6_FILIAL+C6_NUM+C6_ITEM+C6_PRODUTO` | Item do pedido |
| SD1 | 1 | `D1_FILIAL+D1_DOC+D1_SERIE+D1_FORNECE+D1_LOJA+D1_COD+D1_ITEM` | Item de NF de entrada (chave completa) |
| SD2 | 1 | `D2_FILIAL+D2_DOC+D2_SERIE+D2_CLIENTE+D2_LOJA+D2_COD+D2_ITEM` | Item de NF de saída (chave completa) |
| SE1 | 1 | `E1_FILIAL+E1_PREFIXO+E1_NUM+E1_PARCELA+E1_TIPO` | Título por documento |
| SE1 | 2 | `E1_FILIAL+E1_CLIENTE+E1_LOJA+E1_PREFIXO+E1_NUM+E1_PARCELA+E1_TIPO` | Títulos por cliente |
| SE1 | 6 | `E1_FILIAL+DTOS(E1_VENCREA)` | Títulos por data real de vencimento |
| SE2 | 1 | `E2_FILIAL+E2_PREFIXO+E2_NUM+E2_PARCELA+E2_TIPO+E2_FORNECE+E2_LOJA` | Título por documento |
| SF1 | 1 | `F1_FILIAL+F1_DOC+F1_SERIE+F1_FORNECE+F1_LOJA+F1_TIPO` | Cabeçalho NF entrada |
| SF2 | 1 | `F2_FILIAL+F2_DOC+F2_SERIE+F2_CLIENTE+F2_LOJA+F2_TIPO` | Cabeçalho NF saída |

## Alinhar o WHERE ao índice

O otimizador só usa um índice quando as colunas líderes da expressão de chave aparecem no WHERE, na mesma ordem.

**Bom (usa o índice 2 de SE1):**
```sql
SELECT E1_PREFIXO, E1_NUM, E1_VALOR
FROM SE1010 SE1
WHERE SE1.D_E_L_E_T_ = ' '
  AND SE1.E1_FILIAL   = '01'        -- 1º componente
  AND SE1.E1_CLIENTE  = '000001'    -- 2º componente
  AND SE1.E1_LOJA     = '01'        -- 3º componente
  AND SE1.E1_VALOR > 0
```

**Ruim (força varredura completa -- pula os componentes líderes):**
```sql
SELECT E1_PREFIXO, E1_NUM
FROM SE1010 SE1
WHERE SE1.D_E_L_E_T_ = ' '
  AND SE1.E1_PREFIXO = '001'
  AND SE1.E1_VENCTO BETWEEN '20260101' AND '20260131'
-- Correção: adicionar E1_FILIAL, ou usar o índice 6 (DTOS(E1_VENCREA)) se a busca é por data
```

## Anti-padrões (matam o uso de índice ou o plano de execução)

| Anti-padrão | Problema | Correção |
|---|---|---|
| `WHERE campo LIKE '%texto%'` (wildcard à esquerda) | Nenhum índice pode ser usado | Evitar wildcard à esquerda, ou usar full-text search |
| Função sobre a coluna no WHERE (`YEAR(campo)`, `TO_CHAR(campo,...)`) | Desabilita o índice | Reescrever como range (`BETWEEN`) sobre o valor bruto |
| Pular o campo de filial no WHERE de tabela indexada por filial | Otimizador não consegue usar o índice líder | Sempre incluir o filtro de filial primeiro (ou explicitamente decidir omiti-lo por design, ver `protheus-sql-patterns.md`) |
| `OR` entre colunas de índices diferentes | Cai para varredura completa na maioria dos otimizadores | Separar em duas consultas com `UNION ALL`, cada uma usando seu índice |
| `NOT IN (SELECT ...)` em tabela grande | Plano ruim; quebra com `NULL` na subquery | Reescrever como `LEFT JOIN ... WHERE <chave> IS NULL` |
| `SELECT *` | Traz dezenas de colunas de sistema, lento | Listar só as colunas necessárias |
| Buscar existência com `COUNT(*)` ou `RecCount()`-equivalente | Conta tudo antes de responder | `SELECT TOP 1` / `LIMIT 1` + checar se veio linha |
| Loop de uma consulta por item (N+1) para depois juntar em memória | N round-trips ao banco em vez de 1 | Um único `INNER JOIN`/`IN (...)` que resolve tudo de uma vez (ver `relationships.md`) |

## Validando o plano de execução

| Banco | Comando |
|---|---|
| SQL Server | `SET SHOWPLAN_TEXT ON` antes da query, ou Estimated Plan no SSMS |
| PostgreSQL | `EXPLAIN (ANALYZE, BUFFERS) <query>` |
| Oracle | `EXPLAIN PLAN FOR <query>; SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);` |

Procure `Index Seek`/`Index Scan` (bom) vs `Table Scan`/`Seq Scan` (ruim em tabela grande). Se o plano mostrar scan onde se esperava seek, revise a ordem das colunas do WHERE contra a expressão de chave do SIX.

## Leitura sem contenção de lock (SQL Server)

Extração de BI é leitura concorrente com o ERP em produção. No SQL Server, aplicar `WITH (NOLOCK)` em toda tabela do FROM/JOIN de consultas somente-leitura reduz espera por lock (PostgreSQL/Oracle usam MVCC e não precisam disso):

```sql
FROM SF2010 SF2 WITH (NOLOCK)
INNER JOIN SD2010 SD2 WITH (NOLOCK) ON ...
```

`NOLOCK` pode ler dados de uma transação não confirmada (dirty read) -- aceitável para a maioria dos relatórios de BI, mas evite em números que exigem consistência forte (ex. fechamento contábil).

## Se a extração estiver lenta por contenção com o ERP (diagnóstico, não desenvolvimento)

Quando a lentidão não é da query em si, mas de contenção de lock com o ERP em produção, estas consultas identificam quem está bloqueando:

**SQL Server:**
```sql
EXEC sp_who2
EXEC sp_lock
```

**PostgreSQL:**
```sql
SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

**Oracle:**
```sql
SELECT blocking.sid AS blocking_sid, blocking.username AS blocking_user,
       blocked.sid AS blocked_sid, blocked.username AS blocked_user
FROM v$lock l1
JOIN v$session blocking ON blocking.sid = l1.sid
JOIN v$lock l2 ON l1.id1 = l2.id1 AND l1.id2 = l2.id2
JOIN v$session blocked ON blocked.sid = l2.sid
WHERE l1.block = 1 AND l2.request > 0;
```

Isso é diagnóstico de infraestrutura, não desenvolvimento -- útil só quando a extração de dados está sendo bloqueada pelo próprio ERP em produção.

## Referência: códigos de regra de qualidade TOTVS relacionados a dicionário

Ao auditar SQL embutido em rotina ADVPL legada (para entender de onde um relatório tira um número antes de recriá-lo em Power Query), estes códigos aparecem em ferramentas de análise estática TOTVS e indicam acesso direto às tabelas de dicionário -- sinal de que a rotina original não usa view/API padrão e pode ter lógica de negócio escondida vale a pena investigar:

| Código | Tabela de dicionário envolvida |
|---|---|
| CA2001 | SIX (índices) |
| CA2003 | SX2 (tabelas) |
| CA2004 | SX3 (campos) |
| CA2006 | SX9 (relacionamentos) |
| CA2009 | SX5 (domínios) |
| CA2010 | SX6 (parâmetros) |
