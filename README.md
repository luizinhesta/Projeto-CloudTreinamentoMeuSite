# Projeto 2 — MeuSite

## Sobre o projeto

![Objetivos](imagens/imagem(2).png)

Este projeto foi desenvolvido como desafio da mentoria AWS. O objetivo era construir, do zero, uma aplicação web containerizada com deploy totalmente automatizado na AWS, seguindo boas práticas de infraestrutura como código e CI/CD.

O resultado é uma aplicação ASP.NET Core (.NET 8) rodando em containers no ECS Fargate, com toda a infraestrutura provisionada via CloudFormation e deploys orquestrados por AWS CodePipeline + GitHub Actions. O projeto opera em duas contas AWS separadas (DEV e PROD), cada uma com seu próprio pipeline, domínio DNS e certificado HTTPS.

### O que foi construído

- **Aplicação containerizada** — ASP.NET Core empacotada em Docker, rodando no ECS Fargate
- **Infraestrutura 100% como código** — 8 templates CloudFormation que criam VPC, Security Groups, ECR, ECS Cluster, ALB, ECS Service, CloudFront e Route53
- **CI/CD completo** — GitHub Actions dispara CodePipelines que provisionam a infra e fazem build/deploy da aplicação automaticamente
- **Multi-conta AWS** — ambientes DEV e PROD isolados em contas separadas, com domínios DNS delegados a partir de uma conta MASTER
- **HTTPS ponta a ponta** — DEV com certificado no ALB, PROD com HTTPS terminado no CloudFront + ALB protegido por Security Group e header secreto
- **Auto Scaling** — PROD escala de 1 a 4 tasks automaticamente com base em CPU
- **Deploy e destruição com um clique** — workflows do GitHub Actions para criar ou remover toda a infraestrutura

---

## Contas AWS

| Conta | ID | Responsabilidade |
|---|---|---|
| DEV | `111111111111` | Ambiente de desenvolvimento — Hosted Zone `dev.exemplo.com.br` |
| PROD | `222222222222` | Ambiente de produção — Hosted Zone `prod.exemplo.com.br` |
| MASTER | `333333333333` | Domínio raiz `exemplo.com.br` com delegação de subdomínios para DEV e PROD |

![Objetivos](imagens/imagem(10).png)

---

## Arquitetura geral

```
GitHub Actions (deploy-infra.yml)
  └─► Zip do repositório → upload S3
  └─► CloudFormation: cria pipeline de infra na conta alvo
  └─► Dispara o pipeline automaticamente

GitHub Actions (deploy-app.yml)
  └─► Zip do repositório → upload S3
  └─► Dispara o pipeline de app no CodePipeline

Pipeline de Infra DEV (100% CloudFormation nativo)
  Source → VPC → SG → ECR → ECS_Cluster → ALB → ECS_Service → Route53
```

![Objetivos](imagens/imagem(4).png)

```
Pipeline de Infra PROD (100% CloudFormation nativo)
  Source → VPC → SG → ECR → ECS_Cluster → ALB → ECS_Service → CloudFront → Route53
```
![Objetivos](imagens/imagem(3).png)

```
Pipeline de App (DEV ou PROD)
  Source → Docker_Build_Push_ECR → Deploy_ECS
```
![Objetivos](imagens/imagem(7).png)

```
DEV (HTTPS direto no ALB)
  Usuário → Route53 (meusite.dev.exemplo.com.br) → ALB (HTTPS 443) → ECS Tasks (porta 8080)

PROD (HTTPS via CloudFront)
  Usuário → Route53 (meusite.prod.exemplo.com.br) → CloudFront (HTTPS) ──HTTP──► ALB (porta 80, IPs CloudFront only) → ECS Tasks (porta 8080)
```

![Objetivos](imagens/imagem(5).png)

---

## GitHub Actions — o ponto de partida

![Objetivos](imagens/imagem(1).png)

Todo deploy neste projeto começa pelo GitHub Actions. Ele é o gatilho inicial que conecta o código no repositório à infraestrutura na AWS. Sem ele, nada acontece.

O projeto usa dois workflows manuais (`workflow_dispatch`) que o desenvolvedor executa pela interface do GitHub:

### `deploy-infra.yml` — Deploy Infrastructure

