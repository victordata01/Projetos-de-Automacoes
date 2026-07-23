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

## Estrutura do projeto

O pipeline é organizado em módulos, orquestrados pelo `main.py`:

- **`main.py`** — orquestrador do pipeline: carrega o `.env`, roda as etapas de queries, análise e
  envio de e-mail em sequência, com log e alerta automático em caso de falha em qualquer etapa.
- **`queries.py`** — roda as queries (escritas por mim) que puxam os dados do Athena.
- **`athena_query_v2`** — ferramenta interna criada pela empresa para rodar consultas no Athena com
  mais facilidade, sem precisar solicitar acessos adicionais. Como foi desenvolvida pela empresa, o
  código dela não pode ser incluído aqui.
- **`analyzer.py`** — analisa os DataFrames retornados e gera o texto de análise e os gráficos.
- **`mailer.py`** — monta o corpo do e-mail (com apoio da IA) e configura o envio.
- **`preview.html`** — gerado pelo `mailer.py`, permite visualizar o e-mail em HTML antes do envio.
- **`.env`** — armazena os logins e permissões usados pelo pipeline (não versionado no repositório).

## Código: `main.py`

```python
"""
main.py — Orquestrador do pipeline de relatório de cobrança.

Fluxo:
1. Carrega variáveis de ambiente do .env
2. Executa queries no Athena     (queries.py)
3. Analisa os DataFrames         (analyzer.py)
4. Envia o email de relatório    (mailer.py)

Pode ser agendado via:
  - Linux/macOS: crontab  (ex: 0 7 * * * /usr/bin/python3 /caminho/main.py)
  - Windows:     Task Scheduler  apontando para python.exe main.py
"""

import logging
import sys
from datetime import datetime

from dotenv import load_dotenv

# Carrega o .env antes de qualquer import dos módulos do projeto,
# pois eles leem os valores via os.getenv() no momento da importação.
load_dotenv()

from analyzer import analyze
from mailer import send_alert, send_report            # noqa: E402
from queries import run_queries                       # noqa: E402

# --- Configuração de logging: saída no console + arquivo diário
LOG_FORMAT = "%(asctime)s [%(levelname)s] %(name)s - %(message)s"
logging.basicConfig(
    level   = logging.INFO,
    format  = LOG_FORMAT,
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler(
            f"pipeline_{datetime.now().strftime('%Y%m%d')}.log",
            encoding="utf-8",
        ),
    ],
)
logger = logging.getLogger("main")


def main() -> None:
    logger.info("=" * 60)
    logger.info("Iniciando pipeline de relatório de cobrança")
    logger.info("=" * 60)

    # ---------------------------------------------------------
    # ETAPA 1 - Queries no Athena
    # ---------------------------------------------------------
    try:
        logger.info("[1/3] Executando queries no Athena...")
        dataframes = run_queries()
    except Exception as exc:
        logger.exception("Falha na etapa de queries.")
        send_alert(
            subject = "Pipeline falhou na Etapa 1 (Queries Athena)",
            body = (
                f"Erro ao executar queries no Athena.\n\n"
                f"Detalhes: {exc}\n\n"
                f"Verifique os logs para mais informações."
            ),
        )
        sys.exit(1)

    logger.info("Tabelas retornadas: %s", list(dataframes.keys()))

    # ---------------------------------------------------------
    # ETAPA 2 - Análise dos DataFrames
    # ---------------------------------------------------------
    try:
        logger.info("[2/3] Analisando DataFrames...")
        analysis_text, charts = analyze(dataframes)
    except Exception as exc:
        logger.exception("Falha na etapa de análise.")
        send_alert(
            subject = "Pipeline falhou na Etapa 2 (Análise)",
            body = f"Erro ao analisar DataFrames.\n\nDetalhes: {exc}",
        )
        sys.exit(1)

    logger.info("Relatorio gerado: %d caracteres, %d gráficos", len(analysis_text), len(charts))

    # ---------------------------------------------------------
    # ETAPA 3 - Envio do email
    # ---------------------------------------------------------
    try:
        logger.info("[3/3] Enviando email de relatório...")
        send_report(analysis_text, dataframes, charts)
    except Exception as exc:
        logger.exception("Falha na etapa de envio de email.")
        send_alert(
            subject = "Pipeline falhou na Etapa 3 (Envio de Email)",
            body = (
                f"As queries e análise foram concluídas, mas o email "
                f"não foi enviado.\n\nDetalhes: {exc}"
            ),
        )
        sys.exit(1)

    logger.info("=" * 60)
    logger.info("Pipeline concluído com sucesso.")
    logger.info("=" * 60)


if __name__ == "__main__":
    main()
```
