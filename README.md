# aula-ml — Versionamento de Dados e Modelos com Git e DVC (ATP07)

Repositório da Atividade Prática 07 da disciplina **CK0444 – Engenharia de Sistemas Inteligentes (UFC)**. O projeto demonstra o versionamento conjunto de código, conjuntos de dados, partições de teste e artefatos de modelo utilizando **Git** e **DVC**, garantindo reprodutibilidade, rastreabilidade por hashes SHA-256 e avaliação comparativa em partição de teste fixa.

- Repositório remoto Git: `https://github.com/caiosaboia/aula-ml`
- Armazenamento remoto DVC padrão (`storage`): `../dvc-storage` (relativo à raiz do repositório; registrado em `.dvc/config` como `../../dvc-storage`)

## Pré-requisitos

- **Git** (para versionamento de código, metadados `.dvc`, manifestos e tags)
- **Python >= 3.10** (executável `python3` no Linux/macOS/WSL ou `python` no Windows; sem dependências externas no script de treino)
- **DVC 3** (`dvc`, para rastreamento e sincronização de dados e artefatos de modelo)

## Estrutura do Repositório

| Caminho | Função | Versionado por |
| :--- | :--- | :--- |
| `src/train.py` | Script de treinamento, avaliação e geração de manifestos | Git |
| `params.json` | Hiperparâmetros do experimento (`tokenizador`) | Git |
| `data/chamados.csv` | Conjunto de dados rotulado de chamados | DVC (`data/chamados.csv.dvc`) |
| `data/split.csv` | Partição fixa de treino/teste | DVC (`data/split.csv.dvc`) |
| `data/manifesto.json` | Metadados do dataset (`dataset_id`, esquema, regra de rotulagem e alterações) | Git |
| `models/model.json` | Artefato serializado do modelo treinado | DVC (`models/model.json.dvc`) |
| `reports/metrics.json` | Métricas de avaliação na partição de teste fixa | Git |
| `reports/modelo.json` | Manifesto de rastreabilidade (`run_id`, `git_commit` e hashes SHA-256) | Git |
| `reports/comparacao.md` | Relatório comparativo entre `chamados-r1` e `chamados-r2` e decisão de revisão | Git |
| `.dvc/config` | Configuração do armazenamento remoto DVC (`storage`) | Git |

## Resumo das Tags

| Tag | Commit da Tag | Descrição |
| :--- | :--- | :--- |
| `dados-v1` | `e81d82aef417fc81aba6f10c916730aa9705b920` | Primeira versão do dataset (`chamados-v1`) e partição fixa (`data/split.csv`) |
| `modelo-r1` | `26f08c8d962d6458613b2a18e694f20b54111695` | Modelo baseline (`chamados-r1`) treinado sobre `chamados-v1` (acurácia `0.75`, recall urgente `0.50`) |
| `dados-v2` | `e8c6e97f9bddec26661492bbeca18b603795f7a4` | Correção do rótulo do ID `006` de `comum` para `urgente` (`chamados-v2`), mantendo `data/split.csv` inalterado |
| `modelo-r2` | `49c53accbe1219db33ffacc98060a4a8809d8c8f` | Modelo candidato (`chamados-r2`) treinado sobre `chamados-v2` (acurácia `1.00`, recall urgente `1.00`) |

## Configuração do Armazenamento Remoto DVC

O remoto padrão `storage` já está configurado em `.dvc/config` apontando para o diretório irmão `../dvc-storage` em relação à raiz do repositório:

```ini
[core]
    remote = storage
['remote "storage"']
    url = ../../dvc-storage
```

Se o repositório for clonado no mesmo diretório pai onde existe `dvc-storage`, nenhuma reconfiguração é necessária. Caso o armazenamento `dvc-storage` esteja em outro caminho local, ajuste a URL antes de executar `dvc pull`:

```bash
dvc remote modify storage url /caminho/para/dvc-storage
```

No Windows (PowerShell), informe o caminho local correspondente:

```powershell
dvc remote modify storage url C:\caminho\para\dvc-storage
```

## Restauração de uma Versão em Clone Limpo e Verificação SHA-256

Para restaurar qualquer versão versionada (`modelo-r1` ou `modelo-r2`) em um diretório novo e validar a integridade criptográfica dos arquivos restaurados contra `reports/modelo.json`:

### 1. Clonar o repositório e selecionar a tag desejada

```bash
git clone https://github.com/caiosaboia/aula-ml.git aula-ml-restauracao
cd aula-ml-restauracao
git checkout modelo-r1
```

> Substitua `modelo-r1` por `modelo-r2` para restaurar a segunda versão do modelo.

### 2. Baixar os artefatos rastreados pelo DVC

```bash
dvc pull
```

### 3. Verificar os hashes SHA-256 contra o manifesto `reports/modelo.json`

**Linux / macOS / WSL (`python3`):**

```bash
python3 -c '
import hashlib, json, pathlib

manifesto = json.loads(pathlib.Path("reports/modelo.json").read_text(encoding="utf-8"))
checks = [
    ("data/chamados.csv", manifesto["dataset_sha256"]),
    ("data/split.csv", manifesto["split_sha256"]),
    (manifesto["artefato"], manifesto["artifact_sha256"]),
]

for caminho, esperado in checks:
    atual = hashlib.sha256(pathlib.Path(caminho).read_bytes()).hexdigest()
    assert atual == esperado, f"MISMATCH {caminho}: esperado={esperado} atual={atual}"
    print(f"OK {caminho} {atual}")
'
```

**Windows / PowerShell (`python`):**

```powershell
python -c "import hashlib, json, pathlib; m = json.loads(pathlib.Path('reports/modelo.json').read_text(encoding='utf-8')); [print(f'OK {p} {h}') if (h := hashlib.sha256(pathlib.Path(p).read_bytes()).hexdigest()) == e else (_ for _ in ()).throw(AssertionError(f'MISMATCH {p}: {h} != {e}')) for p, e in [('data/chamados.csv', m['dataset_sha256']), ('data/split.csv', m['split_sha256']), (m['artefato'], m['artifact_sha256'])]]"
```

### Saída esperada para `modelo-r1` (`HEAD = 26f08c8d962d6458613b2a18e694f20b54111695`)

```text
OK data/chamados.csv bc3fd86a3a5f598841a58839a9726d901fa7e8361ffe79c5b43873ad8a1328ac
OK data/split.csv 2ef820171c1a4625fe216e6da67c7c8dcf9d97a08288c0239cd23f1a6961d981
OK models/model.json 72a3a67936139a3473838a6dcfa7aae1acb4c23467be1ef1056d51d485b94840
```

## Relatório de Comparação

A análise detalhada das métricas, hashes de rastreabilidade e decisão de aceitação de `chamados-r2` encontra-se em `reports/comparacao.md`.
