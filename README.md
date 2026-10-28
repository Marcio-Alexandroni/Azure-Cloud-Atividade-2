# AV-ICL-02 — Análise de Sentimento com Azure Functions

Aplicação *serverless* na **Azure** que coleta comentários de clientes, avalia o sentimento com o **Azure AI Language** e consolida os totais em um banco **SQL Server**.

## Arquitetura

```
 backend-n75y.onrender.com/comentarios
               │  (GET a cada 10 s)
               ▼
 ┌──────────────────────────┐     analyzeSentiment     ┌─────────────────────┐
 │ timerComentarios (Timer) │ ───────────────────────► │  Azure AI Language  │
 └──────────────────────────┘                          └─────────────────────┘
               │  output binding
               ▼
     Storage Queue: fila-sentimentos
               │  trigger
               ▼
 ┌──────────────────────────────┐   SQL output binding  ┌──────────────────────┐
 │ processarSentimento (Queue)  │ ────────────────────► │ Azure SQL            │
 └──────────────────────────────┘                       │ dbo.SENTIMENTOS      │
                                                        └──────────────────────┘
                                                                   ▲
 ┌──────────────────────────────┐   SQL input binding              │
 │ totaisSentimentos (HTTP GET) │ ─────────────────────────────────┘
 └──────────────────────────────┘
        GET /api/sentimentos  →  { "positive": 0, "neutral": 0, "negative": 1 }
```

| Função | Gatilho | O que faz |
|---|---|---|
| `timerComentarios` | Timer `*/10 * * * * *` | Busca um comentário, analisa o sentimento e envia o resultado para a fila |
| `processarSentimento` | Storage Queue `fila-sentimentos` | Insere o resultado na tabela `dbo.SENTIMENTOS` |
| `totaisSentimentos` | HTTP GET `/api/sentimentos` | Retorna os totais por sentimento em JSON |

> O Azure AI Language foi criado no tier gratuito **F0** (5.000 análises/mês). Com o timer a cada 10 s, a cota dura cerca de 14 horas; após a coleta, o timer pode ser desativado (seção 10) sem afetar o endpoint de consulta.
>
> O Azure AI Language também pode retornar `mixed`. Como o enunciado pede apenas `positive`, `neutral` ou `negative`, nesses casos é gravada a classe com maior *confidence score*.

## Endpoint de entrega

```
https://func-sentimentos-1793228270.azurewebsites.net/api/sentimentos
```

Resposta verificada em 28/10/2026:

```json
{"positive":10,"neutral":5,"negative":8}
```

Evidência do teste de ponta a ponta (endpoint, fila e banco): [`evidencias/teste-ponta-a-ponta.jpg`](evidencias/teste-ponta-a-ponta.jpg)

## Estrutura do repositório

```
├── host.json
├── package.json
├── local.settings.example.json
├── evidencias/
│   └── teste-ponta-a-ponta.jpg
├── sql/
│   └── schema.sql
└── src/functions/
    ├── timerComentarios.js
    ├── processarSentimento.js
    └── totaisSentimentos.js
```

---

## Comandos executados para criar os recursos na Azure

Todos os comandos foram executados no **bash** do **Azure Cloud Shell**, na assinatura **Azure for Students**.

### 1. Login e variáveis de ambiente

```bash
az login
az account show --output table

az policy assignment list --query "[].parameters.listOfAllowedLocations.value[]" -o tsv

export AZURE_CORE_ONLY_SHOW_ERRORS=true
export LOCATION=brazilsouth
export SUFIXO=$(date +%s)

export RESOURCE_GROUP=rg-av-icl-02
export STORAGE_NAME=stsent$SUFIXO
export FUNCTION_APP_NAME=func-sentimentos-$SUFIXO
export LANGUAGE_NAME=lang-sentimentos-$SUFIXO
export SQL_SERVER=sqlsentimentos$SUFIXO
export SQL_DB=db
export SQL_ADMIN=adminuser
export SQL_PASSWORD='<senha-forte>'
export QUEUE_NAME=fila-sentimentos
```

### 2. Registro dos provedores e grupo de recursos

```bash
az provider register --namespace Microsoft.Web
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.CognitiveServices

az group create --name $RESOURCE_GROUP --location $LOCATION
```

### 3. Storage Account e fila

```bash
az storage account create \
  --name $STORAGE_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

export AZURE_STORAGE_CONNECTION_STRING=$(
  az storage account show-connection-string \
    --name $STORAGE_NAME \
    --resource-group $RESOURCE_GROUP \
    --query connectionString \
    --output tsv
)

az storage queue create \
  --name $QUEUE_NAME \
  --account-name $STORAGE_NAME \
  --connection-string $AZURE_STORAGE_CONNECTION_STRING
```

