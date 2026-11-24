# FICHA DE PROPOSTA DE PROJETO — CICLO DE CIÊNCIA DE DADOS
**Disciplina:** Ciência de Dados  
**Professora:** Trícia Santos Souto  
**Entrega:** Semana 1 — Fases 6 e 7 (Entrega 1)

---

### 1. Identificação da Equipe
- **Integrantes:** Lavínia Souza Caló de Figueiredo, Lucas Luige Costa Miranda e Sara Raquel Pinto Brandão Silva
- **Base de Dados:** *Stack Overflow Developer Survey, edição 2025 (módulo de IA e AI Agents)*: <https://survey.stackoverflow.co/2025>
- **Unidade de Análise:** um desenvolvedor profissional por linha. Mantivemos apenas quem respondeu `MainBranch == 'I am a developer by profession'`.

---

### 2. Tarefa Escolhida e Justificativa pela EDA (até 5 linhas)
- **Tarefa:** classificação binária supervisionada (Caminho A).
- **Justificativa:** na análise exploratória (Fases 0 a 5), os 10 preditores testados se associaram ao uso de agentes de IA ($p < 0{,}01$), com forças bem diferentes. As maiores vêm da percepção sobre a própria IA: `AIComplex` ($V = 0{,}34$) e `AIAcc` ($V = 0{,}27$), em que a adoção vai de 14% entre quem desconfia muito das respostas a 65% entre quem confia muito. Depois vêm `SOVisitFreq` ($V = 0{,}13$) e `DevType` ($V = 0{,}12$). Com um desfecho claro e 32% de positivos, o problema pede aprendizado supervisionado.

---

### 3. Variável-Alvo
- **Nome da Coluna:** `AIAgents_bin`
- **Definição Operacional:** $1$ = usa agentes de IA no trabalho, seja diária, semanal ou mensalmente; $0$ = não usa, ou usa IA apenas como autocompletar/copiloto.
- **Amostra Válida:** 24.754 desenvolvedores profissionais com rótulo. Cerca de um terço dos respondentes pulou a pergunta sobre agentes e ficou fora da modelagem, porque preencher o alvo seria inventar justamente a resposta que queremos prever.

---

### 4. Métrica Principal e Justificativa (Custo dos Erros)
- **Métrica Principal:** **F1-Score da classe positiva (adotantes)**.
- **Métricas Secundárias de Diagnóstico:** PR-AUC (*Average Precision*), *Balanced Accuracy* e acurácia global.
- **Justificativa de Negócio:** imaginamos o modelo nas mãos de uma empresa que decide quem recebe licença e treinamento em agentes de IA. Um falso negativo deixa sem a ferramenta um desenvolvedor que a usaria, e a empresa perde a produtividade que ele ganharia. Um falso positivo custa uma licença pouco aproveitada. A acurácia engana nesse cenário: só 31,9% dos casos são positivos, então um modelo que sempre responde "não adota" acerta 68,1% e tem F1 igual a zero. O F1 da classe positiva pune esse atalho e nos obriga a equilibrar *recall* e precisão.
- **Critério de Sucesso:** na validação cruzada do treino, o modelo precisa superar os dois baselines ingênuos, o majoritário (F1 = 0,00) e o estratificado aleatório (F1 = 0,32). Nossa meta é F1 acima de 0,50.

---

### 5. Modelos Pretendidos e Justificativa
- **Modelo de Base (Obrigatório em Sala):** Árvore de Decisão (`DecisionTreeClassifier`).
- **Segundo Modelo Pretendido:** *CatBoost* (`CatBoostClassifier`), um *gradient boosting* de árvores com tratamento nativo de variáveis categóricas.
- **Justificativa Técnica:** as curvas LOESS da Fase 5 mostram relações que seguem sempre no mesmo sentido, mas mudam de inclinação. A adoção cai rápido nos primeiros 5 anos de programação (`YearsCode`), fica estável até cerca de 28 anos e volta a cair depois disso. Em `JobSat`, a taxa quase não se mexe entre as notas 0 e 4 e sobe a partir da 5. Árvores aprendem esses pontos de corte por conta própria e captam interações entre variáveis sem que precisemos especificá-las.
- **Por que CatBoost:** 7 dos 9 preditores finais são categóricos (`DevType`, `OrgSize`, `ICorPM`, `RemoteWork`, `SOVisitFreq`, `AIAcc`, `AIComplex`). O CatBoost os recebe como texto, sem dummies, e codifica cada categoria pelo alvo de forma ordenada: cada linha usa só a média das linhas que vieram antes dela numa permutação aleatória, o que impede que ela veja o próprio rótulo. Isso dispensa o *target encoding* manual de `DevType`. O desbalanceamento é tratado com `auto_class_weights='Balanced'` e os ausentes numéricos são aceitos diretamente. Para a comparação ser justa, a Árvore de Decisão mantém o pipeline com dummies e *target encoding*, o CatBoost recebe as colunas brutas, e os dois são avaliados nas mesmas dobras estratificadas e com as mesmas métricas dos baselines.

---

### 6. Riscos Percebidos e Estratégias de Mitigação
1. **Desbalanceamento moderado de classes (~2,1:1):** na validação cruzada, vamos testar a ponderação de classes (`class_weight='balanced'` na árvore e `auto_class_weights='Balanced'` no CatBoost) e o ajuste do limiar de decisão (*threshold tuning*), sempre comparando com o baseline (`DummyClassifier`).
2. **Dados ausentes nos preditores:** seguimos o método isBAD da análise exploratória, com mediana e coluna indicadora para as numéricas e a categoria `'missing'` para as categóricas. Na Fase 7, levamos esse tratamento para dentro de um `Pipeline`/`ColumnTransformer` do scikit-learn. A separação entre treino e teste acontece antes dele, e cada dobra da validação cruzada recalcula as medianas sem enxergar os dados de validação.
3. **Alta cardinalidade:** `DevType` tem 33 níveis, alguns com poucos respondentes. Usamos *target encoding* com suavização de Laplace ($m=10$), que aproxima os níveis raros da taxa global. O encoding é calculado por dobras (*cross-fitting*), de modo que nenhuma linha é codificada com o próprio rótulo. No CatBoost, essa etapa é substituída pela codificação ordenada que o próprio modelo faz.
4. **Causalidade reversa nos preditores de percepção:** `AIAcc` e `AIComplex` podem ser, em parte, efeito do uso de agentes, já que quem usa tende a avaliar melhor a IA. Vamos comparar os modelos com e sem essas duas variáveis para medir quanto do desempenho depende delas.
