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
| **Silver** | Limpeza, padronização e aplicação de SCD | 🔄 Próxima fase |
| **Gold** | Agregações e visões analíticas finais | 🔄 Pendente |

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

## Estrutura do Projeto

```
mini_projeto/
├── dataset/                        # Arquivos de dados de origem
│   ├── filial-brick_sample.xlsx
│   └── MS_12_2022_sample.xlsx
├── 01_ingestion_bronze.ipynb       # Fase 1: Ingestão → Camada Bronze
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
| `pandas` | Leitura e pré-visualização dos datasets |
| `openpyxl` | Leitura de arquivos `.xlsx` |
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

Execute as células do notebook `01_ingestion_bronze.ipynb` em ordem sequencial:

| Célula | Ação |
|--------|------|
| 1 | Instalação de dependências |
| 2 | Importação de bibliotecas |
| 3 | Configurações do pipeline |
| 4 | Definição das funções utilitárias |
| 5 | Definição do cliente GCS |
| 6 | Definição das funções de upload |
| 7 | Descoberta dos arquivos locais |
| 8 | Pré-visualização dos dados |
| 9 | Conexão com o GCS |
| 10 | Upload para a camada Bronze |
| 11 | Upload do manifesto |
| 12 | Relatório final |
| 13 | Verificação dos arquivos no bucket |
