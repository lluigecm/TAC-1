# Registro de Decisões e Atividades — Entrega até a Fase 12

Resumo das decisões tomadas e do que foi feito no notebook [`eda_pipeline_fase0_12.ipynb`](eda_pipeline_fase0_12.ipynb) (antes `eda_pipeline_fase0_7.ipynb` e `eda_pipeline_fase0_11.ipynb`).
Referência das fases: *Trabalho Final – Pipeline de Ciência de Dados - Detalhe.pdf* (Profa. Trícia, 05/11/2026).

---

## Decisões anteriores (Entrega 1 e ficha de proposta)

| Fase | Decisão | Alternativas consideradas | Evidência / justificativa |
|---|---|---|---|
| 6 | Alvo `AIAgents_bin` (1 = usa agentes diária, semanal ou mensalmente) | `AISelect` (uso de qualquer ferramenta de IA) | `AISelect` é proxy quase direta do alvo; foi descartada como preditor (vazamento) |
| 6 | Métrica principal: F1 da classe positiva | Acurácia, PR-AUC, balanced accuracy | 31,9% de positivos; o majoritário acerta 68,1% com F1 = 0; falso negativo custa mais |
| 7 | Holdout 80/20 estratificado (`random_state=42`) + `StratifiedKFold(5)` no treino | `GroupKFold`, `TimeSeriesSplit` | Uma linha por respondente, corte transversal, sem ordem temporal |
| 11 (prévia) | Segundo modelo: **CatBoost** (antes: Random Forest ou Gradient Boosting) | Random Forest, HistGradientBoosting, LightGBM | Pedido da professora; 7 dos 9 preditores são categóricos e o CatBoost os trata nativamente, com codificação ordenada pelo alvo, sem vazamento |

---

## Decisões da entrega da semana 2 (Seção 3 e Fases 8 a 10)

Todas as comparações usam as mesmas 5 dobras (`cv_esquema`), a métrica F1 da classe 1 (média ± dp) e o **teste t corrigido de Nadeau-Bengio**. Escolhemos esse teste porque, com 5 dobras, o Wilcoxon nunca atinge p < 0,05: o menor p bicaudal possível é 0,0625. Os hiperparâmetros estão fixos (árvore com `max_depth=6` e `min_samples_leaf=50`; CatBoost com 300 iterações, `lr=0.1` e `depth=6`), porque a busca é da Fase 11.

| Fase | Decisão | Alternativas testadas | Evidência | Escolha |
|---|---|---|---|---|
| Seção 3 | Refazer os filtros das Fases 3 e 4 só no treino | Seleção da Etapa 1 × só treino | As 9 variáveis continuam significativas; o VIF remove as mesmas 3 colunas (`RemoteWork_missing`, `OrgSize_missing`, `WorkExp_z`) | Nenhuma variável mudou de status |
| 8 | Escala na árvore | Com × sem z-score | F1 idêntico nas 5 dobras | **Sem z-score** |
| 8 | Outliers na árvore | Manter × retirar o corte 3×IQR | O corte só pega 6 valores de `YearsCode` (75, 82 e 100 anos), implausíveis | **Mantido** (qualidade do dado) |
| 8 | Categóricas no CatBoost | Dummies/TE × nativas | 7 dos 9 preditores são categóricos; a codificação ordenada evita vazamento | **Nativas** |
| 8 | Categorias raras e novas | Agrupar × não agrupar | A suavização do TE e o *prior* do CatBoost já tratam os níveis raros; categoria nova não quebra os pipelines | **Sem agrupamento** |
| 8 | Novos atributos | Criar × não criar | As curvas LOESS mostram pontos de corte que árvores aprendem sozinhas | **Nenhum** |
| 9 | Mecanismo de ausência | — | Em `YearsCode`, `SOVisitFreq` e `AIAcc`, a ausência se liga ao alvo (p < 0,001), então não é MCAR. O bloco OrgSize/ICorPM/RemoteWork falta junto (89%), não se liga ao alvo, mas depende de variáveis observadas (MAR) | Descartar linhas está fora de questão |
| 9 | Imputação na árvore | A: isBAD/'missing'; B: mediana/moda; C: KNN | F1 0,543 / 0,547 / 0,543; p = 0,46 (A×B) e 0,83 (A×C) | **A**: empate estatístico; o enunciado pede manter as isBAD; a ausência é informativa; C é ~100× mais lenta |
| 9 | Imputação no CatBoost | Nativa × mediana/moda | F1 0,553 × 0,553; p = 0,90 | **Nativa** (mais simples) |
| 9 | Dados sintéticos | Gerar × não gerar | O problema é valor faltando, a classe não é rara (31,9%) e a base é grande | **Não gerar** |
| 10 | Desbalanceamento | Nível 0, 1 (peso), 2 (limiar), 1+2 | O nível 0 é ruim (F1 0,19 na árvore, 0,35 no CatBoost); todos os tratamentos ganham do nível 0 com p < 0,001. Nível 2 × 1: +0,011 na árvore (p = 0,26), +0,009 no CatBoost (p = 0,04). Nível 1+2 × 2: sem ganho (p = 0,96 e 0,09). A PR-AUC não muda | **Nível 2, sem peso de classe**: uma intervenção só, maior recall (0,88 e 0,81), não distorce as probabilidades |
| 10 | SMOTE / reamostragem | Não testado | As alternativas que não criam dados bastaram | **Não aplicar** |

