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

> Nota: query simplificada e com nomes fictícios — não reflete a estrutura real de produção da empresa por motivos juridicos.


```sql
SELECT
    cliente_id,
    etapa_jornada,
    dias_atraso,
    canal_preferencial,
    cod_unidade,
    status,
    dt_vencimento,
    saldo_devedor,
    data_saldo
FROM base_cobranca
WHERE status = 'ativo'
  AND etapa_jornada IN ('pre_vencimento', 'vencido')
  AND (REGRAS ALINHADAS COM OS GESTORES PORÉM NÃO POSSO COLOCAR AQUI POR MOTIVOS JURIDICOS)
```
