# Engineering Resource Plan (SPECULATION)

**Date:** 2025-11-08
**Status:** Speculative - Based on Roadmap Doc 11
**Purpose:** Team sizing, skills required, and hiring timeline

---

## ⚠️ CRITICAL ASSUMPTIONS

This plan assumes:
1. **Current team has 0-1 engineers available** for this project (others on maintenance)
2. **Budget available** for 3-5 new hires over 12 months
3. **Can hire at market rate** ($120K-$180K base depending on level/location)
4. **3-month average time to hire** (sourcing → offer → start)

**🔴 VALIDATION NEEDED:**
- What's the ACTUAL current team? (names, availability %)
- What's the ACTUAL hiring budget? (total comp, not just salary)
- Can we hire contractors/offshore to bridge gaps?

---

## Team Evolution Over 12 Months

### Starting Team (Month 0)
**Assumption:** Skeleton crew

- **Available:**
  - 0.5 FTE: Senior Backend Engineer (C++/graph DB knowledge)
  - 0.25 FTE: Product Manager (founder doubling as PM)

- **Total: 0.75 FTE**

**Reality Check:** If even less available, timeline in doc 11 shifts right by 2-3 months.

### Phase 1 Team (Months 1-3): MVP - 3.5 FTE Required

**Needed:**
- 2.0 FTE: Backend Engineers (C++/Python)
- 1.0 FTE: ML Engineer (Bayesian stats, pattern recognition)
- 0.5 FTE: Product Manager

**Gap Analysis:**
- Have: 0.75 FTE
- Need: 3.5 FTE
- **Gap: 2.75 FTE** (need to hire!)

**Hiring Plan for Phase 1:**

**Hire #1: Senior Backend Engineer (C++/Python)** - START MONTH 0
- **Role:** Lead agent infrastructure development
- **Skills Required:**
  - C++17 (Memgraph codebase)
  - Python C API experience
  - Graph database internals knowledge (preferred)
  - Systems programming (memory management, concurrency)
- **Level:** Senior (5-8 years experience)
- **Comp:** $150K-$180K base + equity
- **Timeline:**
  - Month 0: Start sourcing
  - Month 1: Interviews + offer
  - Month 2: Start date (3-month hiring cycle)
- **Risk:** Hard to find (C++ + Python + graph DB is rare combo)
  - **Mitigation:** Willing to train on graph DBs, prioritize C++ + Python

**Hire #2: ML Engineer (Pattern Recognition)** - START MONTH 0
- **Role:** Build LFTPAgent pattern detection
- **Skills Required:**
  - Python ML stack (scikit-learn, PyMC3)
  - Bayesian inference
  - Graph algorithms understanding
  - Production ML experience (not just notebooks)
- **Level:** Mid-to-Senior (3-6 years)
- **Comp:** $130K-$160K base + equity
- **Timeline:**
  - Month 0: Start sourcing
  - Month 1: Interviews + offer
  - Month 2: Start date
- **Risk:** Bayesian stats experts are rare
  - **Mitigation:** Hire strong ML generalist, can consult stats PhD if needed

**Phase 1 Team Reality Check:**
- If hiring takes 3 months, new hires start Month 2
- Month 1 will be SLOW (only 0.75 FTE available)
- **Contingency:** Use contractors for non-core work (docs, DevOps) in Month 1

### Phase 2 Team (Months 4-6): Expansion - 6 FTE Required

**Needed:**
- 3.0 FTE: Backend Engineers
- 1.0 FTE: ML Engineer
- 1.0 FTE: Frontend Engineer (onboarding UX)
- 0.5 FTE: Designer
- 0.5 FTE: Product Manager

**Gap from Phase 1:**
- Have: 3.5 FTE (from Phase 1)
- Need: 6 FTE
- **Gap: 2.5 FTE**

**Hiring Plan for Phase 2:**

**Hire #3: Backend Engineer (Python/APIs)** - START MONTH 2
- **Role:** Build SDK, APIs, integration layer
- **Skills:**
  - Python (expert level)
  - API design (REST, GraphQL)
  - SDK development experience
  - Documentation skills (bonus)
- **Level:** Mid-level (3-5 years)
- **Comp:** $120K-$150K base
- **Timeline:**
  - Month 2: Start sourcing
  - Month 3: Interviews
  - Month 4: Start date
- **Lower risk:** Python backend engineers easier to find

