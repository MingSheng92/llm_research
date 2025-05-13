## Research with LLM 

Will not be sharing the code here but will be talking abit more on the design choices and serving as a simple dev log for myself and anyone interested in this topic.

#### Built with 
cloud service: AWS bedrock, EC2, cognito  <br />
LLM: Claude sonnet, haiku  <br />
Rerank model: cohere rerank  <br />
Embedding model: all-MiniLM-L6-v2/bge-large <br />
API: Patbase <br />
Vector store : FAISS <br />
libraries : llama-index, streamlit, python, Dockers, md2pdf <br />

#### Abstarct 
In a collaboration with R&D, we aim to create an automated way of performing intial research for the researchers and provide a recommendations from the current patent landscape's trend.

#### Architecture Overview
Below is the simplified version of the architechture design of the tool. 
![image](https://github.com/MingSheng92/llm_research/blob/main/images/Infra.webp)

We have devided into 5 distinct steps for this workflow of the tool <br />
- Step 1: Keywords
  * Topic extraction: Based on the initial input prompt from the user, it will extract the relevant topic with LLM. 
  * Keywords generation: Once topic is obtained, we will then make use of it to generate 2 additional distinct keywords with LLM. <br />
With the two pre-processing steps done, user can freely input phrases or single keywords and Generative AI will take care of things. <br />
Example  <br />
Initial prompt : "I would like to know about the latest advancement in Hydrogen Battery" <br />
topic extracted : Hydrogen Battery <br />
keywords generated : Hydrogen Battery, Hydrogen Fuel Cell, Hydrogen Energy Storage <br />

- Step 2: Retrieval of patents
  * A custom retrieval class for patbase was built to retrieve 20 latest patents from each keywords (configurable) with concurrent to maximize efficiency of retrieval.
  * all credentials are stored in sqlite3 database with both username and password encrypted.
  Preprocessing done to remove all unnecessary information and select only english or machine translated (if source is not in english), this is to optimize LLM ulitization where we feed only what we need the LLM to read which in turn lower the cost and shorten the process of inference time.  <br />

- Step 3: Rerank
  * Here we will feed the patent documents to cohere rerank to rank documents based on the relevancy of the patent to the topic itself.
  Started off with building a rag pipeline with llama-index (load docs - chuncking - set up query engine - retrieve top 20 chunks to find the top relevant patents) and we later find out rerank model does similar steps and can achieve better results. <br />
  Rerank model: ~20 sec to rerank and return top 20 docs <br />
  Llama-index/langchain: > 1 min to setup the vector store and return top 20 chunks <br />
  A 5X savings on setup time efficiency gain. <br />
  
  Here are some other methods I explored during the project:
  * Hybrid search: vDB + BM25

   Hybrid search combines semantic vector-based search with traditional keyword-based text search (BestMatch25). It theoretically improves retrieval accuracy by not only considering the meaning of the text but also precise keyword matches. Typically, results from both methods are merged and reranked by a combined score. Based on my experience, it reduced the retrieval quality instead of improving it.

  * Cross-encoder reranking
    
   Reranking the results of vector search using Cross-encoder models seemed promising. In short, Cross-encoders give a more precise similarity score but are slower.Cross-encoders lie between embedding models (bi-encoders) and LLMs. Unlike comparing texts via their vector representations (which inherently lose some information), cross-encoders directly assess semantic similarity between two texts, giving more accurate scores.However, pairwise comparisons of the query with every database element takes too long.

- Step 4: Report generation
  * This is where the main focus of this whole PoC, we wanted to have an AI to write out a report for the researchers so that they are well informed on the current patent landscape but also to recommend the researchers on potential topics to look at.
  * The report will consist of 3 sections: abstract, key overview, recommendations and conclusion 
  * based on the reranked results, we have obtained 20 relavent patents then we will make use of partial patent to write out the abstarct and key overview, where abstract will have a summary of all the company that is involves in the patent. where key overview will discuss the trends and high level overview on the advacement they claimed.
  As we all know patents are really long documents, feeding the full document to analyze and write report can be costly especially when we are dealing with 20 patents. We have to get creative on the approach to save both time and cost in conducting research with LLM. We took some inspiration from chain of verification and corrective rag, but instead of having a evaluation system to check on the response (it will take too much time) we will always make assumption of the report is incomplete and needs validation.<br />
  * Research Question Generation: Based on the first two section, we will create an agent with the role of research to ask a bunch of questions to validate already written sections. It will create a json object will a list of question corresponding to the patent. Next we will set up individual RAG pipeline on each patent listed to perform QA, once all answers retrieved we will then proceed with the writing of the report.  
  * Recommendations : Making use of the QA answers as context we will task the writing agent to write the final report. 
  <br />Why QA system was done to aid in the writing task? LLM getting better and better each day. Getting exceptionally well in needle in the haystack with great recall [link](https://x.com/alexalbert__/status/1764722513014329620) but I found that it needs to have tailored prompt template to do so which may not be possible for our case when everything can be dynamic, a too generice prompt may leads to the LLM losing focus and giving out generic answers and recommendations. <br />
  Hence agent QA with RAG seems to make more sense in this scenario and also great to optimize both cost and inference time, and indeed making the LLM more "focus" when writing out the report.

- Step 5: output pdf/setup chat agent
  * Since we have written the report in markdown formart, we can make use of md2pdf library to convert the final report into pdf for easy distribution among researchers.
  * We also setup a chat agent with llama-inde so that the user can verify in an interactive way in case there is any follow up questions. Note that the chat agent vector store will be set up based on the 60 patents retrieved not just the 20 relavant patents reranked.

####
With this design we managed to achieve comparable results to opensource projects/articles.  
![comparison](https://github.com/MingSheng92/llm_research/blob/main/images/compare.png)

####
Recent study from standford also suggest that LLMs are equally good at proposing ideas, hence validating our work here as a recommender for researchers on potential research topics and improvement suggestions. ![Link](https://www.deeplearning.ai/the-batch/stanford-study-finds-ai-matches-human-experts-at-writing-research-proposals/?utm_campaign=The%20Batch&utm_content=320262824&utm_medium=social&utm_source=linkedin&hss_channel=lcp-18246783)
