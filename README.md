# Mini-Projeto ETL: Processamento de Dados IQVIA com SCD

Pipeline de ETL na nuvem para processamento de dados do setor farmacêutico (Dataset IQVIA), aplicando técnicas de **Slowly Changing Dimension (SCD)** para rastreamento histórico de mudanças em produtos.

---

## Contexto

Empresas farmacêuticas precisam rastrear o histórico de mudanças em seus produtos. Quando o preço ou a categoria de um medicamento muda, o valor anterior não pode ser simplesmente sobrescrito — é necessário preservar o histórico para análises temporais e auditorias. Este projeto aplica SCD para resolver esse problema.

---

## Arquitetura — Medallion (Bronze / Silver / Gold)

```
dataset/ (local)
    └── *.xlsx / *.csv / *.parquet
            │
            ▼
┌─────────────────────────────────────────────┐
│         Google Cloud Storage                │
│   Bucket: etl-iqvia-data-lake-augusto       │
│                                             │
│  bronze/iqvia/<YYYY>/<MM>/<DD>/             │  ← Dados brutos (sem transformação)
│  silver/iqvia/                              │  ← Dados limpos e padronizados
│  gold/iqvia/                                │  ← Dados agregados para consumo
└─────────────────────────────────────────────┘
```

| Camada | Descrição | Status |
|--------|-----------|--------|
| **Bronze** | Dados brutos conforme recebidos, sem transformação | ✅ Concluído |
| **Silver** | Limpeza, padronização e aplicação de SCD | ✅ Concluído |
| **Gold** | SCD Tipo 2 no BigQuery | ✅ Concluído |

---

## Fases do Projeto

### Fase 1 — Ingestão (Bronze)
> Notebook: `01_ingestion_bronze.ipynb`

- Descoberta automática dos arquivos em `dataset/` (`.xlsx`, `.csv`, `.parquet`)
- Pré-visualização dos dados brutos para validação
- Upload para o GCS com particionamento por data (`YYYY/MM/DD`)
- Cálculo de hash MD5 para verificação de integridade
- Geração de manifesto de ingestão (`_manifest.json`) para rastreabilidade

---

### Fase 2 — Transformação (Silver)
> Notebook: `02_transformation_silver.ipynb`

- Leitura dos arquivos `.xlsx` diretamente da camada Bronze no GCS
- Renomeação de colunas para snake_case padronizado
- Extração do código numérico do campo `BRICK`
- Preenchimento de nulos em colunas de volume com `0` (ausência de venda)
- Conversão de tipos (int, float, string)
- Remoção de duplicatas
- Enriquecimento via join entre tabela de vendas e mapeamento de filiais
- Adição de coluna de auditoria `dt_processamento`
- Salvamento em formato **Parquet** na camada Silver do GCS

---

### Fase 3 — Carga com SCD Tipo 2 (Gold / BigQuery)
> Notebook: `03_load_gold_scd.ipynb`

- Leitura do Parquet `vendas_enriquecido.parquet` da camada Silver
- Consolidação dos volumes por produto (agrupamento por EAN)
- Criação automática da tabela `dim_produto_scd2` no BigQuery (se não existir)
- Lógica SCD Tipo 2:
  - **Produto novo** → inserção com `flag_ativo = TRUE` e `data_fim_validade = NULL`
  - **Valor alterado** → encerramento do registro anterior (`flag_ativo = FALSE`, `data_fim_validade = hoje`) + inserção da nova versão
  - **Sem alteração** → ignorado
- Geração de `sk_produto` (Surrogate Key SHA256)
- Relatório final com contagem de inserções, encerramentos e registros sem alteração

---

## Estrutura do Projeto

```
mini_projeto/
├── dataset/                        # Arquivos de dados de origem
│   ├── filial-brick_sample.xlsx
│   └── MS_12_2022_sample.xlsx
├── 01_ingestion_bronze.ipynb       # Fase 1: Ingestão → Camada Bronze
├── 02_transformation_silver.ipynb  # Fase 2: Transformação → Camada Silver
├── 03_load_gold_scd.ipynb          # Fase 3: Carga SCD Tipo 2 → BigQuery Gold
├── 04_simulation_scd_changes.ipynb # Simulação: mudanças para teste do SCD2
├── requirements.txt                # Dependências do projeto
├── venv/                           # Ambiente virtual Python
└── README.md
```

---

## Pré-requisitos

- Python 3.11+
- [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) instalado e configurado
- Conta GCP com acesso ao bucket `etl-iqvia-data-lake-augusto`

---

## Configuração do Ambiente

**1. Criar e ativar a venv:**
```powershell
D:\Python\python.exe -m venv venv
.\venv\Scripts\Activate.ps1
```

**2. Instalar dependências:**
```powershell
pip install -r requirements.txt
```

**3. Autenticar no Google Cloud:**
```powershell
gcloud auth application-default login
gcloud auth application-default set-quota-project proc-de-dados-iqvia-com-scd
```

**4. Selecionar o kernel no notebook:**  
No VS Code, selecione o kernel **"Python (mini_projeto)"** no canto superior direito do notebook.

---

## Dependências

| Pacote | Uso |
|--------|-----|
| `google-cloud-storage` | Conexão e operações com o GCS |
| `google-cloud-bigquery` | Conexão e operações com o BigQuery |
| `db-dtypes` | Suporte a tipos BigQuery no pandas |
| `pandas` | Leitura e transformação dos datasets |
| `openpyxl` | Leitura de arquivos `.xlsx` |
| `pyarrow` | Serialização Parquet para camada Silver/Gold |
| `ipykernel` | Integração da venv com Jupyter/VS Code |

---

## Dataset

| Arquivo | Descrição |
|---------|-----------|
| `MS_12_2022_sample.xlsx` | Dados de vendas IQVIA — Dezembro/2022 |
| `filial-brick_sample.xlsx` | Mapeamento de filiais e regiões (brick) |

---

## Estrutura do Bucket GCS

```
gs://etl-iqvia-data-lake-augusto/
└── bronze/
    └── iqvia/
        └── <YYYY>/
            └── <MM>/
                └── <DD>/
                    ├── MS_12_2022_sample.xlsx
                    ├── filial-brick_sample.xlsx
                    └── _manifest.json        ← Manifesto de ingestão (auditoria)
```

O manifesto `_manifest.json` contém, para cada arquivo carregado:
- Nome e caminho no GCS
- Tamanho em MB
- Hash MD5
- Timestamp de carga
- Status do upload

---

## Como Executar

Execute os notebooks **em ordem**, rodando todas as células sequencialmente em cada um:

| Ordem | Notebook | Descrição |
|-------|----------|-----------|
| 1 | `01_ingestion_bronze.ipynb` | Lê os arquivos locais, faz upload para o GCS (Bronze) e gera o manifesto |
| 2 | `02_transformation_silver.ipynb` | Lê o Bronze, aplica limpeza e enriquecimento, salva Parquet no Silver |
| 3 | `03_load_gold_scd.ipynb` | Lê o Silver e popula a tabela `dim_produto_scd2` no BigQuery com SCD Tipo 2 |
| 4 _(opcional)_ | `04_simulation_scd_changes.ipynb` | Gera um arquivo Silver modificado para demonstrar o SCD2 em ação |

> **Dica:** Após rodar o notebook 4, execute novamente o notebook 3 apontando `SILVER_FILE_NAME = "vendas_enriquecido_v2.parquet"` para ver as alterações e exclusões lógicas sendo processadas.