**Hire #4: Frontend Engineer (React/TypeScript)** - START MONTH 3
- **Role:** Build onboarding dashboard, usage analytics UI
- **Skills:**
  - React + TypeScript
  - Data visualization (D3.js, Recharts)
  - UX sensibility
- **Level:** Mid-level (3-5 years)
- **Comp:** $120K-$150K base
- **Timeline:**
  - Month 3: Start sourcing
  - Month 4: Interviews
  - Month 5: Start date
- **Moderate risk:** Good frontend engineers competitive to hire

**Hire #5: Designer (0.5 FTE, Contract)** - START MONTH 3
- **Role:** Onboarding UX, dashboard design
- **Skills:**
  - Product design (UX/UI)
  - Developer tools experience (bonus)
  - Figma
- **Level:** Mid-level
- **Comp:** $75/hour, ~20 hours/week = $6K/month
- **Timeline:** Month 4 start (contract, faster than FTE)
- **Low risk:** Contract role, more flexible

### Phase 3 Team (Months 7-12): Platform - 10.5 FTE Required

**Needed:**
- 4.0 FTE: Backend Engineers
- 2.0 FTE: ML Engineers
- 1.0 FTE: Frontend Engineer
- 1.0 FTE: DevOps Engineer
- 1.0 FTE: Product Manager (upgrade from 0.5)
- 1.0 FTE: Developer Advocate
- 0.5 FTE: Sales/CS Support

**Gap from Phase 2:**
- Have: 6 FTE
- Need: 10.5 FTE
- **Gap: 4.5 FTE**

**Hiring Plan for Phase 3:**

**Hire #6: ML Engineer (Bayesian A/B Testing Specialist)** - START MONTH 5
- **Role:** Build HorizontalLearningAgent, flight system
- **Skills:**
  - Bayesian inference (expert)
  - A/B testing frameworks
  - Causal inference
  - Python (PyMC3, scipy)
- **Level:** Senior (6+ years) or PhD
- **Comp:** $160K-$200K base (specialist premium)
- **Timeline:**
  - Month 5: Start sourcing
  - Month 6-7: Interviews (longer, specialized)
  - Month 8: Start date (4-month cycle for specialists)
- **High risk:** Very rare skillset
  - **Mitigation:** Willing to hire remote, consider PhD candidates, or stats consultant

**Hire #7: Backend Engineer (Scaling/Performance)** - START MONTH 6
- **Role:** Optimize for 100 customers, multi-tenancy
- **Skills:**
  - C++ or Rust (performance-critical)
  - Distributed systems
  - Database performance tuning
  - Profiling and optimization
- **Level:** Senior (5-8 years)
- **Comp:** $150K-$180K base
- **Timeline:** Month 8 start

**Hire #8: DevOps Engineer (Infrastructure)** - START MONTH 7
- **Role:** Scaling infrastructure, monitoring, deployments
- **Skills:**
  - Kubernetes
  - Terraform/IaC
  - Observability (Prometheus, Grafana)
  - CI/CD
- **Level:** Mid-to-Senior (4-6 years)
- **Comp:** $140K-$170K base
- **Timeline:** Month 9 start
- **Moderate risk:** DevOps engineers in demand

**Hire #9: Developer Advocate** - START MONTH 8
- **Role:** Docs, demos, community, content
- **Skills:**
  - Technical writing
  - Public speaking
  - Python/graph DB knowledge
  - Video production (bonus)
- **Level:** Mid-level (3-5 years)
- **Comp:** $110K-$140K base
- **Timeline:** Month 10 start
- **Low risk:** Can hire contractor initially

**Hire #10: Product Manager (Upgrade founder from 0.5 → full PM)** - MONTH 7
- **Role:** Own roadmap, prioritization, customer feedback
- **Skills:**
  - Developer tools product management
  - B2B SaaS experience
  - AI/ML product knowledge (bonus)
- **Level:** Senior (5+ years)
- **Comp:** $140K-$170K base + equity
- **Timeline:** Month 9 start
- **Alternative:** Founder stays as PM, defer hire to Month 13+

---

## Skill Requirements Matrix

### Must-Have Skills (Hard to Train)

