# Comparação de Modelos: `chamados-r1` vs. `chamados-r2`

Relatório técnico de revisão comparativa referente à correção de rótulo no conjunto de dados de chamados (branch `correcao-rotulo` -> `main`).

## 1. Alteração Realizada em `v2` (`dados-v1` -> `dados-v2`)

A revisão de qualidade dos dados identificou inconsistência de rotulagem frente à regra definida em `data/manifesto.json` (*"Queda, falha crítica e bloqueio são urgentes. Cadastro e dúvida são comuns."*):

- **`data/chamados.csv`**: linha do registro `006` corrigida:
  - Antes (`chamados-v1`): `006,bloqueio de acesso,comum`
  - Depois (`chamados-v2`): `006,bloqueio de acesso,urgente`
- **`data/manifesto.json`**:
  - `dataset_id` atualizado de `chamados-v1` para `chamados-v2`.
  - Campo `alteracao` atualizado para `"Rótulo do ID 006 corrigido de comum para urgente."`.
- **`data/split.csv`**: mantido estritamente inalterado (`git diff dados-v1 dados-v2 -- data/split.csv.dvc` sem diferenças; `split_sha256` constante), preservando a comparabilidade direta na mesma partição de teste fixa.

## 2. Comparação de Métricas (Partição de Teste Fixa)

| Métrica | `chamados-r1` (`chamados-v1`) | `chamados-r2` (`chamados-v2`) | Variação (`r2 - r1`) | Suporte |
| :--- | :---: | :---: | :---: | :---: |
| Acurácia global | `0.75` (3/4) | `1.00` (4/4) | `+0.25` | 4 |
| Recall (`comum`) | `1.00` (2/2) | `1.00` (2/2) | `0.00` | 2 |
| Recall (`urgente`) | `0.50` (1/2) | `1.00` (2/2) | `+0.50` | 2 |

## 3. Rastreabilidade e Hashes SHA-256

| Atributo | `chamados-r1` | `chamados-r2` | Observação |
| :--- | :--- | :--- | :--- |
| `model_id` | `chamados-r1` | `chamados-r2` | Novo artefato gerado |
| `dataset_id` | `chamados-v1` | `chamados-v2` | Correção do ID `006` |
| Tag Git do modelo | `modelo-r1` | `modelo-r2` | Tags anotadas publicadas |
| Commit da tag | `26f08c8d962d6458613b2a18e694f20b54111695` | `49c53accbe1219db33ffacc98060a4a8809d8c8f` | Referência imutável no Git |
| `git_commit` (entrada do treino) | `dfc30821864c9352b231a0c047545e2e70098114` | `e8c6e97f9bddec26661492bbeca18b603795f7a4` | Registrado em `reports/modelo.json` |
| `run_id` | `20d4486c-41e1-49a2-9d66-ab5646bcf776` | `1628c2e9-d795-4fa5-a67e-74ab2d52684a` | Identificador único da execução |
| `dataset_sha256` | `bc3fd86a3a5f598841a58839a9726d901fa7e8361ffe79c5b43873ad8a1328ac` | `4ca2e29cb930a9161c2a851b3c71f5529dbb812db607ee4a5ea95d2223baa14d` | Alterado pela correção em `data/chamados.csv` |
| `split_sha256` | `2ef820171c1a4625fe216e6da67c7c8dcf9d97a08288c0239cd23f1a6961d981` | `2ef820171c1a4625fe216e6da67c7c8dcf9d97a08288c0239cd23f1a6961d981` | **Constante** (mesma partição de teste) |
| `artifact_sha256` | `72a3a67936139a3473838a6dcfa7aae1acb4c23467be1ef1056d51d485b94840` | `7f9d94ca8dd86ccc8ad90b0839613c61a33e2168bd23bf083a4bbe1375f908ac` | Alterado pelo retreinamento de `models/model.json` |

## 4. Evidência de Restauração em Clone Limpo

A reprodutibilidade do versionamento Git + DVC foi validada em um clone limpo independente (`aula-ml-restauracao`), posicionado na tag `modelo-r1`:

- **Commit verificado (`HEAD`)**: `26f08c8d962d6458613b2a18e694f20b54111695`
- **Resultado da verificação SHA-256 contra `reports/modelo.json`**:

```text
OK data/chamados.csv bc3fd86a3a5f598841a58839a9726d901fa7e8361ffe79c5b43873ad8a1328ac
OK data/split.csv 2ef820171c1a4625fe216e6da67c7c8dcf9d97a08288c0239cd23f1a6961d981
OK models/model.json 72a3a67936139a3473838a6dcfa7aae1acb4c23467be1ef1056d51d485b94840
```

Todos os hashes dos artefatos recuperados via `dvc pull` corresponderam exatamente aos valores registrados no manifesto da tag `modelo-r1`.

## 5. Decisão de Revisão

**Decisão: ACEITAR `chamados-r2`.**

### Critério de aceitação

> Melhorar o recall da classe `urgente` sem reduzir o recall da classe `comum`, utilizando a mesma partição de teste fixa.

### Verificação do critério

1. **Melhoria no recall de `urgente`**: passou de `0.50` (1/2) em `chamados-r1` para `1.00` (2/2) em `chamados-r2` (`+0.50`).
2. **Preservação do recall de `comum`**: mantido em `1.00` (2/2) em ambas as versões (`0.00`).
3. **Integridade da partição de teste**: confirmada pelo hash `split_sha256` idêntico (`2ef820171c1a4625fe216e6da67c7c8dcf9d97a08288c0239cd23f1a6961d981`) e pelo `git diff` vazio em `data/split.csv.dvc`.
4. **Rastreabilidade e restauração**: hashes e restauração limpa validados sem divergências.

### Nota metodológica

O conjunto de avaliação utilizado nesta prática contém apenas **4 exemplos sintéticos** (suporte igual a 2 para `comum` e 2 para `urgente`). Em uma amostra desse porte, a alteração na predição de um único exemplo provoca variação de 50 pontos percentuais no recall da respectiva classe e de 25 pontos percentuais na acurácia global. Assim, embora `chamados-r2` atenda integralmente ao critério funcional e metodológico da Atividade Prática 07, os valores absolutos (`1.00`) não permitem inferir desempenho estatístico ou generalização em dados reais de produção sem uma ampliação representativa da base de teste.
