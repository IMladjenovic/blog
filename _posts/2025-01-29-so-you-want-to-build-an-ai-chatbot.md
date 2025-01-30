---
title: So you want to build an AI chatbot
date: 2025-01-29 00:00:00 Z
categories:
  - Artificial Intelligence
  - Sustainability
  - Tech
tags:
  - Artificial Intelligence
  - Sustainability
  - Greenwashing
summary: This blog is for anyone who is creating their own AI chatbot or multi-agent service.
author: imladjenovic
---

Last October a small team at Scott Logic started a project to answer the question “How can we leverage Generative AI to identify and detect ESG [greenwashing](https://blog.scottlogic.com/2024/04/15/how-cxos-can-spot-technology-greenwashing.html)?”.

This challenge was created by FinTech Scotland’s “[FRIL](https://www.fintechscotland.com/what-we-do/financial-regulation-innovation-lab/)” organisation who selected our pitch to build [InferESG](https://github.com/ScottLogic/InferESG) - a [multi-agent service](https://blog.scottlogic.com/2024/06/28/building-a-multi-agent-chatbot-without-langchain.html) for analysing sustainability reports, flagging potential greenwashing and answering ESG questions.

> **Greenwashing:** When a company presents themselves as more sustainable than they really are (maliciously or otherwise...)

> **Multi-Agent Service:** A service containing multiple agents where each agent is specialised to solve a specific task using an LLM such as ChatGPT.

With no prior AI experience, I’ve learned a lot in those short 3 months.

Here are 7 reflections I made on AI that you must read before building your own multi-agent Chatbot.

## 1. Cheap AI models are great – to a point

At the start of the project, we investigated which AI models we should use. At the time, we had accounts with two LLM providers already setup, OpenAI and Mistral, so we started here.

It was imperative to find a balance between performance and cost, and reviewing various AI benchmarks, I found a consistent narrative that OpenAI’s gpt-4o-mini was only slightly less performant than the other LLMs (including their own gpt-4o) whilst being a whopping 10th of the cost.

* [https://epoch.ai/data/ai-benchmarking-dashboard](https://epoch.ai/data/ai-benchmarking-dashboard) shows accuracy scores for gpt-4o with 49%, gpt-4o-mini with 40%, mistral large with 34% and mistral large 2 with 49%. Most models from other providers are in this ballpark.

* [https://artificialanalysis.ai/models](https://artificialanalysis.ai/models) gives both gpt-4o and gpt-4o-mini a Quality score of 73 and mistral large 2 coming in at 74.

We made an early decision to favour the mini model based on the above findings. We made excellent progress and gpt-4o-mini was superb... until it wasn’t.

Imagine, for a moment, you’re an instance of gpt-4o-mini idling in one of OpenAI’s servers; you’ve been prompted to act as the [supervisor of an agentic model](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/), responsible for receiving user input and selecting an agent to handle it.

You receive the following user question: _“What are AstraZeneca’s sustainability goals?”_

Looking through your list of agents, you must select one of the following:

* Web Agent: _“This agent can search the internet to answer questions that require current information or answer general ESG / company related questions.”_

* Materiality Agent: _“This agent can answer questions about industry wide ESG Materiality standards and reporting practices. This agent cannot provide information about companies themselves, only industries and sectors as a whole.”_

Which agent would you choose?

Time and time again, we saw the ill-suited Materiality Agent selected and only able to produce dud answers about Astra Zeneca’s industry, Biotechnology & Pharmaceuticals.

After much effort reworking the agent selection prompt and editing agent descriptions to nudge InferESG in the right direction, I decided to try upgrading the agent model from gpt-4o-mini to gpt-4o. I was reluctant to try this as I felt the problem had to be with my prompt engineering skills, not the model; and from my research into benchmarks, I had little faith this was going to make a difference - but voilà, suddenly it worked! InferESG was “getting it”.

So why did gpt-4o work when the benchmarks said the mini model was just as good? My first thought was to double check the context window size – the total limit across input, output and reasoning tokens, but this was a dead-end, [the limits are the same](https://platform.openai.com/docs/models#gpt-4o).

So, I had to look a little deeper and this is where I first saw the term Small Language Models (SLMs). SLMs are smaller and thus more efficient than their counterpart (and better-known) Large Language Models (LLMs). SLMs are ideal for resource-constrained environments, and this is why the costs are lower.

I don’t believe OpenAI would describe their gpt-4o-mini as an SLM, but I would argue it is SLM adjacent. Various [unofficial estimates](https://explodingtopics.com/blog/gpt-parameters) put gpt-4o-mini at having 8 billion parameters (connections between nodes) and gpt-4o somewhere in the ballpark of 1.8 trillion. Parameters represent the connections between nodes in a neural network, they are weighted and determine how likely a link is to be made. Does having more parameters make a model smarter? Not exactly. DeepSeek, for example, caused waves in the AI space with only [671 billion parameters](https://huggingface.co/deepseek-ai/DeepSeek-R1). But the number of parameters help us paint the picture. 

The mini model is explicitly advertised by OpenAI as being more suited for simple tasks. We took the benchmarks at face value without investigating what or how they performed their testing.

Small models were a good choice to start with, they allowed us to get going and learn more about prompting. But when you begin to feel frustrated with prompt engineering, it may be time to stop hitting your head against the wall and take a look at the tools you are using.

## 2. Break down complex tasks 

In some instances, I found myself asking the LLM to do too many things with a single prompt.

This was a trap I fell into while upgrading the InferESG Datastore Agent. The agent connected to a Neo4j database that contained a single, pre-loaded ESG dataset. There are many ESG datasets out there with different methods and metrics included. I set about upgrading the Datastore Agent to import any kind of ESG csv file into the database using an LLM prompt. The prompt would perform the following:

1. Read the csv file containing ESG data
2. Identify the entities (e.g. companies, industries, types of environment / social / governance data) within
3. Map the relationships between entities
4. Create a Cypher query (Neo4j’s querying language) to import the csv into the database

I spent a lot of time prompt engineering but struggled to create a prompt which could perform all the steps above, even after upgrading the Datastore Agent from gpt-4o-mini to gpt-4o. Eventually, I tried a different tact and broke up the steps into two parts:

1. Modelling: performing steps 1, 2 and 3 to produce a json model output
2. Cypher Query: performing step 4 using the json model as input

By focusing each prompt and LLM call on a simpler task I immediately saw much better results, but more importantly, I was able to review the quality of output from each prompt individually, understanding why they were failing and make iterative improvements. 

## 3. Everyone says “Give Examples” ...

One of the biggest pieces of advice I heard about prompt engineering from my colleagues was to use examples to help the LLM understand what I want it to do.

It was the advice I was most apprehensive of.

Very early on we found InferESG hallucinating over simple questions. You could ask a question about Astra Zeneca, and it would give you an answer about Exxon. The issue lied in our Intent Agent, the agent responsible for reviewing a user’s question and determining what their intent is. In an ideal world, this agent would have a light touch, helping when a user’s question needed context. If they asked, “Where is Mount Everest?” and then followed up with “How tall is it?” our agent would rephrase the second question to “How tall is Mount Everest?”.

In the Intent Agent prompt, we included 10 examples of questions a user might ask and how it should go about answering. Examples such as “What efforts is Exxon making protect sea life?” and “Show me a chart about BP’s GHG emissions”. Unfortunately, these examples were hijacking the Intent Agent’s output, replacing company names given by the user with company names in the examples.

This was concerning, and upon reflection, there are a few possible causes for this misbehaviour which I think we could have investigated:

* Was the size of the system prompt too big?
* Did we have too many examples, or not enough?
* What if we upgraded from gpt-4o-mini to gpt-4o?

When solving this issue, I had neither the time nor AI knowledge to investigate these areas, instead I grew an aversion to the advice of “use examples”. I stuck to instructing the agent carefully and concisely. The only examples I would give would be to demonstrate desired output format, and even then, these would be as generic as possible, e.g.

`Output your answer in the following json format: { “question”: “Show me a chart about COMPANY_NAME GHG emissions” }`

This worked for me, but with more time I would like to have explored the bullet points above.

## 4. Working with AI is fun!

I want to call out the many times we built a new feature and were blown away by how well it worked. Creating agents and seeing the LLM outperform our expectations was exciting. When we completed our final demo to our FRIL challenge corporate sponsor, it far exceeded what we had hoped. AI is very good at what it does, when you keep in mind its limits...

## 5. AI is bleeding-edge

InferESG does a lot of document analysis, reviewing sustainability reports and materiality documents, often in pdf format. When we implemented our first pdf analysis feature into the project, I expected LLM providers would provide APIs for file upload - just like the attach file feature in ChatGPT. I was surprised to find that OpenAI was the only service at the time which provides API support for chatting about uploaded files (they call it “file search”).

One of the core tenants of InferESG was to not tie the application down to any single LLM, to be able to switch between LLMs as we see fit. Unfortunately, to get high-quality document analysis in the time we had available, we needed to use OpenAI file-search and lock several of our agents to this single provider. The decision nearly came back to bite us when the hour before our final demo when OpenAI APIs went down, unable to switch to another provider. Luckily, OpenAI was fixed just in time.

But even when using OpenAI’s file-search I was disappointed to find that this beta feature still hadn’t nailed down citation – the ability to see exactly what content of a file the LLM used to generate its answer. Being able to verify an LLM’s answer by reviewing its source material felt like a core requirement of this service.

To get citations on your LLM calls would require a do-it-yourself approach by building your own vector store. We investigated setting up a simple vector store, but it was not sufficient for the in-depth pdf file analysis that we needed. With more time we would have been excited to build something like the [NVIDIA AI Blueprints multimodal pdf data extraction](https://github.com/NVIDIA-AI-Blueprints/multimodal-pdf-data-extraction?tab=readme-ov-file). A sophisticated vector store service like this would have been perfect for our needs, I was disappointed to find no LLM providers had something like this out-of-the-box.

As you can see, I had several expectations shattered whilst working on this feature. There is a lot of hype surrounding AI, but the technology is moving rapidly (as I write this blog DeepSeek has just shocked the AI industry). I hope the features I’ve listed above become ubiquitous across AI providers, it would certainly have helped us build an even better product, but AI might evolve past these altogether...

During our InferESG project we started hearing rumblings of a new Gemini feature called “Deep Research” which seemed to perform the same service we were building - but with all of Google’s funding behind it. In a years' time there may be no need to build anymore InferESGs at all.

## 6. Testing an Agentic Model

How do we know if we are making progress? Is the quality of our product getting better?

We made a series of improvements that we thought would improve InferESG:

* Enabling OpenAI’s file search feature for better document analysis
* Smarter agent selection
* Better quality web page summarisation in our Web Agent
* Creation of a Materiality Agent for answering industry or sector specific ESG questions.

In theory, these changes improved InferESG, gave the user more accurate information and created more grounded answers to questions. The keywords here are “in theory”, it felt like InferESG was getting better, but we couldn’t be sure we weren’t wearing rose-tinted glasses while exploratory testing our own work. 

Fortunately, David set up a system to evaluate InferESG. He created a list of questions to ask our chatbot after each release and then inspected answers closely, looking for True Positives, True Negatives, False Positives and False Negatives. 

* True Positives: Correctly identified ESG claims or concerns
* True Negatives: Correctly identified non ESG issues
* False Positives: Incorrectly flagged ESG claims or concerns
* False Negatives: Missed ESG issues or potential greenwashing

We then measured a Recall and Precision Score:

* Recall Score – can we find all instances of ESG claims or concerns, measured as: TP / (TP + FN)
* Precision Score – how correct is the information we return: TP / (TP + FP)

We combined these scores to evaluate InferESG’s performance. Low and behold, the results showed that we were right, our changes were improving InferESG, and it was good to have proof.

One final note to share here is the final scoring method Accuracy Score – how often is InferESG right, measured as (TP + TN) / (TP + TN + FP + FN). Ultimately, we didn’t use this scoring system in the project as we didn’t see as much value measuring True Negatives, but it’s worth knowing about. 

## 7. Prompt Engineering with a Prompt Test Suite 

Throughout the development of InferESG we spent a lot of time prompt engineering. In the beginning, we made edits to the prompts and reloaded the app to test. This was a tedious process and as every developer knows, you have to shorten the feedback loop or you will go crazy. And of course, the classic xkcd “[Is It Worth The Time?](https://xkcd.com/1205/)”

We began using [promptfoo](https://www.promptfoo.dev/), the prompt testing framework. Whilst we didn’t have the smoothest setup experience, prompt engineering became a breeze. It was so much faster and a lot more enjoyable.

If I were to start the project over, I’m not sure promptfoo is the framework I would choose, but it certainly was a big boost to productivity at the time. 

## That's it!

Hopefully these lessons learnt will help the next AI rookie on their first AI project!

Now just to create a time machine and post this blog 3 months ago... 