## Decisões da Fase 11 (parcial)

Escopo combinado com a equipe: busca completa para a **árvore** (grade, gap treino–validação, curva de `max_depth`); para o **CatBoost**, só as curvas de validação e o gap, que definem a faixa da busca da próxima rodada.

| Fase | Decisão | Alternativas testadas | Evidência | Escolha |
|---|---|---|---|---|
| 11 | Método de busca | (a) PR-AUC e limiar no fim; (b) F1 com limiar por dentro | (b) custa ~4× por candidato (~80 s no CatBoost, ~1 s na árvore) | **(b) na árvore**; no CatBoost, **(a) na exploração ampla e (b) na grade final** |
| 11 | Paralelismo | Limitar núcleos × usar todos | Todos os 16 núcleos ficam a 100% por alguns minutos; não danifica o hardware | **Usar todos** (`n_jobs=-1`), decisão da equipe |
| 11 | `min_samples_split` na grade | Manter × remover | Só tem efeito quando passa de 2 × `min_samples_leaf`. Nos casos em que poderia agir, trocar 2 por 100 muda o F1 em +0,001 (`min_samples_leaf=10`, p = 0,39) e em 0,000 (`min_samples_leaf=25`, p = 0,95) | **Removido** (grade de 420 → 210 combinações) |
| 11 | Hiperparâmetros da árvore | Grade: `max_depth` 3–12, `min_samples_leaf` 10–200, `ccp_alpha` 0–0,001, gini/entropy | Melhor F1 0,558 ± 0,003; gap treino–validação 0,005 (sem overfitting); contra a configuração fixa +0,004 (p = 0,36) | **entropy, max_depth=8, min_samples_leaf=100, ccp_alpha=0,0001** |
| 11 | Árvore: melhor × regra de 1 desvio-padrão | `max_depth=8` × `max_depth=2` | A curva de `max_depth` é plana a partir de 2 (0,556 contra 0,558) | **Melhor combinação** (decisão da equipe); explicabilidade na Fase 13 |
| 11 | Faixa da busca do CatBoost | depth 4, 6, 8 × 1–800 iterações (lr 0,1) | A PR-AUC de validação atinge o pico em ~100 iterações (depth 4 e 6) e em ~26 (depth 8), depois cai, enquanto o treino sobe (até 0,97). A configuração fixa (6, 300) já passou do pico | **depth 4–6, poucas centenas de iterações, mais `l2_leaf_reg`** (próxima rodada) |

## Decisões da revisão da Etapa 1 (comentários da professora, 23/11/2026)

