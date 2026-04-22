# Multi-Agent Zoo Guide with Remote MCP Connection
Codelab - https://codelabs.developers.google.com/codelabs/cloud-run/use-mcp-server-on-cloud-run-with-an-adk-agent#6

This guide details the steps to create a multi-agent system containing sequential agent using the **Google ADK**, deploying it to **Cloud Run** using **adk deploy** , and connecting it to a remote **MCP (Model Context Protocol)** server and also other langchain tool like wikipedia.

---

## 1. Key Learning Points: Google Cloud IAM & Service Accounts

*   **Project ID as a Container**: Setting the `PROJECT_ID` via `gcloud config` acts as a **"digital workspace."** It ensures all resources (code, databases, AI models) are stored and secured correctly.
*   **Service Account as Identity**: A **Service Account** is a **"virtual employee"** for your code. It allows the agent to perform actions without human credentials.
*   **Permissions vs. Deployment**:
    *   **Deploying**: Requires **Admin** or **Developer** roles.
    *   **Invoking**: Requires the `roles/run.invoker` role to send HTTPS requests to private Cloud Run services.
*   **Least Privilege Principle**: Using a custom service account ensures the agent only has specific keys (e.g., `aiplatform.user` and `run.invoker`) rather than a **"Master Key"** (Editor role).
*   **The "Binding" Process**: **IAM Policy Binding** grants a specific role to a Service Account, which is then **"pinned"** to the agent during deployment.

---

## 2. ADK Deploy vs. Traditional FastAPI deploymentment to cloud RUN
Why this is different from the https://codelabs.developers.google.com/deploy-google-adk-agent-to-cloud-run?hl=en#0 where we deployed a simple fastAPI web app containing an agent into cloud run 


In this workflow, we use `adk deploy` instead of manually writing a `main.py` with FastAPI.
*   **Traditional Way - (Your FastAPI wrapped agent)**: You had to manually write the main.py, import the ADK library, and explicitly tell it to serve the web interface. You were the "manager" of the server.
*   **New ADK Way**: The **ADK CLI** acts as the manager. The `adk deploy` command with the `--with-ui` flag automatically creates an optimized server logic and web interface "behind the scenes."

When you use the `--with-ui flag` in the adk deploy command, it is doing the exact same thing as setting `web=True` in your old code—it's just doing it "behind the scenes" so you don't have to maintain a main.py file yourself.

---

## 3. Project Setup & Environment

### Create Directory & Set Project
```bash
mkdir adk_zoo_guide_agent_connect_remote_mcp_serer
cd adk_zoo_guide_agent_connect_remote_mcp_serer
gcloud config set project vjindal-project-ai-basic
gcloud projects list
```

### Enable Required APIs (Reference)
```bash
gcloud services enable \
    run.googleapis.com \
    artifactregistry.googleapis.com \
    cloudbuild.googleapis.com \
    aiplatform.googleapis.com
```

### Dependencies (`requirements.txt`)
```bash
vi requirements.txt
```
**Add the following:**
```text
google-adk==1.14.0
langchain-community==0.3.27
wikipedia==1.4.0
```

---

## 4. Identity & Access Management (IAM)

### Configure Environment Variables
```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export SA_NAME=lab2-cr-service
export SERVICE_ACCOUNT="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

echo $PROJECT_ID 
echo $PROJECT_NUMBER 
echo $SA_NAME 
echo $SERVICE_ACCOUNT 
```

### Create Service Account & Bind Roles
```bash
gcloud iam service-accounts create ${SA_NAME} --display-name="Service Account for lab 2 "

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/aiplatform.user"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/run.invoker"
```

### Setup Environment Configuration (`.env`)
```bash
cloudshell edit .env
echo -e "\nMCP_SERVER_URL=https://zoo-mcp-server-${PROJECT_NUMBER}.europe-west1.run.app/mcp" >> .env
```
**Note**: Ensure the URL matches the region (e.g., `europe-west1`) where your MCP server was previously deployed.

**Now the IAM page shows like this - 
lab2-cr-service@vjindal-project-ai-basic.iam.gserviceaccount.com	-> Service Account for lab 2	->	Cloud Run Invoker
	and Vertex AI User