Responsável por criar (ou destruir) toda a infraestrutura na conta AWS escolhida.

**Parâmetros de entrada:**
- `environment`: `dev` ou `prod` — define em qual conta AWS o deploy será feito
- `action`: `deploy` ou `delete` — cria tudo ou destrói tudo

**O que faz na ação `deploy`:**
1. Autentica na conta AWS via OIDC (sem access keys estáticas)
2. Cria service-linked roles necessárias (ELB, ECS) se for a primeira vez
3. Limpa stacks em estado de falha (`ROLLBACK_COMPLETE`, etc.) automaticamente
4. Cria o bucket S3 de artefatos com versionamento e criptografia
5. Faz zip do repositório e upload para o S3
6. Busca o ID da Hosted Zone (Route53) na conta alvo
7. Cria/atualiza as stacks CloudFormation dos dois CodePipelines (infra + app)
8. Dispara automaticamente o pipeline de infraestrutura

**O que faz na ação `delete`:**
1. Deleta stacks na ordem inversa de dependência (Route53 → CloudFront → ECS → ALB → ECR → SGs → VPC)
2. Remove as stacks dos CodePipelines
3. Esvazia e deleta o bucket S3 (objetos, versões e delete markers)
4. Desregistra todas as revisões de task definitions do ECS
5. Remove CloudWatch Log Groups

### `deploy-app.yml` — Deploy Application

Responsável por disparar o pipeline de aplicação (build Docker + deploy ECS).

**Parâmetros de entrada:**
- `environment`: `dev` ou `prod`

**O que faz:**
1. Autentica na conta AWS via OIDC
2. Faz zip do repositório e upload para o S3
3. Dispara o CodePipeline de app (`meusite-{env}-app-pipeline`)

O CodePipeline então executa: `Source → Docker_Build_Push_ECR → Deploy_ECS`

### Autenticação — OIDC (sem secrets estáticas)

Os workflows usam o provider OIDC do GitHub (`token.actions.githubusercontent.com`) para assumir uma IAM Role diretamente na conta AWS. Isso elimina a necessidade de armazenar access keys como secrets — apenas o ARN da role é necessário.

**Permissions no workflow:**
```yaml
permissions:
  id-token: write   # permite solicitar o token OIDC
  contents: read    # permite checkout do repositório
```

**Action utilizada:** `aws-actions/configure-aws-credentials@v4`

### GitHub Actions Secrets

Configure em **Settings → Secrets and variables → Actions**:

| Secret | Descrição |
|---|---|
| `AWS_ROLE_ARN_DEV` | ARN da role OIDC na conta DEV (`arn:aws:iam::111111111111:role/GitHubActionsRole`) |
| `AWS_ROLE_ARN_PROD` | ARN da role OIDC na conta PROD (`arn:aws:iam::222222222222:role/GitHubActionsRole`) |

As roles precisam ter trust policy para `token.actions.githubusercontent.com` com `repo:seu-usuario/SeuRepositorio:*`.

### Fluxo completo

```
Desenvolvedor clica "Run workflow" no GitHub
  └─► GitHub Actions autentica via OIDC na conta AWS
  └─► Zip do código → upload S3
  └─► CloudFormation cria/atualiza os CodePipelines
  └─► CodePipeline executa os stages na AWS
```

O GitHub Actions é apenas o disparador — toda a execução pesada (build, deploy, provisionamento) acontece dentro da AWS via CodePipeline e CodeBuild.

---

## CodePipeline — visão detalhada

### Pipeline de Infraestrutura

O CodePipeline usa o provider nativo `CloudFormation` em cada stage. Cada stage cria/atualiza uma stack diretamente, sem CodeBuild intermediário.

Os templates usam `Fn::ImportValue` para buscar outputs das stacks anteriores — sem passar parâmetros via shell script.

#### DEV

| Stage | Stack | Template |
|---|---|---|
| Source | — | S3 zip |
| VPC | `meusite-dev-vpc` | `1-vpc.yaml` |
| Security_Groups | `meusite-dev-security-groups` | `2-security-groups.yaml` |
| ECR | `meusite-dev-ecr` | `3-ecr.yaml` |
| ECS_Cluster | `meusite-dev-ecs-cluster` | `4-ecs-cluster.yaml` |
| ALB | `meusite-dev-alb` | `5-alb.yaml` |
| ECS_Service | `meusite-dev-ecs-service` | `6-ecs-service.yaml` |
| Route53 | `meusite-dev-route53` | `8-route53.yaml` |

