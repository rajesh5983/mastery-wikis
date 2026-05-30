# Layer 7 — Interview

> **Framework:** Communication, problem-solving, and demonstrating ownership and impact.

---

## Clarify Requirements Before Coding

**Status:** ⬜ Not Started

**Definition:** Before writing any code or query in an interview, ask clarifying questions to understand the precise problem: expected input and output, constraints, edge cases, and whether there is a preferred approach. This demonstrates real engineering behaviour — requirements always have ambiguity.

**Mental Model:** A doctor doesn't prescribe before diagnosing. Ask questions first. It shows you know that requirements are ambiguous and that solutions depend on constraints you don't yet know.

**Common Misconceptions:**
- Asking questions wastes time and signals unpreparedness — interviewers want to see this behaviour; diving straight into code without clarification is the red flag.
- All requirements are obvious from the problem statement — ambiguity is often intentional in interview problems; finding and resolving it is part of what's being evaluated.

**Interview Skeleton:**
- What it is: gathering information before committing to a solution approach
- Why it matters: wrong assumptions lead to wrong solutions; interviewers evaluate whether you clarify before diving in
- Example: for a "find duplicate records" question, ask: what defines a duplicate? which row to keep? what is the table size? is this a one-off or recurring task?

**Free Resources:** https://www.techinterviewhandbook.org/coding-interview-cheatsheet/ — Tech Interview Handbook checklist for coding interview communication and preparation

---

## Ask About Scale, Freshness, SLAs, and Consumers

**Status:** ⬜ Not Started

**Definition:** In system design and pipeline questions, always ask about data volume (MB vs TB vs PB), latency requirements (real-time vs hourly vs daily), SLA commitments (what is the cost of being late?), and who consumes the data (BI tools, ML models, APIs). These answers change the entire architecture.

**Mental Model:** Scale, freshness, SLAs, and consumers are the four compass points of data architecture. Without knowing them, any solution you propose is a guess. With them, you can make principled, defensible design choices.

**Common Misconceptions:**
- Propose a solution first, then adjust for scale — always ask about scale first; a solution designed for 1GB is architecturally different from one for 1PB.
- Consumers don't affect pipeline design — a pipeline feeding an ML model has different requirements (format, freshness, schema stability) than one feeding a BI dashboard.

**Interview Skeleton:**
- What it is: the set of context questions every data engineer should ask before designing any system
- Why it matters: demonstrating that your design depends on these answers signals senior engineering thinking
- Example: when asked to design a pipeline, open with: "How much data? How fresh does it need to be? Who is downstream and what do they need?"

**Free Resources:** https://github.com/donnemartin/system-design-primer — System Design Primer covering scalability trade-offs and design interview patterns

---

## Communicate Assumptions and Trade-offs

**Status:** ⬜ Not Started

**Definition:** As you solve a problem, explicitly state the assumptions you are making and the trade-offs of your approach. This shows you understand that engineering decisions have costs and benefits, and that you are not just pattern-matching to a memorised solution.

**Mental Model:** A navigator who says "I'm assuming we want the fastest route, not the shortest" is more trustworthy than one who silently picks a route. Stating assumptions lets others correct you before you go 100 miles in the wrong direction.

**Common Misconceptions:**
- Stating assumptions makes you look uncertain — it makes you look like an experienced engineer who knows real problems have constraints.
- Trade-off discussions are only expected from senior candidates — entry-level engineers who articulate trade-offs stand out immediately in interviews.

**Interview Skeleton:**
- What it is: making your reasoning explicit so interviewers can evaluate your thought process, not just your output
- Why it matters: interviewers hire for the thinking behind the code, not just the code itself
- Example: "I'm using a hash map here, which trades memory for O(1) lookup — if memory is constrained I'd switch to a sorted array with binary search"

**Free Resources:** https://www.interviewcake.com/article/python/data-structures — Interview Cake guide on communicating trade-offs in technical discussions