| Fase | Decisão | Alternativas consideradas | Evidência | Escolha |
|---|---|---|---|---|
| 1 | Remuneração implausível | Manter × piso de domínio | Mínimo de US$ 1/ano; 469 valores abaixo de US$ 1.000 (2,3% dos observados) | **Piso de US$ 1.000/ano** → NA (regra fixa, antes do snapshot do Bloco B) |
| 3 | Candidatas fora de H1–H6 (`Age`, `EdLevel`, `Employment`, `Country`, `Industry`, `AIThreat`, `AISent`, remuneração) | Não testar × testar | As 8 têm p < 0,05; só `AISent` (V = 0,31) e `Country` (V = 0,13) passam de 0,10 | **Testadas e registradas**; ficam fora do conjunto final até o experimento 9 × 17 no Bloco B |
| 3 | Teste da remuneração | Só Mann-Whitney × também por decis | Mann-Whitney p = 0,33 (medianas iguais), mas a curva tem forma de U (40% → 26–29% → 36%); por decis p ≈ 2e-24, V = 0,085 | **Qui-quadrado por decis** (o Mann-Whitney fica como observação) |
| 3 | H3 como tautologia: ausência | — | A ausência de 30% de `AIAcc`/`AIComplex` vem de quem abandonou o questionário (99% também sem rótulo); na base rotulada ela é 0,4% e V < 0,03 | **Não é tautologia** |
| 3 | H3 como tautologia: categoria "I don't use AI tools for complex tasks" | — | 15,5% da base, adoção de 4,5%, 60% declaram não usar IA; sem ela, o V de `AIComplex` cai de 0,34 para 0,24 | **Mantida com ressalva**; o tratamento (recodificar × retirar `AIComplex`) fica para decisão da equipe |
| 5 | Segundo modelo no texto da Fase 5 | — | A ficha e a Fase 8 já usam CatBoost | Texto atualizado para **CatBoost** |
| B | Tratamento da categoria "I don't use AI tools for complex tasks" | (a) manter; (b) recodificar como ausente; (c) retirar `AIComplex` | Escolher de antemão seria arbitrário: (b) preserva a escala ordinal de quem usa IA, (c) elimina também a causalidade reversa, e não há evidência de qual custa menos F1 | **Decidir por experimento** (a × b × c), com o mesmo protocolo das Fases 8–10 — decisão da equipe |
| B | `AISent` na análise de sensibilidade | Incluir × não incluir | É uma atitude em relação à IA (V = 0,31), com o mesmo risco de causalidade reversa de `AIAcc`/`AIComplex` | **Incluir** junto com `AIAcc` e `AIComplex` — decisão da equipe |
| B | Candidatas do experimento de conjunto | 7 × 8 variáveis | A remuneração também é significativa (por decis, V = 0,085) | **As 8**, incluindo a remuneração: experimento 9 × 17 — decisão da equipe |

## Decisões da revisão do conjunto de preditores (experimentos C1–C3, 23/11/2026)

Protocolo: mesmas 5 dobras (`cv_esquema`), F1 da classe 1 com o limiar ajustado (Fase 10), teste de Nadeau-Bengio. Árvore com os hiperparâmetros da Fase 11; CatBoost na configuração fixa (300 iterações, lr 0,1, depth 6), porque a busca dele ainda não foi feita. Conferência: com os 9 preditores, os construtores reproduzem a Fase 11 (árvore, 0,558) e a Fase 10 (CatBoost, 0,561).

