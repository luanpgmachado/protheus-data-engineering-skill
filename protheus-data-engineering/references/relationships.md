# Relacionamentos e Sequência de Documentos

## JOINs centrais verificados

### SE2 (Contas a Pagar) ↔ SA2 (Fornecedores)
```sql
FROM SE2010 SE2
INNER JOIN SA2010 SA2
    ON SE2.E2_FORNECE = SA2.A2_COD
   AND SE2.E2_LOJA    = SA2.A2_LOJA
   AND SA2.A2_FILIAL  = SE2.E2_FILIAL   -- mesma filial (ajustar se SA2 for compartilhada no ambiente)
   AND SA2.D_E_L_E_T_ = ' '
WHERE SE2.D_E_L_E_T_ = ' '
```

### SC5 (Pedido de Venda) ↔ SA1 (Clientes) -- LEFT JOIN (cliente pode ter sido removido/bloqueado)
```sql
FROM SC5010 SC5
LEFT JOIN SA1010 SA1
    ON SC5.C5_CLIENTE = SA1.A1_COD
   AND SC5.C5_LOJACLI = SA1.A1_LOJA
   AND SA1.A1_FILIAL  = SC5.C5_FILIAL
   AND SA1.D_E_L_E_T_ = ' '
WHERE SC5.D_E_L_E_T_ = ' '
```

### SC6 (Item do Pedido de Venda) ↔ SB1 (Produtos)
Substitui um padrão comum e ineficiente de "buscar produto um a um por código": um único JOIN resolve todos os itens de uma vez.
```sql
FROM SC6010 SC6
INNER JOIN SB1010 SB1
    ON SB1.B1_FILIAL = SC6.C6_FILIAL
   AND SB1.B1_COD    = SC6.C6_PRODUTO
   AND SB1.D_E_L_E_T_ = ' '
WHERE SC6.D_E_L_E_T_ = ' '
```

### SD2 (Item NF Saída) ↔ SF2 (Cabeçalho NF Saída)
Chave de junção cabeçalho↔item -- vale o mesmo padrão para SD1↔SF1 (entrada).
```sql
FROM SD2010 SD2
INNER JOIN SF2010 SF2
    ON SF2.F2_FILIAL = SD2.D2_FILIAL
   AND SF2.F2_DOC    = SD2.D2_DOC
   AND SF2.F2_SERIE  = SD2.D2_SERIE
   AND SF2.D_E_L_E_T_ = ' '
WHERE SD2.D_E_L_E_T_ = ' '
```

## Sequência de documentos por módulo

### Compras
`SC1 (solicitação) → SC8 (cotação) → SC7 (pedido de compra) → SD1+SF1 (NF de entrada) → SE2 (título a pagar)`

- SC7 ↔ SC1 por `C7_NUMSC+C7_ITEMSC`
- SD1 ↔ SC7 por `D1_PEDIDO+D1_ITEMPC`
- SF1 ↔ SD1 por `FILIAL+DOC+SERIE+FORNECE+LOJA`
- SE2 é gerado a partir da NF (prefixo/número derivados do documento fiscal)
- `C1_QUJE`/`C7_QUJE` = saldo já convertido em pedido/recebido -- necessário para calcular "quanto falta receber" corretamente

### Faturamento
`SC5+SC6 (pedido de venda) → SC9 (liberação) → SF2+SD2 (NF de saída) → SE1 (título a receber)`

- SD2 ↔ SC6 por `D2_PEDIDO+D2_ITEMPV`
- SC9 ↔ SC5/SC6 por `C9_PEDIDO+C9_ITEM`
- Após faturar, SC5/SC6 recebem `C5_NOTA/C6_NOTA` e `C5_SERIE/C6_SERIE` de volta -- é o caminho inverso (NF→pedido) quando você já tem a nota e precisa achar o pedido de origem

### Financeiro
`SE1/SE2 (título) ↔ SE5 (baixa/movimentação bancária)`

