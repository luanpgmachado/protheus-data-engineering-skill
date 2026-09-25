# Catálogo de Tabelas Padrão do Protheus

Convenção de nomenclatura: alias `XXN` onde `XX` é o prefixo do módulo e `N` um dígito de sequência. Campos seguem `XX_CAMPO` com o mesmo prefixo de duas letras da tabela (ex. tabela `SA1` → campos `A1_*`).

| Prefixo | Módulo/Área | Tabelas típicas |
|---|---|---|
| SA | Clientes/Fornecedores/Vendedores | SA1, SA2, SA3, SA5 |
| SB | Produtos/Estoque (cadastro) | SB1, SB2, SB5, SB8 |
| SC | Pedidos/Ordens | SC1, SC2, SC5, SC6, SC7, SC8, SC9 |
| SD | Documentos/Movimentações | SD1, SD2, SD3, SD4 |
| SE | Financeiro | SE1, SE2, SE5 |
| SF | Notas Fiscais/Livros Fiscais | SF1, SF2, SF3, SFT |
| CT | Contábil | CT1, CT2, CTD, CTT, CTK, CV3 |
| ST | Manutenção de Ativos | ST9, STF, STG, STI, STJ, STL, TQB |
| SG | PCP (estrutura/roteiro) | SG1, SG2 |
| SX/SI | Dicionário de dados | ver `data-dictionary.md` |
| Z* / ZZ* | Customizado pelo cliente | Z01–Z99, ZA0–ZAZ, ZZ1–ZZZ |

O nome físico no banco é o alias + complemento de empresa (ex. `SA1010`). **Nunca hardcode** esse nome ao documentar uma consulta reutilizável -- confirme via `SELECT X2_ARQUIVO FROM SX2010 WHERE X2_CHAVE='SA1'` no ambiente real (ver `data-dictionary.md`).

---

## Tabelas centrais (schema verificado campo a campo)

### SA1 — Clientes (cabeçalho, mestre)
Chave: `A1_FILIAL+A1_COD+A1_LOJA`

| Campo | Tipo/Tam | Descrição |
|---|---|---|
| A1_FILIAL | C 2 | Filial |
| A1_COD | C 6 | Código do cliente |
| A1_LOJA | C 2 | Loja do cliente |
| A1_NOME | C 50 | Nome/razão social |
| A1_NREDUZ | C 20 | Nome reduzido |
| A1_PESSOA | C 1 | Física/Jurídica |
| A1_CGC | C 14 | CNPJ/CPF |
| A1_MUN / A1_EST / A1_CEP | -- | Endereço |
| A1_COND | C 3 | Condição de pagamento padrão |
| A1_VEND | C 6 | Vendedor padrão |
| A1_MSBLQL | C 1 | Cliente bloqueado |

### SA2 — Fornecedores (cabeçalho, mestre)
Chave: `A2_FILIAL+A2_COD+A2_LOJA`. Campos espelham SA1 (`A2_NOME`, `A2_CGC`, `A2_COND`, `A2_CONTA`, `A2_CONTRIB`).

### SB1 — Produtos (cabeçalho, mestre)
Chave: `B1_FILIAL+B1_COD`

| Campo | Descrição |
|---|---|
| B1_COD | Código do produto |
| B1_DESC | Descrição |
| B1_TIPO | Tipo (MP, PA, ME, SV...) |
| B1_UM | Unidade de medida |
| B1_LOCPAD | Armazém padrão |
| B1_GRUPO | Grupo de estoque |
| B1_POSIPI | NCM |
| B1_CODBAR | Código de barras |
| B1_MSBLQL | Produto bloqueado |

### SB2 — Saldos em Estoque (estado atual, não histórico)
Chave: `B2_FILIAL+B2_LOCAL+B2_COD`. `B2_QATU` (saldo atual), `B2_RESERVA`/`B2_QEMP` (reservado/empenhado), `B2_QPEDVEN` (em pedido de venda). **É snapshot de saldo**, não movimentação -- para histórico use SD3.