| Exp. | Decisão | Alternativas testadas | Evidência | Escolha e motivo |
|---|---|---|---|---|
| C1 | Conjunto de preditores | 9 (atual) × 17 (+ 8 candidatas) × 16 (17 sem `AISent`) | Árvore: 17 − 9 = +0,001 (p = 0,80). CatBoost: 17 − 9 = +0,008 (p = 0,02), mas 16 − 9 = +0,005 (p = 0,20). As 8 continuam significativas só no treino; o VIF nos 17 remove 2 dummies novas | **Manter os 9** — decisão da equipe. (1) A árvore não ganha nada; (2) o ganho do CatBoost é pequeno e vem quase todo de `AISent`, que tem o mesmo risco de causalidade reversa da H3; (3) mesmo critério das Fases 9 e 10 (diferença desprezível → o mais simples); (4) adotar os 17 obrigaria a refazer as Fases 8–11 e traria `Country` (169 níveis) e a remuneração (47% de ausência) |
| C3 | Categoria "I don't use AI tools for complex tasks" | Original × categoria → ausente × categoria → nível neutro × sem `AIComplex` | → ausente: Δ ≈ 0 (o modelo continua separando o grupo pelo `'missing'`, que passa a ser dominado por ele; a opção não remove nada). → neutro: −0,016 (árvore, p = 0,05) e −0,017 (CatBoost, p = 0,01). Sem `AIComplex`: −0,033 e −0,028 (p ≤ 0,001) | **Juntar ao nível neutro ("Neither good or bad")** — decisão da equipe. (1) Coerência com o descarte de `AISelect`: a categoria praticamente declara o não uso; (2) custa metade de retirar `AIComplex`; (3) preserva a escala ordinal de quem usa IA. A opção "→ ausente" foi descartada porque não neutraliza a categoria com a estratégia isBAD/`'missing'` |
| C3 | Alcance da recodificação | Refazer as Fases 8–11 × aplicar daqui em diante | As Fases 8–10 comparam estratégias com a mesma base, e as diferenças entre elas continuam válidas (mesmo argumento da Seção 11.3 para o CatBoost fixo); a curva de `max_depth` da árvore é plana | **Daqui em diante** (Fase 11 restante e Fase 12); hiperparâmetros da árvore mantidos. Nova referência de F1: árvore 0,542 ± 0,006; CatBoost 0,545 ± 0,004 |
| C4 | Métrica da exploração do CatBoost | `PRAUC` interna do CatBoost × `average_precision_score` | A `PRAUC` interna interpola de outro jeito e dá valores maiores com empates (num teste: 0,453 × 0,443); na 1ª versão da busca, apontou pico na 1ª iteração para `depth=4` | **`average_precision_score`** nas previsões a cada 10 iterações (mesma métrica das Fases 9 e 10). As curvas da 11.3 ganharam uma nota: conclusão qualitativa mantida, valores absolutos não comparáveis |
| C4 | Hiperparâmetros do CatBoost | Exploração (PR-AUC): depth {4, 5, 6} × l2_leaf_reg {1, 3, 10, 30}, lr 0,05, até 800 it.; grade final (F1 com limiar): 4 melhores + configuração fixa | Exploração plana (PR-AUC 0,514–0,517, dp ≈ 0,017). Grade final: depth 6/l2 30/380 it. = 0,5492 (gap 0,013) e depth 4/l2 3/200 it. = 0,5491 (gap 0,007). Contra a fixa: +0,004 (p = 0,32). Contra a árvore na mesma base: +0,007 (p = 0,003) | **depth=4, l2_leaf_reg=3, 200 it., lr 0,05**. Empate no F1 resolvido pelo menor gap (critério de parcimônia das Fases 9 e 10): metade do overfitting, mais raso, ~⅓ do tempo de treino. O ganho sobre a fixa não é significativo; o que a busca melhora é o gap |
| C2 | Variáveis de percepção (`AIAcc`, `AIComplex`) | Manter × retirar | Sem as duas, o F1 cai para 0,486 (árvore) e 0,489 (CatBoost), −0,07 (p < 0,001), abaixo da meta de 0,50, mas bem acima do baseline (0,322) | **Manter, com ressalva** — decisão da equipe. São o sinal mais forte da base e estão disponíveis no momento da decisão (questionário); a dependência do modelo em relação a elas fica registrada como limitação no relatório e na explicabilidade (Fase 13) |

## Decisões da Fase 12 (comparação estatística e teste único, 24/11/2026)

Protocolo fixado **antes** de abrir o teste: seleção só no treino (mesmas 5 dobras, Nadeau-Bengio); reajuste no treino inteiro com o limiar escolhido de novo por 3 dobras internas; o teste é previsto uma única vez e **não muda nenhuma escolha**.

| Decisão | Alternativas | Evidência | Escolha e motivo |
|---|---|---|---|
| Regra de escolha do modelo final | Maior F1 na validação × modelo mais simples | — | **Maior F1 na validação, se a diferença para a árvore for significativa (p < 0,05); senão, a árvore.** É o mesmo critério das Fases 9 a 11 (diferença desprezível → o mais simples), escrito antes de abrir o teste para que o teste não influencie a escolha |
| Modelo final | Baseline estratificado × Árvore (Fase 11) × CatBoost (Seção 11.6) | Validação: F1 0,549 ± 0,005 (CatBoost) × 0,542 ± 0,006 (árvore) × 0,322 (baseline). CatBoost − árvore: F1 +0,007 (p = 0,003), PR-AUC +0,028 (p = 0,020) | **CatBoost** (depth 4, l2 3, 200 it., lr 0,05). A árvore fica como referência interpretável para a Fase 13 |
| Incerteza no teste | Só valor pontual × bootstrap × McNemar | — | **Bootstrap** (2.000 reamostras das linhas do teste, as mesmas para todos os modelos, para diferenças pareadas) como medida principal; **McNemar** só como complemento, porque compara acurácia e não F1 e favorece o modelo de limiar mais alto |
| Resultado no teste (não muda a escolha) | — | CatBoost F1 0,549 [0,532; 0,566], igual à validação; árvore 0,541 [0,525; 0,556]; baseline 0,312. CatBoost − árvore: F1 +0,008 [−0,001; +0,017], PR-AUC +0,034 [+0,022; +0,046]; McNemar p < 0,001. ROC-AUC ≈ 0,70 | Critério de sucesso da Fase 6 atingido (F1 > 0,50 em todo o IC). A validação cruzada não foi otimista. Poder preditivo **moderado**, com forte dependência de `AIAcc`/`AIComplex` (Seção R.3) |

