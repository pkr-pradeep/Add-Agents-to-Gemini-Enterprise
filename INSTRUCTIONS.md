# Add Agents to Gemini Enterprise: Challenge Lab Guide (GENAI149)

This comprehensive guide details step-by-step instructions, shell commands, console actions, and conceptual explanations for completing the **Add Agents to Gemini Enterprise** challenge lab.

---

## 📋 Table of Contents
1. [Overview & Architecture](#overview--architecture)
2. [Lab Environment Variables](#lab-environment-variables)
3. [Task 1: Install ADK and Set Up Your Environment](#task-1-install-adk-and-set-up-your-environment)
4. [Task 2: Create a No-Code ADK Agent and Deploy to Agent Runtime](#task-2-create-a-no-code-adk-agent-and-deploy-to-agent-runtime)
5. [Task 3: Set Up OAuth Consent Screen & Create OAuth Client](#task-3-set-up-oauth-consent-screen--create-oauth-client)
6. [Task 4: Deploy Gemini Enterprise and Enable Agent Designer](#task-4-deploy-gemini-enterprise-and-enable-agent-designer)
7. [Task 5: Add a No-Code Agent with Agent Designer](#task-5-add-a-no-code-agent-with-agent-designer)
8. [Task 6: Add the ADK Agent (Agent Runtime) to Gemini Enterprise](#task-6-add-the-adk-agent-agent-runtime-to-gemini-enterprise)
9. [Task 7: Test and Chat with Your Deployed Brand Voice Agent](#task-7-test-and-chat-with-your-deployed-brand-voice-agent)

---

## 🏛️ Overview & Architecture

In this lab, you integrate conversational AI agents into **Gemini Enterprise** (the unified workplace AI assistant). You deploy two complementary agent types:
1. **No-Code Web Agent (Agent Designer)**: Created natively within Gemini Enterprise UI and grounded with Google Search.
2. **ADK Agent on Agent Runtime (Agent Engine)**: Configured declaratively with YAML using Google's **Agent Development Kit (ADK)** and hosted on **Vertex AI Agent Engine (Reasoning Engine)**, secured via **OAuth 2.0**.

```mermaid
flowchart LR
    User([User in Gemini Enterprise UI]) --> Auth[OAuth 2.0 Consent / Token]
    User --> AD[Agent Designer: Pool Robot Researcher + Search]
    User --> GE[Gemini Enterprise Assistant]
    GE --> AE[Vertex AI Agent Engine / Reasoning Engine]
    AE --> BV[Brand Voice ADK Agent (Gemini 3.5 Flash)]
```

---

## ⚙️ Lab Environment Variables

Keep these project parameters handy throughout the lab:

| Parameter | Value |
| :--- | :--- |
| **GCP Project ID** | `qwiklabs-gcp-04-a2f7b4f9e952` |
| **Student Username** | `student-01-058018f125a2@qwiklabs.net` |
| **Deployment Region** | `us-east4` |
| **Staging Bucket** | `gs://qwiklabs-gcp-04-a2f7b4f9e952-bucket` |
| **Redirect URI** | `https://vertexaisearch.cloud.google.com/static/oauth/oauth.html` |
| **OAuth Token URI** | `https://oauth2.googleapis.com/token` |

---

## Task 1: Install ADK and Set Up Your Environment

### Objective
Set up Cloud Shell Terminal with Python dependencies, install the Google Agent Development Kit (ADK), and configure `gcloud`.

### Steps & Commands

1. **Download the lab assets from Cloud Storage**:
   ```bash
   gcloud storage cp -r gs://qwiklabs-gcp-04-a2f7b4f9e952-bucket/adk_challenge_lab .
   cd adk_challenge_lab
   ```

2. **Update PATH and install dependencies**:
   ```bash
   export PATH=$PATH:"/home/${USER}/.local/bin"
   python3 -m pip install -r requirements.txt
   ```

3. **Set the active Google Cloud project**:
   ```bash
   gcloud config set project qwiklabs-gcp-04-a2f7b4f9e952
   ```

### 💡 Why are we doing this?
- `requirements.txt` installs `google-cloud-aiplatform[agent_engines,adk]`, `cloudpickle`, and `google-adk[a2a]`. These libraries provide the `adk` CLI tool and Vertex AI serialization tools required to package and upload agents.
- Adding `/home/${USER}/.local/bin` to `PATH` ensures the newly installed `adk` command is directly executable from anywhere in the shell.

---

## Task 2: Create a No-Code ADK Agent and Deploy to Agent Runtime

### Objective
Generate an agent template using ADK's declarative Agent Config (`root_agent.yaml`), define the persona for "Cymbal Pools Brand Voice", and deploy it to Vertex AI Agent Engine.

### Steps & Commands

1. **Scaffold the Agent Config directory**:
   ```bash
   adk create \
     --type=config \
     --project qwiklabs-gcp-04-a2f7b4f9e952 \
     --region global \
     --model gemini-3.5-flash \
     brand_voice
   ```
   > **Note**: This automatically generates `brand_voice/.env` and `brand_voice/root_agent.yaml`.

2. **Configure the agent persona in `brand_voice/root_agent.yaml`**:
   Open `brand_voice/root_agent.yaml` and update `description` and `instruction`:
   ```yaml
   # yaml-language-server: $schema=https://raw.githubusercontent.com/google/adk-python/refs/heads/main/src/google/adk/agents/config_schemas/AgentConfig.json
   name: root_agent
   description: Rewrites content into the Cymbal Pools brand voice.
   instruction: >
     Rewrite text provided to you into the laid-back tone
     of a surfer dude. Include pool puns where possible.
     End each message with a goal of getting outside into
     the sun and water. Periodically add reminders to
     stay hydrated and wear sunscreen.
   model: gemini-3.5-flash
   ```

3. **Deploy the agent to Agent Engine (Agent Runtime)**:
   ```bash
   adk deploy agent_engine \
     --display_name "Cymbal Pools Brand Voice" \
     --region us-east4 \
     --staging_bucket gs://qwiklabs-gcp-04-a2f7b4f9e952-bucket \
     brand_voice
   ```

4. **Record the Deployed Resource Name**:
   When deployment completes (usually 5–7 minutes), copy the output resource name:
   ```text
   projects/<PROJECT_NUMBER>/locations/us-east4/reasoningEngines/<REASONING_ENGINE_ID>
   ```
   *(Example: `projects/275193795176/locations/us-east4/reasoningEngines/7434902025067823104`)*

### 💡 Why are we doing this?
- **No-Code ADK (`--type=config`)**: Allows defining agent prompts, models, and behavior purely in YAML without needing custom Python wrappers.
- **Agent Engine (Reasoning Engine)**: Managed serverless runtime on Google Cloud that securely encapsulates LLM execution, handles state management, and exposes a standardized gRPC/REST API endpoint.
- **Resource Name**: Gemini Enterprise requires this exact resource string to target and query the model.

---

## Task 3: Set Up OAuth Consent Screen & Create OAuth Client

### Objective
Configure Google Cloud OAuth 2.0 infrastructure so that Gemini Enterprise users can securely authorize access to backend reasoning engines.

### Steps in Google Cloud Console

1. **Configure OAuth Consent Screen**:
   - In Cloud Console, search for **Google Auth Platform** (or **APIs & Services > OAuth consent screen**).
   - Click **Get Started** or configure:
     - **App Name**: `Cymbal Pools Auth`
     - **User support email**: `student-01-058018f125a2@qwiklabs.net`
     - **Audience**: Select **Internal** (limits authentication to organizational domain users).
     - **Developer Contact Email**: `student-01-058018f125a2@qwiklabs.net`
   - Complete and save the configuration.

2. **Create OAuth 2.0 Web Client ID**:
   - Go to **APIs & Services > Credentials** (or **Clients** in Google Auth Platform).
   - Click **+ Create Credentials > OAuth client ID**.
   - Fill in the required fields:
     - **Application type**: `Web application`
     - **Name**: `Gemini Enterprise Client`
     - **Authorized redirect URIs**: Add:
       ```
       https://vertexaisearch.cloud.google.com/static/oauth/oauth.html
       ```
   - Click **Create**.
   - **Save both values**:
     - `Client ID`: (e.g., `XXXXXXXXXX.apps.googleusercontent.com`)
     - `Client Secret`: (e.g., `GOCSPX-XXXXXXXXXX`)

### 💡 Why are we doing this?
- Gemini Enterprise acts as an OAuth client. To query the reasoning engine on behalf of a specific user, Google requires an OAuth 2.0 authorization code flow.
- The redirect URI `https://vertexaisearch.cloud.google.com/static/oauth/oauth.html` is the static callback handler where Google Identity returns the auth code to Gemini Enterprise.

---

## Task 4: Deploy Gemini Enterprise and Enable Agent Designer

### Objective
Create and initialize the central Gemini Enterprise application and turn on support for custom conversational and workflow agents.

### Steps in Google Cloud Console

1. Navigate to **Gemini Enterprise** (search for `Gemini Enterprise` in the top search bar).
2. Click **Start 30-day free trial** and select **Continue** to activate the required APIs.
3. Deploy an application with the following parameters:
   - **Name**: `Cymbal Pools GE`
   - **Location**: `Global`
   - **Company name**: `Cymbal Pools`
   - **Identity Provider**: Select `Global Google Identity`.
4. Once the application is created, go to **Settings / Features** and enable:
   - ✅ **Chat Agents**
   - ✅ **Workflow Agents**

### 💡 Why are we doing this?
- Gemini Enterprise is Google Cloud's enterprise gen-AI workspace.
- Enabling Chat and Workflow Agents allows the assistant to invoke external reasoning engines, agents, and custom tools directly from user chat prompts.

---

## Task 5: Add a No-Code Agent with Agent Designer

### Objective
Build an ad-hoc informational agent natively within Gemini Enterprise using natural language instructions and grounded web search.

### Steps in Gemini Enterprise UI

1. Open the **Web Application Preview** of your `Cymbal Pools GE` app.
2. From the left navigation menu, click **Agents** > **Create Agent** (via **Agent Designer**).
3. Use the following natural language creation prompt:
   ```text
   Keep me informed of the meaningful updates and differences between models in new pool robot innovations.
   ```
4. Verify that the **Google Search** connector is enabled (to allow real-time grounding).
5. Click **Create** to deploy the agent.
6. Test the agent by opening a chat and submitting:
   ```text
   What is the latest in pool robot technology, and is it a meaningful improvement over last year's robots?
   ```

### 💡 Why are we doing this?
- Agent Designer allows non-technical team members to build specialized, domain-specific research assistants using natural language without writing code or deployment manifests.
- Grounding with Google Search ensures responses reference current market products rather than stale training cutoff data.

---

## Task 6: Add the ADK Agent (Agent Runtime) to Gemini Enterprise

### Objective
Encode the authorization parameters into an Authorization URI and register your deployed Agent Engine into Gemini Enterprise.

### Steps & Commands

1. **Construct the Authorization URI**:
   - Open `adk_challenge_lab/construct_auth_uri.py`.
   - Update line 24 with your OAuth Client ID from Task 3:
     ```python
     OAUTH_CLIENT_ID = "YOUR_CLIENT_ID_HERE"
     ```
   - Run the script in Cloud Shell:
     ```bash
     cd ~/adk_challenge_lab
     python3 construct_auth_uri.py
     ```
   - Copy the printed URL output:
     ```text
     https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=https%3A%2F%2Fvertexaisearch.cloud.google.com%2Fstatic%2Foauth%2Foauth.html&include_granted_scopes=true&response_type=code&access_type=offline&prompt=consent&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.profile%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email
     ```

2. **Create Authorization in Gemini Enterprise**:
   - In Cloud Console > **Gemini Enterprise** > **Agents** menu > click **Add Agent** > choose **Custom agent via Agent Runtime**.
   - Create a new authorization with:
     - **Authorization name**: `Brand Voice Auth`
     - **Client ID**: `[Your OAuth Web Client ID]`
     - **Client Secret**: `[Your OAuth Web Client Secret]`
     - **Token URI**: `https://oauth2.googleapis.com/token`
     - **Authorization URI**: `[The URL copied from construct_auth_uri.py]`

3. **Register Agent Settings**:
   - Configure the agent details:
     - **Agent name**: `Brand Voice Agent`
     - **Agent description**: `Rewrites content into the Cymbal Pools brand voice.`
     - **Agent Engine reasoning engine**: `projects/<PROJECT_NUMBER>/locations/us-east4/reasoningEngines/<REASONING_ENGINE_ID>` *(from Task 2)*
   - Save and create the agent.

4. **Update Permissions**:
   - In the agents list, select **Brand Voice Agent** > **Permissions**.
   - Grant access to **All Users**.

### 💡 Why are we doing this?
- **URL Encoding**: OAuth authorization endpoints require special query characters (`:`, `/`, spaces) to be percent-encoded (`%3A`, `%2F`, `%20`) so that Google Accounts processes query parameters cleanly without truncation.
- **All Users Permission**: Makes the agent discoverable and executable by any employee in the workspace.

---

## Task 7: Test and Chat with Your Deployed Brand Voice Agent

### Objective
Authorize the agent in the Gemini Enterprise workspace chat and verify the persona translation.

### Steps in Gemini Enterprise UI

1. Refresh the Gemini Enterprise web application tab and navigate to **Agents**.
2. Click the three-dot options menu (`⋮`) on **Brand Voice Agent** and select **Pin** to add it to the quick-access sidebar.
3. Open a chat with **Brand Voice Agent** and send:
   ```text
   hello
   ```
4. A consent prompt will appear. Click **Authorize**.
5. Select your Qwiklabs student account (`student-01-058018f125a2@qwiklabs.net`) and click **Allow**.
6. Send the test prompt:
   ```text
   Please rewrite in our brand voice: The part we needed to fix your pool filter has arrived.
   ```
7. Verify that the agent replies in the expected surfer-dude tone with pool puns and sun/hydration reminders!

### 💡 Why are we doing this?
- The initial `hello` triggers the OAuth 2.0 handshake, exchanging the authorization code for access and refresh tokens.
- Once authorized, user queries flow directly from Gemini Enterprise to the Vertex AI Agent Engine reasoning engine, which returns the stylized response.
