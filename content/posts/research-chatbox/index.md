+++
title = 'Building a RAG-Powered Vulnerability Research Chatbot with Agno'
date = 2025-08-18T13:03:51-04:00
draft = false
+++

Recently, I delved into a personal project: creating a Retrieval-Augmented Generation (RAG) system. This chatbot pulls from a curated set of public security research articles to answer questions about vulnerabilities, functions, and exploitation techniques. This blog post walks through how I built a simple yet surprisingly powerful vulnerability research chatbot using a RAG system powered by the **Agno** Python library.

## Motivation and Goals

Vulnerability research can be time-consuming. You spot a weird function in a Windows DLL or a suspicious behavior in software, and suddenly you're digging through docs, blogs, and CVE reports. What if you could query a chatbot trained on security articles to get quick, relevant insights?

A chatbot that retrieves and synthesizes relevant research could save hours of manual digging. That's where RAG comes in. RAG combines large language models (LLMs) with a knowledge base: it retrieves relevant info from your dataset and generates responses based on it. I used the Agno library (check out their [intro docs](https://docs.agno.com/introduction)) because it's straightforward for building RAG apps in Python.

I had two main goals:

1.  **Explore a Practical Use Case:** I wanted to see if I could create a focused chatbot that provides genuinely useful assistance for vulnerability research tasks, moving beyond generic responses from standard LLMs.
2.  **Understand RAG Systems:** This project was a hands-on way to dive deep into RAG architecture—to understand how it leverages both public and proprietary data to provide actionable, context-aware answers.

To achieve these goals, I built a system with minimal complexity that nonetheless delivers impressive results. Note: The chatbot's quality shines with a rich knowledge base. I started with public blogs, but feeding it proprietary research docs could make it a game-changer for teams.

## Implementation Breakdown

With these goals in mind, let's dive into the technical details of the implementation. The system is built on the Agno library, which simplifies creating RAG applications. My implementation has two core components: a knowledge retrieval script and a web server for the chatbot itself.

### Step 1: Building the Knowledge Base

First, I needed to collect, process, and store the expert knowledge. This is
handled by `kretrieve.py`.  Its job is to take a list of URLs pointing to
security research blogs, transform their content into semantic embeddings
(numerical vector representations of meaning), and store them in a PgVector
database. This allows the chatbot to perform semantic search—finding content
based on meaning rather than exact keyword matches.

The code is straightforward. It defines a `UrlKnowledge` object, specifying the
source URLs and the vector database configuration, including the OpenAI model
used for creating the embeddings.

Here's a snippet of the core code:

```python
from agno.embedder.openai import OpenAIEmbedder
from agno.knowledge.url import UrlKnowledge
from agno.vectordb.pgvector import PgVector, SearchType

knowledge = UrlKnowledge(
    urls=[
        "https://www.zerodayinitiative.com/blog/2022/3/16/abusing-arbitrary-file-deletes-to-escalate-privilege-and-other-great-tricks",
        "https://www.zerodayinitiative.com/blog/2025/5/7/cve-2024-44236-remote-code-execution-vulnerability-in-apple-macos",
        # ... more URLs here
    ],
    vector_db=PgVector(
        db_url="postgresql+psycopg://xxxx:xxxx@localhost:5532/ai",
        table_name="agno_docs",
        search_type=SearchType.hybrid,
        embedder=OpenAIEmbedder(id="text-embedding-3-small", dimensions=1536)),
)

knowledge.load(recreate=True)
print("Knowledge base loaded successfully!")
```

I loaded articles from Zero Day Initiative (ZDI) blogs. Run this once to populate your DB. (Pro tip: Add more URLs for better coverage—quality input means quality output.)

With the knowledge base populated, the next step is to build the chatbot service itself. Here, the focus shifts from ingesting data to retrieving and serving it efficiently.

### Step 2: Serving the RAG Chatbot

With the knowledge base ready, the next step was serving it through a chatbot
API (`serve.py`). This connects the database, the LLM, and a user-facing
interface. This script uses Agno to define an "Agent" that connects the
language model (LLM), the knowledge base, and a user interface.

Key components in this file include:

  * **`PgVector` Connection:** It connects to the same database we populated in the previous step. Notice the urls list is empty here - the ingestion process is separate. The chatbot server’s only job is to retrieve and respond, keeping it lightweight and modular.
  * **`Agent` Configuration:** I defined an agent with specific instructions, such as "Search your knowledge before answering," and told it to cite its sources. It uses an OpenAI model as its reasoning engine.
  * **`FastAPIApp`:** Agno uses FastAPI to quickly spin up a web server, making the agent accessible via an API.

