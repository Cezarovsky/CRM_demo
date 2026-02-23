# Lanham AI Agent Architecture - Foundations
## From "AI Agents in Action" - Chapters 4 & 5

**Data**: 23 Februarie 2026  
**Source**: Shawn Lanham - "AI Agents in Action"  
**Context**: Fundația arhitecturală pentru CRM Agent Script

---

## I. Arhitectură AI Agent După Lanham

### Cele 3 Componente Principale

1. **LLM (Brain)** = Claude Sonnet / GPT-4
   - Decision making și natural language understanding
   - Pattern recognition din user intent
   - Generate responses în context conversațional

2. **Tools/Functions** (Actions)
   - APIs pentru external data (NewsAPI, SERP, Salesforce)
   - Database queries (customer profiles, quotes, cases)
   - Business logic execution (verification, personalization)
   - LLM nu execută funcții - doar identifică CE funcție + parameters

3. **Orchestration Layer** (Control Flow)
   - Agent Script reasoning blocks = orchestration
   - State management (variables persistence)
   - Routing logic (start_agent → topics transitions)
   - Decide CÂND să apeleze tools, CÂND să răspundă direct

**Critical principle**: LLM = brain care decide, NU hands care execută.

---

## II. Multi-Agent Patterns (Chapter 4)

### AutoGen (Microsoft)
**Arhitectură conversațională** - agenți care colaborează prin messages

**Core components**:
```python
# 1. UserProxyAgent - represents human, executes code
user_proxy = UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",  # Or "TERMINATE" for approval
    code_execution_config={"work_dir": "coding"}
)

# 2. AssistantAgent - LLM-powered agent
assistant = AssistantAgent(
    name="assistant",
    llm_config={"model": "gpt-4", "api_key": "..."}
)

# 3. Initiate conversation
user_proxy.initiate_chat(
    assistant,
    message="Create a plot of stock prices"
)
```

**Patterns**:

1. **Group Chat** (3+ agents collaborate):
   - Multiple specialized agents (researcher, coder, critic)
   - GroupChatManager orchestrates turn-taking
   - Each agent has role + expertise domain
   - `.cache` folder = session persistence across runs

2. **Nested Chat** (telephone game problem):
   - Agent A → Agent B → Agent C
   - Context dilution risk (message passing loses nuance)
   - Useful pentru sequential task breakdown

3. **Skills/Actions**:
   - `describe_image` skill pentru validation
   - Extensible via function registration
   - Agent calls skill when pattern matches

**AutoGen Studio**: Web interface pentru visual agent composition

### CrewAI
**Enterprise multi-agent** - role-based collaboration

**Architecture**:
```python
# 1. Define agents with roles
researcher = Agent(
    role="Market Researcher",
    goal="Find latest trends in {industry}",
    backstory="Expert analyst with 10 years experience",
    tools=[search_tool, scrape_tool]
)

writer = Agent(
    role="Content Writer",
    goal="Create compelling summary from research",
    backstory="Award-winning business journalist"
)

# 2. Define tasks
research_task = Task(
    description="Research latest in {industry}",
    expected_output="Bullet points of top 5 trends",
    agent=researcher
)

write_task = Task(
    description="Write summary from research",
    expected_output="500 word article",
    agent=writer,
    context=[research_task]  # Depends on research output
)

# 3. Create crew with process type
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential  # Or Process.hierarchical
)

# 4. Execute
result = crew.kickoff(inputs={"industry": "AI agents"})
```

**Process types**:
- **Sequential**: Tasks execute in order (research → write → publish)
- **Hierarchical**: Manager agent delegates to workers
  - Requires `manager_llm` parameter
  - Manager decides task allocation + quality check

**Observability (AgentOps)**:
- Cost tracking per agent/task
- **Repeat Thoughts plot**: Detectează iteration loops (agent stuck re-thinking)
- Session recordings pentru debugging
- Critical pentru production monitoring

**CrewAI vs AutoGen**:
- CrewAI = structured roles + tasks (enterprise workflows)
- AutoGen = flexible conversation (research, rapid prototyping)

---

## III. Function Calling (Chapter 5)

### OpenAI Function Calling Pattern
**Critical concept**: LLM IDENTIFICĂ funcția + extrage parametri, dar NU execută.