![Objetivos](imagens/imagem(9).png)
![Objetivos](imagens/imagem(15).png)


#### PROD

| Stage | Stack | Template |
|---|---|---|
| Source | — | S3 zip |
| VPC | `meusite-prod-vpc` | `1-vpc.yaml` |
| Security_Groups | `meusite-prod-security-groups` | `2-security-groups.yaml` |
| ECR | `meusite-prod-ecr` | `3-ecr.yaml` |
| ECS_Cluster | `meusite-prod-ecs-cluster` | `4-ecs-cluster.yaml` |
| ALB | `meusite-prod-alb` | `5-alb.yaml` |
| ECS_Service | `meusite-prod-ecs-service` | `6-ecs-service.yaml` |
| CloudFront | `meusite-prod-cloudfront` | `7-cloudfront.yaml` |
| Route53 | `meusite-prod-route53` | `8-route53.yaml` |

![Objetivos](imagens/imagem(8).png)
![Objetivos](imagens/imagem(14).png)


### Pipeline de Aplicação

| Stage | O que faz |
|---|---|
| Source | Lê zip do S3 |
| Docker_Build_Push_ECR | `dotnet restore` → `dotnet publish` → `docker build` → push `latest` + `v{N}` para ECR → gera `imagedefinitions.json` |
| Deploy_ECS | Lê `imagedefinitions.json` → registra nova revisão da task definition → `update-service` → aguarda estabilizar |

O buildspec do `Deploy_ECS` está **inline no CloudFormation** (`pipeline-app-{env}.yaml`).

A cada execução do pipeline de app:
- ECR recebe nova imagem tagueada `v{N}` + `latest`
- Task definition ganha nova revisão
- ECS Service atualizado com `desired-count 1`

### Stacks CloudFormation

| Stack | Nome AWS | Conta |
|---|---|---|
| VPC | `meusite-{env}-vpc` | DEV / PROD |
| Security Groups | `meusite-{env}-security-groups` | DEV / PROD |
| ECR | `meusite-{env}-ecr` | DEV / PROD |
| ECS Cluster | `meusite-{env}-ecs-cluster` | DEV / PROD |
| ALB | `meusite-{env}-alb` | DEV / PROD |
| ECS Service | `meusite-{env}-ecs-service` | DEV / PROD |
| CloudFront | `meusite-prod-cloudfront` | PROD |
| Route53 | `meusite-{env}-route53` | DEV / PROD |
| Pipeline Infra | `meusite-{env}-infra-codepipeline` | DEV / PROD |
| Pipeline App | `meusite-{env}-app-codepipeline` | DEV / PROD |

![Objetivos](imagens/imagem(18).png)
![Objetivos](imagens/imagem(19).png)


---

## DNS e Certificados

### Delegação de subdomínios

A conta MASTER possui o domínio `exemplo.com.br`. Os subdomínios foram delegados via registros NS:

| Subdomínio | Hosted Zone | Conta |
|---|---|---|
| `dev.exemplo.com.br` | Gerenciada na conta DEV | DEV |
| `prod.exemplo.com.br` | Gerenciada na conta PROD | PROD |

Cada conta gerencia seus próprios registros DNS sem precisar de acesso cross-account.

### Certificados ACM

| Ambiente | Certificado | Conta | Usado em |
|---|---|---|---|
| DEV | `*.dev.exemplo.com.br` | DEV (`111111111111`) | ALB (HTTPS 443) |

![Objetivos](imagens/imagem(20).png)

| PROD | `*.prod.exemplo.com.br` | PROD (`222222222222`) | CloudFront (HTTPS) |

![Objetivos](imagens/imagem(21).png)

### URLs de acesso

| Ambiente | URL |
|---|---|
| DEV | `https://meusite.dev.exemplo.com.br` |

![Objetivos](imagens/imagem(24).png)

| PROD | `https://meusite.prod.exemplo.com.br` |

![Objetivos](imagens/imagem(23).png)

---

