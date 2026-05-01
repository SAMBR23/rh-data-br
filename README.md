# 🏢 ACME Corp – Bases de RH (Teste para Engenheiro de Dados)

Conjunto completo de bases fictícias de RH simulando um cenário multi-fonte real, com **dados sujos propositais**, **formatos heterogêneos** (JSON, CSV, XML), **snapshots para SCD2/CDC** e **gabarito de KPIs em SQL**.

**Cenário:** A ACME Corp tem ~200 funcionários e seus dados de RH estão espalhados em múltiplas fontes (ERP, sistema de ponto, plataforma de ATS, sistema legado, planilhas). O time de Dados precisa consolidar isso em um Lakehouse no **Microsoft Fabric**.

---

## 📦 Arquivos disponíveis

### 🥉 Camada Bronze — Fontes principais (JSON)

| Arquivo | Descrição | Chave principal | Observações |
|---|---|---|---|
| `employees.json` | Cadastro mestre de funcionários | `employee_id` | ⚠️ Contém 2 duplicatas e nulls |
| `departments.json` | Departamentos | `department_id` | ⚠️ Wrapper com metadata |
| `positions.json` | Cargos e faixas salariais | `id_cargo` | ⚠️ Campos em **português** |
| `payroll.json` | Folha de pagamento (12 meses de 2024) | `payroll_id` | ⚠️ FK chamada `emp_id`; valores negativos |
| `time_tracking.json` | Registros de ponto (60 dias) | `employee_id` + `date` | ⚠️ **Estrutura aninhada** |
| `performance_reviews.json` | Avaliações de desempenho | `review_id` | — |
| `training.json` | Treinamentos | `training_id` | ⚠️ Datas em formato BR |
| `benefits.json` | Benefícios por funcionário | `id_funcionario` | ⚠️ Aninhado e em PT |
| `recruitment.json` | Candidatos e processos seletivos | `candidate_id` | Link opcional com `hired_employee_id` |
| `terminations.json` | Desligamentos | `termination_id` | ⚠️ Datas em formato **US** |
| `absences.json` | Férias, atestados e faltas | `absence_id` | — |
| `engagement_survey.json` | Pesquisa de engajamento Q3/2024 | `response_id` | ⚠️ `submitted_at` em **timestamp Unix** |

### 🆕 Fontes em formatos heterogêneos

| Arquivo | Formato | Descrição | Desafios |
|---|---|---|---|
| `org_structure.csv` | **CSV** | Hierarquia organizacional (BU, divisão, time, cadeia de reporte) | ⚠️ Separador `;`, campos com espaço, datas formato `dd-Mon-yyyy` |
| `legacy_employees.xml` | **XML** | 30 funcionários do sistema antigo (ex-funcionários) | ⚠️ Tags em PT, histórico salarial aninhado, alguns com `CurrentEmployeeRef` |

### 🔄 Snapshots para SCD2 / CDC

| Arquivo | Descrição | Mudanças vs base |
|---|---|---|
| `employees_2025_01.json` | Snapshot do cadastro em 31/jan/2025 | 5 promoções, 3 transferências, 2 contratações, 2 desligamentos |
| `payroll_2025_01.json` | Folha de janeiro/2025 | Dissídio de 8% aplicado a 80% dos funcionários |
| `_changelog_2025_01.json` | **Ground truth** das mudanças (referência para validação) | Lista exata das 11 mudanças |

### 📚 Documentação e gabarito

| Arquivo | Descrição |
|---|---|
| `README.md` | Este arquivo (visão geral, modelo, desafios, arquitetura sugerida) |
| `solutions_kpis_sql.md` | **Gabarito** com 12 queries SQL dos KPIs principais + validações de qualidade |

---

## 🔗 Modelo de relacionamento (chaves)

