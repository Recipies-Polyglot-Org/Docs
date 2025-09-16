# Candidate Finder System - Complete Data Flow Documentation

## Overview
The Candidate Finder is an AI-powered recruitment platform that uses semantic matching to find the best candidates for job descriptions. It combines GitHub profile analysis, vector embeddings, and intelligent skill extraction to provide evidence-based candidate recommendations.

---

## 🔄 Complete Data Flow Architecture

```
GitHub Search → Profile Processing → Vector Storage → JD Matching → Results Display
     ↓              ↓                  ↓             ↓              ↓
  [Frontend]    [Backend APIs]    [ChromaDB]    [AI Analysis]   [Frontend]
```

---

## 🚀 **PHASE 1: GitHub User Search & Collection**

### 1.1 Frontend - GitHub Search Form
**File:** `frontend/src/components/GithubSearchForm.js`

**User Input:**
- Programming language (e.g., Python, JavaScript)
- Location (e.g., "New York", "India")
- Minimum followers count
- Minimum repositories count
- Maximum users to fetch
- Repositories per user to analyze

```javascript
const payload = {
  language: formData.language || null,
  location: formData.location || null,
  min_followers: formData.minFollowers ? parseInt(formData.minFollowers) : null,
  min_repos: formData.minRepos ? parseInt(formData.minRepos) : null,
  max_users: formData.maxUsers,
  per_user_repos: formData.perUserRepos
};

const response = await githubApi.fetchUsers(payload);
```

### 1.2 API Call to Backend
**Endpoint:** `POST /api/fetch_github_bg`
**File:** `backend/app/routers/github.py`

The frontend sends search criteria to the backend, which starts a background job:

```python
@router.post("/fetch_github_bg")
async def fetch_github_bg(req: GitHubFetchRequest, background_tasks: BackgroundTasks):
    """Start background GitHub user fetching process"""
    return await github_service.start_fetch_job(
        req.language, req.location, req.min_followers, 
        req.min_repos, req.max_users, req.per_user_repos, 
        background_tasks
    )
```

### 1.3 Background Processing - GitHub Data Collection
**File:** `backend/app/services/github_service.py`

**Step 1: Build GitHub Search Query**
```python
query_parts = ["type:user"]
if language: query_parts.append(f"language:{language}")
if location: query_parts.append(f"location:{location}")
if min_followers: query_parts.append(f"followers:>={min_followers}")
if min_repos: query_parts.append(f"repos:>={min_repos}")

query = " ".join(query_parts)
```

**Step 2: Fetch Users from GitHub API**
```python
users = await self.github.search_users(
    query=query, 
    max_users=max_users, 
    per_page=min(100, max_users)
)
```

**Step 3: For Each User - Collect Detailed Profile**
```python
for user in users:
    # Get detailed user info
    user_details = await self.github.get_user_details(user['login'])
    
    # Get user's repositories
    repos = await self.github.get_user_repos(user['login'], per_user_repos)
    
    # Process and store the profile
    await self.process_and_store_profile(user_details, repos)
```

---

## 📊 **PHASE 2: Profile Processing & Skill Extraction**

### 2.1 Profile Text Generation
**File:** `backend/app/features/github/github_connector_async.py`

For each GitHub user, we create a comprehensive text profile:

```python
profile_parts = [
    f"Name: {user_data.get('name', 'N/A')}",
    f"Username: {user_data.get('login', 'N/A')}",
    f"Bio: {user_data.get('bio', 'N/A')}",
    f"Location: {user_data.get('location', 'N/A')}",
    f"Company: {user_data.get('company', 'N/A')}",
    f"Public Repositories: {user_data.get('public_repos', 0)}",
    f"Followers: {user_data.get('followers', 0)}",
    
    # Repository analysis
    "REPOSITORIES:",
    *[f"Repo: {repo['name']} - {repo.get('description', '')} "
      f"Language: {repo.get('language', 'N/A')} "
      f"Stars: {repo.get('stargazers_count', 0)}" 
      for repo in repos]
]

profile_text = "\n".join(profile_parts)
```

### 2.2 AI-Powered Skills Extraction
**File:** `backend/app/features/skills/skills.py`