## Segurança PROD — ALB protegido

O ALB de PROD não é acessível diretamente pela internet. Duas camadas de proteção:

**1. Security Group (nível de rede)**
O SG do ALB PROD usa o AWS Managed Prefix List `pl-xxxxxxxx` (`com.amazonaws.global.cloudfront.origin-facing`).
Apenas IPs de origem do CloudFront conseguem abrir conexão TCP na porta 80.
A AWS mantém essa lista automaticamente — sem necessidade de atualização manual de IPs.

**2. Header secreto (nível de aplicação)**
O CloudFront injeta o header `X-Origin-Verify: seu-segredo-origin-verify` em todas as requisições para o ALB.
O ALB tem uma regra que encaminha para o Target Group apenas se o header estiver presente com o valor correto.
Requisições sem o header recebem `403 Forbidden`.

Para rotacionar o segredo: altere `OriginVerifySecret` nos stages ALB e CloudFront em `pipeline-infra-prod.yaml` e re-deploy.

---

## Configuração do container

| Propriedade | Valor |
|---|---|
| Nome do container | `meusite` |
| Porta da aplicação | `8080` |
| Repositório ECR | `meusite-app` |
| Framework | .NET 8 |
| CPU | 256 |
| Memória | 512 MB |
| Launch type | FARGATE |
| Subnets | Privadas (NAT Gateway para ECR e CloudWatch) |
| Circuit Breaker | Desabilitado |

![Objetivos](imagens/imagem(25).png)
![Objetivos](imagens/imagem(28).png)
![Objetivos](imagens/imagem(26).png)
![Objetivos](imagens/imagem(27).png)



### Auto Scaling (PROD)

| Configuração | Valor |
|---|---|
| Mínimo de tasks | 1 |
| Máximo de tasks | 4 |
| Métrica | CPU média do serviço |
| Target | 70% |
| Scale out cooldown | 60 segundos |
| Scale in cooldown | 300 segundos |

DEV não tem Auto Scaling — para economizar custo use `desired-count 0` quando não estiver em uso.

---

## Parâmetros por ambiente

### dev.json (referência — não lido pelos pipelines)

| Parâmetro | Valor |
|---|---|
| Environment | `dev` |
| DesiredCount | `0` |
| AccountId | `111111111111` |
| DomainName | `meusite.dev.exemplo.com.br` |
| AcmCertificateArn | `arn:aws:acm:us-east-1:111111111111:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |

### prod.json (referência — não lido pelos pipelines)

| Parâmetro | Valor |
|---|---|
| Environment | `prod` |
| DesiredCount | `0` |
| AccountId | `222222222222` |
| DomainName | `meusite.prod.exemplo.com.br` |
| AcmCertificateArn | `arn:aws:acm:us-east-1:222222222222:certificate/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy` |

> Os CIDRs de VPC e subnets estão fixos no template `1-vpc.yaml` via `Mappings` — DEV usa `10.1.0.0/16` e PROD usa `10.2.0.0/16`. Os pipelines passam apenas `Environment` como parâmetro para o stage VPC.

---

## Estrutura do repositório

```
meusite/                                      ← Aplicação ASP.NET Core .NET 8
buildspec.yml                                 ← dotnet restore → dotnet publish → docker build → push ECR
infra/
  cloudformation/
    1-vpc.yaml                                ← VPC + subnets públicas/privadas + NAT Gateway
                                                CIDRs fixos no template via Mappings por ambiente
                                                DEV: 10.1.0.0/16 | PROD: 10.2.0.0/16
    2-security-groups.yaml                    ← SGs: ALB e ECS Tasks
                                                DEV:  ALB aceita HTTP 80 (redireciona para HTTPS) e HTTPS 443 de qualquer origem
                                                PROD: ALB aceita HTTP 80 apenas dos IPs do CloudFront (Managed Prefix List pl-xxxxxxxx)
    3-ecr.yaml                                ← ECR Repository (meusite-app) + EmptyOnDelete
    4-ecs-cluster.yaml                        ← ECS Cluster Fargate
    5-alb.yaml                                ← ALB + Target Group
                                                DEV:  Listener HTTP 80 (redirect 301 para HTTPS) + Listener HTTPS 443 com certificado ACM
                                                PROD: Listener HTTP 80 com validacao header X-Origin-Verify
    6-ecs-service.yaml                        ← ECS Task Definition + Service (DesiredCount=0 na criacao)
                                                PROD: Application Auto Scaling habilitado (min 1, max 4, CPU 70%)
                                                DEV:  Sem Auto Scaling
    7-cloudfront.yaml                         ← CloudFront (PROD only, conta PROD)
                                                Envia header X-Origin-Verify para o ALB
    8-route53.yaml                            ← Registro DNS Alias na propria conta
                                                DEV:  meusite.dev.exemplo.com.br → ALB
                                                PROD: meusite.prod.exemplo.com.br → CloudFront
    parameters/
      dev.json                                ← Referencia de parametros DEV (nao lido pelos pipelines)
      prod.json                               ← Referencia de parametros PROD (nao lido pelos pipelines)
  codepipeline/
    buildspecs/
      buildspec-ecs-deploy-dev.yml            ← Referencia — buildspec do Deploy_ECS esta inline no pipeline
      buildspec-ecs-deploy-prod.yml           ← Referencia — buildspec do Deploy_ECS esta inline no pipeline
    pipelines/
      pipeline-infra-dev.yaml                 ← Pipeline infra DEV: 100% CloudFormation nativo, 8 stages
      pipeline-app-dev.yaml                   ← Pipeline app DEV: Docker build + ECS deploy
      pipeline-infra-prod.yaml                ← Pipeline infra PROD: 100% CloudFormation nativo, 9 stages
      pipeline-app-prod.yaml                  ← Pipeline app PROD: Docker build + ECS deploy
.github/
  workflows/
    deploy-infra.yml                          ← Sobe zip para S3, cria/atualiza pipeline e dispara infra
    deploy-app.yml                            ← Sobe zip para S3 e dispara o pipeline de app
```

---

## Primeiro deploy — passo a passo

### Pré-requisitos

1. **Roles OIDC** criadas manualmente em DEV e PROD com trust policy para GitHub Actions
2. **Hosted Zones delegadas** — `dev.exemplo.com.br` na conta DEV e `prod.exemplo.com.br` na conta PROD, com registros NS criados na conta MASTER em `exemplo.com.br`
3. **Certificados ACM** emitidos:
   - DEV: `*.dev.exemplo.com.br` na conta DEV (`us-east-1`)
   - PROD: `*.prod.exemplo.com.br` na conta PROD (`us-east-1`)
4. **Service-linked roles** — criadas automaticamente pelo workflow no primeiro deploy

![Objetivos](imagens/imagem(13).png)

### Deploy de infraestrutura

```
GitHub → Actions → Deploy Infrastructure → Run workflow
  environment: dev   (ou prod)
  action: deploy
```

O workflow:
1. Busca o ID da Hosted Zone (`dev.exemplo.com.br` ou `prod.exemplo.com.br`) na conta alvo
2. Cria bucket S3 `meusite-codepipeline-{env}-us-east-1` se não existir
3. Faz zip do repo e upload para S3
4. Cria/atualiza stack `meusite-{env}-infra-codepipeline` passando o `HostedZoneId`
5. Dispara `meusite-{env}-infra-pipeline` automaticamente

![Objetivos](imagens/imagem(11).png)

### Deploy de aplicação

```
GitHub → Actions → Deploy Application → Run workflow
  environment: dev   (ou prod)
```
![Objetivos](imagens/imagem(12).png)

### Ordem correta no primeiro deploy

```
1. deploy-infra.yml → cria toda a infraestrutura incluindo Route53
2. deploy-app.yml   → build Docker + push ECR + sobe ECS Service (desired-count 1)
```

---

## Operações comuns

### Ligar / desligar DEV

```bash
# Ligar
aws ecs update-service \
  --cluster meusite-dev-cluster \
  --service meusite-dev-service \
  --desired-count 1 --region us-east-1

# Desligar (economiza custo — NAT Gateway continua cobrado)
aws ecs update-service \
  --cluster meusite-dev-cluster \
  --service meusite-dev-service \
  --desired-count 0 --region us-east-1
```

### Rollback de versão

```bash
# Ver revisões disponíveis
aws ecs list-task-definitions --family-prefix meusite-{env}-task --region us-east-1

# Fazer rollback para revisão específica
aws ecs update-service \
  --cluster meusite-{env}-cluster \
  --service meusite-{env}-service \
  --task-definition meusite-{env}-task:{REVISION} \
  --region us-east-1
```

### Invalidar cache CloudFront (PROD)

```bash
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```

### Destruir infraestrutura

```
GitHub → Actions → Deploy Infrastructure → Run workflow
  environment: dev ou prod
  action: delete
```

Remove tudo nesta ordem:
1. Route53 DEV / Route53 + CloudFront PROD (na própria conta)
2. ECS Service → ALB → ECS Cluster → ECR → SGs → VPC
3. Stacks dos pipelines (app + infra)
4. Bucket S3 (esvazia objetos + versões + delete markers + deleta)
5. Task definitions ECS (todas as revisões)
6. CloudWatch Log Groups

![Objetivos](imagens/imagem(32).png)
![Objetivos](imagens/imagem(31).png)

---

## Troubleshooting

**Stack em ROLLBACK_COMPLETE**
O workflow detecta e deleta automaticamente antes de recriar. Manual:
```bash
aws cloudformation delete-stack --stack-name <STACK> --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name <STACK> --region us-east-1
```

**ECR com imagens bloqueando delete**
O template usa `EmptyOnDelete: true` — o CloudFormation esvazia automaticamente.
Se a stack já está em `DELETE_FAILED`, esvazie manualmente:
```bash
aws ecr delete-repository --repository-name meusite-app --force --region us-east-1
```

**`Could not assume role with OIDC: Request ARN is invalid`**
A trust policy da role OIDC está restringindo a branch errada. Altere o `sub` para:
```
repo:seu-usuario/SeuRepositorio:*
```

**ECS Service não estabiliza após deploy da app**
- Verifique CloudWatch Logs: `/ecs/meusite-{env}-app`
- Verifique Target Group health: Console → EC2 → Target Groups → `meusite-{env}-tg`
- O nome do container no `imagedefinitions.json` deve ser `meusite`

**`Container.image should not be null or empty`**
O `imagedefinitions.json` não foi encontrado. Verifique se o stage `Docker_Build_Push_ECR` gerou o artefato `BuildOutput`.

**ALB PROD retorna 403**
Acesso direto ao DNS do ALB sem passar pelo CloudFront. Acesse sempre via `https://meusite.prod.exemplo.com.br`.

**`error: 'vN' is not a valid version string` no CodeBuild**
A variável de ambiente `VERSION` conflita com o MSBuild/NuGet que a usa internamente como versão do pacote.
O buildspec usa `IMAGE_TAG` em vez de `VERSION` para evitar esse conflito.

---

## Custos estimados (por ambiente ativo)

| Recurso | Custo aproximado/mês |
|---|---|
| NAT Gateway | ~$32 + $0.045/GB |
| ALB | ~$16 + $0.008/LCU-hora |
| ECS Fargate (1 task, 256CPU/512MB) | ~$10 |
| CloudFront (PROD) | Gratuito até 1TB |
| ECR | $0.10/GB armazenado |

**Dica:** Rode `delete` após os testes para evitar cobrança do NAT Gateway.

---

## Observações

- `./meusite` é imutável — nenhum pipeline modifica seus arquivos.
- O build .NET (`dotnet restore` + `dotnet publish`) é feito no CodeBuild, não no Dockerfile. Framework: .NET 8.
- DEV: HTTP redireciona para HTTPS no ALB. Certificado `*.dev.exemplo.com.br`.
- PROD: HTTPS terminado no CloudFront, ALB recebe HTTP apenas de IPs do CloudFront. Auto Scaling min 1 / max 4.
- DEV e PROD iniciam com `DesiredCount=0`. O pipeline de app sobe para `1` automaticamente.
- Imagens ECR são tagueadas como `latest` e `v{BUILD_NUMBER}` a cada build.
- CIDRs: DEV `10.1.0.0/16`, PROD `10.2.0.0/16`.
- Variável `VERSION` não deve ser usada no buildspec — conflita com o MSBuild/NuGet. Use `IMAGE_TAG`.

![Objetivos](imagens/imagem(29).png)
![Objetivos](imagens/imagem(30).png)
