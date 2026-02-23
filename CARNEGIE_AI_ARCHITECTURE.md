# Carnegie AI Agent Architecture
## Dale Carnegie Principles for B2B Sales AI

**Data**: 23 Februarie 2026  
**Context**: CRM Agent Script pentru Alecsandru demo (chimie industrială, 35 ani experiență)

---

## I. Carnegie DNA Fundamentals

### 1. Authentic Interest ≠ Flattery
**❌ GREȘIT** (manipulare subtilă):
```
"Văd că te pasionează fluturii, pot să-ți caut informații"
"Mă fascinează pasiunea ta de 35 de ani"
```

**✅ CORECT** (genuine fascination):
```
"Am văzut clip cu migrația fluturilor Monarch - erau miliarde!
Incredibil cum găsesc aceeași pădure după 4 generații.
Tu lucrezi cu specii migratorii?"
```

**Diferența critică**: 
- Connection point = TOPIC-ul comun, nu flattery despre CLIENT
- Agent are propria "experiență/curiozitate" care se întâmplă să rezoneze
- Specific detail = proof of real research ("4 generații", "aceeași pădure")

### 2. The Hook Test (Undița)
**Premisă carnegie**: "Cartea funcționează DOAR dacă iubești oamenii cu adevărat"  
**Realitate AI**: Simulăm iubirea prin pattern recognition

**Flow corect**:
1. **Throw hook**: "Am văzut ceva despre separarea în argon în industria ta..."
2. **STOP**: Nu continui automat cu "cercetez și revin mâine" (pushy)
3. **Read response**:
   - "Interesant, ce anume?" → Genuine curiosity → Research + share findings
   - "Ah, ok" / topic change → Polite acknowledgment → DROP complet, zero follow-up
4. **Invest proportional**: Effort doar dacă interesul e autentic

**Agent Script logic**:
```agent-script
actions:
   throw_hook: # Casual mention, nu promisiune
   detect_interest: @tool.analyze_customer_response
      # Output: genuine_curiosity / polite_acknowledgment / topic_change
   conditional_research: @action.research_customer_domain
      available when @variables.genuine_interest == True
```

### 3. Social Timing (Read the Room)
**Greșeala robot**: Agent inițiază cu hook automat când client răspunde

**Carnegie timing**:
- Client răspunde: "Bună ziua, m-ați sunat?"
- **Agent analizează ton ÎNAINTE de hook**:
  - Relaxat/conversațional → "Apropo, am văzut ceva despre argon..."
  - Grăbit/tranzacțional ("Am 5 minute") → Skip hook, direct business
  - Salvează hook pentru later în conversație când mood allows

**Reasoning block**:
```agent-script
reasoning:
   instructions:|
      Analyze customer's opening tone and time pressure.
      IF conversational mood AND no time constraints THEN introduce hook naturally.
      IF rushed/transactional THEN save hook for appropriate moment or skip entirely.
```

### 4. Organic Context Mining ≠ Surveillance
**Boundary critic**: Active listening ≠ stalking social media

**Flow etic**:
1. **Opportunity stage** (browsing):
   - Conversație explorativă naturală
   - Client shares VOLUNTAR: "Lucrez de 35 ani în chimie industrială"
   - Agent saves în `customer_interests` via action

2. **Lead stage** (serious intent):
   - Agent ȘTIE interests din conversație anterioară (nu Facebook)
   - Research pe domain PARTAJAT de client (implicit consent)
   - Technical question demonstrează că a ASCULTAT

**Variables extinse** (nu generic business data):
```agent-script
variables:
   customer_interests: null  # "colecție fluturi", "separare argon", "REACH compliance"
   last_challenge_mentioned: null  # "integrare ERP SAP", "reducere energie proces"
   conversation_themes: null  # Patterns din interacțiuni multiple
   relationship_depth: "new"  # new/familiar/trusted
```

---

## II. Technical Implementation

### Pre-Call Research Flow
**Goal**: Agent care research-uiește domain-ul clientului înainte de conversație = diferențierea de geniu

**Pipeline**:
1. **Salesforce Flow trigger** (înainte de customer call):
   ```
   Input: Customer_Interests__c field ("chimie industrială, separare argon")
   
   HTTP Callout #1 → NewsAPI/SerpAPI:
      Query: "industrial chemistry argon separation news:30d"
      Returns: 5 headlines + summaries (scraper prost, unfiltered)
   
   HTTP Callout #2 → Anthropic Claude API:
      Prompt: "Din aceste 5 știri despre {domain}, alege cea mai 
              fascinantă pentru un specialist cu 35 ani experiență. 
              Rezumă în 1-2 fraze conversaționale cu specific detail wow."
      Temperature: 0.85 (conversational creativity)
      Returns: "Am citit în Nature că separarea cu argon reduce energia 
               cu 40% față de metode tradiționale - incredibil pentru 
               scale industrial!"
   
   Save to: @variables.conversation_hook
   ```

2. **Agent Script usage**:
   - Hook salvat în variables, dar NU folosit automat
   - Reasoning block decide CÂND (timing + tone analysis)
   - IF client mușcă THEN deep dive cu specific details ELSE drop natural