**Method 1: LLM-Based Extraction**
```python
def extract_skills(self, text: str) -> List[str]:
    prompt = """Extract the key technical skills from the following text. 
    Return them as a comma-separated list.
    
    Example output: Python, JavaScript, React, Docker
    
    Text: {text}
    
    Skills:"""
    
    response = self.embedding_service.get_text_completion(prompt)
    skills = [s.strip() for s in response.split(",") if s.strip()]
    return list(set(skills))
```

**Method 2: Regex Pattern Matching (Backup)**
```python
SKILL_PATTERNS = {
    "python": [r"\bdef\s+\w+\b", r"\bprint\(", r"\bclass\s+\w+:", r"\bflask\b", r"\bdjango\b"],
    "javascript": [r"\bfunction\s+\w+\(", r"\breact\b", r"\bangular\b", r"\bvue\.js\b"],
    "aws": [r"\bS3\b", r"\bLambda\b", r"\bEC2\b", r"\bRDS\b"],
    "docker": [r"\bdocker\b", r"\bDockerfile\b", r"\bdocker\s+compose\b"],
    # ... 20+ categories with 100+ patterns
}
```

### 2.3 Evidence Collection
For each extracted skill, we find supporting evidence:

```python
def find_evidence(self, text: str, skills: List[str]) -> Dict[str, List[str]]:
    prompt = """Find evidence snippets that demonstrate these technical skills.
    Return ONLY a valid JSON object where keys are skills and values are 
    arrays of relevant text snippets.
    
    Skills: {skills}
    Text: {text}
    
    Evidence:"""
    
    response = self.embedding_service.get_text_completion(prompt)
    return json.loads(response)  # Returns {skill: [evidence_snippets]}
```

---

## 🗄️ **PHASE 3: Vector Storage in ChromaDB**

### 3.1 Embedding Generation
**File:** `backend/app/infrastructure/aws/bedrock_embeddings.py`

Each profile text is converted to a high-dimensional vector using AWS Bedrock:

```python
def get_embedding_for_text(self, text: str) -> List[float]:
    request_body = {"inputText": text}
    
    response = self.client.invoke_model(
        modelId="amazon.titan-embed-text-v1",  # 1536-dimensional vectors
        contentType="application/json",
        body=json.dumps(request_body)
    )
    
    response_body = json.loads(response["body"].read())
    embedding = response_body.get("embedding", [])
    return [float(x) for x in embedding]
```

### 3.2 Storage in ChromaDB
**File:** `backend/app/infrastructure/aws/vectorstore.py`

```python
def upsert_profile(profile_id: str, text: str, vector: list, metadata: dict = None):
    # Store in vector database
    collection.add(
        ids=[profile_id],           # Unique identifier
        metadatas=[metadata],       # Structured data (skills, repos, etc.)
        documents=[text],           # Full profile text
        embeddings=[vector]         # 1536-dimensional vector
    )
    
    # Persist to disk
    if hasattr(collection, "persist"):
        collection.persist()
```

### 3.3 Database Structure

**ChromaDB Collection:**
- **IDs**: GitHub username or unique identifier
- **Documents**: Full profile text for semantic search
- **Embeddings**: 1536-dimensional vectors from Bedrock
- **Metadata**: JSON containing:
  ```json
  {
    "name": "John Doe",
    "username": "johndoe",
    "location": "San Francisco",
    "company": "Tech Corp",
    "skills_list": ["Python", "React", "AWS"],
    "skills_evidence": {"Python": ["def main():", "import pandas"], ...},
    "top_repositories": [...],
    "followers": 150,
    "public_repos": 25
  }
  ```

**SQLite Database (Backup/Cache):**
```sql
CREATE TABLE candidates (
    id VARCHAR PRIMARY KEY,
    source VARCHAR,
    filename VARCHAR,
    profile_text TEXT,
    metadata JSON,
    created_at DATETIME
);
```

---

## 🔍 **PHASE 4: Job Description Search & Matching**

### 4.1 Frontend - Job Description Input
**File:** `frontend/src/components/JobForm.js`

User inputs a job description, which is sent to the backend:

```javascript
const handleSubmit = async (e) => {
  const response = await jobsApi.createJob(jd, k);  // k = number of results
  onJobCreated(response);
};
```

