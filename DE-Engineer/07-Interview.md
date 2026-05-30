# Layer 7 — Interview

> **Framework:** Communication, problem-solving, and demonstrating ownership and impact.

---

## Clarify Requirements Before Coding

**Status:** ⬜ Not Started

**Definition:** Before writing any code or SQL in a technical interview, ask targeted clarifying questions to establish the precise problem parameters: expected input and output shapes, data volume constraints, edge cases that matter, and whether there are specific tools or approaches preferred. This mirrors real engineering behaviour — production requirements are always ambiguous, and surfacing that ambiguity early is a core engineering skill.

**Key Mental Model:** A doctor doesn't prescribe before diagnosing. Clarifying questions are your diagnostic instruments — they tell you which of several valid solutions is actually appropriate. Diving straight into code before clarifying is the equivalent of prescribing without asking about symptoms.

**How It Works:**
- The first 60–90 seconds of a technical interview question should be spent restating the problem in your own words and asking 2–3 targeted questions. Restating the problem in your own language forces you to understand it deeply and surfaces misunderstandings before they become wasted coding effort.
- Questions should be scoped and purposeful, not a laundry list. For a SQL problem: "What defines a duplicate — all columns matching, or just a specific key?" For a pipeline design question: "What's the approximate data volume and how fresh does it need to be?" For a coding problem: "Can I assume the input is sorted, or should I handle unsorted input?"
- Ambiguity in interview problems is often deliberate — the interviewer wants to observe whether you recognise that requirements are incomplete and ask to fill the gaps, or whether you silently make unexamined assumptions and build the wrong thing. Recognising and surfacing ambiguity is evaluated as a positive signal at senior levels.
- After receiving clarifying answers, briefly narrate your updated understanding: "So I'm solving for a 10M-row table, I should keep the most recently updated duplicate, and I don't need to handle NULL primary keys." This confirms alignment before investing time in a solution and gives the interviewer one more opportunity to redirect if your understanding is still off.
- The clarification step also serves as thinking time — asking questions while reviewing the problem buys time to think before committing to a direction, without the negative signal of silence. See [[DE-Engineer/01-Foundation]] for the related communication-while-coding practice.

**Common Misconceptions:**
- Asking clarifying questions wastes limited interview time and signals unpreparedness — interviewers explicitly want to observe this behaviour. Jumping straight into code without clarification is the red flag, not the clarifying question. Most interview rubrics specifically credit the "clarifies requirements" behaviour.
- The problem statement contains all the information you need — interview problem statements are almost always intentionally underspecified. Ambiguity about what constitutes a duplicate, what scale means, and what edge cases to handle is built into problems to test whether candidates recognise and address it.

**Interview Answer Skeleton:**
- **What it is:** The deliberate practice of asking targeted questions to disambiguate problem requirements before committing to a solution direction — demonstrating that you treat interviews as engineering problems with real constraints, not as performance tests with memorised solutions.
- **Why it matters / trade-offs:** Wrong assumptions lead to technically correct but contextually wrong solutions. An interviewer who watches you build an O(n²) solution when they intended a large-scale problem has learned nothing useful about your engineering judgment. Clarifying questions prevent this and create collaborative dialogue. The only downside is the small time cost of 60–90 seconds upfront.
- **Example or context:** Given "find duplicate records in a customer table": ask "What defines a duplicate — identical customer_id, or identical name and email? Which duplicate should I keep — the most recent by created_at, or the most complete by null count? Is this a one-off cleanup query or a recurring pipeline step?" These three questions determine whether the solution is ROW_NUMBER() + filter, a MERGE statement, or a scheduled dbt incremental model.

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — open-source DE interview preparation handbook covering communication frameworks, technical question patterns, and requirement clarification strategies
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference with interview guides covering system design question approaches and requirement gathering in technical interviews

---

## Ask About Scale, Freshness, SLAs, and Consumers

**Status:** ⬜ Not Started

**Definition:** In data system design interviews, the four questions that most dramatically change architectural choices are: how much data (scale), how current must the output be (freshness), what are the consequences of being late or wrong (SLA), and who receives this data and how do they use it (consumers). Every major pipeline architecture decision is downstream of these four parameters.

