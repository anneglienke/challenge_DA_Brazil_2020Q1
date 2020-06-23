# Dicionário de Dados

Este documento contém as especificações de colunas e/ou métricas criadas durante o processo de resolução dos problemas, e qualquer outro conceito assumido para propósitos de análise.

## Tabela sellers

| Coluna  | Descrição                                                         |
|---------|-------------------------------------------------------------------|
| Lojista | Atribuição sequencial aos IDs para melhorar a estética dos dashes.|

## Tabela payments

| Coluna  | Descrição                                                         |
|---------|-------------------------------------------------------------------|
| correct_payment_values | Coluna criada em M que divide o valor original por 100, de forma que correspondesse ao formato adequado para a visualização. |

## Tabela métricas_criadas

Esta tabela contém as métricas criadas em DAX para analisar os dados e montar a visualização.

| Métrica                           | Descrição                                                                                                                                                                              |
|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| pagamentos                        | Soma os valores de pagamentos                                                                                                                                                          |
| pagamentos_ano_anterior           | Calcula o valor de pagamentos no mesmo período do ano anterior                                                                                                                         |
| filtro_limpeza_tabela             | Remove de visualização todos os itens a que não há valores atribuídos no ano selecionado e no ano anterior                                                                             |
| pedidos_por_vendedor              | Calcula a quantidade de pedidos por vendedor                                                                                                                                           |
| perc_crescimento                  | Calcula o percentual de crescimento do período atual em relação ao mesmo período do ano anterior. Nos casos em que ambos os períodos tiverem valor igual a zero, atribui o valor zero. |
| rank_pagamentos                   | Ranqueia os sellers por pagamentos |
| vlr_pagamentos_acumulado_por_rank | Acumula os valores de pagamentos por lojistas na ordem do rank_pagamentos |
| perc_pagamentosxpagamentos_acum   | Acumula o valor da métrica perc_pagamentosxtotal conforme ordem definida pelo ranking. |
| perc_pagamentosxtotal             | Calcula o percentual de pagamentos de um lojista em relação ao valor de pagamentos total (de todos os lojistas), para saber a quanto ele corresponde do todo. |
| rank_abc_pagamentos               | determina o posicionamento do cliente na curva ABC. Em caso de um único seller corresponder a mais de 50% da receita total, este seller recebe valor A. |

### Sintaxe de criação das métricas

A seguir será apresentado o código DAX para criação de cada uma das métricas listadas na tabela acima.

**pagamentos**
```
SUM(payments[correct_payment_values])
```

**pagamentos_ano_anterior**
```
CALCULATE([pagamentos]; SAMEPERIODLASTYEAR(orders[order_approved_at].[Date]))
```

**filtro_limpeza_tabela**
```
IF(
  [pagamentos]=0 && [pagamentos_ano_anterior] = 0;
  "ESCONDER";
  "MOSTRAR"
)
```

**pedidos_por_vendedor**
```
DISTINCTCOUNT(order_items[order_id])/DISTINCTCOUNT(sellers[Lojistas])
```

**perc_crescimento**
```
VAR PERC = IFERROR(([pagamentos]/[pagamentos_ano_anterior])-1; 0)

RETURN IF(
  [pagamentos_ano_anterior]=0 && [pagamentos]=0;
  0;
  PERC
)
```

**rank_pagamentos**
```
RANKX(ALL(sellers[Lojistas]);[pagamentos];DESC)
```

**vlr_pagamentos_acumulado_por_rank**
```
(SUMX(TOPN([rank_pagamentos];ALL(sellers[Lojistas]);[pagamentos]);[pagamentos]))
```

**perc_pagamentosxpagamentos_acum**
```
VAR VLR = [vlr_pagamentos_acumulado_por_rank]/CALCULATE([pagamentos];ALL(sellers[Lojistas]))

RETURN IF(
  HASONEFILTER(sellers[Lojistas]);
  VLR;
  1
)
```

**perc_pagamentosxtotal**
```
[pagamentos]/CALCULATE([pagamentos];ALLSELECTED(sellers[Lojistas]))
```

**rank_abc_pagamentos**
```
VAR PERC = [perc_pagamentosxpagamentos_acum]*100 

VAR PTS = IF( 
  [rank_pagamentos]=1;
  "A";
  IF(
    PERC<50;
    "A";    
    IF(
      AND(PERC<80;PERC>=50);   
      "B";
      "C"
    )
  )
)

VAR CALC = CALCULATE(PTS;FILTER(orders;SELECTEDVALUE(orders[order_approved_at].[Date])))

RETURN IF(
  HASONEFILTER(sellers[Lojistas]);
  CALC;
  ""
)
```