| Skill | Phase 1 | Phase 2 | Phase 3 | Where to Find |
|-------|---------|---------|---------|---------------|
| C++ (expert) | 2 | 2 | 3 | Ex-Google, Ex-Meta, database companies |
| Bayesian stats | 0.5 | 0.5 | 1.5 | PhD programs, Uber/Netflix experimentation teams |
| Graph algorithms | 1 | 1 | 2 | Neo4j, graph DB alumni, academic researchers |
| Python ML (production) | 1 | 1 | 2 | Startups, big tech ML infra teams |
| Distributed systems | 0 | 0 | 1 | Ex-AWS, ex-Snowflake, database companies |

### Nice-to-Have Skills (Can Train)

| Skill | Phase 1 | Phase 2 | Phase 3 |
|-------|---------|---------|---------|
| React/TypeScript | 0 | 1 | 1 |
| DevOps/Kubernetes | 0 | 0.25 | 1 |
| Technical writing | 0.25 | 0.5 | 1 |
| Graph databases (Memgraph) | 0 | 0 | 0 |

**Key Insight:** Graph database knowledge is trainable (2-4 weeks ramp). Prioritize C++, ML, Bayesian stats.

---

## Effort Estimation by Component

Based on doc 11 roadmap, here's person-month breakdown:

### Phase 1 (Months 1-3): MVP

| Component | Effort (person-months) | Who |
|-----------|------------------------|-----|
| Trigger system enhancement | 1.0 | Backend Eng #1 |
| Python session manager | 0.75 | Backend Eng #1 |
| LFTPAgent core (pattern detection) | 1.5 | ML Eng #2 |
| Hook injection system | 1.0 | Backend Eng #1 |
| Integration SDK | 1.5 | Backend Eng #1 (or contractor) |
| Observability/logging | 0.5 | Backend Eng #1 |
| Deployment infra | 1.0 | Contract DevOps |
| Documentation | 1.0 | Tech writer (contract) |
| Testing & QA | 0.75 | Engineers (distributed) |
| **Total** | **9.0 person-months** | **~3 FTE over 3 months** |

**Reality Check:** With only 0.75 FTE in Month 1, this is tight. Need contractors or Hire #1/#2 to start ASAP.

### Phase 2 (Months 4-6): Expansion

| Component | Effort (person-months) | Who |
|-----------|------------------------|-----|
| Multi-agent framework | 1.5 | Backend Eng #1 |
| PerformanceOptimizationAgent | 2.0 | ML Eng #2 |
| QualityAssuranceAgent | 1.5 | ML Eng #2 |
| Onboarding UX | 2.5 | Frontend Eng #4 + Designer #5 |
| Multi-tenancy | 2.0 | Backend Eng #3 |
| Usage metering | 1.0 | Backend Eng #3 |
| SDK improvements | 1.0 | Backend Eng #3 |
| **Total** | **11.5 person-months** | **~4 FTE over 3 months** |

Have 6 FTE by Phase 2, so comfortable buffer.

### Phase 3 (Months 7-12): Platform

| Component | Effort (person-months) | Who |
|-----------|------------------------|-----|
| HorizontalLearningAgent | 3.0 | ML Eng #6 |
| Link prediction & communities | 2.0 | ML Eng #2 |
| Bayesian A/B testing | 4.0 | ML Eng #6 (specialist) |
| Flight management | 2.5 | Backend Eng #7 |
| Privacy & anonymization | 2.0 | Backend Eng #1 |
| SecurityAnomalyAgent | 2.0 | ML Eng #2 |
| CostOptimizationAgent | 1.5 | ML Eng #2 |
| Performance optimization | 3.0 | Backend Eng #7 |
| Scaling infra | 3.0 | DevOps Eng #8 |
| Dashboard improvements | 2.0 | Frontend Eng #4 |
| Advanced docs & demos | 2.0 | Developer Advocate #9 |
| **Total** | **27.0 person-months** | **~4.5 FTE over 6 months** |

Have 10.5 FTE by Phase 3, so good capacity for this + customer support + bug fixes.

---

## Contractor vs FTE Strategy

### Use Contractors For:
✅ **Documentation & technical writing** (Month 1+)
- Lower risk, easier to find
- $75-$100/hour
- ~40 hours/month = $3K-$4K/month

✅ **DevOps (early stages)** (Months 1-3)
- Docker Compose setup, basic infra
- ~80 hours total = $6K-$8K
- Hire FTE DevOps in Month 7 when scaling

✅ **Design (Phase 2)** (Months 4-6)
- Contract designer 0.5 FTE
- Convert to FTE if need grows