**Key Mental Model:** Scale, freshness, SLAs, and consumers are the four compass points of data architecture. Without knowing them, any proposed solution is an educated guess with undefined fit. With them, you can make principled, defensible design choices and explain precisely why you chose streaming over batch, columnar warehouse over data lake, or daily over hourly.

**How It Works:**
- Scale determines the execution model: a 10MB table processes fine with pandas on a single machine; a 10TB daily ingestion requires distributed Spark or a warehouse bulk load; a 10PB historical dataset requires partitioned external tables and careful cost management. These are different tools, different architectures, and different operational models — asking about scale first prevents proposing the wrong one.
- Freshness requirements determine the pipeline pattern: "stakeholders check the dashboard every morning" → daily batch job completing by 6am is sufficient and far cheaper than streaming. "Fraud detection must act within 30 seconds of a transaction" → streaming with sub-minute latency is genuinely required. Most "real-time" requests in interviews are actually "fresh by morning" requirements that daily batch satisfies adequately. See [[DE-Engineer/05-Scale]] for the cost implications.
- SLA context reveals the cost of failure: a dashboard showing marketing spend to a CMO has a "nice to have" SLA — late data is annoying. A data feed powering real-time pricing for an e-commerce platform has an SLA with direct revenue impact — late data loses sales. This context determines how much redundancy, retry logic, and monitoring investment is appropriate.
- Consumer type shapes format and schema decisions: an ML model consuming training data needs dense feature vectors in Parquet or TFRecord format, schema stability over time, and potentially specific temporal splits. A BI dashboard needs well-named columns, aggregated grain, and fast query response. A REST API consumer needs JSON serialisation and versioned schema. The same underlying data needs different shaping for each consumer type.
- Asking these questions in a system design interview also demonstrates that you understand these are the real variables that drive engineering decisions — rather than pattern-matching to a favourite architecture regardless of context. This is one of the clearest signals that differentiates senior engineers from junior ones in interview settings.

**Common Misconceptions:**
- Propose a general solution first, then refine for scale — a 1GB solution and a 1PB solution are architecturally different from the start: different tools, different partitioning strategies, different cost models, different failure modes. Starting with the wrong scale assumption wastes the entire interview and produces a solution that would fail in production.
- Consumer requirements are a post-design detail — the consumer's access pattern and format requirements affect schema design, grain choice, materialisation strategy, and even table engine selection from the beginning. An ML training pipeline that discovers at the end that the model needs daily snapshots rather than latest-state data needs to be redesigned from scratch if that was discovered late.

