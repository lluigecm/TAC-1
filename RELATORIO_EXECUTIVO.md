# Relatório Executivo — Quem adota agentes de IA? Um modelo preditivo com a Stack Overflow Developer Survey 2025

**Disciplina:** Ciência de Dados  
**Professora:** Trícia Santos Souto  
**Equipe:** Lavínia Souza Caló de Figueiredo, Lucas Luige Costa Miranda e Sara Raquel Pinto Brandão Silva  
**Base de dados:** Stack Overflow Developer Survey, edição 2025, com o módulo de IA e AI Agents. Disponível em <https://survey.stackoverflow.co/2025>  
**Escopo desta entrega:** Fases 0 a 12 do pipeline. Os cálculos estão no notebook `eda_pipeline_fase0_12.ipynb`.

---

## 1. Resumo

- **Pergunta:** quais desenvolvedores profissionais usam agentes de IA no trabalho? Dá para prever isso com o perfil e as percepções que eles declaram no questionário?
- **Base:** 24.754 desenvolvedores profissionais que responderam à pergunta sobre agentes de IA. Desses, 31,9% são adotantes.
- **Modelo final:** CatBoost (*gradient boosting*), comparado com uma Árvore de Decisão e com um classificador ingênuo, que serve de *baseline*.
- **Resultado no conjunto de teste:** o modelo foi avaliado uma única vez em dados que não usou durante o desenvolvimento.
  - O F1 da classe "adotante" ficou em **0,549**, com intervalo de 95% entre 0,532 e 0,566. A meta era 0,50.
  - O *baseline* ingênuo fica em 0,312.
  - O resultado no teste é igual ao estimado na validação cruzada, o que indica que a estimativa não foi otimista.
- **Em termos práticos:**
  - **Alcance:** o modelo encontra 78 de cada 100 adotantes.
  - **Acerto:** de cada 100 desenvolvedores que ele aponta, 42 de fato adotam. Escolhendo ao acaso, seriam 32.
- **Principal limitação:** o poder preditivo é moderado (ROC-AUC de cerca de 0,70) e depende muito de duas percepções sobre IA:
  - a confiança na precisão da ferramenta;
  - a avaliação dela em tarefas complexas.

  Essas percepções podem ser tanto causa quanto consequência do uso.

## 2. Problema de negócio

Uma organização que planeja distribuir licenças, programas-piloto ou treinamentos de agentes de IA quer saber, **antes** de decidir, quais desenvolvedores têm mais chance de adotá-los. Essa é a tarefa do projeto, formulada como **classificação binária supervisionada**:

- **Classe 1 (adotante):** usa agentes de IA no trabalho, com frequência diária, semanal ou mensal.
- **Classe 0 (não adotante):** não usa, só planeja usar, ou usa IA apenas como autocompletar.

**Métrica principal: F1 da classe adotante.** A escolha vem do custo dos erros:

| Erro | O que acontece | Custo |
|---|---|---|
| Falso negativo | Deixar de fora alguém que adotaria | Alto: perde-se produtividade e o profissional fica para trás |
| Falso positivo | Incluir alguém que não adota | Baixo a moderado: uma licença ou vaga de treinamento mal aproveitada |

Com 31,9% de adotantes, a acurácia engana: um modelo que sempre responde "não adota" acerta 68%, mas não serve para nada. O F1 equilibra encontrar os adotantes (recall) com não errar demais (precisão).

**Critério de sucesso:** F1 acima de 0,50 e acima do *baseline* aleatório estratificado (≈ 0,32).

## 3. Dados e preparação

| Etapa | Linhas |
|---|---|
| Respostas brutas da pesquisa (172 colunas) | 49.191 |
| Desenvolvedores profissionais (recorte de escopo) | 37.467 |
| Após retirar ausências em idade, escolaridade e vínculo (menos de 2% em cada) | 36.808 |
| Com resposta sobre agentes de IA (base de modelagem) | **24.754** |
| Treino (80%) / teste (20%), estratificados | 19.803 / 4.951 |

**Por que a base de modelagem é menor:** cerca de um terço dos profissionais não respondeu ao bloco sobre agentes. A resposta (o rótulo) nunca foi preenchida artificialmente, e essas linhas ficaram de fora da modelagem.

**Seleção de variáveis:** das 172 colunas, ficaram 20 perguntas de resposta única sobre perfil, empresa, trabalho e percepções de IA.

