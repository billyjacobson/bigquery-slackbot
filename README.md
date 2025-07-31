# BigQuery Slack Bot

Connect your team to your marketing, sales, analytics data and more with natural langauge in the chat app you're already using. Put their common queries directly in their hands in a code-free secure system.

--

This project shows how to set up a Slack bot with access to BigQuery data. It uses an n8n.io workflow to connect Slack to an [MCP Toolbox](https://googleapis.github.io/genai-toolbox/) server allowing for a secure connection to BigQuery for only the queries you want to allow. 

Run this on a hosted n8n deployment with your MCP Toolbox server on Cloud Run or try this out on a local deployment.

This bot uses the [public `bigquery-public-data.thelook_ecommerce` dataset](https://console.cloud.google.com/marketplace/product/bigquery-public-data/thelook-ecommerce). Try it out first, then update it to use your data.

## Prerequisites

*   A Google Cloud Project with the BigQuery API enabled.
*   An [n8n.io](https://n8n.io/) deployment (cloud or self-hosted).
*   A Slack workspace with permissions to add apps.
*   Configured credentials for Slack in n8n. This can take 15-30 minutes to work through but I found these resources to be helpful:
    *   [Documentation to set up](https://docs.n8n.io/integrations/builtin/credentials/slack/)
    *   [Helpful video guide to set up](https://www.youtube.com/watch?v=222VOwDijz4)


### For a hosted deployment:

*   A Google Cloud Project with the Cloud Run API enabled.
*   [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) installed and configured on your local machine.


## Workflow Architecture

1. Slack Trigger: Listens for messages directed at the chatbot.
1. AI Agent: Sends the user's request to the AI model and receives a response or tools to use.
1. BigQuery MCP Client: Lists the available tools on top of the BigQuery database and executes the secure parameterized SQL queries against the data.
1. Slack Node: Sends the results to the user.


## Setup


### 1. Run your MCP Server

Deploy your MCP server to Cloud Run to access it via the internet or run it locally for a local n8n deployment.

#### Local Deployment

1. Download the MCP Toolbox for Databases

curl -O https://storage.googleapis.com/genai-toolbox/v0.10.0/linux/amd64/toolbox
chmod +x toolbox

1. Launch the server 

./toolbox --tools-file "tools.yaml"

#### Cloud Run Deployment

For more detailed steps, view [the deployment documentation](https://googleapis.github.io/genai-toolbox/how-to/deploy_toolbox/).

1. Set your project ID

export PROJECT_ID="my-project-id"


1.  Configure gcloud

gcloud init
gcloud config set project $PROJECT_ID

    
1.  Ensure all necessary APIs are enabled

gcloud services enable run.googleapis.com \
                       cloudbuild.googleapis.com \
                       artifactregistry.googleapis.com \
                       iam.googleapis.com \
                       secretmanager.googleapis.com


1. Create a service account to access the MCP server

gcloud iam service-accounts create toolbox-identity
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:toolbox-identity@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/secretmanager.secretAccessor 
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:toolbox-identity@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/bigquery.user

1. Put the tools.yaml into a secret

gcloud secrets create tools --data-file=tools.yaml

1. Deploy to Cloud Run

export IMAGE=us-central1-docker.pkg.dev/database-toolbox/toolbox/toolbox:latest
gcloud run deploy toolbox \
    --image $IMAGE \
    --service-account toolbox-identity \
    --region us-central1 \
    --set-secrets "/app/tools.yaml=tools:latest" \
    --args="--tools-file=/app/tools.yaml","--address=0.0.0.0","--port=8080"



1. After deployment, the command will output a Service URL. Save this URL for the next step.


### 1. Configure the n8n Workflow

1. Create a new n8n workflow

1. Import the workflow.json

1. Set your credentials for each node (Slack and your AI model)

1. BigQuery MCP Client node, replace `YOUR_MCP_TOOLBOX_URL` with localhost:8080 or the Service URL you copied from the Cloud Run deployment.

1.  Activate the workflow.


## Usage

Invite the bot to a channel in your Slack workspace. Mention the bot and ask it to perform a query. For example:

`@YourBotName SELECT name, retail_price FROM products ORDER BY retail_price DESC LIMIT 5`

The bot will use the MCP Toolbox server to execute the query against the `bigquery-public-data.thelook_ecommerce.products` table and post the results back in the channel.


### Here are the following supported queries and their parameters:

| Query name                             | Parameters                   |
|----------------------------------------|------------------------------|
| Monthly Sales                          | Year Month status            |
| Customers by Country                   | country gender               |
| Customers by Gender                    | Status Min revenue           |
| Customers by Age Group                 | Age group Gender             |
| Most/Least Profitable Brands           | Brand_param most/least limit |
| Most/Least Profitable Product Category | most/least limit             |