## Decisões da preparação da entrega (Fases 0 a 12, 24/11/2026)

| Decisão | Alternativas | Evidência | Escolha e motivo |
|---|---|---|---|
| Edição da pesquisa | 2024 (como estava) × 2025 | As colunas `AIAgents`, `AIAgentChange`, `AIAgent_Uses` etc. só existem no questionário de 2025; a base tem 49.191 respostas (a edição de 2024 teve cerca de 65 mil) | **2025.** Corrigida no cabeçalho do notebook, na Seção 7.2, na ficha de proposta e no README; a nota 14 do cabeçalho registra a correção |
| Link da pesquisa | — | A professora pediu nos comentários da Etapa 1 | <https://survey.stackoverflow.co/2025> (resultados e acesso aos dados) no notebook, na ficha, no README e no relatório |
| Cabeçalho do notebook | Manter "Entrega 1 (Semana 1)" × descrever a entrega atual | O cabeçalho descrevia só as Fases 6 e 7, e as notas de revisão cobriam só o Bloco A | "Entrega atual: Fases 0 a 12" e notas 11 a 14: revisão do conjunto, Fase 11 completa, Fase 12 e correção do ano |
| Nome do notebook | Manter `eda_pipeline_fase0_11.ipynb` × renomear | O arquivo já contém a Fase 12 | **`eda_pipeline_fase0_12.ipynb`**, seguindo o padrão de nome dos arquivos anteriores. Nenhum código do notebook referencia o próprio nome |
| `eda_pipeline_fase0_4.ipynb` | Manter × remover | Foi restaurado do commit `c73c68b` só para consulta e não entra na entrega | **Removido** do diretório. Continua recuperável pelo git (`git show c73c68b:eda_pipeline_fase0_4.ipynb`) |
| Formato do relatório executivo | Markdown no repositório × documento compartilhável × `.docx` | A equipe ainda não definiu o formato; os demais documentos do projeto (ficha e decisões) são Markdown versionado | **`RELATORIO_EXECUTIVO.md`** no repositório. Pode ser convertido para PDF ou Word depois, sem mudar o conteúdo |
| Conteúdo do relatório | — | A professora pediu um relatório sem código; o enunciado pede a declaração de uso de IA no Apêndice B | Público: a professora. Linguagem de negócio, sem código e sem nomes de colunas no texto principal. Seções: resumo, problema, dados, AED, construção do modelo, resultados da Fase 12, limitações e próximos passos. Apêndice A: decisões e descartes. Apêndice B: declaração de uso de IA, a partir do rascunho da equipe (resposta 4 à professora) |

---

## Registro de atividades

### 15/11/2026
- Calculada a proporção do rótulo: 7.889 adotantes (31,9%) × 16.865 não adotantes (68,1%), razão ≈ 2,1:1.
- Ficha de proposta atualizada para o CatBoost; `catboost` adicionado ao `requirements.txt` e `scikit-learn>=1.5` (necessário para o `TunedThresholdClassifierCV`).
- Uma primeira seção "Fase 8" foi escrita comparando os modelos. Ao ler o documento da professora, vimos que ela misturava as Fases 8, 10, 11 e 12, e por isso foi refeita (ver abaixo).
- Lista de pendências até a Fase 10 levantada a partir do documento.
- Seção "Fase 8" antiga removida do notebook e substituída por:
  - **7.6:** VIF refeito só no treino;
  - **Fase 8:** pipelines por família (árvore e CatBoost), evidência sobre escala e outliers, categorias raras e novas;
  - **Fase 9:** diagnóstico de ausência (NA × alvo, teste de MAR no bloco) e comparação de 3 estratégias na árvore e 2 no CatBoost;
  - **Fase 10:** diagnóstico (razão 2,14:1 no treino) e experimento com os níveis 0, 1, 2 e 1+2 nos dois modelos;
  - Quadro de Decisões de Modelagem, Quadro de Descarte estendido e versões das bibliotecas.