**Variáveis excluídas por vazamento:** a pergunta sobre o uso de ferramentas de IA em geral (*AISelect*) foi retirada dos preditores. Ela é uma prévia quase direta da resposta: quem não usa IA nenhuma também não usa agentes.

**Tratamento dos dados:**
- **Valores implausíveis:** foram tratados como ausentes. Por exemplo, remunerações abaixo de US$ 1.000 por ano (a base tinha valores de US$ 1) e outliers extremos (mais de 3 intervalos interquartis).
- **Ausências:** não foram descartadas. Elas foram marcadas com um indicador, porque a ausência também carrega informação. Por exemplo, quem não informou o tempo de programação adota mais.
- **Testes não paramétricos:** nenhuma variável numérica tem distribuição normal, então os testes da análise exploratória são não paramétricos.

## 4. O que a análise exploratória mostrou

Seis hipóteses foram formuladas antes de olhar os dados. Todas mostraram associação estatisticamente significativa, o que é esperado com quase 25 mil respondentes. Por isso, a decisão se baseou no **tamanho do efeito** (V de Cramér ou correlação *rank-biserial*), com 0,10 como limiar de relevância.

| Hipótese | Resultado | Tamanho do efeito | Situação |
|---|---|---|---|
| **H1.** Mais experiência → menos uso | Sentido confirmado | Desprezível (r = −0,07) | Mantida com ressalva |
| **H2.** Cargo e porte da empresa influenciam | Cargo: sim. Porte: no sentido **oposto** (empresas maiores adotam um pouco menos) | Cargo: fraco (V = 0,12). Porte: desprezível | Parcial |
| **H3.** Mais confiança na IA → mais uso | A adoção vai de 14% ("desconfia muito") a 65% ("confia muito") | Moderado (V = 0,27 e 0,34): o maior da base | Mantida com ressalva (ver abaixo) |
| **H4.** Satisfação no cargo associada ao uso | Mais satisfeitos adotam um pouco mais | Desprezível (r = 0,05) | Mantida com ressalva |
| **H5.** Trabalho remoto e contribuidores individuais adotam mais | Remoto: sim, levemente. Mas **gestores adotam mais** que contribuidores individuais (42% contra 30%) | Desprezível a fraco | Parcial |
| **H6.** Quem usa mais o Stack Overflow adota menos | Confirmada: de 40% (visita rara) a 25% (várias vezes por dia) | Fraco (V = 0,13) | Mantida |

**A ressalva sobre a H3:**
- **Uma das respostas descreve o próprio não uso.** A opção "não uso IA para tarefas complexas" tem só 4,5% de adotantes.
- **A causalidade pode ser reversa:** usar agentes pode levar a confiar na IA, e não o contrário. Com dados de um único momento, não é possível separar as duas direções.
- **Consequência:** a H3 foi lida como **associação**, não como causa, e o modelo foi testado com e sem essas percepções (Seção 5).

**Variáveis fora das hipóteses:** oito variáveis também foram testadas:
- **Testadas:** idade, escolaridade, vínculo de trabalho, país, setor, percepção de ameaça da IA, opinião geral sobre IA e remuneração.
- **Destaques:** a remuneração tem relação **em U**. Adotam mais os que ganham menos de cerca de US$ 16 mil por ano e os que ganham mais de US$ 180 mil.
- **Destino:** todas foram avaliadas no modelo (Seção 5) e nenhuma ficou no conjunto final.

**Forma das relações:** as curvas de suavização mostraram platôs e mudanças de inclinação, além da forma em U da remuneração. Por isso o projeto usa modelos baseados em **árvores**, que aprendem pontos de corte sem precisar especificá-los à mão.

## 5. Como o modelo foi construído

**Proteção contra vazamento:**
- **Teste separado no início:** o conjunto de teste (20%) foi separado antes de qualquer etapa que aprende com os dados e ficou lacrado até a avaliação final.
- **Aprendizado só no treino:** todo o pré-processamento que aprende com os dados (medianas, cortes de outlier, codificação de categorias pelo alvo) é reajustado só no treino, dentro de cada dobra da validação cruzada.
- **Filtros refeitos:** os testes de associação e de colinearidade foram refeitos só no treino, e chegaram às mesmas variáveis.

