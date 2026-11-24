# Adoção de agentes de IA por desenvolvedores — Stack Overflow Developer Survey 2025

Trabalho final de Ciência de Dados: pipeline completo (AED, modelagem e avaliação) sobre a
[Stack Overflow Developer Survey 2025](https://survey.stackoverflow.co/2025), para prever quais
desenvolvedores profissionais usam agentes de IA no trabalho.

## Estrutura

- [`eda_pipeline_fase0_12.ipynb`](eda_pipeline_fase0_12.ipynb) — **notebook atual (Fases 0 a 12)**, já executado:
  AED revisada (Bloco A, Fases 0 a 5); formulação, baseline e particionamento sem vazamento (Fases 6 e 7);
  preparação por família, ausentes e desbalanceamento (Fases 8 a 10); revisão do conjunto de preditores;
  busca de hiperparâmetros da árvore e do CatBoost (Fase 11); comparação estatística e avaliação única
  no teste (Fase 12); Quadro de Decisões e Quadro de Descarte.
- [`RELATORIO_EXECUTIVO.md`](RELATORIO_EXECUTIVO.md) — relatório executivo, sem código, com o Apêndice B
  (declaração de uso de IA generativa).
- [`DECISOES.md`](DECISOES.md) — registro resumido das decisões tomadas e do que foi feito.
- [`FICHA_DE_PROPOSTA.md`](FICHA_DE_PROPOSTA.md) — ficha de proposta do projeto (tarefa, alvo, métrica,
  modelos e riscos).
- Versões anteriores do notebook (Fases 0 a 2, 0 a 4, 0 a 5 e 0 a 7) ficam no histórico do git.
- [`eda_stackoverflow_survey.ipynb`](eda_stackoverflow_survey.ipynb) — análise descritiva inicial
  (tipagem de variáveis, nulos, estatísticas descritivas, histogramas, boxplots, outliers, teste de
  normalidade e gráficos de barras para as variáveis categóricas).
- [`survey_results_schema.csv`](survey_results_schema.csv) — dicionário oficial das perguntas da pesquisa.
- `survey_results_public.csv` — dados brutos (**não incluído no repositório**, ver abaixo).

## Dados

O arquivo `survey_results_public.csv` (~134 MB) não está versionado porque excede o limite de 100 MB do
GitHub. Para reproduzir a análise:

1. Baixe os dados da edição 2025 em <https://survey.stackoverflow.co/> e coloque o arquivo
   `survey_results_public.csv` na raiz do repositório, ao lado do notebook.
2. As colunas usadas e o dicionário de dados estão documentados na primeira seção do notebook.

## Recorte e critérios usados

- **Linhas:** filtrado para `MainBranch == "I am a developer by profession"` (só desenvolvedores
  profissionais), removendo estudantes/hobbistas/ex-desenvolvedores que distorceriam as perguntas de
  carreira e remuneração.
- **Colunas:** de 172 colunas originais, foram selecionadas 20 — perguntas de resposta única (não
  perguntas de múltipla escolha/ranking, que exigiriam parsing de listas). O critério completo está
  explicado na primeira célula do notebook.

## Como rodar

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt
jupyter notebook eda_pipeline_fase0_12.ipynb
```

A execução completa leva cerca de 35 minutos (a maior parte é a busca de hiperparâmetros do CatBoost).