- Criado este arquivo de registro.
- Notebook executado do início ao fim sem erros (cerca de 6 min; a Fase 10 com o CatBoost é a parte mais pesada). README atualizado.

- **Fase 11 (parcial):**
  - escopo definido e aprovado pela equipe;
  - busca em grade da árvore, verificação de `min_samples_split`, gap treino–validação e curva de `max_depth`;
  - curvas de validação do CatBoost (depth × iterações);
  - Quadro de Decisões e Quadro de Descarte estendidos para a Fase 11.

  Com isso, o notebook passa a levar ~12 min para rodar inteiro.

### 23/11/2026 — Revisão da Etapa 1, parte A (fechamento da Fase 5)
- Respostas às perguntas da professora sobre a Etapa 1 levantadas com números da base.
- Fase 0: texto da exploração secundária atualizado; mesmo cuidado de `AISelect` aplicado à H3.
- Fase 1: piso de domínio de US$ 1.000/ano na remuneração.
- Fase 2: explicação do efeito do Z-score (distância em KNN/K-Means e penalização em Ridge/Lasso; árvores não são afetadas).
- Fase 3: investigação da H3 como tautologia e teste das 8 candidatas fora de H1–H6, com tamanho de efeito.
- Fase 5: curvas de `WorkExp` e da remuneração (taxa por decil + LOESS `it=0`); CatBoost no lugar de RF/GB no texto; H3 "mantida com ressalva" no quadro de hipóteses; Quadro de Descarte com todas as variáveis.
- Conjunto final continua com 9 variáveis. Notebook executado do início ao fim sem erros (~13 min); todas as métricas do Bloco B ficaram idênticas às da versão anterior (só mudaram os tempos de execução e a numeração do Quadro de Descarte).

### 23/11/2026 — Revisão da Etapa 1, parte C (experimentos do conjunto de preditores)
- Nova seção no notebook, depois da 11.4: **"Revisão do conjunto de preditores"** (R.1 a R.4).
  - **R.1:** conjunto de 17; testes da Fase 3 e VIF da Fase 4 refeitos só no treino.
  - **R.2:** experimento 9 × 17 × 16.
  - **R.3:** tratamento da categoria "I don't use…" e sensibilidade a `AIAcc`/`AIComplex`.
  - **R.4:** decisões, registro no Quadro de Decisões e no Quadro de Descarte, `X_train_final`/`X_test_final` com a categoria no nível neutro.
- Construtores parametrizados por conjunto (`arvore_do_conjunto`, `catboost_do_conjunto`), conferidos: com os 9 preditores, reproduzem a Fase 11 e a Fase 10.
- A opção "categoria → ausente", registrada antes como alternativa, mostrou-se ineficaz com a estratégia isBAD/`'missing'` e foi trocada por "→ nível neutro".
- `AISent` saiu com as demais candidatas; o experimento 16 × 17 mostrou que o ganho do CatBoost vinha dela.
- Notebook executado do início ao fim sem erros (~23 min; a nova seção leva ~10 min). As saídas das Fases 0–11 ficaram idênticas às da versão anterior, exceto os tempos de execução.

### 24/11/2026 — Revisão da Etapa 1, parte C4 (Fase 11 restante: busca do CatBoost)
- Nova seção "Fase 11 (continuação)" depois da revisão do conjunto: 11.5 (exploração pela PR-AUC, com gráfico das curvas) e 11.6 (grade final pelo F1 com o limiar, gap treino–validação, comparação com a configuração fixa e com a árvore).
- Primeira versão da exploração descartada: a métrica `PRAUC` interna do CatBoost apontava pico na 1ª iteração. Trocada pela `average_precision_score`.
- Títulos e escopo da Fase 11 atualizados (deixou de ser parcial); nota sobre a métrica nas curvas da 11.3; Quadro de Decisões com a linha do CatBoost.
- Gap treino–validação, configuração fixa → escolhida: PR-AUC 0,069 → 0,018; F1 0,024 → 0,007.
- Notebook executado do início ao fim sem erros (~32 min). As saídas anteriores ficaram iguais, exceto os tempos e a linha nova no Quadro de Descarte.