**Comparação de alternativas:** cada decisão foi tomada por experimento, sempre nas mesmas 5 dobras de validação cruzada. A comparação usou o teste t corrigido de Nadeau-Bengio, próprio para dobras que compartilham dados. Quando a diferença era desprezível, ficava a opção mais simples.

| Decisão | O que foi comparado | Escolha |
|---|---|---|
| Ausências | Indicador de ausência × mediana/moda × vizinhos mais próximos (KNN) | Indicador de ausência na árvore e tratamento nativo no CatBoost. Houve empate estatístico, e essas opções preservam a informação da ausência |
| Dados sintéticos | Gerar × não gerar | Não gerar: a classe adotante não é rara (31,9%) |
| Desbalanceamento | Nada × peso de classe × ajuste do limiar de decisão × os dois | **Ajuste do limiar** (o limiar ficou em cerca de 0,27, em vez de 0,5). Ganha de todos sem distorcer as probabilidades |
| Conjunto de preditores | 9 variáveis × 17 (com as 8 candidatas) | **9 variáveis.** A árvore não ganha nada. O ganho do CatBoost (+0,008) vem quase todo da opinião geral sobre IA, que tem o mesmo risco de causalidade reversa da H3 |
| Resposta "não uso IA para tarefas complexas" | Manter × tratar como ausente × juntar ao nível neutro × retirar a pergunta | **Juntar ao nível neutro.** É coerente com a exclusão de *AISelect* e custa 0,016 de F1, metade do custo de retirar a pergunta |
| Hiperparâmetros da árvore | 210 combinações | Profundidade 8, folhas com pelo menos 100 pessoas |
| Hiperparâmetros do CatBoost | 12 combinações exploradas e 4 finalistas | Profundidade 4, 200 iterações. Empatou com o melhor e tem o menor sobreajuste |

**As 9 variáveis finais:**
- tempo de programação;
- satisfação no cargo;
- tipo de cargo;
- porte da empresa;
- papel (contribuidor individual ou gestor);
- modelo de trabalho (remoto, híbrido ou presencial);
- frequência de visita ao Stack Overflow;
- confiança na precisão da IA;
- avaliação da IA em tarefas complexas.

**Teste de sensibilidade:** sem as duas percepções sobre IA, o F1 cai cerca de 0,07, para 0,49, abaixo da meta. O modelo depende fortemente delas.

## 6. Resultados (Fase 12)

**Escolha do modelo, só com o treino:** a regra foi fixada antes de abrir o teste.

| Modelo | F1 na validação cruzada (5 dobras) | PR-AUC |
|---|---|---|
| *Baseline* aleatório estratificado | 0,322 | 0,320 |
| Árvore de Decisão | 0,542 | 0,488 |
| **CatBoost** | **0,549** | **0,516** |

O CatBoost foi melhor que a árvore em F1 (+0,007, p = 0,003) e em PR-AUC (+0,028, p = 0,020), e por isso é o **modelo final**. A árvore fica como referência interpretável.

**Avaliação única no conjunto de teste (4.951 desenvolvedores nunca vistos):**

| Modelo | F1 [IC 95%] | Precisão | Recall | PR-AUC | ROC-AUC |
|---|---|---|---|---|---|
| *Baseline* | 0,312 [0,290; 0,332] | 0,315 | 0,309 | 0,318 | 0,50 |
| Árvore | 0,541 [0,525; 0,556] | 0,395 | 0,858 | 0,483 | 0,69 |
| **CatBoost** | **0,549 [0,532; 0,566]** | **0,423** | **0,783** | **0,517** | **0,70** |

Os intervalos de confiança vêm de 2.000 reamostragens (*bootstrap*) do conjunto de teste.

**Leitura:**
- **A meta foi atingida:** o intervalo inteiro do CatBoost fica acima de 0,50, e os dois modelos superam o *baseline* com folga.
- **A estimativa se confirmou:** o desempenho no teste é igual ao da validação cruzada (0,549 nos dois casos para o CatBoost).
- **CatBoost × árvore no teste:**
  - **Ordenação:** o CatBoost ordena melhor quem tem mais chance de adotar (PR-AUC +0,034, intervalo de +0,022 a +0,046).
  - **F1:** a vantagem é pequena (+0,008), e o intervalo vai de −0,001 a +0,017, ou seja, encosta no zero.
  - **Perfis diferentes:** a árvore encontra mais adotantes (86%), ao custo de mais falsos positivos. O CatBoost é mais preciso (42% contra 40%).
