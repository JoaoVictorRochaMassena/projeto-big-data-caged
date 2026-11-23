# Concluir a implantação e a entrega

O serviço PostgreSQL da equipe já existe no Aiven. Ele foi criado no console e importado pelo Terraform. O R trata os dados, o Python valida a base agregada e o painel consulta a carga existente. [README atual](../README.md) · [Registro histórico da Fase 3](historico/LEIA_ME_ETAPA_ATUAL.md)

## 1. Preparar a máquina e validar a base

1. Abra a raiz do repositório ou extraia o pacote `PROJETO_BIG_DATA_ENVIO.zip` e abra a pasta `Projeto_Big_Data`.
2. Instale [Python](https://www.python.org/downloads/windows/) 64 bits (3.12 é a versão testada) e [Terraform](https://developer.hashicorp.com/terraform/install). R só será necessário se decidir refazer o tratamento dos TXT.
3. Abra um PowerShell nessa pasta. Todos os comandos seguintes partem da pasta que contém `README.md`, `codigo/` e `infraestrutura/`.
4. Crie o ambiente Python e instale as versões verificadas:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-lock.txt
.\.venv\Scripts\python.exe codigo/executar.py validar
.\.venv\Scripts\python.exe -m unittest discover -s testes -p "test_*.py" -v
```

Não é necessário ativar a venv. Se o launcher `py` não estiver disponível, use `python -m venv .venv` após conferir a versão instalada. Para outras versões compatíveis de Python, também existe `requirements.txt`; o lock registra as versões usadas nesta validação.

O resultado esperado da validação é **1.006.405 linhas, 51.893.337 movimentações e 12 competências**. O comando extrai o ZIP da base quando necessário e escreve `verificacoes/validacao_base_atual.json`. Os testes que dependem de conexão são habilitados separadamente. [Verificações e testes](VALIDACAO.md)

Não é necessário copiar os TXT ou executar R novamente para carregar a base publicada. Para reproduzir o tratamento integral, instale no R `DBI`, `duckdb`, `digest` e `jsonlite`, coloque os 12 TXT em `dados_originais/` e execute `Rscript codigo/executar.R tratamento`. Depois retorne à validação em Python.

## 2. Conferir o serviço existente

No Aiven Console, abra o serviço PostgreSQL da equipe, confirme que está ativo no plano Free e confira o IP público da equipe na lista de acesso. O Terraform usa a variável `allowed_cidrs` para essa lista e recusa redes `/0`.

## 3. Conferir o plano Terraform

Use o mesmo backend privado e o arquivo de variáveis do serviço existente. Se a pasta ainda não estiver inicializada, aponte o `terraform init -backend-config` para o state usado na importação. Mantenha state, variáveis e planos fora do repositório e de pastas sincronizadas. O lock do provider fica no Git com o código Terraform.

```powershell
.\infraestrutura\configurar_token.ps1
$arquivoVariaveisCaged = Join-Path $env:LOCALAPPDATA 'ProjetoBigData/terraform.entrega.tfvars.json'
terraform -chdir=infraestrutura plan -var-file="$arquivoVariaveisCaged" -detailed-exitcode
$LASTEXITCODE
```

O resultado esperado é código `0`. O código `2` indica mudanças propostas e o código `1`, falha. Revise qualquer mudança antes de aprovar uma aplicação. O recurso tem `prevent_destroy = true` no Terraform. A proteção contra exclusão no serviço Aiven está desativada, e a variável privada `termination_protection` acompanha o valor do serviço importado. [Evidência da infraestrutura](../verificacoes/terraform_entrega.json)

## 4. Conferir a carga existente

No painel do serviço, obtenha host, porta, banco, usuário e senha e baixe a CA. Mantenha o arquivo CA em local privado. É possível usar `segredos/ca.pem`, ignorado pelo Git; o certificado CA serve para verificar o servidor e não substitui a senha.

Na mesma janela PowerShell em que executará Python:

```powershell
.\codigo\configurar_conexao.ps1
.\.venv\Scripts\python.exe codigo/executar.py verificar-nuvem
```

Informe os valores do painel nos prompts. O banco geralmente é `defaultdb`, mas use o valor realmente informado pelo serviço. A senha será solicitada com entrada oculta. Se a conexão falhar, confira o IP público autorizado, a porta e a CA. Não desative a validação de certificado.

O comando usa a conta administrativa para conferir o schema carregado e registrar totais, somas salariais, TLS e tamanho em `verificacoes/carga_nuvem.json`. Em outra janela, use a conta de leitura para iniciar o Streamlit. [Guia do painel](DASHBOARD.md)

## 5. Guardar evidências e atualizar o status

Após uma execução real bem-sucedida:

1. Confirme `verificacoes/carga_nuvem.json` com status `carga_nuvem_validada` e os totais da base.
2. No Aiven, confira o serviço disponível, plano e uso de disco. Não confunda o tamanho do CSV com o tamanho do banco, índices e espaço de operação. O plano gratuito tem capacidade limitada.
3. Registre o resultado do plano e sua data. Se arquivar uma captura, oculte os dados de acesso.
4. Execute [consultas_validacao.sql](../sql/consultas_validacao.sql) para apresentar resultados por mês e a auditoria.
5. Atualize a data da conferência no README e no mapa da entrega após verificar o resultado.

A evidência gerada por Python contém totais, hash, confirmação de TLS, conferência salarial, tamanho do banco e data. Ela não contém host, usuário, senha ou token. `verificacoes/carga_python_teste_local.json` identifica somente o teste em PostgreSQL local e não substitui a evidência de nuvem `carga_nuvem.json`.

## 6. Publicar no GitHub e enviar

Para enviar uma revisão ao GitHub, gere primeiro o pacote e confira a evidência da carga:

```powershell
.\.venv\Scripts\python.exe codigo/preparar_envio.py
```

O comando bloqueia o envio final se não houver a evidência `carga_nuvem.json` da base atual. Ele grava `output/envio/PROJETO_BIG_DATA_ENVIO.zip` sem credenciais, ambiente virtual ou estado Terraform. Não envia arquivos à internet. Extraia **esse ZIP de envio em uma pasta nova** e abra a pasta `Projeto_Big_Data`. No repositório existente do GitHub, use **Add file → Upload files** para enviar o conteúdo dessa pasta limpa para a raiz, preservando os nomes e diretórios.

Confira os arquivos iniciados por ponto, como `.gitignore`, `.env.example` e `infraestrutura/.terraform.lock.hcl`. Se atualizar pelo navegador, retire também os caminhos antigos da raiz do GitHub, incluindo o enunciado em PDF e o arquivo histórico movido para `documentacao/historico/`.

Envie o conteúdo da pasta, sem criar outra pasta `Projeto_Big_Data` dentro do repositório e sem usar o ZIP como substituto dos arquivos navegáveis. Nunca selecione `.git/`, `.terraform/`, `.venv/`, `__pycache__/`, `tmp/`, state, `terraform.tfvars`, senhas, tokens ou a pasta `segredos/` no upload. O `.gitignore` não filtra arquivos arrastados manualmente no navegador. A pasta `resultados/` deve conter a base compactada; o CSV extraído e os intermediários locais não precisam ser enviados.

Envie na plataforma da disciplina o link do repositório atualizado, conforme o procedimento da turma. O pacote e o guia não substituem o banco provisionado com a carga.

Ao terminar, remova `AIVEN_TOKEN` e `PGPASSWORD` da sessão ou feche a janela. Preserve o serviço durante o período em que será avaliado; planos gratuitos podem ser desligados por inatividade, conforme as regras do provedor.

Fontes: [plano gratuito e limites](https://aiven.io/docs/products/postgresql/concepts/pg-free-tier), [criação de serviço](https://aiven.io/docs/products/postgresql/get-started), [provider Terraform](https://registry.terraform.io/providers/aiven/aiven/latest/docs/resources/pg).