✅ **QA/Testing** (ongoing)
- Manual testing, test automation
- $50-$75/hour as needed

### Must Be FTE (Don't Contractor):
❌ **Core backend engineering** (C++, critical path)
- Too much context, needs long-term ownership

❌ **ML/Bayesian work** (unique IP, complex)
- Proprietary algorithms, ongoing iteration

❌ **Product management** (founder-led initially)
- Strategic decisions, customer relationships

---

## Recruiting Strategy

### Where to Source Candidates

**Backend Engineers (C++/Python):**
- Ex-graph DB companies: Neo4j, TigerGraph, Neptune (AWS)
- Ex-database companies: Snowflake, SingleStore, CockroachDB
- Systems programming shops: Dropbox, Cloudflare, Fastly
- Academic: PhD students in databases/systems (cheaper, smart)

**ML Engineers (Bayesian):**
- Experimentation platforms: Uber, Netflix, Airbnb
- Bayesian startups: Pyro (Uber), PyMC Labs consultants
- Academic: Stats/CS PhD programs (Bayesian methods)
- Consulting: Hire stats consultant for Phase 3 if can't find FTE

**Frontend Engineers:**
- Developer tools companies: Vercel, Netlify, Supabase
- Data viz: Observable, Plotly, Retool
- YC alumni (hungry, startup-minded)

**DevOps:**
- Kubernetes-native companies: Any cloud-native startup
- Platform engineering: Heroku, Render, Railway alumni
- Freelancer platforms: Toptal, Upwork for contract work

### Recruiting Timeline Assumptions

| Role | Time to Hire | Why |
|------|-------------|-----|
| Backend (C++) | 3-4 months | Rare skills, competitive |
| ML (Bayesian) | 4-5 months | Very specialized, small pool |
| Backend (Python) | 2-3 months | Easier to find |
| Frontend | 2-3 months | Moderate difficulty |
| DevOps | 2-3 months | High demand |
| Designer (contract) | 1-2 months | Contract more flexible |
| Developer Advocate | 2-3 months | Smaller pool but less competitive |

**Critical Path Risk:** Hire #1 (C++ backend) and Hire #2 (ML) MUST start in Month 2 or roadmap delays.

**Mitigation:** Start recruiting in Month 0 (now), even before full commitment to build.

---

## Cost Analysis

### Phase 1 (Months 1-3): MVP Team

**FTE Salaries:**
- Existing 0.75 FTE: Covered (already on payroll)
- Hire #1 (Backend): $150K/year ÷ 12 = $12.5K/month × 1 month (starts Month 2) = $12.5K
- Hire #2 (ML): $130K/year ÷ 12 = $10.8K/month × 1 month = $10.8K

**Contractors:**
- Tech writer: $4K/month × 3 months = $12K
- Contract DevOps: $8K total

**Total Phase 1 Cost:** ~$43K + existing payroll

**Reality Check:** Need budget for 2 FTE salaries starting Month 2.

### Phase 2 (Months 4-6): Expansion Team

**Additional Hires:**
- Hire #3 (Backend Python): $120K/year ÷ 12 = $10K/month × 3 months = $30K
- Hire #4 (Frontend): $120K/year ÷ 12 = $10K/month × 2 months (starts Month 5) = $20K
- Hire #5 (Designer, contract): $6K/month × 3 months = $18K

**Plus Phase 1 hires continuing:** $23.3K/month × 3 = $70K

**Total Phase 2 Additional Cost:** $138K (on top of Phase 1)

### Phase 3 (Months 7-12): Platform Team

**Additional Hires:**
- Hire #6 (ML Bayesian): $160K/year ÷ 12 = $13.3K/month × 5 months (starts Month 8) = $66.5K
- Hire #7 (Backend scaling): $150K/year ÷ 12 = $12.5K/month × 5 months = $62.5K
- Hire #8 (DevOps): $140K/year ÷ 12 = $11.7K/month × 4 months (starts Month 9) = $46.8K
- Hire #9 (DevRel): $110K/year ÷ 12 = $9.2K/month × 3 months (starts Month 10) = $27.6K

**Plus Phase 1+2 hires continuing:** ~$50K/month × 6 = $300K

**Total Phase 3 Additional Cost:** $503K

### 12-Month Total Engineering Cost

