# Analise-de-Tickets-de-Suporte-de-TI
Projecto de análise de dados que explora padrões, tempos de resolução e tendências em tickets de TI, usando Power BI, para gerar insights e melhorar a eficiência do suporte.

![Power BI](https://img.shields.io/badge/Ferramenta-Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![Status](https://img.shields.io/badge/Status-Concluído-brightgreen)

---

## Resumo Executivo
 
Este projecto analisa **24.749 tickets de suporte de TI** do período de 2024 a 2025, expandidos para **~116.000 linhas** após a normalização das etiquetas técnicas. A análise revela que o maior risco operacional está nos **40,32% de tickets de Alta Prioridade** (9.655 tickets) e que, apesar da estabilidade no volume de chamados entre os dois anos, o **tempo médio de resolução de 47,88 horas** está acima do SLA definido de 36 horas. As principais causas de paragens técnicas são falhas de **Desempenho, Segurança e Interrupção**, e os dados apontam para a necessidade de reforço da equipa às **Quartas-feiras e Domingos**, bem como no mês de **Março**, que historicamente concentra o maior volume de tickets. O projecto foi desenvolvido inteiramente no **Power BI**, com tratamento avançado de dados em **Power Query (Linguagem M)** e um conjunto robusto de **medidas DAX**, incluindo cálculo de MTTR e SLA Compliance Rate.

## Problema do Negócio

Uma equipa de suporte de TI necessitava de compreender melhor a distribuição e o comportamento dos seus tickets de suporte entre 2024 e 2025, com o objectivo de identificar os principais riscos operacionais, avaliar a estabilidade da infraestrutura, perceber as causas das paragens técnicas e planear de forma eficaz a escala da equipa.

O Gestor levantou as seguintes questões organizadas em três visões analíticas:

**Visão Geral**
1. Onde está o nosso maior risco operacional?
2. A nossa infraestrutura está a tornar-se mais instável?
3. O que está a causar a maioria das paragens técnicas?

**Visão Performance**

4. Como devo planear a escala da minha equipa?

**Visão das Tags**

5. Quais são as etiquetas mais associadas aos tickets de maior prioridade?

---

## Contexto

A base de dados utilizada contém registos de tickets de suporte de TI do período de **2024 a 2025**, cobrindo diferentes tipos de incidentes, categorias, prioridades e etiquetas técnicas. Como o dataset original era estático, foi criado um **motor de geração de histórico temporal** em Power Query com Linguagem M, simulando um cenário realista de dois anos de operação. A análise foi organizada em três visões distintas para facilitar a leitura e a tomada de decisão pela equipa de gestão.

> **Fonte dos dados:** [Kaggle](https://www.kaggle.com/datasets/tobiasbueck/multilingual-customer-support-tickets)

---

## Premissas da Análise

- Os dados foram tratados e modelados no **Power BI**.
- O processo de preparação de dados no **Power Query** foi extenso e focado em garantir a integridade dos cálculos e a criação de um cenário realista para análise, conforme descrito abaixo.

---

### Transformações no Power Query

#### 1 — Limpeza e Padronização de Texto
Para evitar categorias duplicadas por erros de digitação ou espaços invisíveis, foram aplicadas as seguintes transformações nas colunas `type`, `queue`, `priority`, `language` e `version`:

- **Aparar (Trim):** Remoção de espaços em branco no início e no fim dos textos.
- **Limpar (Clean):** Remoção de caracteres não imprimíveis como quebras de linha.
- **Capitalização:** Padronização para que variações como *"Alta"* e *"alta"* fossem lidas como a mesma categoria.

#### 2 — Criação do Identificador Único (Chave Primária)
Foi adicionada uma **Coluna de Índice** iniciando em 1, antes do Unpivot das tags. Este passo foi essencial para que cada ticket recebesse um ID único que se repetiria em várias linhas após o desdobramento das etiquetas, viabilizando o uso da função `DISTINCTCOUNT` no Power BI.

#### 3 — Motor de Geração de Histórico (Simulação 2024–2025)
Como o dataset era estático, foi criada uma inteligência temporal com colunas personalizadas em **Linguagem M**:

- **Coluna `opening_date`** (Data de Abertura):
```
#datetime(2024, 1, 1, 0, 0, 0) + #duration(
    Number.Round(Number.RandomBetween(0, 729), 0),
    Number.Round(Number.RandomBetween(0, 23), 0),
    Number.Round(Number.RandomBetween(0, 59), 0),
    0
)
```

- **Coluna `closing_date`** (Data de Fechamento): Criada com base na abertura, somando uma duração aleatória realista entre 1h e 72h:
```
[opening_date] + #duration(
    0,
    Number.Round(Number.RandomBetween(1, 72), 0),
    Number.Round(Number.RandomBetween(0, 59), 0),
    0
)
```

#### 4 — Optimização para Modelagem (reference_date)
A coluna `opening_date` foi duplicada e o tipo de dados alterado para apenas **Data** (removendo as horas), criando a coluna `reference_date`. Esta "ponte" limpa eliminou erros de valores *(Vazio)* nos filtros de ano e mês, preservando as horas originais na coluna principal para os cálculos de MTTR.

#### 5 — Normalização de Tags (Unpivot)
O dataset original continha **8 colunas de tags** (Tags_1, Tags_2, ..., Tags_8) numa estrutura horizontal. Foi realizado o **Unpivot de outras colunas** para transformar essa estrutura em formato vertical:

- **Colunas fixas:** `Índice`, `subject`, `body`, `opening_date`, `closing_date`, `reference_date`, `type`, `queue`, `priority`, `version`
- **Resultado:** O dataset expandiu de **24.749 para aproximadamente 116.000 linhas**, onde cada linha representa uma associação única **Ticket ↔ Tag**

#### 6 — Tratamento de Nulos e Refinação Final
- **Tags nulas:** Remoção automática de linhas onde a tag era `null` ou vazia, resultado do Unpivot de colunas em branco.
- **Renomeação:** A coluna `Valor` foi renomeada para `Tags` e a coluna `Atributo` foi removida.
- **Substituição de nulos:** Nas colunas `priority` e `type`, os valores `null` foram substituídos por *"Não Definido"* para garantir que 100% dos dados fossem categorizados nos filtros.

#### 7 — Tipagem de Dados
Garantia de que cada coluna possui o formato correcto para o motor do Power BI:

| Coluna | Tipo de Dado |
|--------|-------------|
| `Índice` | Número Inteiro |
| `opening_date` | Data/Hora |
| `closing_date` | Data/Hora |
| `reference_date` | Data |
| Colunas de categorias e tags | Texto |

---

### Tabela Calendário (DAX)
Foi criada uma tabela `Calendário` com DAX contendo as seguintes colunas:

`Date`, `Ano`, `Mês Nome`, `Mês Num`, `Mês Ano`, `Mês Ano Sort`, `Weekday`, `Weekday Num`

---

### Medidas DAX

Todas as medidas foram centralizadas numa tabela dedicada `_Medidas` para facilitar a organização e manutenção do modelo:

**Total Tickets**
```dax
Total Tickets = DISTINCTCOUNT(Fato_Tickets[ID_Ticket])
```

**Tickets Alta Prioridade**
```dax
Tickets Alta Prioridade = CALCULATE(
    [Total Tickets],
    'Dim_Prioridade'[priority] = "Alta"
)
```

**AVG Resolution Time (Horas)**
```dax
AVG Resolution time =
AVERAGEX(
    VALUES(Fato_Tickets[ID_Ticket]),
    CALCULATE(
        DATEDIFF(
            MIN(Fato_Tickets[opening_date]),
            MIN(Fato_Tickets[closing_date]),
            MINUTE
        ) / 60
    )
)
```

**AVG Resolution Time (Dias)**
```dax
Avg Resolution Time (Days) = [AVG Resolution time] / 24
```

**SLA Compliance Rate**
```dax
SLA Compliance Rate =
VAR HorasMeta = 36
VAR TicketsNoPrazo =
    COUNTROWS(
        FILTER(
            VALUES(Fato_Tickets[ID_Ticket]),
            CALCULATE(DATEDIFF(MIN(Fato_Tickets[opening_date]), MIN(Fato_Tickets[closing_date]), HOUR)) <= HorasMeta
        )
    )
RETURN
DIVIDE(TicketsNoPrazo, [Total Tickets], 0)
```

**SLA Breach Rate**
```dax
SLA Breach Rate = 1 - [SLA Compliance Rate]
```

**Top Month Tickets**
```dax
Top Month Tickets =
VAR ResumoMensal =
    SUMMARIZE(
        'Calendario',
        'Calendario'[Mês Ano],
        "Volume", [Total Tickets]
    )
VAR Top1 = TOPN(1, ResumoMensal, [Volume], DESC)
RETURN
MAXX(Top1, 'Calendario'[Mês Ano])
```

**Top Volume Tickets**
```dax
Top Volume Tickets =
MAXX(
    ALLSELECTED('Calendario'[Mês Ano]),
    [Total Tickets]
)
```

**Total de Tags**
```dax
Total de Tags = DISTINCTCOUNT(Dim_Tags[Tags])
```

---

## Estratégia da Solução

### Passo 1 — Resumo do Contexto em Pergunta Aberta
> *Qual é o estado operacional do suporte de TI e onde estão os maiores riscos para a continuidade do serviço?*

### Passo 2 — Transformação em Perguntas Fechadas
> - Qual é a distribuição de tickets por prioridade e onde está a maior exposição operacional?
> - O volume e o tempo de resolução estão a piorar ao longo do tempo?
> - Quais categorias técnicas concentram mais falhas críticas?
> - Em que dias e meses a equipa está mais sobrecarregada?

### Passo 3 — Definição da Tabela Facto
A tabela facto central é a **Fato_Tickets**, que regista cada ticket de suporte como uma transacção individual. As suas métricas principais são o **ID_Ticket** — contabilizado via `DISTINCTCOUNT` como unidade base de toda a análise — e as datas de `opening_date` e `closing_date`, que permitem calcular o **AVG Resolution Time**, a métrica de desempenho central do projecto.
 
| Coluna | Papel na Análise |
|--------|-----------------|
| `ID_Ticket` | Unidade de contagem base (`DISTINCTCOUNT`) |
| `opening_date` | Início do cálculo de MTTR e SLA |
| `closing_date` | Fim do cálculo de MTTR e SLA |
| `reference_date` | Ligação com a tabela Calendário |
| `id_category` | Chave estrangeira → `Dim_Categoria` |
| `id_type` | Chave estrangeira → `Dim_Tipo` |
| `id_tags` | Chave estrangeira → `Dim_Tags` |
| `id_priority` | Chave estrangeira → `Dim_Prioridade` |
 
### Passo 4 — Identificação das Dimensões
 
| Tabela de Dimensão | Coluna de Análise | Descrição |
|-------------------|------------------|-----------|
| `Dim_Prioridade` | `priority` | Nível de prioridade do ticket |
| `Dim_Tipo` | `type` | Tipo de incidente registado |
| `Dim_Categoria` | `category` | Categoria técnica do ticket |
| `Dim_Tags` | `Tags` | Etiquetas técnicas associadas |
| `Calendário` | `Ano`, `Mês Nome`, `Weekday` | Análise temporal dos tickets |

### Passo 5 — Hipóteses Analíticas

- H1: A maior parte dos tickets é de Alta Prioridade, representando o maior risco operacional.
- H2: O volume de tickets aumentou de 2024 para 2025, indicando instabilidade crescente da infraestrutura.
- H3: Falhas de Desempenho, Segurança e Interrupção são as principais causas das paragens técnicas.
- H4: Determinados dias da semana e meses do ano concentram mais tickets e maiores tempos de resolução.

### Passo 6 — Critérios de Priorização

As hipóteses foram priorizadas com base em dois critérios:

- **Impacto Operacional** — quanto a hipótese afecta directamente a continuidade do serviço.
- **Relevância para Gestão** — quanto o insight orienta decisões sobre recursos humanos e infraestrutura.

### Passo 7 — Priorização das Hipóteses Analíticas

| Prioridade | Hipótese | Justificativa |
|------------|----------|---------------|
| Alta | H1 — Distribuição por prioridade | Identifica directamente onde está a maior exposição ao risco |
| Alta | H4 — Padrão temporal de sobrecarga | Orienta o planeamento de escala da equipa |
| Média | H3 — Causas das paragens técnicas | Direciona o investimento em prevenção de falhas |
| Baixa | H2 — Evolução do volume 2024 vs. 2025 | Avalia a tendência de estabilidade da infraestrutura |

---

## Insights da Análise

### Visão Geral

**Maior Risco Operacional**
O maior risco está na elevada carga de tickets de **Alta Prioridade**, que representa **40,32% do total**, correspondendo a **9.655 tickets**. Este volume exige atenção imediata e critérios rigorosos de triagem para evitar impactos críticos na operação.

**Estabilidade da Infraestrutura**
Não houve aumento de volume entre 2024 e 2025, o que indica **estabilidade de demanda**. No entanto, o **tempo médio de resolução de 47,88 horas** é considerado alto, sinalizando que o problema não é a quantidade de tickets, mas a **eficiência na resolução** dos mesmos.

**Causas das Paragens Técnicas**
As principais causas das paragens técnicas são as falhas de **Desempenho**, **Segurança** e **Interrupção**, que concentram a maioria dos incidentes críticos registados no período.

### Visão Performance

**Planeamento de Escala da Equipa**
Os dados revelam padrões claros de sobrecarga temporal que devem orientar o planeamento de recursos:

- **Reforço obrigatório** às **Quartas-feiras** e **Domingos**, dias que apresentam o maior tempo médio de resolução, superior a 2 dias.
- **Março** é o mês de maior pressão histórica, conforme evidenciado pelo heatmap de volume de tickets.

### Visão das Tags

A análise das etiquetas técnicas permite identificar os temas e componentes mais recorrentes nos tickets, orientando decisões de investimento em formação da equipa e melhoria de infraestrutura.

---

## Resultados

<img width="1313" height="740" alt="Visão Geral" src="https://github.com/user-attachments/assets/b3025845-364b-4164-af8b-c95e5f87ac54" />
<em>Figura 1: Visão Geral</em>
<br>

<img width="1318" height="737" alt="Visão1 Suporte" src="https://github.com/user-attachments/assets/4308bddd-4094-4c94-ae7a-593b340fd9a4" />
<em>Figura 2: Visão Performance</em>
<br>

<img width="1314" height="742" alt="Visão Tags" src="https://github.com/user-attachments/assets/9ada51cc-3388-426a-a075-51168f163aff" />
<em>Figura 3: Visão Tags</em>
<br>

Com base na análise, é possível concluir que:

- O maior risco operacional está nos **40,32% de tickets de Alta Prioridade** — a equipa deve implementar um processo de triagem mais rigoroso para gerir este volume.
- O problema central não é a instabilidade da infraestrutura, mas sim a **baixa eficiência de resolução** — com 47,88 horas de média, há margem significativa para optimização dos processos internos.
- O investimento em **prevenção de falhas de Desempenho, Segurança e Interrupção** terá o maior impacto na redução de paragens técnicas.
- A equipa deve ser **reforçada às Quartas-feiras e Domingos** e deve ter capacidade adicional reservada para o mês de **Março**, que historicamente concentra o maior volume de tickets.

---

## Estrutura do Projecto

```
analise-de-tickets-de-suporte-ti
│
├── dados
│   └── aa_dataset-tickets-multi-lang-5-2-50-version.csv    # Base de dados original
├── analise
│   └── analise_de_ticket_suporte_ti.pbix                   # Ficheiro Power BI com análises e dashboard
└── README.md                                               # Documentação do projecto
```

---

## Ferramentas Utilizadas

- **Power BI** — Para tratamento e transformação de dados via Power Query (Linguagem M), modelagem relacional, criação de medidas DAX, visualizações interactivas e Dashboard

---

## Autor

**Santiago Casseca**
[LinkedIn](www.linkedin.com/in/santiago-casseca)
