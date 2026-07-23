# Automação de campanhas de cobrança (SQL + Braze + AWS)

**Ferramentas:** SQL Server, AWS Athena, Braze, Power BI

## Contexto

A comunicação de cobrança precisava rodar de forma recorrente para diferentes réguas (pré-vencimento,
vencido, negociação etc.), cada uma com sua própria segmentação de clientes, canal de envio e regra de
disparo. Sem automação, isso exigiria extração e atualização manual da base a cada campanha, o que é
lento e propenso a erro — e sem visibilidade de ponta a ponta, um problema no meio do fluxo só seria
percebido tarde, com o cliente já impactado.

## Como funciona o processo

**1. Extração da base (SQL + AWS Athena)**
Escrevo a query SQL com base na regra de cada régua de cobrança, consolidando os dados direto no
AWS Athena e já trazendo a base segmentada com os clientes elegíveis para aquela campanha.

**2. Fluxo de disparo (Braze)**
A base gerada alimenta um fluxo criado no Braze, que:
- roda a query e traz os clientes elegíveis;
- separa o cliente para a comunicação correta, conforme a regra de negócio definida (ex: qual régua/etapa
  da jornada ele está);
- separa por canal de comunicação (SMS, e-mail, push etc.), conforme definição de cada campanha;
- envia a comunicação para o cliente de forma automática, sem necessidade de disparo manual.

**3. Relatório de acompanhamento (Power BI + AWS Athena)**
Construí um relatório em Power BI, puxando os dados direto do AWS Athena, para acompanhar todas as
campanhas já automatizadas. Ele mostra:
- **% de envio** — com um alerta automático por e-mail quando esse percentual fica abaixo do esperado,
  para eu verificar o que aconteceu;
- **Quantidade de comunicações por canal**;
- **Funil de comunicação** — quantidade de envios, entregues, visualizações e cliques;
- **Quantidade de comunicações por dia**.

## Caso prático: identificação de uma anomalia

Em um dos dias, houve um erro na configuração feita pelo marketing que gerou um pico fora do padrão no
volume de comunicações disparadas. Graças à visão diária do relatório (quantidade de comunicações por
dia + % de envio), foi possível identificar rapidamente que algo estava fora do esperado, investigar a
causa junto ao time de marketing e corrigir a configuração antes que o problema se agravasse.

-- AQUI VAI UM PRINT DO CASO FALADO ACIMA -- 


## Resultado

- Aumento no percentual de envio e no percentual de recebimento/pagamento dos clientes;
- Eliminação do disparo manual das campanhas de cobrança;
- Identificação e correção rápida de problemas no processo, graças ao acompanhamento diário do relatório.

## Exemplo de estrutura da consulta (ilustrativo)

> Nota: baseada na query real utilizada no processo, com nomes de tabelas/colunas e regras de negócio específicas da empresa substituídos por versões genéricas — a estrutura (CTEs, joins, filtros e ranking) reflete fielmente o que foi construído.


```sql
-- Nota: nomes de tabelas, colunas e regras de negócio específicas da empresa foram
-- substituídos por versões genéricas. Onde a regra em si é proprietária, o trecho
-- foi marcado como "regra de negócio da empresa" no lugar da lógica original.

WITH cte_codigo_cliente AS (
    SELECT
        id,
        created_at AS data_adesao,
        RANK() OVER (ORDER BY created_at) AS cod_cliente
    FROM tabela_clientes
),

cte_parcelas AS (
    SELECT
        t.cliente_id,
        cc.cod_cliente,
        -- regra de negócio da empresa (classificação da parcela/etapa da jornada)
        t.cpf,
        t.id_transacao,
        t.valor_compra,
        t.nome_loja,
        t.data_vencimento,
        t.due_date,
        t.nome_cliente,
        t.email,
        t.telefone,
        t.valor_parcela,
        t.data_compra,
        t.data_pagamento,
        -- regra de negócio da empresa (cálculo de dias em atraso)
        t.dias_atraso,
        -- regra de negócio da empresa (elegibilidade para comunicação)
        t.fl_comunicacao
    FROM tabela_transacoes t
    INNER JOIN tabela_lojas       ON tabela_lojas.id = t.loja_id
    INNER JOIN tabela_clientes    ON tabela_clientes.id = t.cliente_id
    INNER JOIN tabela_pagamentos  ON tabela_pagamentos.transacao_id = t.id_transacao
    INNER JOIN cte_codigo_cliente cc ON cc.id = t.cliente_id
    WHERE status = 2
      AND data_pagamento IS NULL
),

cte_canal AS (
    SELECT DISTINCT
        cpf,
        canal_id,
        id_conta,
        sistema
    FROM tabela_canais_comunicacao
    INNER JOIN tabela_clientes ON tabela_clientes.cpf = tabela_canais_comunicacao.cpf
    INNER JOIN tabela_contas   ON tabela_contas.id = tabela_canais_comunicacao.id_conta
)

SELECT
    parcelas.*,
    -- regra de negócio da empresa (definição do grupo da régua)
    NULL AS grupo,
    -- regra de negócio da empresa (definição do tipo de comunicação/canal)
    NULL AS fl_tipo_comunicacao
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY cliente_id ORDER BY data_compra) AS ordem
    FROM cte_parcelas
    WHERE dias_atraso BETWEEN -5 AND 87
      AND fl_comunicacao = 1
) AS parcelas
LEFT JOIN cte_canal cp ON cp.cpf = parcelas.cpf
WHERE parcelas.ordem = 1
ORDER BY parcelas.cliente_id, parcelas.data_compra 
```