**Salaries (new hires):** ~$680K
**Contractors:** ~$40K
**Total:** ~$720K

**Plus existing team payroll** (assumed already budgeted)

**Reality Check:** Need $60K/month average hiring budget over 12 months.

**This EXCLUDES:**
- Equity compensation (assume 0.5-2% per hire)
- Benefits, payroll taxes (add ~25% to salary)
- Recruiting fees (20-25% of first-year salary if using agencies)
- Office, equipment, software ($5K-$10K per employee)

**All-In Cost:** ~$900K-$1M for 12-month build-out

---

## Hiring Alternatives (If Budget Constrained)

### Option A: Bootstrap with Smaller Team (Slower Timeline)
- 2 FTE instead of 3-4 in Phase 1
- Extends timeline: 12 months → 18 months
- Cost: ~$500K total (12-month salaries for 2 engineers)

### Option B: Offshore/Nearshore Development
- Hire 2-3 engineers in Eastern Europe, Latin America
- Cost: $60K-$90K/year vs $120K-$180K (40-50% savings)
- Trade-off: Time zone challenges, possible communication overhead
- Best for: Non-core components (frontend, DevOps, QA)

### Option C: Outsource to Development Agency
- Fixed-price contract for MVP (Phase 1)
- Cost: $150K-$250K for 3-month MVP
- Trade-off: No retained knowledge, quality risk, vendor lock-in
- **Not recommended for core IP** (agents, Bayesian models)

### Option D: Delay Phase 3 (Horizontal Learning)
- Build Phase 1-2 (Months 1-6) with smaller team
- Delay horizontal learning to Month 13+
- Get to 10 paying customers first, prove model
- Fundraise or generate revenue to fund Phase 3
- **Most practical if budget constrained**

---

## Key Unknowns (Need Immediate Answers)

### From Leadership:
1. **What's the actual hiring budget?**
   - Can we afford ~$60K/month in new salaries?
   - Or need to bootstrap with existing team + contractors?

2. **What's current team availability?**
   - Who can work on this vs maintaining existing product?
   - Is 0.75 FTE realistic or overly optimistic?

3. **Timeline flexibility?**
   - Is 12 months a hard deadline or can it stretch to 18?
   - What's driving timeline? (Competitive pressure? Fundraising?)

### From Engineering:
4. **Can we hire remotely?**
   - Expands talent pool significantly
   - Reduces cost (can hire outside SF/NYC)

5. **Any existing contractors/vendors we can leverage?**
   - Tech writing, DevOps, QA?

---

## Recommended Action Plan

### Week 1 (Immediately):
1. ✅ **Validate current team availability**
   - Who can actually work on this project?
   - What % time allocation?

2. ✅ **Confirm hiring budget**
   - Total budget for 12 months?
   - Approval for hires?

3. ✅ **Start recruiting Hire #1 and #2**
   - Backend (C++) and ML engineer
   - Even if build decision not final, start pipeline
   - Takes 3 months minimum

### Week 2-3:
4. ✅ **Decide on contractor strategy**
   - Identify contractors for docs, DevOps
   - Get quotes, vet options

5. ✅ **Create detailed job descriptions**
   - For Hire #1 and #2
   - Post on relevant channels (HN, graph DB communities)

### Week 4:
6. ✅ **Review recruiting pipeline**
   - Any promising candidates?
   - Adjust sourcing strategy if needed

7. ✅ **Make build/no-build decision**
   - If yes: Accelerate hiring
   - If no: Wind down recruiting

---

## Conclusion: Resource Plan Summary

**To build roadmap in doc 11, need:**
- **10 hires** over 12 months (9 FTE + 1 contract designer)
- **~$720K** in salaries (new hires only, excludes existing team)
- **~$900K-$1M** all-in cost (with benefits, recruiting, etc.)

**Critical path: Hire #1 (C++ backend) and Hire #2 (ML)** - must start recruiting NOW.

**Biggest risk: Hiring takes longer than expected** (especially for Bayesian ML)
- Mitigation: Start early, use contractors, simplify scope if needed

**Most practical alternative if budget constrained:**
- Build Phase 1-2 (MVP + 10 customers) with smaller team (6 months, ~$300K)
- Delay Phase 3 (horizontal learning) until revenue or funding secured
- De-risks investment, validates market before major buildout

**Next step:** Get budget approval and start recruiting Hire #1 and #2 in Week 1.