### 4. Azure AI Language (Cognitive Service)

```bash
az cognitiveservices account list-skus --location $LOCATION

az cognitiveservices account create \
  --name $LANGUAGE_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --kind TextAnalytics \
  --sku F0 \
  --yes

export LANGUAGE_ENDPOINT=$(az cognitiveservices account show \
  --name $LANGUAGE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query properties.endpoint \
  -o tsv)

export LANGUAGE_KEY=$(az cognitiveservices account keys list \
  --name $LANGUAGE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query key1 \
  -o tsv)

echo $LANGUAGE_ENDPOINT
```

### 5. Azure SQL Server, firewall e banco de dados

```bash
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $SQL_ADMIN \
  --admin-password $SQL_PASSWORD

az sql server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --server $SQL_SERVER \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

export MEU_IP=$(curl -s https://api.ipify.org)

az sql server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --server $SQL_SERVER \
  --name AllowCloudShell \
  --start-ip-address $MEU_IP \
  --end-ip-address $MEU_IP

az sql db create \
  --resource-group $RESOURCE_GROUP \
  --server $SQL_SERVER \
  --name $SQL_DB \
  --service-objective Basic \
  --backup-storage-redundancy Local

az sql db list --resource-group $RESOURCE_GROUP --server $SQL_SERVER --output table
```

### 6. Criação da tabela

```bash
wget https://github.com/microsoft/go-sqlcmd/releases/download/v1.10.0/sqlcmd-linux-amd64.tar.bz2
tar -xjf sqlcmd-linux-amd64.tar.bz2 sqlcmd

~/sqlcmd -S $SQL_SERVER.database.windows.net -d $SQL_DB -U $SQL_ADMIN -P "$SQL_PASSWORD" -i sql/schema.sql

~/sqlcmd -S $SQL_SERVER.database.windows.net -d $SQL_DB -U $SQL_ADMIN -P "$SQL_PASSWORD" \
  -Q "SELECT name FROM sys.tables"
```

### 7. Function App

```bash
az functionapp create \
  --resource-group $RESOURCE_GROUP \
  --consumption-plan-location $LOCATION \
  --runtime node \
  --runtime-version 22 \
  --functions-version 4 \
  --os-type Windows \
  --name $FUNCTION_APP_NAME \
  --storage-account $STORAGE_NAME

export SQL_CONNECTION_STRING="Server=tcp:$SQL_SERVER.database.windows.net,1433;Initial Catalog=$SQL_DB;User ID=$SQL_ADMIN;Password=$SQL_PASSWORD;Encrypt=True;"

az functionapp config appsettings set \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    "COMENTARIOS_URL=https://backend-n75y.onrender.com/comentarios" \
    "LANGUAGE_ENDPOINT=$LANGUAGE_ENDPOINT" \
    "LANGUAGE_KEY=$LANGUAGE_KEY" \
    "SqlConnectionString=$SQL_CONNECTION_STRING"

az functionapp show \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --query "{Name:name,State:state,Host:defaultHostName}" \
  --output table
```

### 8. Publicação do código

```bash
cd av-icl-02-sentimentos

npm install --omit=dev

zip -rq function.zip . -x "sql/*" "function.zip"

az functionapp deployment source config-zip \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --src function.zip

az functionapp function list \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --query "[].{Nome:name}" \
  --output table
```

### 9. Testes

```bash
HOSTNAME=$(az functionapp show \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --query defaultHostName \
  --output tsv)

echo "https://$HOSTNAME/api/sentimentos"

curl "https://$HOSTNAME/api/sentimentos"

az storage message peek \
  --queue-name $QUEUE_NAME \
  --connection-string $AZURE_STORAGE_CONNECTION_STRING \
  --num-messages 5

~/sqlcmd -S $SQL_SERVER.database.windows.net -d $SQL_DB -U $SQL_ADMIN -P "$SQL_PASSWORD" \
  -Q "SELECT TOP 10 Id, Sentimento, Confianca, Comentario FROM dbo.SENTIMENTOS ORDER BY Id DESC"
```

### 10. Desativar o timer e remover os recursos

```bash
az functionapp config appsettings set \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings "AzureWebJobs.timerComentarios.Disabled=true"

az functionapp config appsettings set \
  --name $FUNCTION_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings "AzureWebJobs.timerComentarios.Disabled=false"

az group delete --name $RESOURCE_GROUP --yes --no-wait
```

---

## Execução local (opcional)

```bash
npm i -g azure-functions-core-tools@4 azurite
azurite --skipApiVersionCheck --location ./data &

cp local.settings.example.json local.settings.json   # preencher endpoint, chave e connection string
npm install
func start
```

Endpoint local: `http://localhost:7071/api/sentimentos`