---

## Write Neat SQL and Test Edge Cases

**Status:** ⬜ Not Started

**Definition:** In SQL interviews, write readable, consistently formatted queries with meaningful aliases. After writing, proactively test your solution against edge cases: empty tables, NULL values, duplicate rows, ties in rankings, empty result sets, and boundary conditions.

**Mental Model:** Write the query as if a colleague will maintain it tomorrow. Test it as if the data will be as messy as real production data — NULLs everywhere, duplicates you didn't expect, empty result sets at 2am.

**Common Misconceptions:**
- Formatting doesn't matter in interviews — messy SQL is hard to follow and signals habits that create maintenance problems in production; interviewers notice.
- A query that works for the happy path is done — production data always has edge cases; interviewers want to see you proactively identify and handle them.

**Interview Skeleton:**
- What it is: code quality and testing habits applied to SQL in interview settings
- Why it matters: neat, tested SQL demonstrates professional habits and catches bugs before they reach production
- Example: after writing a deduplication query, ask "what happens if all rows have NULL in the ordering column? Let me add a tiebreaker"

**Free Resources:** https://sqlstyle.guide/ — SQL Style Guide for consistent, readable SQL formatting

---

## Use STAR Stories for Project Depth

**Status:** ⬜ Not Started

**Definition:** STAR (Situation-Task-Action-Result) is a framework for answering behavioural interview questions. For data engineering roles, a strong STAR story describes a pipeline or data problem (Situation), your specific ownership (Task), the technical decisions you made (Action), and the measurable impact (Result).

**Mental Model:** STAR is a story structure. Situation sets the scene, Task is your role in it, Action is what you actually did (the technical core), Result is why it mattered. Without Result, the story has no punch.

**Common Misconceptions:**
- Behavioural questions are separate from technical depth — interviewers ask STAR questions to assess real experience and impact; your stories should be specific and technical.
- Any project story works — choose stories that demonstrate scale, complexity, problem-solving under constraints, and quantifiable impact on the business.

**Interview Skeleton:**
- What it is: a structured approach to presenting past experience that demonstrates technical depth and business impact
- Why it matters: unstructured answers lose the interviewer; STAR keeps your answer focused and credible
- Example: "Our daily report was silently failing (S). I owned the fix and monitoring (T). I rewrote incremental logic with idempotency and added row count assertions with Slack alerts (A). Reduced data incidents by 80% (R)."

**Free Resources:** https://www.levels.fyi/blog/star-method-interview.html — Guide to the STAR method specifically for technical engineering interviews

---

## Show Ownership, Impact, and Debugging Skill

**Status:** ⬜ Not Started

**Definition:** Ownership means taking end-to-end responsibility for a system — from design through deployment, monitoring, and on-call support. Impact means connecting technical work to business outcomes. Debugging skill is the ability to systematically diagnose and fix production issues under pressure.

**Mental Model:** Ownership is being the landlord, not the tenant — you don't just live in the system, you're responsible for the pipes and structure. Debugging skill means walking through your diagnostic process step by step, not just producing the fix.

**Common Misconceptions:**
- Ownership is only about building new things — ownership includes monitoring, handling incidents, improving reliability, and documenting for the next person.
- Impact is only measurable in revenue — impact includes SLA improvements, cost reductions, team velocity gains, reduced on-call burden, and stakeholder trust built over time.

**Interview Skeleton:**
- What it is: the qualities that distinguish engineers who ship reliable, trusted systems from those who just complete assigned tasks
- Why it matters: senior engineers want teammates they can trust to own a system end-to-end without supervision
- Example: "When the pipeline broke at 3am, I caught it via a freshness alert, diagnosed skewed partition joins, fixed and backfilled, then added a skew detection check to catch it earlier next time"

**Free Resources:** https://staffeng.com/guides/work-on-what-matters — Staff Eng guide on demonstrating ownership, impact, and leverage at work
