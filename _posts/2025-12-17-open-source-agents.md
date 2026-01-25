---
title: "Open Source Agents"
date: 2025-12-08
---

I spent some time this week sorting out a free / open source stack for adding RAG / LLM based on ESP32 temperature / humidity data.

* Working repo here \- [esp32\_api](https://github.com/adrmac/esp32_api)  
* Course examples here \- [google-agentic-ai](https://github.com/adrmac/google-agentic-ai)

**Ollama** \- downloads open source models and runs them locally

* Ollama directory of models available: [https://ollama.com/search?c=embedding\&q=embed](https://ollama.com/search?c=embedding&q=embed) 

**LLMs** \- pretrained models that generate conversational responses to prompts, and can be constrained to specific domains and structured outputs

* gemma2 \- Google’s open weight LLM  
* llama3 \- Meta’s open weight LLM, NOT related to Ollama  
* gpt-oss \- Open AI’s open weight LLM 

**Text embedding models** \- small dedicated models for converting text to vectors so an LLM can read them: 

* bge-m3 or nomic-embed-text

**LangChain** \- standardized types and language wrapper for AI agents in python and typescript 

* langchain-ollama integration \- Makes it simpler to switch between model APIs without changing code   
* langchain-text-splitters \- packages for splitting up PDFs and websites as embeddable chunks of text with metadata

**PostgreSQL** \- 

* supabase \- not open source but generous free tier cloud db  
* pgvector \- python extension that enables SQL similarity search  
* chromaDB \- more advanced storage option specifically for vectors \-- for now using supabase / pgvector instead

**Retrieval Augmented Generation (RAG) bibliography** \- starting with a small literature corpus on the subject of indoor environmental sensing for the LLM to consult when providing an answer

* [Mobaraki et al.](https://www.mdpi.com/2075-5309/11/8/336), “Application of Low-Cost Sensors for Building Monitoring: A Systematic Literature Review (Buildings 2021\)  
* [Mobaraki et al.,](https://www.mdpi.com/2075-5309/12/9/1411) “Application of Low-Cost Sensors for Accurate Ambient Temperature Monitoring” (Buildings 2022\)  
* [Venkata et al.](https://www.mdpi.com/1424-8220/24/11/3650?utm_source=chatgpt.com), “Challenges and Opportunities in Calibrating Low-Cost Environmental Sensors” (Sensors, 2024\)  
* Integrating Low-cost Sensor Systems and Networks to Enhance Air Quality Applications (World Meterological Organization, 2024\) \- [PDF](https://www.developmentaid.org/api/frontend/cms/file/2024/06/GAW-293_report_en.pdf?utm_source=chatgpt.com)  
* ASHRAE 55 Thermal Conditions for Human Occupancy \- outdated 2013 standard is available free in [PDF](https://ierga.com/hr/wp-content/uploads/sites/2/2017/10/ASHRAE-55-2013.pdf) and/or use [wikipedia](https://en.wikipedia.org/wiki/ASHRAE_55?utm_source=chatgpt.com)  
* International Performance Measurement & Verification (M\&V) Protocol \- Concepts and Practices for Improved Indoor Environmental Quality Volume II \- [PDF](https://docs.nrel.gov/docs/fy01osti/29565.pdf)  
* M\&V Guidelines: Measurement and Verification for Performance Based Contracts Version 4.0 (US Department of Energy, November 2015\) \- [PDF](https://www.energy.gov/sites/prod/files/2016/01/f28/mv_guide_4_0.pdf)
