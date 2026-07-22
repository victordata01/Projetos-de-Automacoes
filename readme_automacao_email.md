# Automação de relatório semanal de metas (Python + IA)

**Ferramentas:** Python, Pandas, AWS Athena, Claude (IA)

## Contexto

Toda quarta-feira, uma pessoa da equipe montava manualmente o relatório semanal de acompanhamento de
metas: olhava todos os relatórios, analisava os dados e escrevia o e-mail para os funcionários da
cobrança e gestores. Esse processo era totalmente manual e consumia basicamente a terça-feira inteira
(preparação) e a quarta-feira inteira (finalização e envio).

## Como funciona o processo

**1. Fonte de dados**
As metas ficam em uma planilha Excel, alimentada semestralmente pelo time. Os demais dados de
acompanhamento vêm do AWS Athena.

**2. Consolidação e disparo (Python)**
Um script em Python:
- puxa os dados direto do AWS Athena;
- monta as visões/consolidados que antes eram feitos manualmente;
- dispara o e-mail automaticamente para todos os funcionários da cobrança e gestores, sem
  intervenção manual.

**3. Geração do resumo (Claude/IA)**
Utilizei o Claude para gerar o resumo em texto do e-mail, com base em um modelo de e-mail manual que
eu já escrevia antes — usei esse modelo como referência para a IA analisar os dados semanais e
produzir um resumo no mesmo padrão e estilo.

## Resultado

- Eliminação de um processo que consumia terça e quarta-feira inteiras;
- Envio automático e recorrente do relatório, sem depender de alguém montar tudo manualmente toda semana;
- Padronização do resumo enviado, mantendo o mesmo estilo do e-mail que era feito manualmente.

## Exemplo de estrutura do script (ilustrativo)

> Nota: script simplificado para fins de portfólio — substitua pelos dados/fontes reais (sem
> informações sensíveis da empresa) antes de publicar.

```python
import pandas as pd

def carregar_dados_athena(query):
    # conexão com AWS Athena e execução da query
    return pd.read_sql(query, conexao_athena)

def carregar_metas_excel(caminho_arquivo):
    return pd.read_excel(caminho_arquivo)

def gerar_resumo_semanal(df_dados, df_metas):
    resumo = df_dados.merge(df_metas, on="area")
    resumo["percentual_atingido"] = resumo["realizado"] / resumo["meta"] * 100
    return resumo

def gerar_texto_com_ia(resumo):
    # chamada à API do Claude, usando o modelo de e-mail manual como referência de estilo
    return gerar_email_com_claude(resumo)

if __name__ == "__main__":
    dados = carregar_dados_athena("SELECT * FROM cobranca_semanal")
    metas = carregar_metas_excel("metas.xlsx")
    resumo = gerar_resumo_semanal(dados, metas)
    texto_email = gerar_texto_com_ia(resumo)
    # enviar_email(texto_email, destinatarios)
```
