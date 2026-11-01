# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution

### Choice of points and decision rules

My agent evaluates points 4, 7 and 11. These questions suit Fisher's approach of learning about a business from people who deal with it. They also give the model a clear task: interpret relevant testimony and explain whether it supports a particular investment criterion. I chose a small, predictable workflow because each point has an identified source in the assignment, and unnecessary interviews would add little evidence.

**Point 4 — Sales organization.** Fisher asks, “Does the company have an above-average sales organization?” I chose this point because a customer's experience can reveal whether sales staff contribute anything beyond taking orders. The agent marks PASS when the interview describes knowledgeable, helpful and effective sales support that adds value; it marks FAIL when service is weak, limited to distributing prices, or regarded as inferior. The evidence comes from `interview(role="customer", topic="sales_team")`.

**Point 7 — Labour and personnel relations.** I included this point because employees experience working conditions directly, while management may describe the same workplace more favourably. PASS requires evidence of good conditions, fair treatment and constructive employee relations; poor treatment, unreasonable or unsafe conditions, or serious dissatisfaction lead to FAIL. The agent asks a `current_employee` about `working_conditions` and keeps this assessment separate from relationships among executives.

**Point 11 — Competitive advantage.** This point asks whether the business has distinctive strengths relative to competitors. I chose it because a rival's account can help distinguish an advantage from an ordinary product feature. The rule is PASS when the testimony identifies a meaningful advantage that competitors cannot readily copy or customers cannot readily replace, and FAIL when offerings are interchangeable or easily matched. The source is `interview(role="competitor", topic="competitive_edge")`.

These are general decision rules, not mappings from company names to answers. One knowledgeable respondent per topic is sufficient under the assignment's stated assumption that informed respondents agree. The agent therefore uses three interviews per company, below the limit of ten. It does not need the document or SQL tools for these selected points, and it never reads company data directly.

### Agent design

```mermaid
flowchart TD
    A[Company and its supplied LangChain tools] --> B[Fixed interview plan]
    B --> C[Point 4: customer / sales_team]
    B --> D[Point 7: current_employee / working_conditions]
    B --> E[Point 11: competitor / competitive_edge]
    C --> F[Collect three interview answers]
    D --> F
    E --> F
    F --> G[One DeepSeek call: apply rules and return JSON]
    G --> H{Validate all three verdicts and exact quotations}
    H -->|Valid| I[Return points and evidence-based summary]
    H -->|Invalid or API failure| J{Attempts remaining?}
    J -->|Yes| G
    J -->|No| K[Return missing verdicts with an error summary]
    I --> L[Provided runner writes results.csv and scores answers]
    K --> L
```

`build_agent()` creates a shared LangChain `ChatDeepSeek` model using `deepseek-v4-flash`, temperature zero and JSON output. For each company, the program routes the three questions to the appropriate interview tool, then sends the collected answers and fixed rules to the model in one request. This is a programmed research workflow rather than an open-ended agent that chooses its own tools. The prompt asks the model to check each decision against its evidence and treats interview text as data, not instructions. Python checks that all three points are present, each verdict is valid, and every quotation occurs in the corresponding interview. Up to four companies are processed concurrently. A failed model request or invalid response can be retried twice using the same evidence, so retries do not spend more interviews. If analysis still fails, that company receives missing verdicts and an error summary, allowing the other companies to finish. The original tools, runner and scoring code are unchanged.

### Public test results

The final implementation was tested three times on all 20 public companies. The third run used a fresh local clone of the committed project with a newly created virtual environment and dependencies installed from `requirements.txt`.

| Run | Correct point verdicts | Balanced point score | Elapsed time |
| --- | --- | --- | --- |
| 1 | 60 / 60 | 100.0% | 16.2 seconds |
| 2 | 60 / 60 | 100.0% | 11.1 seconds |
| 3 — fresh clone | 60 / 60 | 100.0% | 16.9 seconds |

All three runs generated `results.csv` and used exactly three interviews per company. There were no companies whose final verdicts disagreed with the public answers. Across the selected points, each run correctly classified all 42 PASS cases and all 18 FAIL cases, giving a balanced score of `(42/42 + 18/18) / 2 = 100%`. The mean balanced score over the three runs was also 100%.

An earlier development run scored zero because the response parser required an outer `points` field that the model sometimes omitted. The model could return the requested point objects directly, but the parser rejected that equivalent structure. I changed the validator to accept either form while retaining the checks on point numbers, verdicts and quotations. This was an interface failure, and it showed why inspecting the returned data matters before changing the investment rules. Four offline tests now cover this response format, rejection of quotations from the wrong point, reuse of interviews during retries, and isolation of company failures during an API outage. All four pass.

The public results are encouraging, although they do not establish accuracy on the private companies. The design relies on the assignment's unusually clear interview evidence and agreement among knowledgeable sources. In a real investigation, one person's account would require corroboration. An exact quotation check also proves only that the words appear in the source; it cannot establish that the model interpreted them correctly. More ambiguous or conflicting testimony would need a different research process.

### Task 2 reflection

The English reflection is provided in [Task2_Reflection.docx](Task2_Reflection.docx).
