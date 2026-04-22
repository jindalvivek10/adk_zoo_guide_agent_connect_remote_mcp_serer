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

## 2. ADK Deploy vs. Traditional FastAPI

In this workflow, we use `adk deploy` instead of manually writing a `main.py` with FastAPI.
*   **Traditional Way**: You manually manage the server, imports, and web interface logic.
*   **New ADK Way**: The **ADK CLI** acts as the manager. The `adk deploy` command with the `--with-ui` flag automatically creates an optimized server logic and web interface "behind the scenes."

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

**Deployment Checklist:**
*   Select **Y** to create the Artifact Registry repository.
*   Select **y** to allow unauthenticated invocations (for testing purposes).
*   Note the **Service URL** provided at the end (e.g., `https://zoo-tour-guide-...-us-west1.run.app`).

---

## 7. Testing & Agent Flow

### How to Test
1.  Open the **Service URL** in your browser.
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