### 24/11/2026 — Fase 12
- Nova seção "Fase 12" antes do Quadro de Decisões:
  - protocolo fixado antes de abrir o teste;
  - **12.1:** comparação estatística no treino;
  - **12.2:** ajuste final no treino inteiro;
  - **12.3:** avaliação única no teste, com IC por bootstrap, diferenças pareadas, McNemar, curva precisão–recall e matrizes de confusão;
  - **12.4:** registro.
- Quadro de Decisões, Quadro de Descarte e "Próximos passos" atualizados.
- Antes de entrar no notebook, a seção foi testada num runner reduzido. Os números coincidiram com a C4.
- Notebook executado do início ao fim sem erros (~34 min). As saídas anteriores ficaram iguais, exceto os tempos e as linhas novas no Quadro de Decisões e no Quadro de Descarte.

### 24/11/2026 — Preparação da entrega (itens 1 a 5)
- Ano da pesquisa corrigido para 2025 e link incluído (notebook, ficha, README).
- Cabeçalho do notebook atualizado (entrega atual e notas 11 a 14).
- Notebook renomeado para `eda_pipeline_fase0_12.ipynb`; `eda_pipeline_fase0_4.ipynb` removido; README reescrito com a estrutura atual.
- Relatório executivo (`RELATORIO_EXECUTIVO.md`) escrito, com os Apêndices A e B. Os números foram conferidos com as saídas do notebook.
- O notebook não foi executado de novo: só mudaram células de texto (cabeçalho e Seção 7.2), e os resultados continuam os da execução anterior.

### 24/11/2026 — Ajustes cosméticos e versão compartilhável do relatório
- **p-valores muito pequenos:** o Quadro de Descarte e o Quadro de Decisões mostravam "p = 0.00". A função `delta_p` (Seção R.4) passou a escrever "p < 0.001" abaixo de 0,001 e três casas abaixo de 0,01. Motivo: "p = 0.00" sugere p exatamente zero.
- **Comentário da célula de registro da Fase 11:** saiu "(parcial)", porque a Fase 11 está completa.
- **Título deste arquivo:** "Entrega até a Fase 12".
- **Relatório também como documento compartilhável:**
  - **Onde:** documento do Claude Docs, privado até que a equipe o compartilhe: <https://claude.ai/artifact/JHxhaaMUZfYQ1qvdpjrYdL>.
  - **Conteúdo:** igual ao `RELATORIO_EXECUTIVO.md`.
  - **Motivo:** exporta para PDF ou Word e aceita comentários da equipe.
  - **Atenção:** o `.md` continua sendo a versão versionada. Uma mudança em um precisa ser repetida no outro.
- **Notebook executado do início ao fim sem erros (~34 min).** As saídas ficaram iguais às da execução anterior, exceto os tempos e a linha de `AIAcc` e `AIComplex` no Quadro de Descarte, que agora mostra "p < 0.001" no lugar de "p = 0.00".

### Pendências para as próximas entregas
- ~~**Fase 12:** avaliação única no teste~~ — feita (CatBoost, F1 0,549 no teste).
- **Fase 13:** explicabilidade do CatBoost final, com a árvore como referência, incluindo a dependência em relação a `AIAcc` e `AIComplex`.
- **Fase 14:** conclusão; atualizar o relatório executivo.
- *(Opcional)* validação cruzada aninhada.
- ~~Conferir o ano da edição da pesquisa~~ — 2025.
- ~~Entregas: nomes da equipe, ano e link da pesquisa, `.ipynb` e relatório executivo sem código~~ — feitas para as Fases 0 a 12.
- ~~Declarar o uso de IA generativa no Apêndice B do relatório~~ — feito; a equipe deve revisar o texto.
- Atualizar a resposta 3 à professora.
- ~~Cosméticos: "p = 0.00" no Quadro de Descarte, comentário "(parcial)" na célula de registro da Fase 11, título deste arquivo~~ — feitos.
- ~~*(Opcional, Seção 3)* Comparar o conjunto filtrado com o completo, sem o filtro bivariado~~ — coberto pelo experimento 9 × 17.
