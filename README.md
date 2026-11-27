# dataops-curso-historia

Projeto prático do curso **Engenharia de Dados: DataOps, CI/CD e Liderança Técnica** (Alura), com Bruno Corrêa.

Ao longo das aulas, um pipeline de dados de uma loja (pedidos, clientes e produtos) sai do nível 1 de maturidade, com scripts que funcionam mas sem testes nem automação, e chega a um projeto com testes, CI/CD, padrões de código e métricas.

## O caso

Na reunião de resultados, o relatório mostrou um faturamento de outubro diferente do que o financeiro fechou, e ninguém soube explicar de onde veio o número. Você é o engenheiro de dados da loja: ao longo das aulas, vai investigar o erro, corrigir e garantir que ele não volte.

## Roteiro de gravação · comandos de todas as aulas

Esta é a cópia de gravação. Aqui estão os comandos das 6 aulas, na ordem. Cada branch também tem no README só os comandos da própria aula.

- [Aula 1 · Princípios de DataOps](#aula-01-inicio)
- [Aula 2 · Testes automatizados](#aula-02-testes)
- [Aula 3 · CI/CD para pipelines de dados](#aula-03-ci-cd)
- [Aula 4 · Padrões técnicos e code review](#aula-04-padroes)
- [Aula 5 · Mentoria e liderança técnica](#aula-05-lideranca)
- [Aula 6 · Medindo a transformação DataOps](#aula-06-metricas)

<a name="aula-01-inicio"></a>

### Aula 1 · Princípios de DataOps

Roteiro da aula: copie um bloco de cada vez (botão de copiar no canto do bloco), cole no terminal do Codespace e confira o resultado. O comando `code` abre o arquivo no editor.

#### 1.4 · O ambiente no Codespaces

**1. Criar o ambiente.** No GitHub, escolha a branch `aula-01-inicio` e clique em **Code → Codespaces → Create codespace**. Espere o terminal aparecer.

**2. Ver a receita do ambiente.** Mostra a imagem com Python 3.12, o comando que instala as dependências e a variável que faz o dbt achar o `profiles.yml`.

```bash
code .devcontainer/devcontainer.json
```

**3. Ver as versões fixadas.** Cada ferramenta tem a versão travada, para todo mundo ter o mesmo resultado.

```bash
code requirements.txt
```

**4. Conferir as versões instaladas.** Mostra a versão do Python e do dbt que o ambiente instalou.

```bash
python --version
dbt --version
```

Resultado esperado: `Python 3.12` e `dbt 1.12.6`. Se aparecer um aviso de versão mais nova, tudo bem: a versão foi fixada de propósito.

**5. Ao terminar de estudar.** Pare o codespace no menu **Codespaces → Stop**, para não gastar a cota gratuita.

#### 1.5 · Régua de maturidade

O faturamento chegou errado na reunião e ninguém sabe por quê. Antes de mexer em qualquer coisa, o engenheiro de dados investiga, nesta ordem: o número, os arquivos, o banco, a regra e os testes.

**1. Reproduzir o problema.** O primeiro comando carrega os CSVs no banco. O segundo constrói os modelos do dbt. O terceiro mostra o faturamento que foi para o relatório.

```bash
python -m src.ingestao
dbt run
python scripts/faturamento.py
```

Resultado esperado: `1000` clientes, `1000` produtos e `3000` pedidos carregados, `PASS=3` no dbt e `Faturamento de outubro: R$ 333.739,80 em 2546 pedidos`. O pipeline roda sem erro, mas o financeiro fechou `R$ 315.488,80`.

**2. Olhar os arquivos.** O que chegou da origem: 3 mil pedidos de outubro. Não dá para conferir no olho, por isso o próximo passo é perguntar ao banco.

```bash
code data/pedidos.csv
```

**3. Olhar o banco.** O primeiro comando conta os pedidos de cada status na camada `raw`, do jeito que chegaram. O segundo mostra desde quando chega o status novo.

```bash
python scripts/consultar.py "select status, count(*) as pedidos from raw.pedidos group by status order by status"
python scripts/consultar.py "select min(criado_em) as primeiro_estorno from raw.pedidos where status = 'estornado'"
```

Resultado esperado: aberto 328, cancelado 454, estornado 135 e pago 2083, somando os 3000 pedidos. O dado chegou inteiro, então o problema não está na ingestão. Existe um status que ninguém conhecia, `estornado`, e ele começou a chegar em `2026-10-20` sem ninguém avisar.

**4. Seguir o caminho do dado.** A ingestão lê cada CSV e grava na camada `raw`. O staging padroniza: converte a data e deixa o status em minúsculo.

```bash
code src/ingestao.py
code models/staging/stg_pedidos.sql
```

**5. Olhar a regra de negócio.** Valor total é quantidade vezes preço, e só o pedido cancelado sai do faturamento. A regra não sabe o que é estorno.

```bash
code models/marts/fct_pedidos.sql
```

**6. Confirmar a suspeita.** Soma o valor dos pedidos de cada status.

```bash
python scripts/consultar.py "select status, count(*) as pedidos, round(sum(quantidade * preco_unitario), 2) as valor from main.stg_pedidos group by status order by status"
```

Resultado esperado: os estornados somam `18251.0`. É exatamente a diferença entre `R$ 333.739,80` e `R$ 315.488,80`: dinheiro que voltou para o cliente foi somado no faturamento.

**7. Olhar os testes.** Procura a pasta de testes do projeto.

```bash
ls tests
```

Resultado esperado: `No such file or directory`. Não existe nenhum teste, por isso nada avisou antes da reunião.

**8. Avaliar na régua.** A avaliação é feita nos slides da aula: volte para a apresentação.

Branch desta aula: `aula-01-inicio`.

---

<a name="aula-02-testes"></a>

### Aula 2 · Testes automatizados

Roteiro da aula: copie um bloco de cada vez (botão de copiar no canto do bloco), cole no terminal do Codespace e confira o resultado. Você começa da branch `aula-01-inicio` e traz os arquivos desta branch com `git checkout`. Na Aula 2 você trabalha direto na `main`.

Na Aula 1 você achou a causa: o pedido estornado entrou no faturamento, e nenhum teste avisou. Agora os testes provam o erro, a regra é corrigida e o projeto passa a barrar status desconhecidos antes do relatório.

#### 2.1 · A pirâmide de testes

**1. Trazer a regra de negócio e os testes unitários.** Copia só estes arquivos da branch `aula-02-testes` para o seu projeto.

```bash
git checkout origin/aula-02-testes -- src/transformacoes.py tests/__init__.py tests/test_transformacoes.py
```

**2. Ler a regra e os testes.** A regra é uma função pura: recebe uma lista e devolve uma lista. São quatro testes, e o `parametrize` roda o mesmo teste com vários cenários. O `test_remove_pedidos_estornados` é o erro da reunião virando teste.

```bash
code src/transformacoes.py
code tests/test_transformacoes.py
```

**3. Rodar os testes.** O `pytest` procura os testes da pasta `tests/` e roda um por um. O `-v` mostra o nome de cada teste.

```bash
pytest -v
```

Resultado esperado: `6 passed`.

**4. Ver o teste falhar, de propósito.** Em `src/transformacoes.py`, troque `round(p["quantidade"] * p["preco_unitario"], 2)` por `p["quantidade"] * p["preco_unitario"]`, salve e rode:

```bash
pytest -v
```

Resultado esperado: `1 failed`, com o valor `0.30000000000000004`. Desfaça com **Ctrl+Z**, salve e rode `pytest -v` de novo: `6 passed`.

#### 2.2 · Integração e idempotência

**1. Trazer o pipeline e o teste de integração.** O pipeline lê do banco, aplica a regra e grava o resultado.

```bash
git checkout origin/aula-02-testes -- src/pipeline.py tests/test_integracao.py
code src/pipeline.py
code tests/test_integracao.py
```

**2. Rodar os testes.** A fixture cria um DuckDB em memória: um banco real, mas descartável.

```bash
pytest -v
```

Resultado esperado: `8 passed`.

**3. Ver a idempotência quebrar, de propósito.** Em `src/pipeline.py`, troque `CREATE OR REPLACE TABLE` por `CREATE TABLE IF NOT EXISTS`, salve e rode:

```bash
pytest -v
```

Resultado esperado: `1 failed`, com `assert 2 == 1`. O pedido apareceu duas vezes. Desfaça com **Ctrl+Z**, salve e rode `pytest -v` de novo: `8 passed`.

#### 2.3 · Testando dados com dbt

**1. Trazer os testes de dados.** Testes do staging, unit tests do mart e o teste singular. A regra do `fct_pedidos` ainda é a antiga, a mesma que gerou o número errado.

```bash
git checkout origin/aula-02-testes -- models/staging/_staging.yml models/marts/_marts.yml tests/assert_valor_total_positivo.sql
code models/staging/_staging.yml
code models/marts/_marts.yml
code tests/assert_valor_total_positivo.sql
```

No `_staging.yml`, o `accepted_values` lista os status que a loja conhece. No `_marts.yml`, o unit test `valor_total_ignora_estornados` diz o que o faturamento deveria fazer com um estorno.

**2. Rodar modelos e testes juntos.** O primeiro comando carrega os CSVs. O `dbt build` roda os modelos e os testes na ordem do grafo.

```bash
python -m src.ingestao
dbt build
```

Resultado esperado: `FAIL 1` no unit test `valor_total_ignora_estornados`, `ERROR=1` e `SKIP=2`. O teste reproduziu o erro da reunião: o pedido estornado apareceu no resultado, somado ao faturamento.

**3. Corrigir a regra.** Traz o `fct_pedidos` corrigido: cancelados e estornados ficam fora do faturamento.

```bash
git checkout origin/aula-02-testes -- models/marts/fct_pedidos.sql
code models/marts/fct_pedidos.sql
dbt build
python scripts/faturamento.py
```

Resultado esperado: `PASS=12` e `Faturamento de outubro: R$ 315.488,80 em 2411 pedidos`. O número agora bate com o financeiro.

**4. Chegar um status novo, de propósito.** Acrescenta um pedido com o status `devolvido`, que a loja ainda não conhece, carrega de novo e roda o build.

```bash
echo "3001,1,1,1,5.90,devolvido,2026-10-31" >> data/pedidos.csv
python -m src.ingestao
dbt build
```

Resultado esperado: `FAIL 1` no teste `accepted_values` do status, `ERROR=1` e `SKIP=4`. O teste funciona como um porteiro: barrou o status desconhecido antes de ele chegar ao relatório. O teste não corrige o dado, ele avisa. O que fazer com o status novo é decisão do engenheiro com o negócio.

**5. Desfazer.** Volta o CSV ao original, carrega de novo e roda o build.

```bash
git checkout data/pedidos.csv
python -m src.ingestao
dbt build
```

Resultado esperado: `PASS=12`.

#### 2.4 · Cobertura de testes

**1. Medir a cobertura.** Mostra quais linhas os testes executaram. Abaixo de 80%, falha.

```bash
pytest --cov=src --cov-report=term-missing --cov-fail-under=80
```

Resultado esperado: `42%` e `FAIL`. A ingestão inteira está sem teste.

**2. Trazer o teste que falta.** O teste da ingestão roda numa pasta temporária, sem tocar nos dados reais.

```bash
git checkout origin/aula-02-testes -- tests/test_ingestao.py
code tests/test_ingestao.py
```

**3. Medir de novo.**

```bash
pytest --cov=src --cov-report=term-missing --cov-fail-under=80
```

Resultado esperado: `9 passed` e `96%`.

**4. Salvar a aula no Git.** Prepara os arquivos, registra o commit e envia para o GitHub.

```bash
git add .
git commit -m "fix: estornado fora do faturamento, com testes"
git push
```

Branch desta aula: `aula-02-testes`.

---

<a name="aula-03-ci-cd"></a>

### Aula 3 · CI/CD para pipelines de dados

Roteiro da aula: copie um bloco de cada vez (botão de copiar no canto do bloco), cole no terminal do Codespace e confira o resultado. A partir daqui, toda mudança vai por branch e pull request.

O faturamento foi corrigido na Aula 2, mas nada impede que alguém traga a regra antiga de volta sem rodar os testes. Nesta aula, o GitHub passa a rodar os testes sozinho e a barrar esse erro antes da produção.

Antes de começar: para rodar o CI e mexer nas configurações, você precisa de uma cópia do repositório na sua conta. Clique em **Fork**, desmarque **Copy the main branch only** (para vir com todas as branches) e, na aba **Actions** da sua cópia, habilite os workflows.

O vídeo 3.1 é só de slides, sem comandos.

#### 3.2 · Os blocos de um workflow

**1. Criar a branch e trazer o workflow.** O `git switch -c` cria uma branch nova e já entra nela.

```bash
git switch -c ci/primeiro-workflow
git checkout origin/aula-03-ci-cd -- .github/workflows/ci.yml
code .github/workflows/ci.yml
```

Leia de cima para baixo: `on` (quando roda), `jobs` e `runs-on` (o que roda e em qual máquina), `steps` (a sequência de passos).

**2. Enviar a branch para o GitHub.**

```bash
git add .
git commit -m "ci: workflow de qualidade"
git push -u origin ci/primeiro-workflow
```

**3. Abrir o pull request.** No GitHub, clique em **Compare & pull request → Create pull request** e espere o check `qualidade` ficar verde.

**4. Ver o CI ficar vermelho, de propósito.** O primeiro comando traz de volta a regra antiga do faturamento, da Aula 1, a que gerou o número errado da reunião. Os outros enviam a mudança.

```bash
git checkout origin/aula-01-inicio -- models/marts/fct_pedidos.sql
git commit -am "test: regra antiga do faturamento de volta"
git push
```

Resultado esperado: o check fica vermelho no pull request. No log, o step **Build e testes do dbt** mostra o `FAIL` do unit test `valor_total_ignora_estornados`. O erro da reunião foi barrado antes da produção.

**5. Desfazer.** O `git revert` cria um commit que desfaz o anterior.

```bash
git revert HEAD --no-edit
git push
```

Resultado esperado: o check volta a ficar verde. Não faça o merge ainda.

#### 3.3 · Environments, secrets e aprovação

**1. Criar os ambientes e o secret no GitHub.** Em **Settings** do seu repositório:

- **Environments → New environment:** `staging` → **Configure environment**.
- **Environments → New environment:** `production` → marque **Required reviewers**, adicione o seu usuário → **Save protection rules**.
- **Secrets and variables → Actions → New repository secret:** nome `EXEMPLO_SECRET`, valor `ensaio123`.

**2. Trazer os arquivos do deploy.** Um banco por ambiente, o script de deploy e o workflow de CD.

```bash
git checkout origin/aula-03-ci-cd -- profiles.yml scripts/deploy.sh .github/workflows/cd.yml
code profiles.yml
code scripts/deploy.sh
code .github/workflows/cd.yml
```

**3. Testar o deploy no Codespace.** Roda o deploy simulado no ambiente de staging.

```bash
EXEMPLO_SECRET=teste bash scripts/deploy.sh staging
```

Resultado esperado: `PASS=12` e `Deploy em staging concluído`.

**4. Enviar para o GitHub.**

```bash
git add .
git commit -m "ci: deploy com aprovacao"
git push
```

**5. Fazer o merge e aprovar a produção.** No pull request, com o check verde: **Merge pull request → Confirm merge**. Depois vá em **Actions → Deploy do pipeline**, clique em **Review deployments → production → Approve and deploy**. No log, o secret aparece como `***`.

**6. Voltar para a main atualizada.**

```bash
git switch main
git pull
```

#### 3.4 · Protegendo a main

**1. Criar a regra de proteção.** Em **Settings → Branches → Add classic branch protection rule**:

- **Branch name pattern:** `main`.
- Marque **Require a pull request before merging** e desmarque **Require approvals**.
- Marque **Require status checks to pass before merging** e escolha `qualidade`.
- Marque **Do not allow bypassing the above settings**.
- Clique em **Create**.

**2. Criar uma branch de teste.**

```bash
git switch -c teste/ci-vermelho
```

**3. Quebrar de propósito e enviar.** Traz de novo a regra antiga do faturamento e envia a branch.

```bash
git checkout origin/aula-01-inicio -- models/marts/fct_pedidos.sql
git commit -am "test: merge bloqueado"
git push -u origin teste/ci-vermelho
```

**4. Ver o merge bloqueado.** Abra o pull request no GitHub: com o CI vermelho, o botão de merge não libera. Depois clique em **Close pull request**.

**5. Limpar.** Volta para a main e apaga a branch de teste.

```bash
git switch main
git branch -D teste/ci-vermelho
```

Branch desta aula: `aula-03-ci-cd`.

---

<a name="aula-04-padroes"></a>

### Aula 4 · Padrões técnicos e code review

Roteiro da aula: copie um bloco de cada vez (botão de copiar no canto do bloco), cole no terminal do Codespace e confira o resultado.

Na reunião, ninguém soube explicar a regra do faturamento: ela só existia no código. Nesta aula, o time combina padrões, revisa o código em todo pull request e escreve as decisões em ADRs.

#### 4.1 · Padrões técnicos do time

**1. Criar a branch e ver o histórico.** O `git log --oneline` mostra um commit por linha, com os prefixos `feat`, `fix`, `test` e `ci`.

```bash
git switch -c padroes/pre-commit
git log --oneline
```

**2. Trazer os ADRs.** O primeiro registra por que o time escolheu o dbt. O segundo escreve a regra do faturamento, a que ninguém soube explicar na reunião.

```bash
git checkout origin/aula-04-padroes -- docs/adr/0001-escolha-do-dbt.md docs/adr/0002-regra-do-faturamento.md
code docs/adr/0001-escolha-do-dbt.md
code docs/adr/0002-regra-do-faturamento.md
```

#### 4.2 · Automatizando com pre-commit

**1. Trazer a configuração.** O `.pre-commit-config.yaml` tem os hooks de higiene, o ruff e o sqlfluff. O `.sqlfluff` diz que o SQL é DuckDB e usa o dbt.

```bash
git checkout origin/aula-04-padroes -- .pre-commit-config.yaml .sqlfluff
code .pre-commit-config.yaml
code .sqlfluff
```

**2. Ativar e rodar em todos os arquivos.** O `install` liga o pre-commit nos commits. O `run --all-files` roda as verificações agora.

```bash
pre-commit install
pre-commit run --all-files
```

Resultado esperado: na primeira vez ele instala as ferramentas (alguns minutos), e o `ruff format` mostra `Failed` com `4 files reformatted`.

**3. Ver o que ele arrumou.**

```bash
git diff
```

**4. Rodar de novo.**

```bash
git add .
pre-commit run --all-files
```

Resultado esperado: tudo `Passed`.

**5. Trazer o CI que roda o pre-commit.**

```bash
git checkout origin/aula-04-padroes -- .github/workflows/ci.yml
```

#### 4.3 · Code review eficaz

**1. Trazer o template de pull request e o CODEOWNERS.**

```bash
git checkout origin/aula-04-padroes -- .github/pull_request_template.md .github/CODEOWNERS
code .github/pull_request_template.md
code .github/CODEOWNERS
```

**2. Enviar para o GitHub.** Repare que o pre-commit roda sozinho no commit.

```bash
git add .
git commit -m "chore: padroes automatizados com pre-commit"
git push -u origin padroes/pre-commit
```

**3. Revisar o pull request.** Abra o pull request: a descrição já vem com o template. Na aba **Files changed**, revise seguindo a ordem: correção, impacto em dados, testes, performance, segurança e legibilidade.

**4. Merge e aprovação.** Com o check verde: **Merge pull request**, depois **Actions → Review deployments → Approve and deploy**.

Branch desta aula: `aula-04-padroes`.

---

<a name="aula-05-lideranca"></a>

### Aula 5 · Mentoria e liderança técnica

Roteiro da aula: copie um bloco de cada vez (botão de copiar no canto do bloco), cole no terminal do Codespace e confira o resultado.

O caminho que você seguiu na Aula 1 para achar o erro do faturamento estava só na sua cabeça. Nesta aula, ele vira documentação e um runbook, para que qualquer pessoa do time saiba agir quando um número chegar errado.

O vídeo 5.1 é só de slides, sem comandos.

#### 5.2 · Os quatro formatos de pair

**1. Atualizar a main e criar a branch.**

```bash
git switch main
git pull
git switch -c docs/documentacao
```

**2. Trazer as descrições dos modelos.** A source, o staging e o mart, cada um com a sua descrição.

```bash
git checkout origin/aula-05-lideranca -- models/staging/_sources.yml models/staging/_staging.yml models/marts/_marts.yml
code models/staging/_sources.yml
code models/staging/_staging.yml
code models/marts/_marts.yml
```

**3. Conferir que nada quebrou.**

```bash
dbt build
```

Resultado esperado: `PASS=12`.

#### 5.3 · Knowledge sharing

**1. Gerar e abrir a documentação.** O primeiro comando gera o site da documentação. O segundo abre o site na porta 8080.

```bash
dbt docs generate
dbt docs serve --no-browser
```

Clique em **Open in Browser** no aviso da porta 8080. Veja a descrição do `fct_pedidos` e o grafo de linhagem (botão azul no canto inferior direito): é o caminho do faturamento, do CSV até o relatório. Para parar, volte ao terminal e aperte **Ctrl+C**.

**2. Trazer o runbook e o README.** O passo 1 do runbook é a investigação da Aula 1: o número, os arquivos, o banco, a regra e os testes.

```bash
git checkout origin/aula-05-lideranca -- docs/runbooks/incidente-de-dados.md README.md
code docs/runbooks/incidente-de-dados.md
```

**3. Enviar para o GitHub.**

```bash
git add .
git commit -m "docs: documentacao dos modelos e runbook de incidente de dados"
git push -u origin docs/documentacao
```

**4. Merge e aprovação.** Abra o pull request, espere o check verde, faça o merge e aprove a produção em **Actions → Review deployments**.

Branch desta aula: `aula-05-lideranca`.

---

<a name="aula-06-metricas"></a>

### Aula 6 · Medindo a transformação DataOps

Roteiro da aula: copie um bloco de cada vez (botão de copiar no canto do bloco), cole no terminal do Codespace e confira o resultado.

A diretoria quer saber o que aconteceu com o faturamento e se o time melhorou. Nesta aula, o incidente vira um postmortem e as entregas viram métricas.

Antes de começar, no GitHub: **Issues → Labels → New label**, nome `incidente` → **Create label**.

#### 6.1 · Postmortem sem culpados

**1. Abrir a issue do incidente.** No GitHub: **Issues → New issue**, título `Faturamento de outubro divergente na reunião`, label `incidente` → **Submit new issue**. A hora de abertura vira o "detectado" das métricas.

**2. Criar a branch e trazer o postmortem.** Conta a história da reunião, sem culpados: o que aconteceu, a causa raiz e as ações.

```bash
git switch main
git pull
git switch -c incidente/faturamento-divergente
git checkout origin/aula-06-metricas -- docs/postmortems/2026-11-faturamento-divergente.md
code docs/postmortems/2026-11-faturamento-divergente.md
```

Repare na lista de ações: cada causa virou um teste, uma automação ou um documento, e quase tudo já foi feito nas aulas anteriores.

**3. Enviar para o GitHub.**

```bash
git add .
git commit -m "docs: postmortem do faturamento divergente"
git push -u origin incidente/faturamento-divergente
```

**4. Publicar e fechar a issue.** Pull request, check verde, merge, aprovação da produção e, por fim, feche a issue do incidente.

#### 6.2 · Métricas DORA

**1. Criar a branch e trazer o script.**

```bash
git switch main
git pull
git switch -c metricas/dora
git checkout origin/aula-06-metricas -- scripts/metricas_dora.py
code scripts/metricas_dora.py
```

**2. Apontar para o seu repositório.** Na linha do `REPO`, coloque o nome do seu repositório, no formato `usuario/repositorio`.

**3. Rodar o script.** Busca os dados na API do GitHub e calcula os deploys por semana e o lead time.

```bash
python scripts/metricas_dora.py
```

Com poucas semanas de histórico o número é pequeno. O que importa é acompanhar a tendência.

#### 6.3 · Tempo de recuperação (MTTR)

**1. Ver a função do MTTR.** No mesmo `scripts/metricas_dora.py`, veja a função `recuperacao_horas`. Depois, no GitHub, abra a issue do faturamento divergente e compare com o tempo de recuperação que o script mostrou no terminal.

**2. Fechar a história.** Roda o pipeline inteiro de novo e mostra o faturamento do relatório.

```bash
python -m src.ingestao
dbt build
python scripts/faturamento.py
```

Resultado esperado: `PASS=12` e `Faturamento de outubro: R$ 315.488,80 em 2411 pedidos`. O número do relatório bate com o do financeiro.

**3. Trazer o README final.**

```bash
git checkout origin/aula-06-metricas -- README.md
```

**4. Enviar para o GitHub.**

```bash
git add .
git commit -m "feat: metricas DORA"
git push -u origin metricas/dora
```

**5. Publicar.** Pull request, check verde, merge e aprovação da produção.

#### 6.4 · A régua, antes e depois

Só slides, sem comandos.

Branch desta aula: `aula-06-metricas`.


## Como abrir o ambiente

**Pelo navegador, sem instalar nada (recomendado):**

1. Escolha a branch da aula no seletor de branches do GitHub.
2. Clique em **Code → Codespaces → Create codespace**.
3. Espere o ambiente ficar pronto. Python 3.12, dbt, DuckDB e as demais ferramentas são instalados sozinhos.
4. Ao terminar de estudar, pare o codespace (menu Codespaces → Stop) para não gastar a sua cota gratuita.

**No seu computador:** instale Python 3.12 e Git, depois rode:

```bash
python -m venv .venv
source .venv/bin/activate          # no Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
export DBT_PROFILES_DIR=.          # no Windows: $env:DBT_PROFILES_DIR = "."
```

## Primeiros comandos

```bash
python -m src.ingestao   # carrega os CSVs da pasta data/ no DuckDB
dbt run                  # constrói os modelos
```

## Documentação do time

- **Modelos e linhagem:** `dbt docs generate` e depois `dbt docs serve --no-browser`. No codespace, abra a porta 8080.
- **Quando um número chegar errado ou o pipeline falhar:** [docs/runbooks/incidente-de-dados.md](docs/runbooks/incidente-de-dados.md).
- **Decisões técnicas:** [docs/adr/](docs/adr/).

## Uma branch por aula

Cada branch guarda o estado do projeto **ao final** da aula. Para acompanhar uma aula, comece pela branch da aula anterior.

| Branch | Estado do projeto |
| --- | --- |
| `aula-01-inicio` | Scripts funcionando, sem testes e sem CI. O faturamento chega errado na reunião |
| `aula-02-testes` | Testes com pytest, testes de dados do dbt, cobertura e a correção do faturamento |
| `aula-03-ci-cd` | CI em todo pull request barrando a regra antiga e deploy em staging e production com aprovação |
| `aula-04-padroes` | pre-commit, template de pull request, CODEOWNERS e ADRs, incluindo a regra do faturamento |
| `aula-05-lideranca` | Documentação dos modelos e runbook de incidente de dados |
| `aula-06-metricas` | Postmortem do faturamento divergente e métricas DORA |

## Estrutura

```text
dataops-curso-historia/
├── .devcontainer/          ambiente do Codespaces
├── .github/
│   ├── workflows/          ci.yml · cd.yml
│   ├── pull_request_template.md
│   └── CODEOWNERS
├── data/                   CSVs de origem
├── src/                    ingestao.py · transformacoes.py · pipeline.py
├── tests/                  testes pytest e teste singular do dbt
├── models/                 staging/ · marts/ (dbt)
├── scripts/                faturamento.py · consultar.py · deploy.sh · metricas_dora.py
├── docs/                   adr/ · runbooks/ · postmortems/
├── .pre-commit-config.yaml
├── .sqlfluff
├── dbt_project.yml
├── profiles.yml
└── requirements.txt
```