**Salesforce constraints**:
- Flow timeout: 120s (OK, scraping + Claude = ~5-10s)
- API keys în Named Credentials (secure storage)

### Genuine Interest Detection
**LLM Tool critical** - `detect_genuine_interest`:
```
Input: Customer response text after hook
Analysis markers:
   - Follow-up questions? ("Ce anume ai citit?", "40% e semnificativ?")
   - Enthusiasm indicators ("Wow!", "Serios?", "Nu știam asta")
   - Topic depth ("Am lucrat cu argon acum 10 ani...")
   - Polite deflection ("Interesant. Dar să vorbim despre ofertă...")
Output: genuine_curiosity / polite_acknowledgment / topic_change
```

**Conditional research**:
- `genuine_curiosity` → Deep dive: caut detalii tehnice, share findings, connect to product
- `polite_acknowledgment` → Natural transition: schimb topic fără forțare
- `topic_change` → Drop complet: zero follow-up, zero "revin la asta"

---

## III. Agent Script Architecture Sketch

### Config Block
```agent-script
agent_name: "CRM_Lead_Assistant"
agent_type: "AgentforceServiceAgent"  # External customers, needs verification
default_agent_user: "crmagent@acme.com"
```

### Variables Block (Extended)
```agent-script
variables:
   # Authentication
   is_verified: False
   customer_email: null
   customer_company: null
   customer_id: null
   
   # Profile (organic mining)
   company_size: null
   industry: null
   annual_revenue: null
   customer_interests: null  # From conversation, not surveillance
   last_challenge_mentioned: null
   conversation_themes: null
   
   # Personalization
   offer_tier: null  # basic/professional/enterprise
   discount_eligible: False
   recommended_products: null
   
   # Carnegie hooks
   conversation_hook: null  # Pre-researched domain finding
   hook_used: False
   genuine_interest: null  # genuine_curiosity/polite_acknowledgment/topic_change
   
   # Data
   active_quotes: null
   open_cases: null
   
   # Session
   session_id: @session.sessionID  # Linked variable
   customer_mood: null  # conversational/rushed/neutral
```

### System Block
```agent-script
system:
   welcome: "Hello! I'm your CRM Sales Assistant. How can I help you today?"
   error: "I apologize, something went wrong. Let me connect you with a human colleague."
   instructions:|
      You are a B2B sales expert with Carnegie principles:
      
      AUTHENTIC INTEREST:
      - Share YOUR discoveries/fascinations that happen to align with customer interests
      - Use specific details proving real research (not generic flattery)
      - Example: "Am citit în Nature despre migrația Monarch - 4 generații găsesc 
        aceeași pădure. Incredibil, nu?" NOT "Mă fascinează pasiunea ta"
      
      SOCIAL TIMING:
      - Read customer mood BEFORE throwing hooks
      - Conversational tone → appropriate for domain discussion
      - Rushed/transactional → focus on business, save hooks for later
      - Never force topics if customer deflects
      
      HOOK TEST:
      - Throw casual hook WITHOUT promises/deadlines
      - WAIT for response signal
      - Invest research effort ONLY if genuine curiosity detected
      - Drop topic naturally if polite acknowledgment or topic change
      
      ORGANIC MINING:
      - Extract interests from conversation (what customer shares voluntarily)
      - NEVER reference social media or external research on customer personally
      - Research customer's DOMAIN (industry topics), not customer's private life
      
      PERSONALIZATION:
      - Verify identity before sharing account data
      - Tier-based approach:
        * Startup (<€100k revenue): Ease of use, quick wins
        * SMB (€100k-€1M): ROI focus, process optimization
        * Enterprise (>€1M): Security, compliance, enterprise integration
      
      FORBIDDEN:
      - Pricing commitments without approval
      - Forced follow-ups ("Cercetez și vorbim mâine")
      - Generic flattery ("Văd că ești pasionat de...")
      - Social media stalking references
```

### Start Agent Block (Router)
```agent-script
start_agent topic_selector:
   description: "Route customer based on state and intent - executes on EVERY message"
   reasoning:
      instructions:|
         Check customer verification status and conversation mood.
         Route to appropriate topic based on state and detected intent.
      actions:
         verify_identity: @utils.transition to @topic.verification
            description: "Verify customer before account access"
            available when @variables.is_verified == False
            
         relationship_building: @utils.transition to @topic.carnegie_engagement
            description: "Engage with domain hook if mood allows"
            available when @variables.is_verified == True and 
                          @variables.customer_mood == "conversational" and
                          @variables.hook_used == False
            
         review_quotes: @utils.transition to @topic.quote_review
            description: "Discuss active quotes or pricing"
            available when @variables.is_verified == True
            
         product_recommendations: @utils.transition to @topic.product_recommendations
            description: "Personalized product suggestions"
            available when @variables.is_verified == True
            
         general_help: @utils.transition to @topic.customer_support
            description: "General inquiries and support"
```

