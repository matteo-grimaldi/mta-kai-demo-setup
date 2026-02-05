# MTA + KAI on OpenShift: Installation & VS Code Setup

This guide provides a complete walkthrough for installing the **Migration Toolkit for Applications (MTA)** with **Konveyor AI (KAI)** on OpenShift and configuring the VS Code extension for AI-powered code analysis and migration assistance.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Step 1: Create MTA Namespace](#step-1-create-mta-namespace)
  - [Step 2: Install MTA Operator](#step-2-install-mta-operator)
  - [Step 3: Configure LLM Integration](#step-3-configure-llm-integration)
  - [Step 4: Deploy MTA with KAI](#step-4-deploy-mta-with-kai)
- [VS Code Configuration](#vs-code-configuration)
  - [Configure Solution Server](#configure-solution-server)
  - [Configure Provider Settings](#configure-provider-settings)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This setup connects three components:

1. **MTA Hub** - Runs on OpenShift, provides the migration analysis platform
2. **KAI Solution Server** - AI component that processes code analysis requests
3. **LLM Provider** - Large Language Model (via OpenShift AI MaaS, OpenAI, Azure, etc.)

The VS Code extension communicates with both the LLM model and the KAI Solution Server, which in turn uses itself the configured LLM to provide intelligent code migration suggestions for future use.

---

## Architecture

```
                          ┌─────────────────────┐
  41 +                    │   VS Code           │
  42 +                    │   MTA Extension     │
  43 +                    └──────┬──────────┬───┘
  44 +                           │          │
  45 +                 (Path 1)  │          │  (Path 2)
  46 +        Solution Server ───┘          └─── Direct LLM Access
  47 +                 HTTPS                       HTTPS
  48 +                           │           │
  49 +                           ▼           │
  50 +         ┌──────────────────────────┐  │
  51 +         │   OpenShift Cluster      │  │
  52 +         │                          │  │
   53 +        │  ┌──────────────────┐    │  │
   54 +        │  │  MTA Hub         │    │  │
   55 +        │  │  Route: /hub/    │    │  │
   56 +        │  │  services/kai/api│    │  │
   57 +        │  └────────┬─────────┘    │  │
  58 +         │           │              │  │
  59 +         │           ▼              │  │
  60 +         │ ┌──────────────────┐     │  │
  61 +         │ │ KAI Solution     │     │  │
  62 +         │ │ Server           │     │  │
  63 +         │ │ - kai-api        │     │  │
  64 +         │ │ - kai-db         │     │  │
  65 +         │ │ - kai-importer   │     │  │
  66 +         │ └────────┬─────────┘     │  │
  67 +         │          │               │  │
  68 +         └──────────┼─────────────-─┘  │
  69 +                    │                  │
  70 +                    │ HTTPS            │
  71 +                    │ (kai-api-keys)   │
  72 +                    ▼                  ▼
  73 +        ┌─────────────────────────────────────┐
  74 +        │   LLM Provider                      │
  75 +        │   - OpenShift AI MaaS               │
  76 +        │   - OpenAI                          │
  77 +        │   - Azure OpenAI                    │
  78 +        │   - AWS Bedrock                     │
  79 +        │   - Other OpenAI-compatible APIs    │
  80 +        └─────────────────────────────────────┘
```

**Authentication Flow:**
- VSCode - MTA - Solution server - LLM
   1. VS Code extension → KAI Solution Server (via MTA Hub route)
   2. KAI Solution Server → LLM Provider (using credentials from `kai-api-keys` secret)

- VSCode - LLM
   1. VS Code extension → LLM (using credentials from `provider-settings.yaml`)
---


## Prerequisites

### OpenShift Requirements
- **OpenShift Cluster:** Version 4.12 or higher
- **Cluster Admin Access:** Required to install operators
- **Internet Access:** For pulling operator images and accessing LLM APIs

### VS Code Requirements
- **VS Code:** Version 1.85.0 or higher
- **Java Extensions:** Version 1.45.0 or below (if analyzing Java applications)
- **MTA Extension:** [MTA 8.0.x VS Code Extension](https://marketplace.visualstudio.com/items?itemName=redhat.mta-vscode-extension)

### LLM Provider
- Access to one of the following:
  - **OpenShift AI MaaS** (used for this guide)
  - OpenAI API
  - Azure OpenAI
  - AWS Bedrock
  - Any OpenAI-compatible API endpoint

---

## Installation

### Step 1: Create MTA Namespace

Create the dedicated namespace for MTA:

```bash
oc apply -f 00-mta-namespace.yaml
```

**Verify:**
```bash
oc get namespace openshift-mta
```

### Step 2: Install MTA Operator

Install the OperatorGroup and Subscription:

```bash
oc apply -f 01-mta-operatorgroup.yaml
oc apply -f 02-mta-operator-subscription.yaml
```

**Verify the operator is running:**
```bash
oc get csv -n openshift-mta | grep mta-operator
```

Expected output:
```
mta-operator.v8.0.1   Migration Toolkit for Applications Operator   8.0.1   Succeeded
```

### Step 3: Configure LLM Integration

#### 3.1 Obtain LLM API Credentials

**For OpenShift AI MaaS:**

1. Log into your OpenShift AI cluster
2. Navigate to **Model-as-a-Service (MaaS)**
3. Select and subscribe to a model (e.g., `granite-3-2-8b-instruct`)
4. Go to **API Keys** page and create a new API key
5. Copy the **API URL** and **API Key**

**For other providers:**
- **OpenAI:** Get your API key from https://platform.openai.com/api-keys
- **Azure OpenAI:** Use your Azure endpoint and API key
- **AWS Bedrock:** Configure AWS credentials

#### 3.2 Create the LLM API Secret

**Option B: Using oc command directly**

```bash
oc create secret generic kai-api-keys -n openshift-mta \
  --from-literal=OPENAI_API_BASE='<YOUR_LLM_API_URL>' \
  --from-literal=OPENAI_API_KEY='<YOUR_API_KEY>'
```

**Verify the secret:**
```bash
oc get secret kai-api-keys -n openshift-mta
```

### Step 4: Deploy MTA with KAI

Deploy the Tackle custom resource to create the MTA Hub and KAI Solution Server:

```bash
oc apply -f 03-tackle.yaml
```

This will deploy:
- MTA Hub UI and API
- KAI Solution Server components (kai-api, kai-db, kai-importer)
- Keycloak for authentication
- PostgreSQL databases

**Monitor the deployment:**
```bash
# Watch all deployments
oc get pods -n openshift-mta -w

# Check KAI components specifically
oc get deploy,svc -n openshift-mta | grep -E 'kai-(api|db|importer)'
```

**Wait for all pods to be Running/Ready:**
```bash
oc get pods -n openshift-mta
```

**Get the MTA Hub URL:**
```bash
oc get route mta -n openshift-mta -o jsonpath='{.spec.host}'
```

Save this route URL - you'll need it for VS Code configuration.

---

## VS Code Configuration

### Configure Solution Server

1. **Get your MTA Hub route:**
   ```bash
   MTA_ROUTE=$(oc get route mta -n openshift-mta -o jsonpath='{.spec.host}')
   echo "https://${MTA_ROUTE}/hub/services/kai/api"
   ```

2. **Edit `.vscode/settings.json`** and replace `<YOUR_MTA_ROUTE>` with your actual route, for example:
   ```json
   {
       "mta-vscode-extension.genai.agentMode": false,
       "mta-vscode-extension.solutionServer": {
           "enabled": true,
           "url": "https://mta-openshift-mta.apps.cluster-xyz.opentlc.com/hub/services/kai/api",
           "auth": {
               "enabled": false
           }
       }
   }
   ```

### Configure Provider Settings

1. **Copy the provider settings template:**
   ```bash
   cp provider-settings.yaml.template provider-settings.yaml
   ```

2. **Edit `provider-settings.yaml`** and update:
   - Replace `<YOUR_MAAS_MODEL_API_URL>` with your model API url
   - Replace `<YOUR_API_KEY>` with your LLM API key
   - Verify the model name matches your deployment

   ```yaml
   openshift-ai-granite-model: &active
     environment:
       OPENAI_API_KEY: "<YOUR_API_KEY>"
     provider: "ChatOpenAI"
     args:
       model: "granite-3-2-8b-instruct"
       configuration:
         baseURL: "<YOUR_MAAS_MODEL_API_URL>"

   active: *active
   ```

3. **Configure the extension:**
   - Open VS Code
   - Go to **Settings** → **Extensions** → **MTA**
   - Under **Provider Settings File**, browse and select your `provider-settings.yaml`
   - Restart the MTA extension or reload VS Code

---

## Verification

### Verify OpenShift Deployment

1. **Check all MTA pods are running:**
   ```bash
   oc get pods -n openshift-mta
   ```

2. **Verify KAI components:**
   ```bash
   oc get deploy -n openshift-mta | grep kai
   ```

   Expected output:
   ```
   tackle-kai-api          1/1     1            1
   tackle-kai-importer     1/1     1            1
   ```

3. **Check KAI API logs:**
   ```bash
   oc logs -n openshift-mta deployment/tackle-kai-api --tail=50
   ```

4. **Test the route:**
   ```bash
   MTA_ROUTE=$(oc get route mta -n openshift-mta -o jsonpath='{.spec.host}')
   curl -k https://${MTA_ROUTE}/hub/services/kai/api/health || echo "Health check endpoint may not be available - this is normal"
   ```

### Verify VS Code Extension

1. **Open a Java/Spring project** in VS Code
2. **Open the MTA view** (left sidebar)
3. **Run an analysis:**
   - Right-click on your project
   - Select **MTA: Analyze**
   - Choose a transformation target (e.g., "Quarkus")
4. **Check for AI suggestions:**
   - After analysis completes, you should see AI-powered suggestions
   - Look for code snippets and migration recommendations

---

## Troubleshooting

### KAI Pods Not Starting

**Check pod status:**
```bash
oc get pods -n openshift-mta | grep kai
```

**Check pod logs:**
```bash
# Check API logs
oc logs -n openshift-mta deployment/tackle-kai-api

# Check importer logs
oc logs -n openshift-mta deployment/tackle-kai-importer
```

**Common issues:**

1. **Missing secret:** Ensure `kai-api-keys` secret exists
2. **Wrong model name:** Verify `kai_llm_model` in `05-tackle.yaml` matches your LLM
3. **LLM provider unreachable:** Check network connectivity from OpenShift to your LLM provider

### Extension Not Loading Provider Settings

1. **Verify file path:** Check that `provider-settings.yaml` is in the correct location
2. **Check YAML syntax:** Validate the YAML is properly formatted
3. **Restart VS Code:** Reload the window or restart VS Code
4. **Check extension logs:**
   - View → Output → Select "MTA" from dropdown
   - Look for errors related to provider configuration

### Model Name Mismatch

Ensure consistency across all configurations:

| Location | Field | Example Value |
|----------|-------|---------------|
| `05-tackle.yaml` | `kai_llm_model` | `granite-3-2-8b-instruct` |
| `provider-settings.yaml` | `args.model` | `granite-3-2-8b-instruct` |
| LLM Provider | Model subscription | `granite-3-2-8b-instruct` |

---

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

---

## Additional Resources

- [MTA Documentation](https://access.redhat.com/documentation/en-us/migration_toolkit_for_applications/)
- [Konveyor Community](https://www.konveyor.io/)
- [OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai/)
- [VS Code MTA Extension](https://marketplace.visualstudio.com/items?itemName=redhat.mta-vscode-extension)