### 4.2 Backend - Job Processing
**File:** `backend/app/services/job_service.py`

**Step 1: Generate JD Embedding**
```python
async def create_job(self, jd: str, k: int) -> dict:
    # Convert job description to vector
    jd_vec = get_embedding_for_text(jd)
    print(f"[DEBUG] Embedding vector length: {len(jd_vec)}")  # 1536 dimensions
```

**Step 2: Vector Similarity Search**
```python
# Find similar candidates using cosine similarity
candidates = query_similar(jd_vec, k=k)
print(f"[DEBUG] Query returned {len(candidates)} candidates")
```

**Step 3: Calculate Match Percentages**
```python
enhanced_results = []
for candidate in candidates:
    # Get candidate's embedding
    candidate_text = candidate.get("document", "")
    candidate_vec = get_embedding_for_text(candidate_text)
    
    # Calculate cosine similarity
    from numpy import dot
    from numpy.linalg import norm
    
    similarity = dot(jd_vec, candidate_vec) / (norm(jd_vec) * norm(candidate_vec))
    
    # Convert to percentage and assign confidence
    similarity_score = round(similarity * 100, 2)
    confidence = "HIGH" if similarity >= 0.45 else ("MEDIUM" if similarity >= 0.35 else "LOW")
```

**Step 4: Extract Matching Skills**
```python
# Extract skills that match between JD and candidate
candidate_skills = extract_keywords_from_jd(candidate_text)
skill_evidence = find_evidence_for_skills([candidate], candidate_skills)

enhanced_result = {
    **candidate,
    "similarity_score": similarity_score,  # 0-100%
    "confidence": confidence,              # HIGH/MEDIUM/LOW
    "matched_skills": candidate_skills,    # ["Python", "React", ...]
    "skill_evidence": skill_evidence       # {skill: [evidence]}
}
```

### 4.3 Vector Search Implementation
**File:** `backend/app/infrastructure/aws/vectorstore.py`

```python
def query_similar(query_vector, k=10):
    """Find k most similar candidates using cosine similarity"""
    try:
        # Query ChromaDB with embedding vector
        res = collection.query(
            query_embeddings=[query_vector], 
            n_results=k
        )
        
        # Normalize results
        return _normalize_query_result(res)
    except Exception as e:
        logger.error(f"Vector query failed: {str(e)}")
        raise
```

---

## 📱 **PHASE 5: Frontend Display & User Interface**

### 5.1 Results Rendering
**File:** `frontend/src/components/CandidateList.js`

For each candidate, display:

```javascript
// Match percentage and confidence
<Badge colorScheme={getConfidenceColor(candidate.confidence)}>
  {candidate.similarity_score}% Match - {candidate.confidence} Confidence
</Badge>

// Skills with evidence
{skills.map((skill) => (
  <Badge key={skill} colorScheme="blue">{skill}</Badge>
))}

// Detailed evidence
{candidate.metadata?.skills_evidence_json && (
  <Box>
    <Text fontWeight="semibold">Detailed Evidence:</Text>
    {Object.entries(skillsEvidence).map(([skill, evidence]) => (
      <Box key={skill}>
        <Text fontWeight="bold" color="blue.600">{skill}:</Text>
        <UnorderedList>
          {evidence.slice(0, 3).map((snippet, idx) => (
            <ListItem key={idx}>{snippet}</ListItem>
          ))}
        </UnorderedList>
      </Box>
    ))}
  </Box>
)}

// Repository analysis
{candidate.metadata?.top_repositories && (
  <SimpleGrid columns={3} spacing={3}>
    {JSON.parse(candidate.metadata.top_repositories).map((repo) => (
      <Box key={repo.name} p={4} bg="gray.50" borderRadius="lg">
        <Heading size="sm">{repo.name}</Heading>
        <Text fontSize="sm">{repo.description}</Text>
        <Badge>{repo.language}</Badge>
        <Text>★ {repo.stars}</Text>
      </Box>
    ))}
  </SimpleGrid>
)}
```

### 5.2 Three Main Views

**Tab 1: Job Description Search**
- Input job description → Get ranked candidates
- Show match percentages and confidence levels
- Display matching skills with evidence