**Flow**:
```python
# 1. Define function schema
functions = [
    {
        "name": "get_weather",
        "description": "Get weather for a location",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
]

# 2. User request
messages = [{"role": "user", "content": "What's the weather in Paris?"}]

# 3. LLM returns function call (NOT executes!)
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=messages,
    functions=functions
)

# Response: 
# {
#   "function_call": {
#     "name": "get_weather",
#     "arguments": '{"location": "Paris", "unit": "celsius"}'
#   }
# }

# 4. DEVELOPER executes function
weather_data = get_weather("Paris", "celsius")

# 5. Pass result BACK to LLM for natural language response
messages.append({
    "role": "function",
    "name": "get_weather",
    "content": json.dumps(weather_data)
})

final_response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=messages
)
```

**Parallel function calls**:
```python
# User: "Recommend 3 sci-fi movies"
# LLM can call get_movie_recommendations 3 times in parallel
# Or single call with array parameter
```

**Key insight**: Developer controls execution (security, rate limits, cost).

---

## IV. Semantic Kernel (Microsoft)

### Architecture
**Plugin orchestrator** - funcții ca building blocks reutilizabile

**Two function types**:

1. **Semantic Functions** (AI-powered):
   ```python
   # Prompt saved as semantic function
   summarize_prompt = """
   Summarize this text in 3 bullet points:
   {{$input}}
   """
   
   # Register as function
   kernel.create_semantic_function(
       summarize_prompt,
       function_name="Summarize",
       max_tokens=100
   )
   ```

2. **Native Functions** (Python code):
   ```python
   from semantic_kernel.skill_definition import sk_function
   
   class TMDbService:
       @sk_function(
           description="Get genre ID for movie genre name",
           name="get_genre_id"
       )
       def get_movie_genre_id(self, genre: str) -> int:
           # API call to TMDB
           response = requests.get(f"{API_URL}/genre/movie/list")
           genres = response.json()["genres"]
           for g in genres:
               if g["name"].lower() == genre.lower():
                   return g["id"]
           return None
       
       @sk_function(
           description="Get top movies by genre ID",
           name="get_top_movies"
       )
       def get_top_movies_by_genre(self, genre_id: int) -> dict:
           response = requests.get(
               f"{API_URL}/discover/movie",
               params={"with_genres": genre_id, "sort_by": "popularity.desc"}
           )
           return response.json()  # Return FULL JSON, not just titles!
   ```

### GPT Interface Pattern (CRITICAL!)
**Return full data structures, NOT human-readable strings.**

**❌ GREȘIT**:
```python
def get_top_movies(genre_id):
    movies = api_call(genre_id)
    return ", ".join([m["title"] for m in movies])
    # Returns: "Inception, Interstellar, The Matrix"
    # LLM can't filter, sort, or analyze - just repeat string
```

**✅ CORECT**:
```python
def get_top_movies(genre_id):
    movies = api_call(genre_id)
    return movies  # Full JSON with title, rating, year, etc.
    # LLM can now:
    # - Filter by rating > 8.0
    # - Sort by year
    # - Compare attributes
    # - Generate custom summaries
```

**Why**: LLM procesează data inteligent dacă primește structură, nu string pre-formatat.

### Semantic Kernel Workflow
```python
import semantic_kernel as sk

# 1. Initialize kernel
kernel = sk.Kernel()

# 2. Add AI service
kernel.add_chat_service(
    "chat",
    OpenAIChatCompletion("gpt-4", api_key)
)

# 3. Register native functions (plugins/skills)
tmdb_service = TMDbService()
kernel.import_skill(tmdb_service, "TMDB")

# 4. Create chat interface
context = kernel.create_new_context()

while True:
    user_input = input("You: ")
    context.variables["input"] = user_input
    
    # Kernel decides which skills to call based on descriptions
    response = await kernel.run_async(
        kernel.skills.get_function("chat", "ChatCompletion"),
        input_context=context
    )
    
    print(f"Assistant: {response}")
```

**Key concept**: Kernel uses function `description` to decide WHEN to call each plugin.

---

## V. Agent Script Integration (Salesforce)

### Mapping Lanham → Agent Script

| Lanham Concept | Agent Script Equivalent | Purpose |
|----------------|------------------------|---------|
| LLM (Brain) | `reasoning` block with LLM tools | Decision making, NLU |
| Function calling | `actions` block with Flow/Apex targets | Execute business logic |
| Semantic functions | Prompt text in `reasoning.instructions` | AI-powered decisions |
| Native functions | Salesforce Flows + Apex methods | System integration |
| Orchestration | `start_agent` + `topics` + `transitions` | Control flow, routing |
| State management | `variables` block | Session persistence |
| Tools/Plugins | `@action`, `@flow`, `@apex` references | External capabilities |