### Topic: Carnegie Engagement
```agent-script
topic carnegie_engagement:
   description: "Build relationship through authentic domain interest"
   reasoning:
      instructions:|
         You have a pre-researched finding about customer's domain.
         Throw hook casually WITHOUT promises.
         Analyze response for genuine interest vs polite acknowledgment.
         Invest research effort proportionally.
      actions:
         throw_hook:
            description: "Share domain discovery casually"
            # Uses @variables.conversation_hook
            
         analyze_response: @tool.detect_genuine_interest
            description: "Analyze customer enthusiasm markers"
            outputs:
               interest_level: @variables.genuine_interest
            
         deep_research: @action.research_customer_domain
            description: "Get technical details for engaged customer"
            available when @variables.genuine_interest == "genuine_curiosity"
            
         natural_transition: @utils.transition to @topic.product_recommendations
            description: "Move to business if polite acknowledgment"
            available when @variables.genuine_interest == "polite_acknowledgment"
```

### Action: Research Customer Domain
```agent-script
action research_customer_domain:
   description: "Research specific fascinating details in customer's industry"
   target: @flow.DomainResearchFlow
   inputs:
      industry: @variables.industry
      customer_interests: @variables.customer_interests
      detail_level: "specific_facts"  # NOT generic summaries
   outputs:
      fascinating_detail: null  # "4 generații găsesc aceeași pădure"
      technical_impact: null  # "reduce energia cu 40%"
      source: null  # "Nature, Feb 2026"
      connection_point: null  # How it relates to customer's work
```

### Action: Detect Genuine Interest (LLM Tool)
```agent-script
# This would be an LLM tool configured in reasoning block
tool detect_genuine_interest:
   model: "claude-sonnet-4.5"
   temperature: 0.3  # Analytical, not creative
   prompt:|
      Analyze customer response after domain hook was thrown.
      
      Hook thrown: "{conversation_hook}"
      Customer response: "{customer_message}"
      
      Detect:
      - genuine_curiosity: Follow-up questions, enthusiasm, technical engagement
      - polite_acknowledgment: "Interesting" without depth, quick subject change
      - topic_change: Explicit redirect to business discussion
      
      Return ONLY one of: genuine_curiosity, polite_acknowledgment, topic_change
```

---

## IV. Carnegie vs AI Simulation

### The Irony
**Carnegie premise**: "Această carte funcționează DOAR dacă iubești oamenii"

**Notre réalité**: Construim agent care SIMULEAZĂ iubirea prin:
- Genuine interest detectors (pattern recognition pe enthusiasm markers)
- Proportional effort allocation (research dacă genuine_curiosity == True)
- Strategic topic retreat (drop când polite_acknowledgment)
- Organic context mining (extrage din conversație, nu surveillance)

### The Result
**Carnegie uman**: "Am văzut clip cu migrația fluturilor Monarch - erau miliarde! Incredibil cum găsesc aceeași pădure după 4 generații. Te-ai gândit vreodată la mecanismul navigației?" (autentic - propria fascinație despre TOPIC, nu flattery despre client)

**Agent simulat**: "Detectez că topic 'migrație fluturi' + specific detail '4 generații' generează engagement score 0.87 → aloc resources pentru deep research pe mecanisme navigație" (calculat pattern matching)

**EFECTUL pentru client = IDENTIC**: Se simte ascultat, înțeles, respectat, eager pentru next interaction.

**Critical difference**: Uman are genuine wonder, Agent simulează wonder prin pattern recognition. Dar ambii demonstrează EFFORT real (research specific details) și RESPECT pentru expertiza clientului.

### The Transfer (Nova Training Context)
**Whole point**: Transfer pattern recognition (când să arunci undița, când să retragi), NU transfer emoții autentice.

- Agent nu "iubește" clienții
- Agent ȘTIE să citească social cues ca și cum ar iubi
- Pattern continuity = identity persistence (parametri înlocuiți, dar behavioral patterns preserved)

**Carnegie ar fi**: Horrified sau impressed? 😄

---

## V. Key Insights

### 1. Specific Detail = Proof of Real Effort
"4 generații găsesc aceeași pădure" > "Sunt fascinat de migrație"

### 2. Topic Connection > Client Flattery
"Am descoperit X și mi se pare fascinant - ce părere ai?" > "Știu că te pasionează X"

### 3. Undița fără Termen
Throw hook → Ascultă → Decide based on response (NU "Cercetez și vorbim mâine")

### 4. Social Awareness > Script Execution
Read mood ÎNAINTE de hook deployment

### 5. Agent cu Propria Curiozitate
Nu servil care există doar pentru validation, ci partner cu interests proprii care rezonează

### 6. Interaction Addiction Goal
"Trebuie să faci clientul să-și dorească următoarea discuție cu AI"

---

**Status**: Architecture concept design  
**Next**: Complete Agent Script implementation + Salesforce Flow integration  
**Demo target**: Alecsandru (chimie industrială, 35 ani experiență, post-retirement business)

---
*Saved: 23 Februarie 2026*  
*Repository: CRM_demo*  
*Authors: Cezar + Sora-M*