**Interview Answer Skeleton:**
- **What it is:** The four essential context dimensions — scale, freshness, SLA severity, and consumer access patterns — that determine which data architecture is appropriate. These should be asked before any solution is proposed, because each parameter eliminates entire categories of architectural choices.
- **Why it matters / trade-offs:** Demonstrating that your architecture depends on these parameters (not on technology preference) signals senior engineering thinking. The interviewers want to see that you understand these variables drive decisions. The only downside is taking 2–3 minutes upfront — this investment consistently produces better-fitted solutions.
- **Example or context:** Asked to "design a pipeline for sales data": open with "How much sales data per day — thousands of orders or millions? How fresh does it need to be — end of day, hourly, or sub-minute? Who consumes it — a dashboard checked at 9am, an ML model training daily, or a real-time pricing engine? And what happens if the data is 2 hours late — is it an inconvenience or a P0 incident?" These four questions alone narrow the architecture from ten possible approaches to one or two clearly appropriate ones.

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — covers system design interview patterns for data engineering, including how to structure design questions and evaluate architectural trade-offs
- [Confluent Developer](https://developer.confluent.io) — covers streaming vs batch decision frameworks, consumer pattern design, and architectural thinking for event-driven systems

---

## Communicate Assumptions and Trade-offs

**Status:** ⬜ Not Started

**Definition:** Throughout a technical interview, explicitly narrate the assumptions you are making ("I'm assuming the dataset fits in memory") and the trade-offs of your chosen approach ("this is O(n) space but O(1) lookup — if memory is constrained I'd use a different structure"). This demonstrates that you understand engineering decisions have costs and benefits, and that you are reasoning about the problem rather than recalling a memorised answer.

**Key Mental Model:** A navigator who says "I'm assuming we want the fastest route, not the shortest — should I switch?" is more trustworthy than one who silently picks a direction. Voicing assumptions lets others correct you before you commit 100 miles in the wrong direction. In interviews, stated assumptions invite the interviewer into the problem-solving process.

**How It Works:**
- State assumptions at the moment you make them, not retrospectively. "I'm treating NULL customer_ids as invalid and filtering them out — is that the right approach for this business context?" This invites real-time feedback and demonstrates that you understand assumptions are choices, not obvious facts.
- Trade-off communication follows a consistent structure: state the approach, name the advantage, name the disadvantage, state when you'd choose differently. "I'm using a CTE here for readability — it may be re-evaluated multiple times, but for this query volume that's fine. If this ran at high frequency I'd materialise it as a temp table instead."
- Compare alternatives explicitly rather than presenting your solution as the only option: "I could solve this with ROW_NUMBER() in a CTE or with GROUP BY + HAVING — I'm choosing ROW_NUMBER because I need to control which duplicate to keep, which GROUP BY can't do. If I only needed to identify duplicates without caring which row survives, GROUP BY would be cleaner."
- Complexity analysis is a specific form of trade-off communication: "This solution is O(n log n) due to the sort — I could get O(n) with a hash map but that would use O(n) extra memory. For the expected data sizes in this problem, the sort is fine." Stating Big-O signals algorithm literacy and shows you understand performance at scale.
- When you encounter a trade-off you genuinely don't know the answer to, naming it is still valuable: "I'm not sure whether Snowflake materialises this CTE or inlines it — I'd check the query profile in production to verify. In Postgres I know it inlines by default." This demonstrates intellectual honesty and practical verification habits. See [[DE-Engineer/02-SQL]] for SQL trade-off depth.

**Common Misconceptions:**
- Stating assumptions out loud signals uncertainty and lack of confidence — the opposite is true. Unstated assumptions that turn out to be wrong signal poor judgment. Engineers who name their assumptions demonstrate awareness that real engineering always involves tradeoffs and incomplete information — exactly what senior roles require.
- Trade-off discussions are only appropriate for senior-level interview questions — entry-level candidates who articulate trade-offs stand out immediately because most candidates pattern-match to memorised solutions without reasoning. At any level, showing reasoning earns credit beyond what the code alone demonstrates.

**Interview Answer Skeleton:**
- **What it is:** The practice of making implicit engineering reasoning explicit throughout an interview — stating the assumptions behind design choices and the trade-offs of the chosen approach versus alternatives, turning a performance into a collaborative problem-solving conversation.
- **Why it matters / trade-offs:** Interviewers are evaluating the quality of your engineering judgment, not just the correctness of the output. A solution with explained trade-offs and acknowledged limitations demonstrates more engineering maturity than a "correct" solution presented without any reasoning. The only cost is slowing down slightly to narrate — a worthwhile exchange.
- **Example or context:** Writing a window function to deduplicate a large table: "I'm using ROW_NUMBER() here over a hash-partitioned window — this works well when the deduplication key is evenly distributed. If customer_id is heavily skewed (e.g., 40% of rows share the same customer_id), this creates a data skew problem in Spark that I'd handle with salting or a different partition strategy. For the warehouse SQL case, skew doesn't affect window functions the same way — the PARTITION BY just organises the computation, not the physical data distribution."

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — includes interview preparation content covering trade-off communication, system design reasoning, and how to articulate engineering decisions
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference with frameworks for communicating engineering trade-offs in interview contexts

---

## Write Neat SQL and Test Edge Cases

**Status:** ⬜ Not Started

**Definition:** In SQL interviews, write consistently formatted, readable queries with meaningful table aliases and clear structure. After writing, proactively walk through edge cases that could cause incorrect results or errors: empty tables, NULL values in filter or join columns, duplicate rows, ties in rankings, mismatched data types in joins, and boundary dates in time-based filters.

**Key Mental Model:** Write the query as if a colleague will maintain it in six months — indented, aliased, and readable. Test it as if the data will be as unpredictable as real production data — NULLs everywhere, duplicates nobody documented, timestamps in the wrong timezone, empty result sets at 2am.

**How It Works:**
- SQL formatting conventions that signal professional habits: uppercase reserved words (SELECT, FROM, WHERE, GROUP BY), one clause per line, consistent 4-space indentation for nested subqueries, meaningful aliases (o for orders, c for customers, not a and b), and CTEs named for what they represent (deduplicated_orders, not cte1).
- Edge case methodology: after writing a query, trace through it mentally with a minimal dataset that covers each boundary condition. For a deduplication query: what happens when all rows for a customer have the same updated_at (tie in the ordering column)? ROW_NUMBER() assigns an arbitrary winner — is that acceptable or do you need a secondary tiebreaker?
- NULL propagation edge cases are the most common sources of incorrect interview SQL. For any JOIN: what happens to rows in the left table with no match in the right table? For any WHERE clause with a negative condition (`!= 'X'`): are NULLs correctly handled with an explicit OR IS NULL clause? For any AVG(): does the business want the average over all rows or only the non-NULL rows?
- Time-based edge cases: a date filter `WHERE order_date = '2024-01-15'` on a TIMESTAMP column will miss records at midnight and records with timezone offsets. The correct form is `WHERE order_date >= '2024-01-15' AND order_date < '2024-01-16'`. Asking "should this include the full day including midnight UTC, or local timezone?" demonstrates production thinking.
- Explicitly narrate edge case testing during the interview: "Let me check what happens if two rows for the same customer have the same updated_at... ROW_NUMBER() would pick one arbitrarily. If I need deterministic tie-breaking I should add a secondary sort on customer_id or a hash of all columns. Is that required here?" This thought process, narrated aloud, demonstrates production readiness. See [[DE-Engineer/02-SQL]] for NULL handling depth.

**Common Misconceptions:**
- SQL formatting is aesthetic preference and irrelevant in interviews — formatting legibility directly affects whether the interviewer can follow your logic and catch bugs alongside you. Messy, unindented SQL with single-letter aliases is genuinely harder to review and signals habits that create maintenance problems in production codebases.
- A query that produces the correct result for the provided example data is complete — interview example datasets are small and clean by design. Real production data has NULLs, duplicates, timezone edge cases, and empty results. Proactively identifying these edge cases before the interviewer asks demonstrates experience with actual production data quality.

**Interview Answer Skeleton:**
- **What it is:** SQL code quality habits — consistent formatting, meaningful naming, and clean structure — combined with edge case testing methodology that systematically checks for NULL propagation, tie-breaking, empty inputs, and boundary conditions before declaring a solution complete.
- **Why it matters / trade-offs:** Clean, tested SQL demonstrates professional habits and production-readiness. A query that produces the right answer on the happy path but fails silently on NULLs or duplicates is a liability in a production pipeline. The trade-off is the 3–5 minutes spent on edge case review, which consistently surfaces real bugs in interview solutions.
- **Example or context:** After writing a top-3 products by revenue per country query using RANK(): "Let me check ties — if two products have identical revenue they'll both receive rank 2, meaning I might return 4 products for a country with a tie at rank 3. Is that acceptable, or should I use DENSE_RANK() to strictly return the top 3 revenue tiers? Also, products with NULL revenue will be excluded from RANK() entirely — should they be treated as zero or excluded?"

**Free Resources:**
- [SQLZoo Interactive Exercises](https://sqlzoo.net) — hands-on SQL practice with immediate result feedback, useful for testing edge cases and validating query logic on real data
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — covers SQL patterns and includes query formatting conventions and debugging approaches for complex analytical queries

---

## Use STAR Stories for Project Depth

**Status:** ⬜ Not Started

**Definition:** STAR (Situation-Task-Action-Result) is a framework for answering behavioural interview questions with structured, credible depth. For data engineering roles, a strong STAR story describes a concrete pipeline or data problem (Situation), your specific scope of ownership (Task), the exact technical decisions and trade-offs you navigated (Action), and the measurable business or operational outcome (Result).

**Key Mental Model:** STAR is a story structure that gives your experience a beginning, a middle, and an ending with impact. Situation sets the scene, Task establishes your role in it, Action is the technical core where depth lives, and Result is why it mattered to the business. Without a Result, the story has no punch — it's a description of activity without evidence of value delivered.

**How It Works:**
- Situation should be specific and concise — 1–2 sentences. Name the system, the scale, and the pain. "We had a daily customer churn report that was silently returning incorrect numbers due to a deduplication bug in the customer dimension. The model had been wrong for 6 weeks before anyone noticed."
- Task clarifies your specific ownership, not the team's. "I was the on-call engineer that week and took ownership of the full fix — not just the immediate hotfix but the root cause prevention and the monitoring gap that allowed it to go undetected."
- Action is where technical depth lives and distinguishes strong from weak STAR stories. Be specific about the technical decisions: "I identified the bug using dbt model lineage to trace which upstream table was providing duplicates. The fix was a ROW_NUMBER() deduplication CTE in the staging model with a secondary sort on updated_at. I then added a dbt unique test on customer_id and a row count assertion checking within 5% of the previous day's count." Name tools, patterns, and trade-offs you considered.
- Result must be quantified when possible and connected to business impact. "The deduplication fix reduced reported churn rate from 8.3% to 6.1% — the real number. I also added freshness monitoring that would have caught this 5 days in rather than 6 weeks in. The incident became the basis for the team's data quality standards document." Numbers, before/after comparisons, and downstream business effects all strengthen the result.
- Prepare 3–4 strong STAR stories covering different dimensions: a complex technical problem you solved, a production incident you owned and recovered, a cross-team collaboration, and a proactive improvement you initiated. These four cover the most common behavioural interview categories for senior DE roles.

**Common Misconceptions:**
- Behavioural questions test soft skills and are separate from technical depth — at senior levels, behavioural questions are evaluated for technical substance, not just communication style. Interviewers are probing for: did you actually understand the technical problem, did you make principled decisions or just try things until something worked, and did your work produce measurable outcomes? Weak STAR stories at senior level signal shallow ownership.
- Any completed project story is appropriate STAR material — choose stories that demonstrate scale (data volumes that required real engineering decisions), complexity (multiple viable approaches with trade-offs), and quantifiable impact. A story about adding a column to a table is unlikely to demonstrate the depth interviewers are looking for at senior levels.

**Interview Answer Skeleton:**
- **What it is:** A structured narrative framework that packages past technical experience into a credible, specific, impact-driven story — Situation (the problem context), Task (your specific ownership), Action (the exact technical decisions), Result (the measurable outcome) — evaluated as evidence of real engineering capability.
- **Why it matters / trade-offs:** Unstructured answers to "tell me about a time when..." lose the interviewer in irrelevant details and fail to demonstrate the technical depth and ownership that senior roles require. STAR keeps the answer focused and surfaces impact clearly. The trade-off is that good STAR stories require preparation — they must be chosen and practised before the interview.
- **Example or context:** "Our daily report pipeline was silently failing (S). I owned the full fix including root cause and monitoring (T). I traced the bug to a missing filter in a CDC merge operation that was creating duplicate rows, rewrote the incremental logic with proper unique_key merge and a 3-day lookback window, added dbt not_null + unique tests on the primary key, and set up a row count alert comparing daily counts to a 7-day rolling average (A). Data incidents on that pipeline dropped from 4 per month to 0 in the following quarter (R)."

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — covers behavioural interview preparation for data engineering roles, including STAR story frameworks and example stories with technical depth
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference with interview preparation guides covering both technical and behavioural question strategies for data engineering roles

---

## Show Ownership, Impact, and Debugging Skill

**Status:** ⬜ Not Started

**Definition:** Ownership in data engineering means taking end-to-end accountability for a system — from initial design through deployment, monitoring, on-call response, and continuous improvement — without needing to be assigned each step. Impact means connecting technical work to business outcomes with specific metrics. Debugging skill is the ability to systematically diagnose production failures using logs, metrics, and code inspection rather than guessing.

**Key Mental Model:** Ownership is being the landlord, not the tenant — you don't just live in the system, you're responsible for its plumbing, its structure, and what happens when something breaks at 3am. Debugging skill means walking through your diagnostic steps methodically, not just producing the fix.

**How It Works:**
- Demonstrated ownership signals in interviews: "I added monitoring after deploying it" (not just delivered the feature), "I wrote the runbook when I handed it off" (thought about operational continuity), "I stayed on the incident until data was backfilled and validated" (saw the problem through to full resolution, not just the code fix). Each of these signals that you treat systems as responsibilities, not deliverables.
- Impact quantification framework: connect your technical action to a metric change. "Reduced pipeline runtime from 4 hours to 35 minutes" (2× faster) is good. "Reduced pipeline runtime from 4 hours to 35 minutes, which moved the dashboard SLA from 10am to 6:30am and eliminated 3 weekly escalation tickets from the CMO's team" is excellent — it connects the technical improvement to a business experience change and an operational improvement.
- Systematic debugging demonstrates the same kind of thinking production engineers use under pressure. Name the diagnostic steps: "I checked the Airflow logs first to confirm the task failed vs succeeded with empty output. Then I looked at the row count in the destination table versus yesterday's count. Then I examined the source table in the Snowflake query history to see if the upstream load had run." This step-by-step narration is more impressive than just naming the root cause.
- Proactive improvement after incidents signals senior ownership: "After fixing the bug, I added a check that would have caught it 5 days earlier. I also documented the incident as a runbook entry so the next on-call engineer has a faster path to diagnosis." This type of follow-through distinguishes engineers who build sustainable systems from those who fix problems and move on.
- Interviewers specifically listen for agency markers in ownership stories: "I decided to...", "I proposed that...", "I took the on-call rotation for that system even though it wasn't assigned to me" — versus passive language like "we were asked to...", "the team decided...", "I was assigned to fix it." Active voice and specific decision ownership demonstrate the accountability interviewers want.

**Common Misconceptions:**
- Ownership is only relevant when discussing leadership or senior roles — ownership behaviours (adding monitoring unprompted, writing runbooks, seeing incidents through to completion) are valued signals at every level. A mid-level engineer who demonstrates strong ownership typically advances faster than a more technically skilled one who doesn't.
- Impact means only revenue metrics — impact includes SLA improvement (shifted from 10am to 6:30am), cost reduction (cut warehouse spend by 40%), operational improvement (reduced on-call pages from 8/week to 1/week), quality improvement (reduced data incidents from 4/month to 0), and trust built over time ("stakeholders stopped double-checking our numbers"). All of these are legitimate, credible impact statements.

**Interview Answer Skeleton:**
- **What it is:** The three qualities that distinguish engineers who build and maintain trusted systems from those who complete assigned tasks — ownership (end-to-end accountability including monitoring and incidents), impact (technical work connected to quantified business outcomes), and debugging skill (systematic diagnosis narrated as a reproducible process).
- **Why it matters / trade-offs:** Senior engineers need teammates who can be given a system to own, not just tasks to complete. Demonstrating ownership, impact, and debugging skill in interviews signals that you are the former. There is no meaningful trade-off to demonstrating these qualities — they are universally valued at all data engineering levels.
- **Example or context:** "At 3am I was paged by a freshness alert on our revenue pipeline. I checked Airflow first — the task had succeeded, so it wasn't a task failure. Then I checked the destination table row count — 0 rows for the partition, versus 2M rows the previous day. I traced it to a date filter bug using execution_date instead of the correct data_interval_start after an Airflow version upgrade. I hotfixed the filter, triggered a backfill for the affected 3 days, validated row counts matched expectations, and resolved the page. Then I added a row count assertion to the dbt model to catch this class of bug earlier."

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — covers ownership signals, impact framing, and debugging frameworks for senior data engineering interviews and day-to-day practice
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference with guides on demonstrating ownership, quantifying impact, and systematic debugging in data engineering contexts

---