**Tab 2: GitHub User Search**
- Search GitHub by criteria → Build candidate database
- Real-time progress tracking
- Background processing status

**Tab 3: View All Candidates**
- Browse stored candidates
- Filter by skills
- Manage database (clear, refresh)

---

## 🔧 **Technical Implementation Details**

### Similarity Calculation Formula
```python
# Cosine similarity between two vectors
similarity = dot(vector1, vector2) / (norm(vector1) * norm(vector2))

# Where:
# - dot(v1, v2) = sum of element-wise multiplication
# - norm(v) = square root of sum of squared elements
# - Result ranges from -1 to 1 (higher = more similar)
```

### Confidence Thresholds
```python
if similarity >= 0.45:    # 45%+ = HIGH confidence
    confidence = "HIGH"
elif similarity >= 0.35:  # 35-44% = MEDIUM confidence  
    confidence = "MEDIUM"
else:                     # <35% = LOW confidence
    confidence = "LOW"
```

### Performance Optimizations
1. **Caching**: Embeddings cached to avoid re-computation
2. **Background Processing**: GitHub data collection doesn't block UI
3. **Persistent Storage**: ChromaDB maintains data between sessions
4. **Batch Processing**: Multiple profiles processed efficiently

---

## 🎯 **Key Benefits of This Approach**

### 1. Semantic Understanding
- Goes beyond keyword matching
- Understands context and meaning
- Can match "Python developer" with "Django backend engineer"

### 2. Evidence-Based Results
- Not just claiming skills, but proving them
- Shows actual code examples and project usage
- Transparent reasoning for matches

### 3. Scalable Architecture
- Vector search scales to thousands of candidates
- Efficient similarity calculations
- Real-time search capabilities

### 4. Multi-Modal Analysis
- Combines profile text, repository analysis, and project descriptions
- AI-powered skill extraction with regex backup
- Comprehensive candidate understanding

---

## 📊 **Sample Data Flow Example**

### Input: Job Description
```
"Looking for a Senior Python Developer with experience in Django, 
REST APIs, PostgreSQL, and AWS deployment. Must have 3+ years 
experience and strong problem-solving skills."
```

### Process:
1. **JD → Vector**: `[0.1, -0.3, 0.8, ..., 0.2]` (1536 dimensions)
2. **Search ChromaDB**: Find candidates with similar vectors
3. **Calculate Similarity**: 
   - Candidate A: 0.67 → 67% match (HIGH confidence)
   - Candidate B: 0.42 → 42% match (MEDIUM confidence)
4. **Extract Skills**: `["Python", "Django", "REST API", "PostgreSQL", "AWS"]`
5. **Find Evidence**: 
   ```json
   {
     "Python": ["def create_user(self):", "import django.contrib.auth"],
     "Django": ["from django.db import models", "class UserSerializer"],
     "AWS": ["boto3.client('s3')", "EC2 deployment script"]
   }
   ```

### Output: Ranked Results
```
🥇 Candidate A - 67% Match (HIGH)
   Skills: Python, Django, PostgreSQL, AWS, Docker
   Evidence: 15 code snippets across 8 repositories
   
🥈 Candidate B - 42% Match (MEDIUM)  
   Skills: Python, Flask, MySQL, Heroku
   Evidence: 8 code snippets across 5 repositories
```

---

This system provides recruiters with intelligent, evidence-based candidate recommendations that go far beyond traditional keyword matching, ensuring better hiring decisions through semantic understanding and transparent reasoning.






## Why LangChain is not used
1. The Current Implementation is Already Sophisticated
there are custom features that LangChain doesn't provide:

* Advanced skill extraction with evidence: Your regex patterns + AI analysis
* Custom similarity scoring: Cosine similarity with confidence levels (HIGH/MEDIUM/LOW)
* GitHub integration: Specialized for developer profiles
* Caching system: Performance optimizations
* Background job processing: For GitHub data collection

2. Performance Considerations
Current Direct Approach:

* Direct AWS SDK calls are faster
* No abstraction layer overhead
* You control caching, retries, timeouts

3. LangChain Approach:

* Additional abstraction layer
* May be slower for high-volume requests
* Less control over performance optimizations
