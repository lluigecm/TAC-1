# Resumo da Base de Dados

## 1. Descrição da base de dados

A base utilizada é a **Stack Overflow Developer Survey**, na edição que inclui um módulo específico
sobre **IA e AI Agents**. É uma pesquisa anual aplicada pela Stack Overflow com desenvolvedores de
software ao redor do mundo, cobrindo perfil demográfico e profissional, tecnologias utilizadas,
remuneração, satisfação no trabalho e, nesta edição, percepção e adoção de ferramentas/agentes de IA
no processo de desenvolvimento.

A base é composta por dois arquivos:

- **`survey_results_public.csv`** — as respostas individuais da pesquisa (uma linha por respondente).
- **`survey_results_schema.csv`** — dicionário oficial das perguntas: nome da coluna, texto completo
  da pergunta e, quando aplicável, o texto de cada sub-item (usado para entender o significado de cada
  coluna antes de selecioná-las).

É uma pesquisa **opcional e paginada** — o respondente pode pular perguntas ou parar antes do fim —,
por isso boa parte das colunas tem valores faltantes gravados como a string literal `"NA"`, e a taxa de
preenchimento cai conforme a pergunta aparece mais tarde no formulário.

## 2. Quantidade de colunas e linhas

| | Linhas | Colunas |
|---|---|---|
| Base original | 49.191 | 172 |
| Após o recorte usado no trabalho | 37.467 | 20 (+ `ResponseId`) |

## 3. Dataset foi usado como um todo ou delimitado?

Foi **delimitado**, em duas dimensões — linhas e colunas.

### Linhas (filtro)

Mantivemos apenas respondentes com `MainBranch == "I am a developer by profession"`
(37.467 de 49.191 respostas, ~76% da base).

- **Por quê:** a pesquisa inclui estudantes, hobbistas e ex-desenvolvedores, cujas respostas distorcem
  perguntas de carreira, mercado de trabalho e remuneração. O foco do trabalho é o perfil do
  desenvolvedor profissional.
- **Como:** filtro direto pelo valor da coluna `MainBranch`.

### Colunas (seleção)

De 172 colunas originais, foram selecionadas **20**.

- **Por quê:** a maior parte das colunas é de perguntas de múltipla escolha ("selecione todas que se
  aplicam", ex.: linguagens, frameworks, plataformas) ou de rankings compostos (`TechEndorse_*`,
  `JobSatPoints_*`), com valores separados por `;` — exigem um parsing próprio (explosão de listas) fora
  do escopo desta primeira análise exploratória.
- **Como:** priorizamos colunas de **resposta única** (numéricas ou categóricas simples), formando um
  eixo temático coerente — perfil profissional, remuneração/satisfação e adoção/percepção de IA.
  Resultado:
  - **4 contínuas:** `WorkExp`, `YearsCode`, `JobSat`, `ConvertedCompYearly`
  - **16 categóricas:** `Age`, `EdLevel`, `Employment`, `DevType`, `OrgSize`, `ICorPM`, `RemoteWork`,
    `Country`, `Industry`, `AIThreat`, `AISelect`, `AISent`, `AIAcc`, `AIComplex`, `AIAgents`,
    `SOVisitFreq`

O detalhamento completo (descrição de cada coluna e o tipo escolhido) está na primeira seção do notebook
[`eda_stackoverflow_survey.ipynb`](eda_stackoverflow_survey.ipynb).