**
---

## 5. Agent Development

### Initialize Package
```bash
cloudshell edit __init__.py
```
**Add:** `from . import agent`

### Create `agent.py`
Create the `agent.py` file to define your multi-agent system using **SequentialAgent**. This includes:
1.  **Zoo Greeter**: Captures user input.
2.  **Comprehensive Researcher**: Queries MCP and Wikipedia.
3.  **Response Formatter**: Synthesizes the final answer.

---

## 6. Deployment to Cloud Run

Deploy the agent using the ADK CLI. Since there is no custom FastAPI wrapper, **ADK** will spin up its own server.

```bash
uvx --from google-adk==1.14.0 \
adk deploy cloud_run \
  --project=$PROJECT_ID \
  --region=us-west1 \
  --service_name=zoo-tour-guide \
  --with_ui \
  . \
  -- \
  --labels=dev-tutorial=codelab-adk \
  --service-account=$SERVICE_ACCOUNT
```

**Output**
```
Start generating Cloud Run source files in /tmp/cloud_run_deploy_src/20260417_222634
Copying agent source code...
Copying agent source code completed.
Creating Dockerfile...
Creating Dockerfile complete: /tmp/cloud_run_deploy_src/20260417_222634/Dockerfile
Deploying to Cloud Run...
Deploying from source requires an Artifact Registry Docker repository to store built containers. A repository named 
[cloud-run-source-deploy] in region [us-west1] will be created.

**Done**.                                                                                                                   
Service [zoo-tour-guide] revision [zoo-tour-guide-00001-k49] has been deployed and is serving 100 percent of traffic.
Service URL: https://zoo-tour-guide-658050955671.us-west1.run.app

```

**Deployment Checklist:**
*   Select **Y** to create the Artifact Registry repository.
*   Select **y** to allow unauthenticated invocations (for testing purposes).
*   Note the **Service URL** provided at the end (e.g., `https://zoo-tour-guide-...-us-west1.run.app`).

---

## 7. Testing & Agent Flow

### How to Test
1.  Open the **Service URL** in your browser to access the ADK's web interface and interact with the agent..
2.  Toggle **Token Streaming** in the upper right.
3.  Type `hello` to trigger the **Greeter Agent**.
4.  Ask a complex question: *"Where can I find the polar bears in the zoo and what is their diet?"*

### Multi-Agent Architecture Explained
1.  **The Zoo Greeter**: Starts the conversation and uses `add_prompt_to_state` to save the user's intent before handing off to the workflow.
2.  **The Comprehensive Researcher**: The "brain." It decides whether to use the **MCP Server** (Internal Zoo Data) or the **Wikipedia API** (General Knowledge), or both.
3.  **The Response Formatter**: The "presenter." It takes raw **RESEARCH_DATA** and formats it into a friendly, conversational response.

---

## 8. Version Control: Pushing to GitHub

If using **Cloud Shell UI** for the first time:

1.  **Prepare**: Create a `.gitignore` (add `__pycache__/`, `.env`).
2.  **Initialize**: Click the **Source Control** icon -> **Initialize Repository**.
3.  **Commit**: Stage changes with `+` and click **Commit** with a message.
4.  **Publish**: Click **Publish Branch** -> Choose **Public** or **Private**.
5.  **Verify**: Follow the browser prompts to authorize and view your repo on GitHub.

## 9. Cleanup
```bash
gcloud run services delete zoo-tour-guide --region=us-west1 --quiet
gcloud artifacts repositories delete cloud-run-source-deploy --location=us-west1 --quiet
```

## Prompt for AI to update the README doc
I have followed the codelab and copied some technical information and the commands and steps I took to complete the codelab for ccreating a multi agent and deploying to cloud run and agent connecting to MCP server deployed already on cloud run.

Some rough instructions are in README.md file and I want you to do the following

Reorganize the entire content to make it more understandable and easy to comprehend

Don't delete any commands but only delete any repeated commands if done by mistake and remove any unncessary whitelines

hightlight all the commands using ``` escape quotes

Make all other important stuff as bold

Do any other important highlighting to make this a good looking readme file but make sure no content is deleted unncessarily from my readme.md

