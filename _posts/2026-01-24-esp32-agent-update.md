This week I’m coming full circle.

I’ve been immersed in backend engineering using Python and SQL the last few months, seeking to fill out my full-stack knowledge, and prototype a complete data system. 

- Used MicroPython to set up an ESP32 microcontroller with a temperature/humidity sensor that streams measurements every 2 seconds over wi-fi.   
- Wrote a small Python API using the FastAPI framework that receives the sensor measurements and stores them in a Postgres database.   
- Added a feature that SQL queries the database and makes hourly snapshots of average/min/max readings.   
- Deployed the app first using a Cloudflare Tunnel from my computer, and then to a free-tier container on Google Cloud Run

So I now have a few weeks of continuous temperature / humidity data taken from my home office, which I am visualizing using Looker Studio. 

Simultaneously, I’ve been studying agentic AI – basically using pretrained LLMs (Large Language Models) in orchestrated ways to achieve specific things. My goal is to build a chatbot that talks to me about trends and anomalies in my home office data, that I can later scale up to Orcasound hydrophones. A few highlights from this journey included:

- Took a Google Agentic AI [course intensive](https://www.kaggle.com/learn-guide/5-day-agents) in Python, delivered through the Kaggle platform.   
- Pulled the course material into my local environment, stripping out the Kaggle-specific code and calling the Gemini API from my computer – this was my first Python repo.  
- Installed Ollama and adapted the course examples to run with an open source LLM on my computer (gemma2) instead of Gemini API.   
- Mapped the Google ADK (agent development kit) course examples to the open source LangChain agent framework using the LangChain-Ollama integration.  
- Got annoyed with some of LangChain’s complexity for my task and switched to the roughly equivalent LlamaIndex framework. LlamaIndex includes a built-in feature for translating natural language chat questions to SQL queries and combining the results with qualitative text references (‘SQL Auto Vector Query Engine’).  
- I got some results with LlamaIndex, but the SQL/vector feature felt a little clunky, and the agent was taking typically 30s to answer. I couldn’t easily find any observability data about why it was taking so long or how resource heavy it was. The LLM’s answers were also far from perfect and I was also getting distressed by having to build my own platform to evaluate and improve the agent. This led me to looking at Arize Phoenix, an open source agent observability platform in Python that is supposed to help with experimentation, evaluation, and troubleshooting. 

But now I’m at a turning point with Python. I’m wary of taking on more packages, adding complexity, and spreading myself thinner into data science territory. I spent the better part of last year getting a Next / React prototype ([next.orcasound.net](http://next.orcasound.net)) launched, and I have burning UX hypotheses I want to test around integrating an LLM into data visualization and research flows. I want to get back to that. 

The solution was there all along – the [Next / Vercel AI SDK](https://ai-sdk.dev) turns out to be a TypeScript-based agent development kit, essentially equivalent to Google ADK, LangChain, or LlamaIndex, with observability built-in on the browser. Amazing\! It even has a whole UI kit for chatbot interactions. 

Am I loving this? I think so. Like, more Python-oriented types might argue for skipping TypeScript entirely, using Flutter/Dash for the UI, and carrying on with data science as if front-end engineering didn’t exist. I can see that, but I’m also kind of invested in Next, at least until I get to the point of trying to do mobile native. I’d love to try Flutter, or Arize Phoenix for that matter – but I think I owe it to my 6 months-ago self to finish what I started.

Either way, the agent orchestration is basically ‘stateless’ so it doesn’t really matter where it runs. The real issue is where to run the LLM inferences. I have the FastAPI app deployed for free on Cloud Run, but the container is only large enough to process tasks like relaying data from the ESP32 to Postgres, or computing snapshot statistics. It wouldn’t be a good place to run Ollama with open source LLMs that require ‘stateful’ memory – I think it’s possible to do, but not advised. 

If I were using a commercial LLM like Gemini or OpenAI, I could trigger the queries for free from Cloud Run, and then pay per query. But by committing to an open source LLM, you also commit to owning it yourself and hosting it yourself, and there is no such thing as a free cloud server (aka virtual machine / VM), that is large enough to run LLM inferences. 

A virtual machine from Google Compute Engine starts at about $25 / month for 2 vCPU, 1 core, 4 GB RAM, and 10 GB disk space. The monthly pricing is just an estimate, it is actually prorated per second, but the machine is either on or off, it won’t spin up automatically whenever it gets a request.

To figure out how much CPU, RAM, and disk space I would need, I had ChatGPT analyze logs from running the LlamaIndex-Ollama chatbot with Qwen2.5, a relatively lightweight LLM. For each question prompt, the agent needed to generate a response that incorporated both a natural language-to-SQL query, and a RAG vector retrieval step. 

The results suggested the right size VM would be at least 30 GB of disk space ($3 / mo), and something like 4-8 vCPU, 16 GB. That fits one of these Compute Engine machines:

- $97.84 / mo – 4 vCPU, 2 core, 16 GB (one request at a time)  
- $195.67 / mo – 8 vCPU, 4 core, 32 GB (two requests simultaneously)

For comparison, my macbook pro is Apple M1 chip, 10 core, 16GB – so compared with these, it has outsized CPU capacity relative to the amount of RAM. But the VMs have more RAM than needed for my agent. And any old Intel chip office computer would probably meet these specs. This is still not production grade in terms of processing requests for thousands of users, but other strategies should also come into play at that scale. We could also start talking about buying a GPU such as the Nvidia A100, or the cheaper RTX 4090, which would take things to another level.

Bottom line:

I figured out how to steer my agentic AI trajectory back to front-end / UX. The Vercel AI SDK is a front-end agent development kit using Next/React/Typescript — [https://ai-sdk.dev](https://ai-sdk.dev) — It functions the same as Python libraries like LangChain, LlamaIndex, or Google ADK, runs via Next api routes, comes with UI primitives, and gives observability metrics in the browser. 

This seems like a good tool for what I am trying to build: a chatbot that responds to UI filter state, and can change filters in response to chat input. The real issue is how to scale it, even for a small project like Orcasound. I will run the agent on my laptop for now to prove out concepts and keep monitoring performance, while staying on the lookout for a cheap office computer or even GPU to start my home data center. 