```
employees (employee_id) ──┬── department_id ──> departments
                          ├── position_id ──> positions
                          ├── manager_id ──> employees (auto-relacionamento)
                          │
                          ├──< payroll (emp_id)            ⚠️ nome diferente
                          ├──< time_tracking (employee_id)
                          ├──< performance_reviews (employee_id)
                          ├──< training (employee_id)
                          ├──< benefits (id_funcionario)   ⚠️ nome diferente
                          ├──< terminations (employee_id)
                          ├──< absences (employee_id)
                          ├──< engagement_survey.responses (employee_id)
                          ├──< org_structure (Employee ID) ⚠️ campo com espaço
                          └──< legacy_employees (CurrentEmployeeRef) — opcional

recruitment.hired_employee_id ──> employees.employee_id (opcional)
employees_2025_01 ──> compara com employees (SCD2)
payroll_2025_01 ──> append em fact_payroll (CDC)
```

---

## 🧪 Desafios propositais (dados sujos)

O candidato deve identificar e tratar:

1. **Duplicatas** em `employees.json` (2 registros: caixa diferente, espaço em email)
2. **Nomes de chaves inconsistentes:**
   - `employee_id` (padrão), `emp_id` (payroll), `id_funcionario` (benefits), `Employee ID` (CSV), `legacy_id` (XML)
3. **Formatos de data variados:**
   - ISO 8601 (`2024-03-15`)
   - Brasileiro (`15/03/2024`)
   - Americano (`03-15-2024`)
   - Timestamp Unix (`1726401234`)
   - Mixed (`02-Nov-2024` no CSV)
4. **Formatos de arquivo diferentes:** JSON, JSON aninhado, CSV (separador `;`), XML
5. **Idiomas misturados** (alguns campos em PT, outros em EN)
6. **Estruturas aninhadas** (time_tracking, benefits, engagement_survey, XML history)
7. **Wrapper com metadata** em `departments.json` e `employees_2025_01.json`
8. **Valores negativos suspeitos** em `net_salary`
9. **FKs opcionais** entre legacy_employees e employees atuais
10. **SCD2 / CDC**: comparar dois snapshots para detectar mudanças

---

## 🏗️ Arquitetura sugerida (Medallion no Fabric)

### 🥉 BRONZE — Raw Layer
- Ingerir os 17 arquivos **como estão**, sem transformações
- Para cada formato, usar o conector apropriado:
  - **JSON**: REST API ou file source
  - **CSV**: file source com inferência de schema
  - **XML**: leitura via `spark.read.format("xml")` ou parsing manual
- Adicionar colunas de auditoria: `_ingestion_timestamp`, `_source_file`, `_source_url`
- Salvar em formato **Delta** preservando estrutura original

### 🥈 SILVER — Cleaned & Conformed
- **Padronizar nomes de colunas** (snake_case em inglês)
- **Normalizar datas** para `DATE` ou `TIMESTAMP`
- **Deduplificar** `employees`
- **Achatar estruturas aninhadas** (time_tracking, benefits, engagement, XML)
- **Validar integridade referencial**
- **Tratar nulls e valores inválidos** (ex: `net_salary < 0` → flag)
- Aplicar **SCD Tipo 2** comparando snapshots de dezembro/2024 vs janeiro/2025
- **Unir** legacy_employees com employees atuais quando possível (via CurrentEmployeeRef)
- **Enriquecer** dim_employee com dados do org_structure.csv (BU, divisão, time, escritório)

**Tabelas Silver esperadas:**
- `silver.dim_employee` (com SCD2: valid_from, valid_to, is_current)
- `silver.dim_department`, `silver.dim_position`
- `silver.fact_payroll`, `silver.fact_time_tracking`
- `silver.fact_performance_review`, `silver.fact_training`
- `silver.fact_benefits`, `silver.fact_recruitment`
- `silver.fact_termination`, `silver.fact_absence`
- `silver.fact_engagement_response`
- `silver.dim_org_structure` (do CSV)
- `silver.fact_legacy_employee` (do XML)

### 🥇 GOLD — Business / Analytics Layer
Modelagem dimensional (star schema) pronta para Power BI:

**Dimensões:** `dim_employee`, `dim_department`, `dim_position`, `dim_date`, `dim_manager`

**Fatos / Marts:** `fact_headcount_monthly`, `fact_payroll_monthly`, `fact_turnover`, `fact_performance`, `fact_engagement`, `fact_absenteeism`, `fact_training_investment`, `fact_recruitment_funnel`

---

## 📊 KPIs principais (gabarito completo em `solutions_kpis_sql.md`)

