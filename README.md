# Análise Exploratória — Stack Overflow Developer Survey

Trabalho de análise exploratória de dados (EDA) sobre a Stack Overflow Developer Survey
(edição com módulo de IA/AI Agents), com foco em perfil profissional, remuneração/satisfação
e adoção/percepção de IA entre desenvolvedores.

## Estrutura

- [`eda_pipeline_fase0_7.ipynb`](eda_pipeline_fase0_7.ipynb) — **notebook atual (Entrega 1)**, já executado:
  revisão da AED (Bloco A, Fases 0 a 5) + formulação do problema, baseline e particionamento sem
  vazamento (Bloco B, Fases 6 e 7).
- [`FICHA_DE_PROPOSTA.md`](FICHA_DE_PROPOSTA.md) — ficha de proposta do projeto (tarefa, alvo, métrica,
  modelos e riscos).
- [`eda_pipeline_fase0_5.ipynb`](eda_pipeline_fase0_5.ipynb) — versão anterior do pipeline (Fases 0 a 5),
  mantida como histórico.
- [`eda_stackoverflow_survey.ipynb`](eda_stackoverflow_survey.ipynb) — análise descritiva inicial
  (tipagem de variáveis, nulos, estatísticas descritivas, histogramas, boxplots, outliers, teste de
  normalidade e gráficos de barras para as variáveis categóricas).
- [`survey_results_schema.csv`](survey_results_schema.csv) — dicionário oficial das perguntas da pesquisa.
- `survey_results_public.csv` — dados brutos (**não incluído no repositório**, ver abaixo).

## Dados

O arquivo `survey_results_public.csv` (~134 MB) não está versionado porque excede o limite de 100 MB do
GitHub. Para reproduzir a análise:

1. Obtenha o arquivo `survey_results_public.csv` da mesma fonte usada no trabalho (Stack Overflow
   Developer Survey) e coloque-o na raiz do repositório, ao lado do notebook.
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
jupyter notebook eda_pipeline_fase0_7.ipynb
```
