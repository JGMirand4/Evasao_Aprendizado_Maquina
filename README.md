# Predição de Evasão Escolar em Pernambuco
**Disciplina:** Aprendizagem de Máquina | **Entrega:** 11/05/2026

---

## O Problema

A evasão escolar é um dos maiores desafios da educação pública brasileira. Identificar **quais contextos educacionais concentram maior risco de abandono** permite que gestores públicos direcionem recursos e intervenções com mais precisão.

Este projeto treina modelos de classificação binária para prever se um dado contexto (município × localização × dependência administrativa) apresenta **alto risco de evasão**, usando dados reais do INEP de todos os 5.570 municípios brasileiros.

---

## Os Dados

### Dataset 1 — Taxas de Rendimento Escolar (`tx_rend_brasil_regioes_ufs_2024.xlsx`)

**Fonte:** [INEP — Taxas de Rendimento Escolar](https://www.gov.br/inep/pt-br/acesso-a-informacao/dados-abertos/indicadores-educacionais/taxas-de-rendimento-escolar)

Contém os percentuais anuais de aprovação, reprovação e abandono escolar para cada combinação de:

| Dimensão | Valores |
|---|---|
| Unidade geográfica | Brasil, 5 regiões e 27 UFs |
| Localização | Total, Urbana, Rural |
| Dependência administrativa | Total, Federal, Estadual, Municipal, Privada |
| Etapa de ensino | Ensino Fundamental (total, anos iniciais, anos finais) e Ensino Médio (total, 1ª, 2ª, 3ª série) |

**Dimensões após limpeza:** 488 linhas × 27 colunas de indicadores

---

### Dataset 2 — Microdados de Infraestrutura Escolar (`microdados_ed_basica_2024.csv`)

**Fonte:** [INEP — Censo Escolar da Educação Básica 2024](https://www.gov.br/inep/pt-br/acesso-a-informacao/dados-abertos/microdados/censo-escolar)

Cadastro completo de todas as escolas do Brasil com informações de infraestrutura física. O arquivo original tem **215.545 escolas × 426 colunas** (208 MB).

Para este projeto, foram selecionadas **23 colunas** de infraestrutura e localização, e os dados foram **agregados por UF × localização × dependência administrativa**, calculando o percentual de escolas com cada recurso em cada grupo.

| Recurso medido | Exemplo de coluna |
|---|---|
| Conectividade | `pct_internet`, `pct_banda_larga`, `pct_internet_alunos` |
| Pedagógico | `pct_biblioteca`, `pct_sala_leitura`, `pct_lab_ciencias`, `pct_lab_informatica` |
| Infraestrutura básica | `pct_energia_rede`, `pct_agua_potavel`, `pct_esgoto_rede`, `pct_banheiro` |
| Carências | `pct_sem_energia`, `pct_sem_agua`, `pct_sem_esgoto`, `pct_sem_acessibilidade` |
| Esporte e alimentação | `pct_quadra`, `pct_alimentacao` |

**Dimensões após agregação:** 399 linhas × 22 colunas de infraestrutura

---

### Dataset Integrado (`data/processed/dataset_integrado.csv`)

Os dois datasets são unidos pela chave `cod_municipio × localizacao × dependencia_adm` via `merge left`.

**Resultado final: 50.140 instâncias × 84 colunas** — 126× mais dados que a versão anterior por UF.

```
Dataset 2 (rendimento por município)    Dataset 2 (infraestrutura agregada)
Município × Localização × Dep.  →merge← Município × Localização × Dep.
abandono, reprovação...                  % internet, % biblioteca...
5.570 municípios × ~12 combos            5.570 municípios × ~12 combos
                      ↓
              50.140 instâncias
```

Cada linha representa um **contexto educacional municipal**: um município visto por uma combinação de localização (urbana/rural) e rede de ensino (municipal/estadual/privada). Pernambuco contribui com **185 municípios** (vs. 1 UF antes).

> **Referência legada:** `data/processed/dataset_uf_legado.csv` mantém a versão com 398 instâncias por UF para comparação metodológica.

---

## A Variável-Alvo: `risco_evasao`

A variável-alvo é binária e foi construída a partir da **taxa de abandono geral** (média entre abandono do Ensino Fundamental e do Ensino Médio):

```
risco_evasao = 1  se abandono_geral > percentil 75 nacional
risco_evasao = 0  caso contrário
```

| Classe | Instâncias | Interpretação |
|---|---|---|
| 0 — Baixo risco | 37.715 (75,2%) | Abandono abaixo do P75 nacional |
| 1 — Alto risco | 12.425 (24,8%) | Abandono acima do P75 nacional |

O dataset é **desbalanceado** (~3:1), o que reflete a realidade: a maioria dos contextos tem abandono moderado, e os de alto risco são a minoria que queremos identificar.

---

## O que buscamos no dataset

### Hipótese central
> Contextos com **pior infraestrutura** e **maior reprovação** concentram maior risco de abandono escolar.

### Hipóteses específicas

| # | Hipótese | Feature relevante |
|---|---|---|
| H1 | Maior reprovação aumenta risco de evasão | `reprov_fund_total`, `reprov_med_total` |
| H2 | Escolas rurais têm maior risco | `localizacao = Rural` |
| H3 | Rede municipal tem maior risco que estadual e privada | `dependencia_adm` |
| H4 | Ensino Médio tem risco maior que Fundamental | `reprov_med_*` vs `reprov_fund_*` |
| H5 | Menos internet e biblioteca → mais abandono | `pct_internet`, `pct_biblioteca` |
| H6 | Municípios rurais municipais concentram o maior risco | interação `Rural × Municipal` |

### Recorte em Pernambuco

Com a granularidade por município, Pernambuco passa de 15 linhas (versão por UF) para **185 municípios** no dataset, permitindo identificar quais localidades específicas concentram o maior risco.

| Município de PE | Abandono geral | Risco |
|---|---|---|
| Brejão | 3,00% | **Alto** |
| Santa Filomena | 2,80% | **Alto** |
| Santa Cruz | 2,75% | **Alto** |
| Exu | 2,70% | **Alto** |
| Itaquitinga | 2,55% | **Alto** |
| Recife | — | — |

A análise foca em quais municípios pernambucanos estão acima do P75 nacional e quais fatores de infraestrutura e reprovação explicam esse risco.

---

## Estrutura do projeto

```
├── data/
│   ├── raw/                  # dados brutos originais (INEP)
│   └── processed/            # dados gerados pelo pipeline
├── models/                   # modelos e artefatos treinados
├── notebooks/
│   ├── notebook1_definicao_coleta.ipynb       # contexto e fontes
│   ├── notebook2_limpeza_refinamento.ipynb    # limpeza e integração
│   ├── notebook3_eda.ipynb                    # análise exploratória
│   ├── notebook4_preprocessamento.ipynb       # features e split
│   └── notebook5_modelagem.ipynb             # modelos e avaliação
├── pyproject.toml
└── uv.lock
```

## Como executar

```bash
# Instalar dependências
uv sync

# Iniciar Jupyter
uv run jupyter notebook
```

Os notebooks devem ser executados em ordem (1 → 5). Cada um salva seus outputs em `data/processed/` para o próximo.