### SD3 — Movimentações Internas de Estoque (fato transacional, sem chave única)
Campos: `D3_FILIAL`, `D3_TM` (tipo de movimento), `D3_COD` (produto), `D3_LOCAL` (armazém), `D3_QUANT`, `D3_CF` (código fiscal do tipo de requisição/devolução), `D3_DOC`, `D3_OP` (ordem de produção), `D3_NFISCAL`+`D3_SERIE` (NF de origem, quando aplicável). `D3_CF` na faixa `RE0-RE9` = saída (500-998), `DE0-DE9` = entrada (001-499), `999` = transferência/sistema.

### SC5 — Pedidos de Venda (cabeçalho)
Chave: `C5_FILIAL+C5_NUM`. `C5_CLIENTE+C5_LOJACLI`, `C5_LIBEROK` (liberado total), `C5_BLQ` (bloqueio), `C5_NOTA`/`C5_SERIE` (preenchidos após faturamento).

### SC6 — Itens dos Pedidos de Venda
Chave: `C6_FILIAL+C6_NUM+C6_ITEM+C6_PRODUTO`. `C6_QTDVEN` (vendida) vs `C6_QTDENT` (já faturada) = saldo a faturar. `C6_BLQ` (bloqueio por item), `C6_NOTA` (NF gerada).

### SC7 — Pedidos de Compra (item; não há cabeçalho SC7 separado além do próprio item)
Chave: `C7_FILIAL+C7_NUM+C7_ITEM+C7_SEQUEN+C7_ITEMGRD`. `C7_QUANT`/`C7_PRECO`/`C7_TOTAL`, `C7_FORNECE+C7_LOJA`, `C7_RESIDUO` (PC com resíduo eliminado), `C7_ENCER` (encerrado).

### SF1 — Cabeçalho NF Entrada
Chave: `F1_FILIAL+F1_DOC+F1_SERIE+F1_FORNECE+F1_LOJA+F1_FORMUL`. `F1_CHVNFE` (chave NFe), `F1_TIPO`, `F1_STATUS`.

### SD1 — Itens NF Entrada
Chave: `D1_FILIAL+D1_DOC+D1_SERIE+D1_FORNECE+D1_LOJA+D1_ITEM+D1_FORMUL+D1_ITEMGRD`. `D1_PEDIDO` liga ao pedido de compra (SC7), `D1_TES`/`D1_CF` (regra fiscal), `D1_LOCAL` (armazém).

### SF2 — Cabeçalho NF Saída
Chave: `F2_FILIAL+F2_DOC+F2_SERIE+F2_CLIENTE+F2_LOJA`. `F2_CHVNFE`, `F2_TIPO` (Venda/Devolução), `F2_VALBRUT`/`F2_VALMERC`.

### SD2 — Itens NF Saída
Chave: `D2_FILIAL+D2_DOC+D2_SERIE+D2_CLIENTE+D2_LOJA+D2_ITEM`. `D2_PEDIDO` liga a SC6, `D2_TES`/`D2_CF`, `D2_LOCAL`.

### SE1 — Contas a Receber
Chave: `E1_FILIAL+E1_PREFIXO+E1_NUM+E1_PARCELA+E1_TIPO`. `E1_CLIENTE+E1_LOJA`, `E1_VALOR`/`E1_SALDO`, `E1_VENCTO`/`E1_VENCREA`, `E1_STATUS`.

### SE2 — Contas a Pagar
Chave: `E2_FILIAL+E2_PREFIXO+E2_NUM+E2_PARCELA+E2_TIPO+E2_FORNECE+E2_LOJA`. Campos espelham SE1.

### SE5 — Movimentação Bancária (baixa de título, sem chave única)
`E5_PREFIXO+E5_NUMERO+E5_PARCELA+E5_TIPO` liga a SE1/SE2. `E5_RECPAG` (R=recebimento/P=pagamento), `E5_VALOR`. **Nota:** em versões mais recentes do Protheus, SE5 pode estar fragmentada na família de tabelas `FKx` -- confirme no ambiente antes de assumir que todo o histórico de baixa está em SE5.

### CT1 — Plano de Contas
Chave: `CT1_FILIAL+CT1_CONTA`. `CT1_CLASSE` (só contas analíticas recebem lançamento), `CT1_BLOQ`.