- **Na prática, com o CatBoost:**
  - **Adotantes:** de cada 100, o modelo aponta 78.
  - **Não adotantes:** metade recebe um alerta falso. É um custo aceitável, porque o erro caro é deixar um adotante de fora.

## 7. Limitações

1. **Poder preditivo moderado.** ROC-AUC de 0,70 e PR-AUC de 0,52 indicam que, com perfil e percepções, a adoção é previsível **só em parte**. O modelo serve para priorizar, não para decidir sozinho.
2. **Dependência das percepções sobre IA.** Sem a confiança na precisão da IA e a avaliação em tarefas complexas, o F1 cai para cerca de 0,49. Essas percepções podem ser consequência do uso (causalidade reversa), então o modelo **não deve ser lido como causal**. Ele mostra quem tende a adotar, não por quê.
3. **Corte transversal e autodeclaração.** Os dados são de um único momento, respondidos pelos próprios desenvolvedores, sem verificação externa.
4. **Viés de amostra.**
   - **Quem responde:** quem responde à pesquisa do Stack Overflow não representa todos os desenvolvedores.
   - **Quem abandona:** cerca de um terço dos profissionais abandonou o questionário antes do bloco de IA e ficou fora da modelagem.
5. **Hipóteses parcialmente refutadas.** Porte da empresa e papel do profissional (H2 e H5) foram em sentido oposto ao esperado. Os achados foram registrados como estão, sem reformular as hipóteses depois de ver os dados.

## 8. Próximos passos

- **Fase 13 — Explicabilidade:**
  - quais variáveis mais pesam no CatBoost;
  - como cada uma muda a probabilidade de adoção;
  - quanto o modelo depende das percepções sobre IA.

  A árvore entra como referência, com regras legíveis.
- **Fase 14 — Conclusão:** recomendações de uso do modelo e consolidação final.

---

## Apêndice A — Principais decisões e descartes

O registro completo, com a evidência de cada decisão, está no Quadro de Decisões e no Quadro de Descarte do notebook, e no arquivo `DECISOES.md` do repositório.

| Item | Decisão | Motivo |
|---|---|---|
| *AISelect* (uso de ferramentas de IA) | Excluída | Prévia quase direta do rótulo (vazamento) |
| Experiência profissional (anos) | Excluída | Redundante com o tempo de programação (colinearidade) |
| Idade, escolaridade, vínculo, país, setor, ameaça da IA, opinião sobre IA e remuneração | Excluídas | Testadas no modelo: nenhum ganho na árvore; no CatBoost, ganho pequeno vindo só da opinião sobre IA |
| Resposta "não uso IA para tarefas complexas" | Juntada ao nível neutro | Descreve o próprio não uso |
| Confiança na IA e avaliação em tarefas complexas | Mantidas com ressalva | Sinal mais forte da base, disponível no momento da decisão; dependência registrada como limitação |
| Rótulo ausente (cerca de 1/3) | Linhas fora da modelagem | O rótulo nunca é imputado |
| Dados sintéticos e reamostragem (SMOTE) | Não usados | A classe não é rara, e o ajuste do limiar bastou |
| Modelo final | CatBoost | Melhor F1 na validação (p = 0,003), confirmado no teste |

## Apêndice B — Declaração de uso de IA generativa

A equipe usou o **Claude Code** (assistente de programação da Anthropic), dentro do VS Code, para:

- escrever, refatorar e otimizar o código do notebook (pipelines do scikit-learn, testes estatísticos e gráficos);
- executar o notebook e corrigir erros;
- gerar números de conferência;
- revisar o texto das seções e redigir a primeira versão deste relatório.

**As decisões de modelagem foram tomadas pela equipe:**
- o alvo e a métrica;
- o escopo de cada fase;
- a escolha da melhor combinação da árvore, em vez da regra de um desvio-padrão;
- a manutenção dos 9 preditores;
- o tratamento da resposta "não uso IA para tarefas complexas";
- a manutenção, com ressalva, das percepções sobre IA.

Cada decisão está registrada, com a evidência, no arquivo `DECISOES.md`.

**Conferência dos resultados:**
- o notebook foi executado do início ao fim a cada alteração;
- os números deste relatório foram conferidos com as saídas do notebook.