- SE5 ↔ SE1/SE2 por `PREFIXO+NUMERO+PARCELA+TIPO`
- `E5_SITUACA = 'E'` marca estorno -- **nunca somar SE5 sem filtrar estorno**, pois o estorno é um registro complementar, não uma exclusão
- Em ambientes atualizados, verificar se o histórico de baixa está fragmentado na família `FKx` em vez de SE5 puro

### Contábil
Todos os módulos alimentam `CT2` (lançamento) via integração automática (regra de Lançamento Padrão, tabela `CT5`, fora do escopo de BI pois é configuração, não dado).

- `CT2_DEBITO`/`CT2_CREDIT` = `CT1_CONTA`
- Rastreamento origem→lançamento: `CTK`/`CV3` -- `CV3_TABORI+CV3_RECORI` aponta de volta para a tabela e registro de origem (SD1, SE2, SF2 etc.). Use isso para reconciliar dado fiscal/financeiro com o contábil via SQL, em vez de tentar inferir por data+valor.

### Estoque
`SD1/SD2 (movimento de compra/venda) → SD3 (movimentação interna, fato) → SB2 (saldo atual)`

- SD3 é o fato central: toda entrada/saída de estoque passa por ele, com `D3_CF` identificando o tipo (RE=saída, DE=entrada, faixas 500-998 e 001-499 respectivamente)
- SB2 é snapshot de saldo, não histórico -- para série temporal de estoque, use SD3 agregado por data, não SB2

### PCP
`SG1 (estrutura/BOM) → SC2 (ordem de produção) → SD4 (empenho) → SD3 (consumo real) → SB2 (saldo)`

- SC2 ↔ SG1 via explosão de `C2_PRODUTO`
- SC2 ↔ SC5/SC6 (venda) por `C2_PEDIDO+C2_ITEMPED` -- chave rara que liga produção diretamente a um pedido de venda específico (produção sob encomenda)

## Armadilhas de cardinalidade

| Situação | Risco | Como evitar |
|---|---|---|
| Somar `C6_QTDVEN` sem olhar `C9` | Um item pode estar fragmentado em várias linhas SC9 (bloqueio parcial de crédito/estoque) -- somar direto do SC6 ignora isso mas geralmente ainda está correto para "quantidade vendida"; o erro comum é o inverso: tentar derivar "quantidade liberada" de SC6 quando ela só existe em SC9 | Se a pergunta for sobre liberação/bloqueio, junte com SC9; se for sobre quantidade vendida total, SC6 basta |
| `INNER JOIN` com tabela auxiliar que pode não ter linha (ex. cotação de moeda) | Zera o resultado inteiro silenciosamente | Preferir `LEFT JOIN` quando a tabela auxiliar não é garantida, e registrar/alertar quando faltar |
| JOIN cabeçalho↔item sem todos os componentes da chave (esquecer `SERIE` ou `LOJA`) | Duplica linha ou casa item errado quando dois documentos têm o mesmo número em séries diferentes | Sempre usar a chave composta completa documentada em `standard-tables.md`, nunca só `FILIAL+DOC` |
| Agregar CT2 sem filtrar estorno/cancelamento | Débito e crédito de um lançamento cancelado ainda entram na soma | Filtrar `D_E_L_E_T_` e o indicador de estorno relevante da tabela |
| Tratar SC1/SC2/STJ como se tivessem tabela de item separada | Essas três são "cabeçalho único" -- cada linha já é o documento inteiro, não há SC1-item ou STJ-item companion | Confirmar granularidade antes de fazer GROUP BY assumindo 1:N |
| `NOT IN (SELECT ...)` para achar "sem pedido"/"sem baixa" | Plano de execução ruim em tabela grande, e quebra silenciosamente se a subquery tiver `NULL` | Reescrever como `LEFT JOIN ... WHERE <chave_direita> IS NULL` |
