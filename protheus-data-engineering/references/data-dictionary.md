# Dicionário de Dados do Protheus (Tabelas SX)

O Protheus é um sistema **dictionary-driven**: nenhuma tabela ou campo existe sem um registro correspondente nas tabelas de metadado (prefixo `SX`). Para Engenharia de Dados isso é uma vantagem -- o dicionário pode ser consultado via SQL puro, exatamente como qualquer outra tabela, sem precisar de ADVPL.

## As tabelas do dicionário

| Tabela | Nome | O que define | Campos-chave para consulta |
|---|---|---|---|
| **SX2** | Tabelas | Registro mestre de cada alias (`SA1`, `SE1`, `ZA1`...): nome físico, descrição, modo de compartilhamento | `X2_CHAVE` (alias), `X2_ARQUIVO` (nome físico base), `X2_NOME` (descrição), `X2_MODO` (C/E/M) |
| **SX3** | Campos | Todo campo de toda tabela: tipo, tamanho, decimais, título, obrigatoriedade, F3 | `X3_ARQUIVO` (tabela), `X3_CAMPO`, `X3_TIPO` (C/N/D/L/M), `X3_TAMANHO`, `X3_DECIMAL`, `X3_TITULO`, `X3_DESCRIC`, `X3_OBRIGAT` |
| **SIX** | Índices | Toda ordem de índice de toda tabela -- a base real de qualquer decisão de performance | `INDICE` (tabela), `ORDEM`, `CHAVE` (expressão da chave, ex. `A1_FILIAL+A1_COD+A1_LOJA`), `NICKNAME` |
| **SX9** | Relacionamentos | O mais próximo de uma FK documentada -- liga um campo de uma tabela a outra | `X9_DOM` (tabela pai), `X9_CDOM` (campo pai), `X9_EXP` (expressão do lado filho), `X9_CONDSQL` |
| **SX5** | Tabelas genéricas | Pares código→descrição usados como domínio (ex. UF, condição de pagamento) | `X5_TABELA` (código da tabela genérica, 2 chars), `X5_CHAVE`, `X5_DESCRI` |
| **SX6** | Parâmetros (`MV_*`) | Configuração do ambiente -- moeda, formato, comportamento por módulo | `X6_VAR` (nome `MV_...`), `X6_CONTEUD`, `X6_DESCRIC` |
| **SX7** | Gatilhos | Preenchimento automático de campo (ex. nome do cliente ao digitar o código) -- raramente relevante para BI, útil só para explicar por que um campo denormalizado existe | `X7_CAMPO`, `X7_CDOMIN`, `X7_ALIAS` |
| **SXB** | Consultas padrão (F3) | Define o lookup de tela; não tem tradução direta em SQL de BI | -- |
| **SX1** | Perguntas de relatório | Parâmetros de tela de relatório -- não é dado de negócio | -- |

## Consultando o dicionário via SQL

As tabelas SX também são tabelas físicas normais no banco (com o mesmo sufixo de empresa, ex. `SX3010`). Isso significa que você pode **descobrir a estrutura real do ambiente do cliente com SQL puro**, sem precisar de ADVPL -- essencial quando o ambiente tem customização:

```sql
-- Todos os campos de uma tabela (ex.: SA1), como o dicionário do CLIENTE realmente define
SELECT X3_CAMPO, X3_TIPO, X3_TAMANHO, X3_DECIMAL, X3_TITULO, X3_DESCRIC, X3_OBRIGAT
FROM SX3010
WHERE D_E_L_E_T_ = ' '
  AND X3_ARQUIVO = 'SA1'
ORDER BY X3_ORDEM

-- Todos os índices de uma tabela (para alinhar WHERE/ORDER BY)
SELECT ORDEM, CHAVE, NICKNAME
FROM SIX010
WHERE D_E_L_E_T_ = ' '
  AND INDICE = 'SA1'
ORDER BY ORDEM

-- Nome físico e modo de compartilhamento de um alias
SELECT X2_CHAVE, X2_ARQUIVO, X2_NOME, X2_MODO
FROM SX2010
WHERE D_E_L_E_T_ = ' '
  AND X2_CHAVE = 'SA1'

-- Relacionamentos documentados a partir de uma tabela
SELECT X9_DOM, X9_CDOM, X9_EXP, X9_CONDSQL
FROM SX9010
WHERE D_E_L_E_T_ = ' '
  AND X9_DOM = 'SA1'

-- Descobrir todos os campos customizados de uma tabela (prefixo diferente do padrão)
SELECT X3_ARQUIVO, X3_CAMPO, X3_TITULO
FROM SX3010
WHERE D_E_L_E_T_ = ' '
  AND X3_ARQUIVO = 'SA1'
  AND X3_CAMPO NOT LIKE 'A1\_%' ESCAPE '\'
```

> O sufixo físico (`SX3010`, `SX2010`...) varia por ambiente/empresa -- confirme o nome real consultando `SX2010` primeiro (`X2_CHAVE = 'SX3'`) ou peça ao usuário/DBA. Trate o `010` como exemplo, não como constante universal.

**Regra prática:** sempre que o usuário mencionar um campo ou tabela que você não reconhece, ou quando o comportamento dos dados não bater com o padrão documentado, rode a consulta de descoberta acima antes de assumir que o campo não existe ou que é erro do usuário -- customização é a explicação mais comum.

## Compartilhamento e filial

`X2_MODO` (em SX2) define como a tabela se comporta entre empresas/filiais:

| X2_MODO | Significado | Exemplos típicos | Implicação para SQL |
|---|---|---|---|
| `E` | Exclusivo por filial | SA1, SE1, SF2, SC5 (a maioria das tabelas transacionais) | Cada filial só enxerga seus próprios registros; se o requisito é "todas as filiais", **não filtre por filial** -- apenas agrupe/identifique por ela |
| `C` | Compartilhado entre todas as empresas/filiais | Tabelas de parâmetro e domínio globais | Filtro de filial não se aplica ou é irrelevante |
| `M` | Misto (depende de configuração adicional) | Casos multi-tenant específicos | Confirmar o comportamento real no SX2 do ambiente antes de assumir |

**Regra de ouro herdada do dicionário:** o campo de filial é sempre o primeiro componente de todo índice (`SIX`) de tabela exclusiva -- por isso toda consulta bem otimizada também começa o WHERE pelo campo de filial, mesmo quando a intenção é trazer todas as filiais (nesse caso, o filtro de filial é omitido, mas a ordenação/particionamento lógico dos dados continua sendo por filial primeiro).

**Não confunda "identificar o modo de compartilhamento" com "sempre filtrar uma filial só".** A regra correta, conforme o cenário de BI: *identifique a estratégia de filial e compartilhamento da tabela e aplique o filtro de filial conforme o escopo funcional pedido* -- que pode ser uma filial, um subconjunto, ou nenhuma (consolidado).

## Como as tabelas do dicionário se conectam

```
SX2 (Tabela) ──┬──> SX3 (Campos desta tabela)
               │        ├──> X3_F3 ──> SXB (lookup F3)
               │        ├──> X3_TRIGGER ──> SX7 (gatilho)
               │        └──> X3_CBOX ──> SX5 (domínio de combo)
               ├──> SIX (Índices desta tabela)
               └──> SX9 (Relacionamentos com outras tabelas)

SX6 (Parâmetros MV_*) ──> configuração geral do ambiente
```

Ao investigar uma tabela desconhecida, a sequência é: **SX2** (existe? nome físico? modo?) → **SX3** (quais campos, tipos, tamanhos?) → **SIX** (quais índices, para alinhar WHERE) → **SX9** (relacionamento documentado com outras tabelas, se existir).