### CT2 — Lançamentos Contábeis (fato, cada linha é um débito+crédito already pareado)
Chave composta longa incluindo `CT2_DATA+CT2_LOTE+CT2_SBLOTE+CT2_DOC+CT2_LINHA+...`. `CT2_DEBITO`/`CT2_CREDIT` (contas), `CT2_VALOR`, `CT2_HIST`. Rastreamento de origem via `CTK`/`CV3` (ligam o lançamento de volta à tabela/documento que o gerou).

### SF3 — Livros Fiscais (resumo por NF, sem chave única)
`F3_NFISCAL+F3_SERIE+F3_CLIEFOR+F3_LOJA`, `F3_VALICM`/`F3_BASEICM`, `F3_CFO`.

### SFT — Livro Fiscal por Item de NF / SPED (o mais granular para BI fiscal)
Chave: `FT_FILIAL+FT_TIPOMOV+FT_SERIE+FT_NFISCAL+FT_CLIEFOR+FT_LOJA+FT_ITEM+FT_PRODUTO`. Traz ICMS/IPI (base e valor) por item, `FT_TIPOMOV` (E=entrada/S=saída), `FT_CFOP`.

---

## Tabelas adicionais por módulo (confirmar schema real via SX3 antes de usar em produção)

As tabelas abaixo foram identificadas a partir da documentação de módulo do Protheus; use-as para orientar **qual tabela procurar**, mas valide campo a campo no dicionário do ambiente (`data-dictionary.md`) antes de codificar, pois esta lista não passou pela mesma verificação campo-a-campo do bloco acima.

| Módulo | Tabela | Papel | Observação |
|---|---|---|---|
| Compras | SC1 | Solicitação de compra (cabeçalho+item na mesma linha, chave `C1_NUM+C1_ITEM`) | Alimenta SC7/SC8 |
| Compras | SC8 | Itens de cotação de compra | Liga a SC1 via `C8_NUMSC+C8_ITEMSC` |
| Compras | SA5 | Produto x Fornecedor | Preço/prazo por fornecedor |
| Faturamento | SC9 | Itens liberados do pedido de venda | Um item de SC6 pode ter várias linhas SC9 (bloqueio parcial por crédito/estoque) -- não some SC6 direto sem entender isso |
| Faturamento | DA0 / DA1 | Tabela de preço (cabeçalho/item) | |
| PCP | SG1 | Estrutura de produto (BOM) | `G1_QUANT`, `G1_PERDA`; pai/componente |
| PCP | SC2 | Ordem de produção (cabeçalho único, sem item separado) | `C2_STATUS` (P/N/I/E), `C2_PEDIDO+C2_ITEMPED` liga produção a venda |
| PCP | SD4 | Empenho/reserva de componente por OP | Liga `D4_OP` a SC2; **confirme neste ponto se o ambiente usa SD4 também para contagem de inventário -- há divergência entre documentações e isso deve ser validado no SX2 real** |
| PCP | SG2 | Roteiro de operações | |
| Manutenção | ST9 | Cadastro de bem/ativo | `T9_SITMAN` (ativo/inativo) |
| Manutenção | STJ | Ordem de serviço (cabeçalho único, custo agregado) | `TJ_CODBEM` liga a ST9 |
| Manutenção | STL | Insumo realizado da O.S. (é o "item" real da O.S.) | Liga a STJ via `TL_ORDEM+TL_PLANO+TL_SEQRELA` |
| Manutenção | TQB | Solicitação de serviço | No máx. 1 O.S. por solicitação |
| Fiscal | SF4 | TES -- regra fiscal que decide se a operação atualiza estoque/financeiro/credita imposto | `D1_TES`/`D2_TES/C6_TES/C7_TES` apontam para cá; é a tabela de decisão mais importante do módulo fiscal |
| Contábil | CTT | Centro de custo | |
| Contábil | CTK / CV3 | Rastreamento origem→lançamento contábil | `CV3_TABORI+CV3_RECORI` aponta de volta para a tabela/documento de origem (SD1, SE2 etc.) |

Para qualquer tabela desta segunda lista, antes de gerar SQL de produção: rode a consulta de descoberta de `data-dictionary.md` (`SX3010 WHERE X3_ARQUIVO = '<alias>'`) para confirmar campos, tipos e nome físico reais.
