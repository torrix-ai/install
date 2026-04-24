# Demo: AI Support Triage with n8n and Torrix

This demo shows how to build an AI-powered support ticket router in n8n and use Torrix to observe every LLM call: what was sent, what came back, how much it cost, and how long it took.

The workflow classifies an incoming support ticket into a category (bug, billing, feature, or general) and routes it to the appropriate response. Bug reports go to a more capable model for a detailed reply. Everything else uses a cheaper model. Torrix groups both LLM calls into a single trace so you can compare cost and latency across steps.

## Prerequisites

Make sure you have the following running before starting.

**Torrix** installed and running at [http://localhost:8088](http://localhost:8088). If you have not installed it yet:

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/torrix-ai/install/main/docker-compose.community.yml
docker compose up
```

**n8n** running locally or on a server. To run it quickly with Docker:

```bash
docker run -it --rm --name n8n -p 5678:5678 n8nio/n8n
```

**Torrix community node for n8n** installed. In n8n go to Settings, then Community Nodes, click Install, and enter `@torrix-ai/n8n-nodes-torrix`.

## Step 1: Configure the Torrix credential

1. In n8n, go to **Credentials** and click **Add Credential**
2. Search for **Torrix API** and select it
3. Fill in the following fields

**Torrix Base URL:** Use `http://host.docker.internal:8088` on Mac and Windows when both n8n and Torrix run in Docker. On Linux use `http://<your-machine-ip>:8088`.

**Torrix API Key:** Copy this from the Torrix Settings page at [http://localhost:8088](http://localhost:8088).

**Default LLM Provider URL:** The endpoint for your LLM provider. For OpenAI use `https://api.openai.com/v1/chat/completions`.

**Default Provider API Key:** Your API key for the provider (OpenAI, Anthropic, Groq, etc).

**Default Model:** The model to use when a node does not specify one, for example `gpt-4o-mini`.

4. Click **Save** and name the credential **Torrix account** so the imported workflow connects to it automatically.

## Step 2: Import the workflow

1. In n8n, go to **Workflows** and click **Add Workflow**
2. Click the three-dot menu in the top right and select **Import from file**
3. Select `torrix-support-triage.json` from this folder
4. The workflow opens with six nodes already connected and ready to run

## Step 3: Understand the workflow

**Set Ticket** holds the sample support ticket text and generates a unique trace ID from the n8n execution ID. Open this node and edit the ticket text to test different scenarios.

**Classify Ticket** sends the ticket to a cheap model with a system prompt that returns a single classification word: bug, billing, feature, or general.

**Extract Classification** reads the model response and passes the category to the next step.

**Route by Type** checks whether the classification is "bug". If yes, the workflow takes the top branch. If no, it takes the bottom branch.

**Bug Response** uses a more capable model to write a professional reply with root cause analysis and clear next steps for the customer.

**General Response** uses the cheaper model to write a friendly, concise reply for non-bug tickets.

Both LLM nodes share the same trace ID so Torrix groups their calls into a single trace.

## Step 4: Run the demo

1. Click **Execute Workflow** in the top right
2. Watch the nodes turn green as they execute in sequence
3. Click the final active node (Bug Response or General Response) to see the generated reply in the output panel

To try a different ticket type, open the **Set Ticket** node, change the ticket text to a billing question or feature request, and run the workflow again.

## Step 5: Explore in Torrix

Go to [http://localhost:8088](http://localhost:8088) and open the Runs list. You will see an entry with a trace badge showing **2**, meaning two LLM calls were grouped into a single trace for this workflow execution.

Click on the **classify** run to open its detail view. In the Event Timeline:

**REQUEST** shows the full request that was sent to the LLM, including the system prompt and the ticket text as the user message. Click **Show request** to expand it and see exactly what the model received.

**RESPONSE** shows what the model returned, in this case a single word like "bug".

**SUMMARY** shows the model name, latency in milliseconds, token count, and cost in USD for this step.

Now go back and open the **respond-bug** run. This step costs more because it uses a more capable model and generates a longer response. Torrix makes that cost difference visible at a glance without any instrumentation in your code.

Click the trace badge on either run to open the full trace view and see both steps side by side with their combined cost, latency, and token usage.

## What to try next

Change the ticket in **Set Ticket** to a billing or feature request and run again. The classification changes, the workflow routes to General Response instead, and the trace in Torrix still groups both calls together.

Set the model on the **Bug Response** node to a more expensive model and run again. Compare the cost in Torrix to see the difference immediately.

Add a third Torrix Proxy node to the workflow for a follow-up action and give it the same trace ID. Torrix will include it in the same trace automatically.
