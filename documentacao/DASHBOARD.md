# Executar o painel

## Preparar a conexão

1. Use o PostgreSQL já carregado no Aiven Free e a view `caged.v_indicadores_emprego`. Confira a carga com `python codigo/executar.py verificar-nuvem` na sessão administrativa. O comando registra contagens, hash, TLS e tamanho. [Guia de conferência](GUIA_IMPLANTACAO.md)
2. Use a conta de leitura existente `caged_dashboard`. Para preparar outra instalação, o proprietário dos objetos pode conceder uso do schema e leitura da view conforme [permissoes_dashboard.sql](../sql/permissoes_dashboard.sql). Guarde os parâmetros e a CA fora do Git. [Gerenciamento de usuários no Aiven](https://aiven.io/docs/products/postgresql/howto/manage-service-users)
3. Instale as dependências com `python -m pip install -r requirements.txt` em um ambiente virtual Python 3.12. Copie `.env.example` para `.env` e preencha `PG*` com a conta do painel. Em `PGSSLROOTCERT`, use o caminho local do certificado CA. O aplicativo também aceita as variáveis diretamente no ambiente e sempre exige TLS com `verify-full`.

## Executar com um comando

Na raiz do projeto, com o ambiente, a conta de leitura e o IP da equipe em `allowed_cidrs`:

```sh
python -m streamlit run dashboard/app.py --server.address localhost
```

Abra o endereço local indicado pelo Streamlit. O botão **Atualizar dados** força uma nova consulta ao PostgreSQL. Em caso de erro, confira o estado do serviço, o filtro de IP, as variáveis e a CA. O painel apresenta um aviso genérico e preserva os dados de acesso.

Para guardar a configuração fora de uma pasta sincronizada, defina `CAGED_ENV_FILE` com o caminho de um arquivo dotenv privado. O `.env` do projeto pode conter apenas esse apontamento.

## Reconciliar o Terraform do banco existente

Use o mesmo backend e arquivo de variáveis privados do projeto, conforme o [guia de infraestrutura](GUIA_IMPLANTACAO.md). Carregue o token Aiven na sessão e confirme o recurso com `terraform -chdir=infraestrutura state list`. Se `aiven_pg.caged` já apontar para o serviço correto, faça um plano com atualização do estado. Se estiver ausente, importe o serviço existente uma vez com a configuração correspondente.

```powershell
$arquivoVariaveisCaged = Join-Path $env:LOCALAPPDATA 'ProjetoBigData/terraform.entrega.tfvars.json'
terraform -chdir=infraestrutura plan -var-file="$arquivoVariaveisCaged" -detailed-exitcode
```

Numa instalação nova, prepare o arquivo privado a partir de [terraform.tfvars.example](../infraestrutura/terraform.tfvars.example) e aponte para ele em `-var-file`. Use extensão `.tfvars` para HCL ou `.tfvars.json` para JSON.

O código de saída 0 indica ausência de mudanças, 2 indica mudanças propostas e 1 indica falha. Para mudanças esperadas, como a inclusão de um IP específico no filtro, faça o `apply` do plano revisado e rode o plano novamente. Uma proposta de substituição exige verificar os nomes, o plano, o backend e o serviço antes de aplicar. O bloco `prevent_destroy` protege o serviço no Terraform. A variável `termination_protection` controla a proteção no Aiven e pode refletir o valor existente quando o serviço é importado.

Na entrega foi usado o state privado existente e um arquivo privado de variáveis com os parâmetros conferidos na API. O ajuste adicionou o IPv6 do navegador como `/128`. A API, o state e o plano final foram conferidos depois da aplicação. [Registro da reconciliação](../verificacoes/terraform_entrega.json) · [Autorização de IP no PG Studio](https://aiven.io/docs/products/postgresql/howto/pg-studio/security-connections)

## Hospedagem opcional

A demonstração desta entrega roda localmente. As capturas do README servem de apoio se o Aiven ficar indisponível. Uma publicação futura no Streamlit Community Cloud exige incluir os IPs de saída do serviço em `allowed_cidrs`. A validação do Terraform recusa `/0` em IPv4 e IPv6.

1. Disponibilize a branch de entrega no GitHub. No Community Cloud, selecione essa branch e `dashboard/app.py` como ponto de entrada. O serviço instala `requirements.txt` da raiz. [Implantação no Community Cloud](https://docs.streamlit.io/deploy/streamlit-community-cloud/deploy-your-app/deploy)
2. Em **Advanced settings → Secrets**, defina `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER` e `PGPASSWORD` com os dados da conta de leitura. Defina `PGSSLROOTCERT_PEM` com o PEM da CA, em uma string TOML multilinha. A aplicação escreve a CA em um arquivo temporário durante a conexão e o remove em seguida. [Segredos do Community Cloud](https://docs.streamlit.io/deploy/streamlit-community-cloud/deploy-your-app/secrets-management)
3. Confira a lista publicada pelo Streamlit e autorize os CIDRs aplicáveis no arquivo privado do Terraform. Revise o plano antes de aprovar qualquer alteração no serviço. [IPs do Community Cloud](https://docs.streamlit.io/deploy/streamlit-community-cloud/status#ip-addresses)
4. Abra o painel publicado e teste um recorte com mudanças de filtros em cada indicador. Use **Atualizar dados** para verificar nova consulta ao Aiven. Registre o link ativo no README após a conferência.

Os recursos do PostgreSQL estão em [infraestrutura/](../infraestrutura/). A publicação opcional do painel usa a configuração Streamlit versionada e os segredos da plataforma.
