# TODO

export RANDOM_ID="$(openssl rand -hex 3)"
export RG_NAME="myPostgresResourceGroup$RANDOM_ID"
export REGION="centralus"

export POSTGRES_SERVER_NAME="mydb$RANDOM_ID"
export PGHOST="${POSTGRES_SERVER_NAME}.postgres.database.azure.com"
export PGUSER="dbadmin$RANDOM_ID"
export PGPORT=5432
export PGDATABASE="azure-ai-demo"
export PGPASSWORD="$(openssl rand -base64 32)"

export OPEN_AI_SERVICE_NAME="openai-service-$RANDOM_ID"
export EMBEDDING_MODEL="text-embedding-ada-002"
export CHAT_MODEL="gpt-4-turbo-2024-04-09"

az group create \
    --name $RG_NAME \
    --location $REGION \
    --subscription $SUBSCRIPTION_ID

az postgres flexible-server create \
    --admin-password $PGPASSWORD \
    --admin-user $PGUSER \
    --location $REGION \
    --name $POSTGRES_SERVER_NAME \
    --database-name $PGDATABASE \
    --resource-group $RG_NAME \
    --sku-name Standard_B2s \
    --storage-auto-grow Disabled \
    --storage-size 32 \
    --tier Burstable \
    --version 16 \
    --yes -o JSON \
    --public-access 0.0.0.0

az postgres flexible-server parameter set \
    --resource-group $RG_NAME \
    --subscription $SUBSCRIPTION_ID \
    --server-name $POSTGRES_SERVER_NAME \
    --name azure.extensions --value azure_ai,vector

az cognitiveservices account create \
    --name $OPEN_AI_SERVICE_NAME \
    --resource-group $RG_NAME \
    --location $REGION \
    --kind OpenAI \
    --sku s0 \
    --subscription $SUBSCRIPTION_ID

az cognitiveservices account deployment create \
    --name $OPEN_AI_SERVICE_NAME \
    --resource-group  $RG_NAME \
    --deployment-name $EMBEDDING_MODEL \
    --model-name $EMBEDDING_MODEL \
    --model-version "1"  \
    --model-format OpenAI \
    --sku-capacity "1" \
    --sku-name "Standard"

az cognitiveservices account deployment create \
    --name $OPEN_AI_SERVICE_NAME \
    --resource-group  $RG_NAME \
    --deployment-name $CHAT_MODEL \
    --model-name $CHAT_MODEL \
    --model-version "0125-Preview" \
    --model-format OpenAI \
    --sku-capacity "1" \
    --sku-name "Standard"

psql -c "CREATE EXTENSION IF NOT EXISTS vector;"

psql \
    -c "CREATE TABLE embeddings(id int PRIMARY KEY, data text, embedding vector(1536));" \
    -c "CREATE INDEX ON embeddings USING hnsw (embedding vector_ip_ops);"

pip install -r requirements
python chat.py --populate