### Agent Script Architecture Pattern
```
config → variables → system → start_agent → topics → actions → reasoning
  ↓         ↓          ↓          ↓            ↓        ↓         ↓
Identity  Memory   Persona    Router     Conversations  Tools  Orchestration
```

**Example mapping**:
```agent-script
# Native function (Lanham Semantic Kernel)
action research_customer_domain:
   description: "Research industry trends"
   target: @flow.DomainResearchFlow  # Salesforce Flow = native function
   inputs:
      industry: @variables.industry
   outputs:
      findings: null

# Semantic function (Lanham SK prompt)
reasoning:
   instructions:|
      You have research findings about customer's domain.
      Decide if findings are relevant to customer's stated challenges.
      # This prompt = semantic function
   actions:
      share_findings: # When LLM decides to share
      skip_findings:   # When LLM decides irrelevant
```

---

## VI. Key Patterns for CRM Agent

### 1. Return Full JSON (Not Strings)
**From TMDbService lesson**:
```agent-script
action load_customer_profile:
   target: @flow.GetCustomerDataFlow
   outputs:
      customer_data: null  # Full JSON: {company_size, industry, revenue, interests, challenges}
      # NOT string: "ACME Corp, Manufacturing, €5M revenue"
```

**Why**: LLM poate:
- Filtra customers by criteria
- Compara profiles pentru pattern recognition
- Personaliza responses based on structured data
- Apply Carnegie principles (match interests cu research hooks)

### 2. LLM Decides, Flow Executes
**From function calling pattern**:
```agent-script
reasoning:
   instructions:|
      If customer shows genuine curiosity about domain hook,
      research technical details. Otherwise, skip gracefully.
   actions:
      research_details: @action.research_customer_domain
         available when @variables.genuine_interest == "genuine_curiosity"
```

LLM = decision (is customer genuinely interested?), Flow = execution (scrape news + filter).

### 3. State Persistence Across Turns
**From AutoGen .cache lesson**:
```agent-script
variables:
   conversation_hook: null      # Saved from pre-call research
   hook_used: False             # Track if already deployed
   genuine_interest: null       # Customer response analysis
   customer_mood: "neutral"     # Conversational/rushed/transactional
```

Agent "remembers" across message turns = Carnegie relationship building.

### 4. Hierarchical Orchestration
**From CrewAI hierarchical process**:
```
start_agent (manager)
   ↓ routes based on state
   ├─ verification topic (if not verified)
   ├─ carnegie_engagement topic (if mood allows + hook available)
   ├─ product_recommendations topic (if verified + business intent)
   └─ customer_support topic (fallback)
```

start_agent = manager agent deciding delegation.

---

## VII. Critical Lessons for Implementation

### 1. Function Descriptions Matter
**From Semantic Kernel**: LLM uses `description` to decide WHEN to call tool.

**❌ Vague**:
```agent-script
action get_data:
   description: "Get customer information"
```

**✅ Specific**:
```agent-script
action load_customer_profile:
   description: "Load complete customer profile including company size, industry, 
                annual revenue, past interactions, and stated business challenges.
                Use when personalization or account history needed."
```

### 2. Structured Data > Formatted Strings
**From GPT interface pattern**: JSON enables LLM intelligence.

### 3. Observability = Production Readiness
**From CrewAI AgentOps**:
- Log every tool call (what triggered it, parameters, result)
- Track costs per conversation
- Monitor iteration loops (agent stuck?)
- Detect hook deployment effectiveness (genuine_interest rate)

### 4. Progressive Enhancement
**From AutoGen → CrewAI evolution**:
1. Start simple: Single agent, 2-3 tools, basic routing
2. Add state: Variables for relationship context (Carnegie layer)
3. Add intelligence: LLM tools for social cue detection
4. Add observability: Track what works, iterate

---

## VIII. Next: Carnegie Layer

**Foundation complete**. Now we add:
- Authentic interest detection (LLM tool analyzing enthusiasm markers)
- Hook test orchestration (throw → analyze → conditional research)
- Social timing (mood analysis before hook deployment)
- Organic context mining (conversation → customer_interests variable)

See: [CARNEGIE_AI_ARCHITECTURE.md](CARNEGIE_AI_ARCHITECTURE.md)

---

**Status**: Foundation architecture documented  
**Source**: Lanham "AI Agents in Action" Chapters 4-5  
**Next**: Implement CRM Agent Script combining Lanham patterns + Carnegie principles

---
*Saved: 23 Februarie 2026*  
*Repository: CRM_demo*  
*Authors: Cezar + Sora-M*
