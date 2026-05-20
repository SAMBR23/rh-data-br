# 🏢 SH Corp – Case | Data Engineer

**Cenário:** A SH Corp tem ~200 funcionários e seus dados de pessoas estão espalhados em múltiplas fontes. Você precisa consolidar tudo em um OneLake no **Microsoft Fabric**, aplicando a arquitetura Medallion.

Os dados estão em formatos e sistemas diferentes. Parte do trabalho é justamente explorar, entender e tratar o que você encontrar.

---

## 📦 Arquivos disponíveis

### Fontes principais

| Arquivo | Descrição |
|---|---|
| `employees.json` | Cadastro mestre de funcionários |
| `departments.json` | Departamentos |
| `positions.json` | Cargos e faixas salariais |
| `payroll.json` | Folha de pagamento (12 meses de 2024) |
| `time_tracking.json` | Registros de ponto (60 dias) |
| `performance_reviews.json` | Avaliações de desempenho |
| `training.json` | Treinamentos |
| `benefits.json` | Benefícios por funcionário |
| `recruitment.json` | Candidatos e processos seletivos |
| `terminations.json` | Desligamentos |
| `absences.json` | Férias, atestados e faltas |
| `engagement_survey.json` | Pesquisa de engajamento Q3/2024 |
| `org_structure.csv` | Hierarquia organizacional (BU, divisão, time, cadeia de reporte) |
| `legacy_employees.xml` | Funcionários do sistema legado |

### Snapshots

| Arquivo | Descrição |
|---|---|
| `employees_2025_01.json` | Snapshot do cadastro em 31/jan/2025 |
| `payroll_2025_01.json` | Folha de pagamento de janeiro/2025 |

---

## 🏗️ O que construir

### 🥉 Bronze
Ingira todos os arquivos como estão, sem transformações. Adicione colunas de auditoria (`_ingestion_timestamp`, `_source_file`) e salve em formato Delta.

### 🥈 Silver
Dados limpos, padronizados e integrados. Espera-se ao menos:

- Padronização de nomes de colunas e tipos de dados
- Tratamento de inconsistências encontradas
- Integração entre as fontes com integridade referencial
- Aplicação de **SCD Tipo 2** a partir dos snapshots disponíveis

### 🥇 Gold
Star schema pronto para consumo no Power BI, com dimensões e fatos adequados para os KPIs listados abaixo.

---

## 📊 KPIs esperados (Diferencial)

Calcule ao menos **8 dos 12**:

| # | KPI |
|---|---|
| 1 | Headcount Ativo |
| 2 | Headcount Mensal por Departamento |
| 3 | Turnover Anual |
| 4 | Custo Médio por Funcionário |
| 5 | eNPS por Departamento |
| 6 | Taxa de Absenteísmo |
| 7 | Time-to-Hire |
| 8 | Taxa de Promoção |
| 9 | Investimento em Treinamento per capita |
| 10 | Funil de Recrutamento (conversão por etapa) |
| 11 | Diversidade por Nível |
| 12 | Mudanças detectadas entre os dois snapshots (SCD2) |

---

## 📬 Entregáveis

Ao concluir o case, envie:

1. **Repositório** com todo o código produzido (notebooks, scripts de pipeline, queries)
2. **Camadas Bronze, Silver e Gold** implementadas e funcionais no Fabric
3. **Documentação** — um README ou doc explicando:
   - Quais inconsistências você encontrou nos dados e como tratou cada uma
   - Decisões de modelagem que tomou (e por quê)
   - O que ficou fora do escopo e o que faria diferente com mais tempo
4. **Dashboard no Power BI** com os KPIs calculados (ao menos 8 dos 12) | Diferencial

**Formato de entrega:** `Workspace Fabric + Doc`

**Prazo:** `24/05/26`

---

## ✅ Critérios de avaliação

### Básico (obrigatório)
- [ ] Ingestão de todos os arquivos (JSON, CSV e XML)
- [ ] Identificação e tratamento das inconsistências nos dados
- [ ] Modelagem Silver com integridade referencial
- [ ] SCD Tipo 2 aplicado corretamente
- [ ] Modelagem Gold em star schema


### Intermediário
- [ ] Documentação das transformações (data lineage)
- [ ] Validações de qualidade implementadas
- [ ] Pipeline orquestrado e idempotente


### Avançado (bônus)
- [ ] CDC real (não apenas snapshot diff)
- [ ] Testes automatizados (Great Expectations ou similar)
- [ ] Schedule de atualização configurado
- [ ] Histórico salarial do legado unificado com o payroll atual
- [ ] Pelo menos 8 dos 12 KPIs calculados e documentados
- [ ] Dashboard no Power BI com 5+ visuais

---

## 🚀 Como acessar os arquivos

URL base:
```
https://raw.githubusercontent.com/<usuario>/<repo>/main/<arquivo>
```

Os arquivos podem ser lidos diretamente em notebooks do Fabric via `requests` ou conectores nativos do Spark.

Sua credencial é:
