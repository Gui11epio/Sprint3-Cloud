# MottuFind — Docker / ACR / ACI Deploy (files)

Este documento reúne os arquivos e instruções prontos para você copiar para o repositório: `Dockerfile` corrigido, `docker-compose.yml`, `container-group.yml` para ACI e um `README_deploy.md` com todos os comandos passo-a-passo.

> **Observação:** Substitua os placeholders (ex.: `motturegistry123`, senhas, nomes) antes de rodar os comandos.

---

## 1) Dockerfile (MottuFind/MottuFind/Dockerfile)
```dockerfile
# ---------- Build stage ----------
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# copia csproj e restauração
COPY MottuFind/*.sln ./
COPY MottuFind/MottuFind/*.csproj MottuFind/
# se tiver outros projetos no solution, copie-os também antes do restore

RUN dotnet restore MottuFind/MottuFind.csproj

# copy everything and publish
COPY MottuFind/ MottuFind/
WORKDIR /src/MottuFind
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish -c ${BUILD_CONFIGURATION} -o /app/publish

# ---------- Runtime stage ----------
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
ARG APP_UID=1000
ENV APP_UID=${APP_UID}

WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# Copia os arquivos publicados
COPY --from=build /app/publish .

# Garante que a aplicação rode como non-root
# As imagens oficiais do aspnet costumam já ter um usuário 'app' com UID 1000.
# Se preferir criar explicitamente, descomente os comandos abaixo (opcional):
# RUN addgroup --system appgroup && adduser --system --ingroup appgroup --uid ${APP_UID} app

USER ${APP_UID}

ENTRYPOINT ["dotnet", "MottuFind_C_.API.dll"]
```

**Notas:**
- Ajuste o caminho do `COPY` para corresponder exatamente à estrutura do seu repositório se for diferente.
- `APP_UID` padrão = `1000` (usuario não-root). Você pode passar outro `APP_UID` no build se for necessário.

---

## 2) docker-compose.yml (na raiz do repositório)
```yaml
version: "3.8"
services:
  mysql:
    image: mysql:8.0
    container_name: mottu_mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: mottufind
      MYSQL_USER: mfuser
      MYSQL_PASSWORD: MfPassw0rd!
      MYSQL_ROOT_PASSWORD: RootPassw0rd!
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: MottuFind/MottuFind/Dockerfile
      args:
        APP_UID: 1000
        BUILD_CONFIGURATION: Release
    container_name: mottufind_api
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "8080:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: "Production"
      # ConnectionStrings:DefaultConnection (ASP.NET Core) via env var
      ConnectionStrings__DefaultConnection: "Server=mysql;Port=3306;Database=mottufind;User=mfuser;Password=MfPassw0rd!;TreatTinyAsBoolean=true;"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

**Notas:**
- A variável `ConnectionStrings__DefaultConnection` usa `__` para mapear `ConnectionStrings:DefaultConnection` do ASP.NET Core.
- Ajuste `healthcheck` conforme sua rota de health endpoint.

---

## 3) container-group.yml (para `az container create --file container-group.yml`)
```yaml
apiVersion: 2019-12-01
location: brazilsouth
name: mottu-group
properties:
  containers:
  - name: mysql
    properties:
      image: motturegistry123.azurecr.io/library/mysql:8.0
      resources:
        requests:
          cpu: 0.5
          memoryInGb: 1
      environmentVariables:
      - name: MYSQL_ROOT_PASSWORD
        value: RootPassw0rd!
      - name: MYSQL_DATABASE
        value: mottufind
      - name: MYSQL_USER
        value: mfuser
      - name: MYSQL_PASSWORD
        value: MfPassw0rd!
  - name: api
    properties:
      image: motturegistry123.azurecr.io/mottufind/api:v1
      resources:
        requests:
          cpu: 1
          memoryInGb: 1.5
      ports:
      - port: 8080
      environmentVariables:
      - name: ConnectionStrings__DefaultConnection
        value: "Server=localhost;Port=3306;Database=mottufind;User=mfuser;Password=MfPassw0rd!;"
  osType: Linux
  ipAddress:
    type: Public
    ports:
    - protocol: tcp
      port: 8080
  restartPolicy: Always
```

**Importante:**
- `motturegistry123.azurecr.io` é um placeholder — substitua pelo seu `ACR_LOGIN_SERVER` real.
- Containers no *container group* compartilham rede — por isso a API pode se conectar ao MySQL via `localhost` dentro do mesmo grupo.
- Para persistência do banco, use Azure Files e monte um volume; ACI por si só não garante persistência robusta como uma VM ou um serviço PaaS.

---

## 4) README_deploy.md (passo-a-passo)
```markdown
# README — Deploy MottuFind com ACR e ACI

Este README contém os passos para construir localmente, subir imagens para o Azure Container Registry (ACR) e criar um Container Group no Azure Container Instances (ACI) com a sua API .NET e o MySQL.

## Pré-requisitos
- Azure CLI instalada e autenticada (`az login`).
- Docker instalado (se for build local) ou usar `az acr build`.
- Acesso ao repositório com os arquivos: `Dockerfile`, `docker-compose.yml`, `container-group.yml`.

## 1) Teste local com Docker Compose
```bash
# na raiz do repo (onde docker-compose.yml foi colocado)
docker compose build --pull
docker compose up -d
# ver logs da api
docker compose logs -f api
```
Verifique se a API responde: `http://localhost:8080/` (ou endpoint de health configurado).

## 2) Criar ACR e preparar imagens
```bash
# login no azure
az login

# criar resource group (exemplo)
az group create -n rg-mottufind -l brazilsouth

# criar ACR (nome único)
az acr create --resource-group rg-mottufind --name motturegistry123 --sku Basic --location brazilsouth

# login no acr (opcional)
az acr login --name motturegistry123

# tag e push (caso tenha construído localmente)
docker tag mottufind_api:local motturegistry123.azurecr.io/mottufind/api:v1
docker push motturegistry123.azurecr.io/mottufind/api:v1

# alternativa: build direto no ACR (recomendado quando preferir build na nuvem)
az acr build --registry motturegistry123 --image mottufind/api:v1 --file MottuFind/MottuFind/Dockerfile .
```

## 3) Fazer deploy no ACI (container group)
```bash
# recuperar credenciais do ACR para ACI poder puxar imagens privadas
ACR_NAME=motturegistry123
RG=rg-mottufind
ACR_LOGIN_SERVER=${ACR_NAME}.azurecr.io
ACR_USERNAME=$(az acr credential show -n $ACR_NAME --query "username" -o tsv)
ACR_PASSWORD=$(az acr credential show -n $ACR_NAME --query "passwords[0].value" -o tsv)

# criar container group via YAML
az container create --resource-group $RG --file container-group.yml --name mottu-group \
  --registry-login-server $ACR_LOGIN_SERVER --registry-username $ACR_USERNAME --registry-password $ACR_PASSWORD

# verificar status
az container show --resource-group $RG --name mottu-group --query "{state:properties.instanceView.state,containers:properties.containers[*].properties.instanceView.currentState}" -o json

# ver logs do container api
az container logs --resource-group $RG --name mottu-group --container-name api
```

## 4) Observações e recomendações
- **Segredos:** Não versionar senhas. Use Azure Key Vault e injete secrets nas variáveis de ambiente com Managed Identity quando possível.
- **Banco em produção:** Para produção, prefira **Azure Database for MySQL** (PaaS) ao invés de rodar MySQL em ACI.
- **Persistência:** Se precisar de persistência para MySQL no ACI, monte um share do Azure Files no container group.
- **Non-root:** A imagem da API foi configurada para rodar com `USER ${APP_UID}` (não-root). Garanta que `APP_UID` exista na imagem base (as imagens oficiais `aspnet` geralmente expõem o usuário `app` com uid 1000).

```

---

## 5) Pequenas dicas para ajustar no projeto
- Confirme a porta exposta pela sua API (`8080` no Dockerfile). Se a aplicação usa `ASPNETCORE_URLS` você pode setar `ASPNETCORE_URLS=http://+:8080` via env.
- Garanta que o `appsettings.json` não contenha connection strings sensíveis — prefira injetar via env var.
- Se sua solução tem múltiplos projetos referenciados, ajuste os comandos `COPY` no Dockerfile para incluir todos antes do `dotnet restore`.

---

> Se quiser, eu já posso:
> - Gerar um `secret` no ACR para esconder a senha do MySQL no `container-group.yml` (mostrando como usar Azure Key Vault/ACI).
> - Gerar a versão do `Dockerfile` que cria explicitamente o usuário `app` dentro da imagem runtime (em vez de confiar no `app` existente).


***FIM DO DOCUMENTO***