| # | KPI | Resultado esperado |
|---|---|---|
| 1 | Headcount Ativo | ~175 funcionários |
| 2 | Headcount Mensal por Depto | série temporal |
| 3 | Turnover Anual | ~12-14% |
| 4 | Custo Médio por Funcionário | varia por mês |
| 5 | eNPS por Depto | varia |
| 6 | Taxa de Absenteísmo | ~3-5% |
| 7 | Time-to-Hire | ~30-90 dias |
| 8 | Taxa de Promoção | ~10-20% |
| 9 | Investimento em Treinamento per capita | varia |
| 10 | Funil de Recrutamento | conversão por etapa |
| 11 | Diversidade por Nível | distribuição % |
| 12 | Mudanças entre Snapshots (SCD2) | **~11 mudanças** |

---

## 🚀 Como consumir via API (Fabric)

URL raw padrão:
```
https://raw.githubusercontent.com/<usuario>/<repo>/main/<arquivo>
```

**Notebook PySpark — JSON:**
```python
import requests
import json

url = "https://raw.githubusercontent.com/SEU_USUARIO/rh-data-test/main/employees.json"
data = requests.get(url).json()
df = spark.createDataFrame(data)
df.write.mode("overwrite").format("delta").saveAsTable("bronze.employees_raw")
```

**Notebook PySpark — CSV:**
```python
df = (spark.read
      .option("header", "true")
      .option("delimiter", ";")
      .option("encoding", "UTF-8")
      .csv("https://raw.githubusercontent.com/SEU_USUARIO/rh-data-test/main/org_structure.csv"))
df.write.mode("overwrite").format("delta").saveAsTable("bronze.org_structure_raw")
```

**Notebook PySpark — XML (parsing manual):**
```python
import xml.etree.ElementTree as ET
import requests

url = "https://raw.githubusercontent.com/SEU_USUARIO/rh-data-test/main/legacy_employees.xml"
xml_text = requests.get(url).text
root = ET.fromstring(xml_text)

records = []
for emp in root.findall(".//Employee"):
    rec = {
        "legacy_id": emp.get("legacy_id"),
        "active": emp.get("active"),
        "nome": emp.find("Nome").text,
        "matricula": emp.find("Matricula").text,
        "data_admissao": emp.find("DataAdmissao").text,
        "data_desligamento": emp.find("DataDesligamento").text,
        "cargo": emp.find("Cargo").text,
        "departamento": emp.find("Departamento").text,
        "salario_final": int(emp.find("SalarioFinal").text),
        "current_employee_ref": emp.find("CurrentEmployeeRef").text 
            if emp.find("CurrentEmployeeRef") is not None else None
    }
    records.append(rec)

df = spark.createDataFrame(records)
df.write.mode("overwrite").format("delta").saveAsTable("bronze.legacy_employees_raw")
```

---

## ✅ Critérios de avaliação

### Básico (obrigatório)
- [ ] Ingestão funcional dos 17 arquivos (3 formatos: JSON, CSV, XML)
- [ ] Tratamento dos problemas documentados
- [ ] Modelagem Silver com integridade referencial
- [ ] **SCD Tipo 2** comparando os dois snapshots
- [ ] Modelagem Gold em star schema
- [ ] Pelo menos **8 dos 12 KPIs** calculados

### Intermediário
- [ ] Documentação das transformações (data lineage)
- [ ] Validações de qualidade implementadas
- [ ] Pipeline orquestrado e idempotente
- [ ] Dashboard no Power BI com 5+ visuais

### Avançado (bônus)
- [ ] CDC real (não apenas snapshot diff)
- [ ] Testes automatizados (Great Expectations ou similar)
- [ ] Schedule de atualização configurado
- [ ] Histórico salarial do XML legacy unificado com payroll atual

---

## 📝 Volumes resumidos

- **17 arquivos** totalizando ~3 MB
- ~200 funcionários ativos + 30 do sistema legado
- ~2.300 registros de folha (incluindo janeiro/2025)
- ~10.000 registros de ponto
- ~530 avaliações de desempenho
- ~530 registros de treinamento
- ~800 registros de absenteísmo
- ~350 candidatos
- ~130 respostas de pesquisa de engajamento
- ~11 mudanças de SCD2 entre snapshots