Here’s the minimal setup that wires everything together into a FastAPI app.

```python
from agno.agent import Agent
from agno.app.fastapi.app import FastAPIApp
from agno.embedder.openai import OpenAIEmbedder
from agno.models.openai import OpenAIChat
from agno.knowledge.url import UrlKnowledge
from agno.storage.sqlite import SqliteStorage
from agno.vectordb.pgvector import PgVector, SearchType

knowledge = UrlKnowledge(
    urls=[],  # Empty since DB is pre-populated
    vector_db=PgVector(
        # ... same DB config as above
    ),
)

storage = SqliteStorage(table_name="agent_sessions", db_file="tmp/agent.db")

basic_agent = Agent(
    name="Basic Agent",
    agent_id="basic_agent",
    model=OpenAIChat(id="gpt-5-nano"),
    instructions=[
        "Search your knowledge before answering the question.",
        "If an article from the knowledge base is highly relevant to the user's question, try to mention the title and URL/link of this article in the response.",
        # ... more instructions
    ],
    knowledge=knowledge,
    storage=storage,
    # ... other configs
)

fastapi_app = FastAPIApp(
    agents=[basic_agent],
    # ... app details
)

if __name__ == "__main__":
    fastapi_app.serve(app="serve:app", port=8001, reload=True)
```

Run this, and your chatbot API service is live at `http://localhost:8001`. You can now interact with it in several ways: query it directly via curl or Postman for testing, or build a simple web frontend using HTML and JavaScript to create a user-friendly chat interface. The API endpoints provided by FastAPI make it straightforward to integrate with any frontend framework of your choice. 

Once the server is running, you can explore the chatbot API through the auto-generated Swagger UI.

![The FastAPI docs page (Swagger UI)](fig-api.png)

This makes it easy to test endpoints interactively before integrating with a frontend.

With the server running, the chatbot is ready for queries. To demonstrate its capabilities, I tested it with real vulnerability research scenarios.

## Real-World Examples: How It Performs

Despite the simple implementation, the chatbot delivered detailed, accurate, and context-aware answers that directly referenced its knowledge base. Here’s a sample query in the simple frontend interface:

![Example query](fig-frontend.png)

Below are two representative examples that show how effectively it retrieves relevant information and synthesizes responses for vulnerability research tasks.

### Example 1: Digging into a Windows Function

As an N-day researcher, I often need context on obscure functions. For instance, I asked about `KpsCheckKerbResponse()` function in a Windows DLL.

**My Query:** "Can you explain the KpsCheckKerbResponse() function? Specifically, I would like to know the conditions under which this function can be invoked."

**Chatbot Response:**
```markdown
- The KpsCheckKerbResponse function is a validation step used when the KDC
  Proxy processes a forwarded Kerberos response from a domain controller.
- Where it’s invoked:
  - After KDC Proxy reads a Kerberos response from the domain controller,
    KpsSocketRecvDataIoCompletion() hands control to KpsPackProxyResponse()
    with the _KPS_IO context.
  - KpsPackProxyResponse() calls KpsCheckKerbResponse() to validate the
    Kerberos response as part of packaging the KDC proxy response.
- The actual validation behavior (i.e., whether it does anything) depends on
  the bytes in the Kerberos message:
  - KpsCheckKerbResponse() validates the Kerberos response only if the byte
    immediately following the message length prefix is 0x7E or 0x6B.
  - If that following byte is 0x7E or 0x6B, it verifies that the Kerberos
    message is properly constructed.
  - If the following byte is any value other than 0x7E or 0x6B,
    KpsCheckKerbResponse() does not perform any validation and returns without
    error.
- In short:
  - Invocation context: within the Kerberos response handling path of the KDC
    Proxy (via KpsPackProxyResponse() after a full Kerberos response has been
    read).
  - Activation of validation: only when the byte after the 4-byte length prefix
    is 0x7E or 0x6B; otherwise, no validation is performed.
- This behavior is described in the ZDI write-up on CVE-2024-43639, which
  explains how KpsCheckKerbResponse() is used during the KDC Proxy processing
  flow and how the validation can be bypassed depending on that specific byte
  value.
- Reference article (highly relevant): CVE-2024-43639 — Microsoft Windows KDC
  Proxy
  - URL: https://www.zerodayinitiative.com/blog/2025/3/3/cve-2024-43639
```

It pulled directly from a ZDI article—spot-on and cited!

### Example 2: Exploring Exploitation Paths

