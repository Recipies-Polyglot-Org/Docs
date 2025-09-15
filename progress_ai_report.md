# Progress Report for AI project - 13th Sept, 2025

I’m building a candidate-finder application that helps discover developers based on their GitHub activity and match them to job descriptions using a RAG (Retrieval-Augmented Generation) pipeline. The idea is to fetch candidate profiles, analyze their skills from repositories and READMEs, and make them searchable through structured metadata.

So far, I’ve set up a clean project structure with FastAPI, LangChain, and a vector database. I implemented GitHub connectors to fetch and index user data, extract skills evidence, and store both plain text and structured skill metadata. On top of that, I built several endpoints: fetching and indexing candidates, checking the collection, filtering by skills, and running RAG queries against job descriptions. I’ve already validated these endpoints, and filtering by skills like PyTorch works smoothly.




### Some of the end points i have created 

**1. Fetch candidates from GitHub**

 Start a background job:

```
 curl -s -X POST "http://127.0.0.1:8000/api/fetch_github_bg" \
      -H "Content-Type: application/json" \
      -d '{"query":"language:python location:India","max_users":5,"per_user_repos":2}' | jq
``` 

This gives you a job_id, since fetching many users make take some time,

Check job status:

```
curl "http://127.0.0.1:8000/api/fetch_github_job/{job_id}" | jq
```
    
**2. Inspect collection (see indexed profiles)**

```
curl http://127.0.0.1:8000/api/collection | jq
```

**3. Filter by skill**
```
curl "http://127.0.0.1:8000/api/filter_by_skill?skill=pytorch" | jq
```
(or **?skill=git**, **?skill=docker**, etc.)

**4. Create a JD search job**
```
curl -s -X POST "http://127.0.0.1:8000/api/job" \
  -H "Content-Type: application/json" \
  -d '{"jd":"Deep learning researcher with PyTorch experience","k":3}' | jq
```
This returns a job_id and candidate shortlist.

**5. Run RAG for the JD**
```
curl "http://127.0.0.1:8000/api/rag/{job_id}" | jq
```

This gives you the final answer with reasoning + evidence.
    
