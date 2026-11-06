# Configuração do laboratório — AWS Academy + GitHub Actions

## 1. Autenticação do laboratório

Inicie a sessão no AWS Academy e abra **AWS Details > AWS CLI > Show**.
Salve as três credenciais temporárias em `~/.aws/cloud-pipeline/credentials`,
na seção `[cloud-pipeline]`: `aws_access_key_id`, `aws_secret_access_key` e
`aws_session_token`. O arquivo deve ter permissão `600`.

O arquivo separado `~/.aws/cloud-pipeline/config` contém:

```ini
[profile cloud-pipeline]
region = us-east-1
output = json
```

Use os caminhos explicitamente para preservar os perfis de outros projetos:

```bash
AWS_CONFIG_FILE="$HOME/.aws/cloud-pipeline/config" \
AWS_SHARED_CREDENTIALS_FILE="$HOME/.aws/cloud-pipeline/credentials" \
aws --profile cloud-pipeline sts get-caller-identity
```

As credenciais representam a sessão do papel `voclabs`; não é necessário criar
outro usuário IAM. Quando expirarem, copie as novas credenciais e atualize também
os secrets do environment `production` no GitHub. Não use `aws login` para esta
sessão do laboratório nem substitua `~/.aws/credentials`.

## 2. Infraestrutura EC2

| Recurso | Configuração |
|---|---|
| Região | `us-east-1` |
| Instância | `i-0f24b556a34e501ae` (`cloud-pipeline-api`) |
| Sistema | Ubuntu Server 24.04 LTS, x86_64 |
| Capacidade | `t3.micro`, créditos de CPU em modo Standard |
| Disco | 8 GiB gp3, criptografado, excluído ao terminar a instância |
| Security Group | `sg-021dd23fb174a5b6b` |
| Entrada HTTP | TCP/80, `0.0.0.0/0`, para demonstração pública |
| Entrada SSH | Fechada; nenhuma regra TCP/22 e nenhuma chave SSH |
| Perfil de instância | `LabInstanceProfile`, com o `LabRole` existente |
| Identificação | Tag `Project=cloud-pipeline-starter` |

O perfil existente permite que o SSM Agent registre a instância no AWS Systems
Manager. A implantação usa **Run Command**, sem runner próprio e sem abrir SSH
para os IPs variáveis dos runners GitHub.

O bootstrap executa `scripts/setup-ec2-ubuntu.sh` como root, com `USER=ubuntu`,
instalando Docker e curl. Confira o término do cloud-init e o estado Online no
Systems Manager antes do primeiro deploy.

O IPv4 público pode mudar ao reiniciar a sessão do laboratório. O CD consulta o
endereço atual pela API EC2; se a instância for recriada, atualize seu ID no GitHub.

## 3. GitHub

No repositório, crie o environment **production**, restrito à branch `main`.
Configure nele:

| Tipo | Nome | Valor |
|---|---|---|
| Secret | `AWS_ACCESS_KEY_ID` | Access key temporária do laboratório |
| Secret | `AWS_SECRET_ACCESS_KEY` | Secret key temporária do laboratório |
| Secret | `AWS_SESSION_TOKEN` | Session token temporário do laboratório |
| Variable | `AWS_REGION` | `us-east-1` |
| Variable | `EC2_INSTANCE_ID` | `i-0f24b556a34e501ae` |

Nunca versionar esses secrets nem incluí-los em logs ou screenshots. O workflow
não usa `EC2_SSH_KEY`, `EC2_KNOWN_HOSTS`, `EC2_HOST` ou `EC2_USER`.

Mantenha `main` protegida e desenvolva em branches `feature/...` com PR. A CI
executa pytest e build Docker; apenas pushes na `main` publicam imagens no GHCR.
O CD roda somente após uma CI bem-sucedida originada de push na `main`, faz checkout
do mesmo SHA e envia `scripts/deploy.sh` à instância via SSM. Deploys são
serializados para evitar recriar o contêiner simultaneamente.

## 4. GHCR e persistência

A imagem é `ghcr.io/melloirl/cloud-pipeline-starter:COMMIT_SHA` e deve ser pública
para permitir `docker pull` anônimo na EC2. A visibilidade do repositório, por si
só, não garante a visibilidade do pacote; confira nas configurações do GHCR.

`scripts/deploy.sh` publica a porta 80, monta o volume Docker `cloud-tasks-data`
em `/data` e mantém o SQLite em `/data/tasks.db`. Recriar o contêiner preserva
esse volume; terminar a instância também exclui seu disco e os dados do volume.

## 5. Validação e evidências

1. Abra um PR e registre sua revisão e o run de CI aprovado, sem publicação/deploy.
2. Integre na `main` e registre o SHA, os runs de CI/CD e a imagem com a mesma tag.
3. Consulte `http://IP_PUBLICO/health` e demonstre as operações em `/tasks`.
4. Crie uma tarefa com título identificável e registre seu ID. Execute novamente
   o deploy da mesma imagem via SSM e confira o mesmo ID e título após recriar o
   contêiner, sem excluir o volume.
5. Guarde links e respostas HTTP para o relatório e a apresentação.

Se o CD falhar na autenticação, renove os três secrets do laboratório. Se a
instância aparecer Offline no SSM, confira a sessão do lab, o perfil de instância,
o agente e a saída HTTPS. Para investigar o deploy, consulte stdout/stderr do
Run Command; falhas do comando também são exibidas no run do GitHub.

## 6. Custos e encerramento

Monitore o saldo do laboratório. A aplicação usa uma instância, seu disco e um
IPv4 público; não usa RDS, NAT Gateway, balanceador ou uma segunda máquina para
runner. O banco SQLite não deve receber dados reais.

Após a apresentação, termine a instância e confirme a exclusão do disco. Exclua
o Security Group quando não tiver mais interfaces associadas, desabilite o CD e
remova os secrets temporários do environment. Preserve a infraestrutura de
gerenciamento criada pelo AWS Academy.