Next, I explored whether the chatbot could explain exploitation possibilities from an arbitrary file delete bug in Windows.

**My Query:** "I am developing a proof of concept for a vulnerability that allows arbitrary file deletion by exploiting a Windows vulnerability. Can you explain whether it is possible to achieve privilege escalation from this situation? Please provide detailed information."

**Chatbot Response:**
```markdown
Yes, it can be possible to escalate from arbitrary file deletion to privilege
escalation on Windows, but it’s highly dependent on the exact vulnerability,
the involved components, and timing. Public research has shown several paths
that use an arbitrary delete primitive to gain SYSTEM or other high-privilege
access, usually by abusing trusted rollback/cleanup flows or privileged tasks.

High-level concepts (from public research)
- Windows Installer path (Config.Msi / rollback):
  - An attacker who can delete or replace certain system folders can cause
    Windows Installer to trust a modified folder (e.g., C:\Config.Msi) that the
    attacker controls.
  - The attacker can place manipulated rollback data (e.g., .rbs and .rbf
    files) into that folder. Because the installer preserves the folder’s
    security descriptor and the registry state indicates trust, the
    rollback/install logic can execute attacker-controlled code during
    uninstall/rollback, potentially delivering a SYSTEM-level payload.
  - This often requires a two-stage flow: stage 1 executes a flow that leads to
    the folder being recreated with weak permissions, stage 2 triggers an
    uninstall/rollback that consumes attacker-controlled data, culminating in
    privilege escalation.
  - There are later refinements to improve reliability (race-condition
    avoidance, precise timing, etc.) and to chain the technique with additional
    primitives (e.g., targeting the uninstall rollback window).

- Scheduled task and other privileged cleanup paths:
  - Researchers have shown that abusing privileged cleanup paths (e.g., a
    system-owned cleanup task) can be combined with arbitrary deletes to
    perform a similar escalation, often by manipulating the contents that the
    cleanup task would delete or by steering the flow to execute
    attacker-controlled code under SYSTEM.

- Advanced chaining techniques:
  - Some proofs-of-concept chain multiple primitives (e.g., arbitrary file
    deletion, moving content, and creating/joining special namespaces or
    alternate data streams) to reach a reliable SYSTEM-level execution path.
  - Examples discussed include using file/folder deletion to impact the state
    of C:\Config.Msi and then triggering rollback to drop a malicious DLL.

What to read for details
- The widely cited work: Abusing Arbitrary File Deletes to Escalate Privilege
  and Other Great Tricks
  - Link:
    https://www.zerodayinitiative.com/blog/2022/3/16/abusing-arbitrary-file-deletes-to-escalate-privilege-and-other-great-tricks

Defensive considerations
- Patch and mitigate the underlying issue(s) where possible (e.g., components
  that trust folders based on transient states or registry hints).
- Harden executable and installer flows:
  - Apply the latest OS and security updates.
  - Restrict or monitor Windows Installer behavior and uninstall/rollback
    paths.
- Minimize attack surface:
  - Limit which users/services can delete or replace privileged folders.
  - Use least-privilege for services and protect sensitive directories with
    robust ACLs.
- Detection and monitoring:
  - Monitor for suspicious changes to C:\Config.Msi and related rollback paths,
    unusual folder recreation with weak DACLs, and unexpected .rbs/.rbf
    artifacts.
  - Alert on anomalous activity in installer-related flows and on operations
    around SYSTEM-level cleanup tasks.
- Hardening features:
  - Consider application control/WDAC, AppLocker policies, and Controlled
    Folder Access to reduce exposure to such tampering.
  - Use EDR rules to detect and block risky sequences (e.g., loading
    attacker-controlled DLLs via rollback mechanisms).

If you want, I can summarize the high-level flow further without actionable
exploit steps, or help you map these concepts to defenses in your environment.
```

Again, it cited a key ZDI post and provided practical advice.

These examples highlight how even a minimal RAG implementation can produce
highly practical results.

## Final Thoughts and Next Steps

This project demonstrated that a lightweight RAG pipeline can deliver real
value for vulnerability research. The Agno library made it incredibly easy to
wire up the components without getting bogged down in boilerplate code.

For future work, I plan to:

  * **Expand the Knowledge Base:** Add more data sources, including local documents and proprietary research notes.
  * **Build a Simple UI:** Create a web-based frontend to make the chatbot easier to interact with.
  * **Experiment with Models:** Test different embedding and language models to see how they impact the quality of the responses.

By combining retrieval with generation, security teams can accelerate vulnerability research and focus more on analysis rather than manual information gathering.
