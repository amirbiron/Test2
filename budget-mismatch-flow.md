# זרימת "מעט פניות" → אבחון תקציב → "הגדל תקציב" (Problem 4 / `budget_mismatch`)

> ## ⚠️ עודכן ב-Session 7.6.5 — קרא קודם
> חלקים מהמסמך שלהלן מתארים את המנגנון **לפני** 7.6.5. הדלתא (מקור-אמת: `/root/.claude/plans/goofy-plotting-hopcroft.md`):
> - **אין יותר יעד-לידים** (`monthly_lead_goal` הוסר מהשאלון/מודלים). הצ'יפ "מעט פניות" **בלתי-מותנה**.
> - **אין ענף feasible/infeasible ואין "עצירה חכמה"** (הפניה-לבעיה-1). האבחון מחשב **צפי בלבד**
>   (`budget_feasibility_service.expected_leads(daily, market_cpl)`) ומציג מסר-הצעה יחיד (`offer.txt`).
> - **המשתמש מקליד את התקציב החדש** (turn חדש: `raise-intent` שואל → `preview` → `approve`). ולידציה שרתית:
>   `> הנוכחי` ו-`≤ הנוכחי+100₪` (`_assess_budget_increase(target_daily)`); `agorot=round(target×100)` (לא ceil).
>   endpoint חדש `POST .../budget/raise-intent`; `preview`+`approve` מקבלים גוף `{target_daily}` (`BudgetRaiseRequest`).
>   type תשובה חדש `budget_input` (שאלה-חוזרת: too-low/too-high/unparseable).
> - **דיסקליימר** ("דורש למידה חוזרת, לא מבטיח כמות פניות") על **מסך האישור** (preview) **וגם** על הודעת ההצלחה.
> - **התראה חדשה `budget_increased`** (מייל חי + וואטסאפ רדום עד WABA) אחרי approve מוצלח — best-effort,
>   `anchor=budget:{campaign_id}:{agorot}` (idempotent), נספרת במכסת 30/חודש (migration 0111 + RPC).
> - **audit `budget_agreements`**: `monthly_lead_goal_at_time=None`, `required_budget=None`; `new_budget`=הסכום שהוקלד.
> - **נשמר ללא שינוי:** reserve-first+rollback, idempotency ערך-מוחלט, sync-quiz, לולאת reactive, force-creative/decline.

> מסמך תפעולי מפורט מקצה-לקצה של הבעיה הרביעית בפרוטוקול-הסוכן: הלקוח מתלונן על **מעט פניות**,
> הסוכן מאבחן **מתמטית** אם התקציב מספיק ליעד, ואם לא — מציע להעלות תקציב. המסמך ממפה עיצוב↔קוד:
> כל שלב עם הקובץ, הפונקציה, ה-endpoint, הנוסחה, ומעברי-המצב בפועל.
>
> **מקורות-אמת:** `docs/ROADMAP.md` (Session 7.6.5, חלקים 1–11) = העיצוב המוצרי; `docs/problem4-force-creative-plan.md`
> = ענף הסירוב (force-creative); הקוד עצמו = המימוש. **ענף "הגדל תקציב" (approve) הוא המוקד כאן.**
> **הבחנה חשובה:** ה-"value-turn" שייך ל**בעיה 2** (`low_quality_leads`), **לא** לבעיה זו — ראה §12.

---

## 0. TL;DR — התמונה בשורה אחת

```
צ'יפ "מעט פניות" (מותנה ביעד)
   → diagnose_problem_4  (אבחון מתמטי דטרמיניסטי — "חוק 7": הקוד מחשב, ה-LLM רק מנסח)
        expected_leads  = ⌊(daily_budget × 30) / market_cpl⌋
        required_budget = (monthly_lead_goal × market_cpl) / 30
        feasible        = (daily_budget × 30) / market_cpl ≥ monthly_lead_goal
   → 3 תוצאות:
        • None (חסר benchmark/תקציב/יעד)  → unavailable
        • feasible (התקציב מספיק)         → "הבעיה כנראה לא בתקציב" → הפניה לבעיה 1 (high_cpl)
        • infeasible (התקציב נמוך)         → מציג את הפער + 2 צ'יפים:
              [הגדל תקציב] / [השאר את התקציב הקיים]
   → "הגדל תקציב":  preview (אישור מפורש) → approve
                     approve = record(budget_increase_approved) לפני-Meta  →  update_ad_set(daily_budget=agorot)  →  sync quiz
   → "השאר את התקציב הקיים":  decline (record budget_increase_declined) → אופציונלי force_creative (ניסיון קריאייטיב חד-פעמי)
   → לולאה reactive:  בכל תלונה חוזרת, שולפים את ההסכמה האחרונה מ-budget_agreements ומאבחנים מחדש (בלי cron)
```

**עקרונות-ליבה:**
1. **חוק 7 — האבחון דטרמיניסטי.** ה-LLM **לא** מעריך היתכנות; הוא מקבל `feasible/expected_leads/required_budget` מוכנים ומנסח את המספרים בלבד.
2. **ההחלה סינכרונית.** אין worker/cron למסלול ה-approve — הכל בתוך ה-HTTP request.
3. **התיעוד הוא הזיכרון.** טבלת `budget_agreements` היא audit-log append-only (אין עמודת `status`); הלולאה ה-reactive נשענת עליה, לא על cron.
4. **reserve-first.** רושמים את ההסכמה ל-DB **לפני** הקריאה ל-Meta; אם Meta נכשל — rollback לרשומה (אסור להשאיר "אושר" לפעולה שלא קרתה).

---

## 1. תנאי הכניסה — יעד לידים (`monthly_lead_goal`)

השדה הזה הוא **השער** לכל הבעיה. בלעדיו — הצ'יפ "מעט פניות" לא מופיע, והבעיה כולה לא נגישה.

| היכן | פרט |
|------|-----|
| **עמודה** | `campaigns.monthly_lead_goal integer` (nullable) — migration `0102`. |
| **נאסף** | בשאלון-יצירה, שלב התקציב `#wiz-budget`, קלט אופציונלי `#wiz-monthly-goal` (`app/web/index.html:1901-1913`). תווית: "🎯 יעד לידים לחודש (אופציונלי)". |
| **נקרא ל-quiz** | `_readMonthlyLeadGoal()` (`index.html:5112-5117`) → נארז כ-`monthly_lead_goal` ב-`_buildQuizBody()` (`index.html:5145`). ריק/0/לא-מספרי → `null`. |
| **נחשף בשיחה** | `ConversationResponse.monthly_lead_goal` (`app/models/agent.py:41-42`) → `CHAT.monthlyLeadGoal` ב-`_enterConversation` (`index.html:2581`). |

> **מסקנה:** כשנכנסים לבעיה — היעד **תמיד** קיים. אין null-handling בתוך הזרימה עצמה (ה-null נחסם כבר בשלב הצגת הצ'יפ).

---

## 2. הצ'יפ המותנה "מעט פניות" + הניתוב

**רינדור התפריט** — `_renderProblemMenu()` (`index.html:2588-2596`):
- צ'יפים בסיסיים תמיד: `['עלות לפנייה גבוהה', 'פניות לא איכותיות']`.
- הצ'יפ `'מעט פניות'` **נדחף רק אם** `CHAT.monthlyLeadGoal != null` (`index.html:2593`).

**מיפוי label→problem_type** — `_PROBLEM_PT` (`index.html:2540`):
```js
const _PROBLEM_PT = {
  'עלות לפנייה גבוהה':'high_cpl',
  'פניות לא איכותיות':'low_quality_leads',
  'מעט פניות':'few_leads',
};
```

**לחיצה** → `_chatDispatch` (`index.html:2621-2622`) מזהה `few_leads` → `CHAT.problemType='few_leads'` → `runDiagnose()` (`index.html:2626-2666`).

**קריאת ה-API** — `runDiagnose` → `window.api.diagnose(CHAT.conv, 'few_leads', {})`:
- **api.js** — `diagnose(conversationId, problemType, opts)` (`app/web/api.js:231-241`).
- **Endpoint** — `POST /me/agent/conversations/{conversationId}/diagnose`, גוף `{problem_type:'few_leads'}`. `few_leads` מתעלם מ-`subcategory`/`value`/`override` — קריאה חד-שלבית.

**ניתוב בשרת** — `app/routers/agent.py:286-287`:
```python
elif body.problem_type == "few_leads":
    decision = await agent_orchestrator.diagnose_problem_4(user.id, conversation_id)
```
(מודל הבקשה: `problem_type: Literal["high_cpl","low_quality_leads","meta_rejection","few_leads"]`, `app/models/agent.py:114`. מראה סנדבוקס: `app/services/sandbox_service.py:296-297`.)

---

## 3. האבחון המתמטי הדטרמיניסטי (חוק 7)

**פונקציה:** `diagnose_problem_4(user_id, conversation_id) -> AgentDecision` — `app/services/agent_orchestrator.py:634-690`.
Docstring: "אבחון מתמטי דטרמיניסטי (חוק 7): האם (תקציב×30)/market_cpl ≥ יעד. אין benchmark-stop, אין מנוע-שלבים, אין סדרה — רק חישוב + ניסוח."

### 3.1 הקלטים (כולם נאספים בתוך הפונקציה, `:641-652`)

| משתנה | מקור | הערה |
|-------|------|------|
| `campaign_id` | `agent_service.get_owned_campaign_id` (`:641`) | 404 אם לא-בעלים |
| `last_agreement` | `budget_agreement_service.fetch_last(campaign_id)` (`:643`) | ללולאה ה-reactive (§7) |
| `industry` | `campaigns.industry` (`:646`) | לקביעת `market_cpl` |
| `goal` | `campaigns.monthly_lead_goal` (`:647`) | היעד |
| `market_cpl` | `benchmark_service.market_cpl_for_industry(industry)` (`:649`) | ראה §3.3 |
| `budget` | `_budget_for_campaign` → `quiz.answers.budget` (`:650`, `_budget_for_campaign` `:552-559`) | **מהשאלון, לא מ-Meta** |
| `daily_budget` | `_normalize_daily_budget(budget)` (`:651`) | ראה §3.2 |
| `feasibility` | `budget_feasibility_service.assess(daily_budget, market_cpl, goal)` (`:652`) | ראה §3.4 |

> **קריטי:** ה-**CPL בפועל של הקמפיין אינו קלט** לאבחון. בעיה 4 מניחה שהקמפיין **בריא** (עלות-ליד תקינה) ומשתמשת ב-`market_cpl` של הענף. ה-CPL החי נשלף **רק** בענף ה-reactive עבור רשומת `creative_against_advice` קודמת (`:668-675`).

### 3.2 נרמול התקציב היומי — `_normalize_daily_budget` (`:562-575`)

`quiz.answers.budget` הוא dict `{mode, amount, days}`:
- `mode == "daily"` → `float(amount)`.
- `mode == "lifetime"` → `float(amount) / float(days)` אם `days > 0`, אחרת `None`.
- `amount` לא מספר-ממשי (או bool) → `None`.

### 3.3 עלות-ליד השוק — `benchmark_service.market_cpl_for_industry` (`app/services/benchmark_service.py:101-112`)

```python
return (row["amazing_max"] + row["expensive_above"]) / 2
```
`benchmarks.json` מחזיק שני ספים לענף בלבד (`amazing_max`, `expensive_above`) — אין ערך "ממוצע" יחיד — לכן `market_cpl` = **אמצע הטווח האמצעי**. ענף לא-מוכר/`None` → `None` (→ degradation).

| industry | amazing_max | expensive_above | `market_cpl` = (a+b)/2 |
|----------|-------------|-----------------|------------------------|
| beauty | 25 | 50 | **37.5** |
| courses | 50 | 100 | **75** |
| fitness | 30 | 60 | **45** |
| local_services | 40 | 85 | **62.5** |
| professional | 90 | 160 | **125** |
| realestate | 100 | 200 | **150** |
| b2b_hr | 120 | 250 | **185** |

(7 מפתחות סגורים — `app/models/benchmark.py:15-25`.)

### 3.4 החישוב — `budget_feasibility_service.assess` (`app/services/budget_feasibility_service.py:43-62`)

קבוע: `_DAYS_PER_MONTH = 30`.
שער-תקינות `_finite_positive` (`:33-40`): כל קלט חייב להיות `int`/`float` (לא `bool`), `math.isfinite`, ו-`> 0`. כשל של **אחד** → `assess` מחזירה `None` (degradation, לא קריסה).

```python
expected        = (daily_budget * 30) / market_cpl
required        = (monthly_lead_goal * market_cpl) / 30
return BudgetFeasibility(
    expected_leads = int(expected),          # floor — "לא מבטיחים יותר ממה שיש"
    required_budget = required,              # התקציב היומי הדרוש ליעד
    feasible = expected >= monthly_lead_goal, # מושווה על ה-float (לא ה-int)
)
```

- **`expected_leads`** = ⌊(daily × 30) / market_cpl⌋ — כמה לידים התקציב הנוכחי קונה בחודש.
- **`required_budget`** = (goal × market_cpl) / 30 — התקציב היומי שיגיע בדיוק ליעד.
- **`feasible`** ⇔ `daily_budget ≥ required_budget`. **"mismatch"** (infeasible) ⇔ `expected < goal`.
- מבנה התוצאה: `BudgetFeasibility(expected_leads:int, required_budget:float, feasible:bool)` — dataclass frozen (`:24-30`).

**דוגמה (beauty):** market_cpl=37.5, goal=40, daily=₪30 → expected=⌊900/37.5⌋=⌊24⌋=24 < 40 → **infeasible**; required=40×37.5/30=**₪50/יום**.

### 3.5 שלוש התוצאות + הצ'יפים (`:654-690`)

| תנאי | `type` | הודעה | צ'יפים |
|------|--------|-------|--------|
| `feasibility is None` (`:654-657`) | `unavailable` | `_BUDGET_NO_DATA_MSG` ("אין לי כרגע מספיק נתונים… נסה שוב מאוחר יותר", `:540-543`) — **בלי LLM** | `[חזרה לתפריט]` |
| **feasible** (`:682-683`) | `budget_options` | "התקציב אמור להספיק → הבעיה כנראה לא בתקציב, בוא נבדוק את עלות-הליד" | `[עלות לפנייה גבוהה, חזרה לתפריט]` — הפניה לבעיה 1 |
| **infeasible + lifetime** (`:684-686`) | `budget_options` | הודעת-פער + `_BUDGET_LIFETIME_HINT` (`:546-549`) | `[השאר את התקציב הקיים, חזרה לתפריט]` — **בלי "הגדל"** (approve לא מעלה lifetime) |
| **infeasible + daily** (`:687-688`) | `budget_options` | הודעת-הפער המתמטית | `[הגדל תקציב, השאר את התקציב הקיים]` |

**קבועי צ'יפים** (`agent_orchestrator.py:529-534`, `id, label`):
- `_CHIP_RAISE_BUDGET = ("raise_budget", "הגדל תקציב")` → `POST .../budget/preview`
- `_CHIP_KEEP_BUDGET = ("keep_budget", "השאר את התקציב הקיים")` → `POST .../budget/decline`
- `_CHIP_REFER_HIGH_CPL = ("high_cpl", "עלות לפנייה גבוהה")` → חוזר ל-`diagnose_problem_1`
- `_CHIP_BACK = ("back_to_menu", "חזרה לתפריט")`

**מבנה ההחזרה** — `AgentDecision` (`agent_orchestrator.py:92-106`): לבעיה 4 תמיד `type ∈ {"budget_options","unavailable"}`, ו-`session_id/proposal_variation/proposal_step = None`. ה-router אורז ל-`DiagnoseResponse` (`app/models/agent.py:133-149`, `Chip={id,label}`).

---

## 4. ניסוח ה-LLM (`prompts_service`) — feasible / infeasible

הקוד חישב את המספרים; ה-LLM רק מנסח אותם למשפט אנושי.

**בונה:** `_phrase_budget_mismatch(...)` (`agent_orchestrator.py:584-631`) → `prompts_service.build_agent_prompt(mode="chip", issue_type="budget_mismatch", state_key, system_state=...)` (`prompts_service.py:186-229`).
- `state_key = "feasible" if feasibility.feasible else "infeasible"` (`:659`).
- **קבצי הפרומפט:** `app/prompts/agent/modules/budget_mismatch/feasible.txt` ו-`.../infeasible.txt`.

**מלכודת חשובה:** קבצי המודול **אינם מכילים placeholders מסוג `{}`** — שמות המשתנים מופיעים כטקסט חופשי בהנחיה העברית. הערכים המספריים מגיעים ל-LLM דרך בלוק נפרד `[SYSTEM_PAYLOAD]` (מרונדר ע"י `_render_payload`, `:178-184`, שמשמיט כל ערך `None`). ההנחיה מפרטת איזה שם מתאים לאיזו שורת-payload.

**המשתנים המוזרקים** (`agent_orchestrator.py:601-610`, דרך `agent_service._fmt_num`):

| מפתח ב-`[SYSTEM_PAYLOAD]` | ערך | infeasible | feasible |
|---------------------------|-----|:----------:|:--------:|
| `service_name` | שם השירות | ✓ | ✓ |
| `market_average_cost_per_lead` | `market_cpl` | ✓ | ✓ |
| `current_daily_budget` | `daily_budget` | ✓ | ✓ |
| `monthly_lead_goal` | היעד | ✓ | ✓ |
| `required_daily_budget_for_goal` | `feasibility.required_budget` | ✓ | (מוזרק, לא מוזכר) |
| `leads_the_current_budget_allows_per_month` | `feasibility.expected_leads` | ✓ | ✓ |

- `_fmt_num` (`agent_service.py:195`): `None → "לא ידוע"`; שלם ללא עשרוני; אחרת ספרה עשרונית אחת.
- אין משתנה `gap` מפורש — הפער משתמע: המודל משווה `leads_..._per_month` מול `monthly_lead_goal` (וב-infeasible גם `required_..._for_goal` מול `current_daily_budget`), ומנסח "פחות מהיעד". (פער מספרי, אם צריך: `goal − expected_leads` לידים, או `required_budget − daily_budget` ש"ח.)
- פרמטרי LLM: trigger `_BUDGET_TRIGGER` (`:526`), `settings.openai_agent_model`, `temperature=0.5`, `max_tokens=600`, `json_mode=False`. Rate-limit → `AgentTransientError`(503); אחר → `AgentPermanentError`(500).

**נוסח היעד (ROADMAP חלק 4, למה ה-LLM מכוון):**
> "בוא נסתכל על המספרים יחד. עלות ליד ממוצעת בשוק לענף שלך היא ₪[X]. כדי להגיע ליעד שלך של [Z] לידים בחודש, נדרש תקציב יומי של כ-₪[W]. התקציב הנוכחי שלך, ₪[Y] ביום, מאפשר מתמטית כ-[expected_leads] לידים — פחות מהיעד. זה לא עניין של איכות הקמפיין; הוא עובד תקין. זה פער מתמטי בין התקציב ליעד."

---

## 5. מסלול א' — "הגדל תקציב" (המוקד: preview → approve)

זהו **double-confirm**: הצ'יפ פותח תצוגה-מקדימה, ורק אישור מפורש שני מבצע את החיוב.

### 5.1 שלב 1 — Preview (אישור מפורש)

```
צ'יפ 'הגדל תקציב'
  → _chatDispatch (index.html:2615):  runBudgetAction('preview')
  → runBudgetAction (index.html:2670-2682):  window.api.budgetAction(CHAT.conv, 'preview')
  → api.js budgetAction (api.js:245-250):  POST .../budget/preview
  → router budget_preview (agent.py:319-329):  agent_orchestrator.preview_budget_increase (:758-775)
```
- `preview_budget_increase` קורא ל-`_assess_budget_increase` (§5.2.1) ומחזיר **רק הודעה** — **בלי record, בלי Meta**.
- הודעת ה-preview (`:768-771`): "אני מעלה את התקציב היומי מ-₪[Y] ל-₪[W]. שים לב: התשלום למטא ייגבה ישירות מהאשראי שלך לפי התקציב החדש. מאשר?"
- צ'יפים: `[_CHIP_CONFIRM_RAISE=("confirm_raise","מאשר"), _CHIP_CANCEL_RAISE=("cancel_raise","ביטול")]` (`:533-534`).
- **דגל אנטי-זיוף:** אחרי ה-preview, `runBudgetAction` מציב `CHAT.budgetConfirm = chips.some(c=>c.id==='confirm_raise')` (`index.html:2679`). הדגל עולה **רק** אם השרת החזיר בפועל את צ'יפ `confirm_raise`.

### 5.2 שלב 2 — Approve (ביצוע החיוב)

```
צ'יפ 'מאשר'
  → _chatDispatch (index.html:2616):  if(budgetConfirmPending) runBudgetAction('approve') else _freeText
       budgetConfirmPending נצרך אטומית בראש ה-dispatch (index.html:2602) — כל ניתוב אחר מאפס אותו
  → POST .../budget/approve  → router budget_approve (agent.py:341-352)
  → agent_orchestrator.approve_budget (:778-827)
```

#### 5.2.1 החישוב-מחדש המשותף — `_assess_budget_increase` (`:720-755`, server-authoritative)

- טוען קמפיין, קורא `industry`, `goal`, ו-**`ad_set_id = camp["meta_ad_set_id"]`** (`:729`).
- מחשב מחדש `market_cpl`, `daily_budget`, `feasibility` (אותה מתמטיקה כמו האבחון — השרת לא סומך על מספרים מהלקוח).
- **שערים** (מוחזרים כ-`(False, AgentDecision)`):
  - `feasibility is None` או אין `ad_set_id` → degradation (`_BUDGET_NO_DATA_MSG`).
  - `feasibility.feasible` → "כבר מספיק" → הפניה לבעיה 1.
  - `budget.mode == "lifetime"` → הודעת-עדכון-ידני (approve לא נוגע ב-lifetime).
- **הצלחה:** `agorot = math.ceil(feasibility.required_budget * 100 - 1e-9)` (`:750`) — עיגול **כלפי מעלה** (round רגיל היה עלול להשאיר infeasible). מחזיר dict עם `new_daily = agorot/100`, `ad_set_id`, וכו'.

#### 5.2.2 רצף ה-approve (`:778-827`)

1. **חישוב-מחדש** — `ok, result = _assess_budget_increase(...)`; אם `not ok` → מחזיר את ה-decision של השער.
2. **תיעוד reserve-first (לפני Meta)** (`:793-798`):
   ```python
   agreement_id = await _guard(budget_agreement_service.record(
       campaign_id=campaign_id, user_id=user_id,
       agreement_type="budget_increase_approved",
       market_cpl=result["market_cpl"], daily_budget_at_time=result["daily_budget"],  # התקציב הישן
       monthly_lead_goal_at_time=result["goal"],
       required_budget=new_daily, expected_leads=feasibility.expected_leads,
       new_budget=new_daily,        # התקציב החדש
   ))                               # cpl_before נשאר NULL (רלוונטי רק ל-creative_against_advice)
   ```
3. **החלה על Meta** (`:800-802`):
   ```python
   client = await meta_client_factory.get_push_client(user_id, campaign_id)
   await client.update_ad_set(ad_set_id, {"daily_budget": agorot})
   ```
4. **כשל → rollback** (`:803-817`, ראה §5.3).
5. **הצלחה → סנכרון השאלון** (`:823`): `quiz_service.update_budget_to_daily(campaign_id, new_daily)` — **fail-loud** (`_guard`; transient→503/unexpected→500). שומר על `quiz.answers.budget` (ולולאת ה-reactive) עקבי עם Meta.
6. **אישור** (`:825-827`): `_format_budget_approved_message(new_daily)` → "מצוין — עדכנתי את התקציב היומי ל-₪[W]. התשלום למטא ייגבה מעתה לפי התקציב החדש, ואתה אמור לראות עלייה בכמות הפניות בהתאם." · `chips=[חזרה לתפריט]`.

צ'יפ **'ביטול'** → `_renderProblemMenu()` (`index.html:2617`).

#### 5.2.3 ההחלה על Meta — יחידות ומקור ה-ad_set

- `update_ad_set(token, ad_set_id, params) -> None` (`app/integrations/meta.py:664-672`): `AdSet(fbid=ad_set_id).api_update(params)`. **התקציב היומי הוא ברמת ה-ad_set, לא ה-campaign.**
- **יחידות: אגורות (×100).** `agorot = ceil(required_budget × 100)`; ה-param הוא `{"daily_budget": agorot}`. (צד היצירה עקבי: `campaign_push_service.py:163-170`, `int(round(amount*100))`.)
- **מקור `ad_set_id`:** `campaigns.meta_ad_set_id` (ב-`_PUSH_COLUMNS`, `campaign_service.py:33`), נקרא ב-`_assess_budget_increase:729`.

#### 5.2.4 ה-seam של MetaClient + sandbox

| שכבה | קובץ | התנהגות |
|------|------|---------|
| Protocol | `app/services/meta_client.py:60` | `async def update_ad_set(self, ad_set_id, params) -> None` |
| Real | `meta_client.py:113-114` | `await meta.update_ad_set(token, ad_set_id, params)` |
| Sandbox | `sandbox_meta_client.py:91-92` | **no-op מלוגג** (אין קריאת Meta) |
| Factory | `meta_client_factory.py:39-51` | `get_push_client`: אם `fetch_is_sandbox(campaign_id)` → `SandboxMetaClient` (**נבדק לפני שליפת token** — סנדבוקס לא צריך חיבור Meta); אחרת token → `RealMetaClient`; `token is None` → `MetaClientNotConnectedError` |

> קמפיין סנדבוקס: **רושם** את ההסכמה ו**מסנכרן** את השאלון, אבל עדכון ה-Meta הוא no-op.

### 5.3 טיפול בכשלים + מעברי-מצב (אין עמודת `status`!)

**אין `pending → applied/failed`.** אין עמודת `status` בטבלה. ה"מכונת-מצבים" האפקטיבית היא **קיום-שורה**:

| שלב Meta נכשל | פעולה | תוצאה |
|---------------|--------|-------|
| `MetaClientNotConnectedError` | `_safe_rollback(agreement_id)` (מחיקת השורה) | `_BUDGET_NOT_CONNECTED_MSG` · `type=unavailable` (`:803-806`) |
| `MetaTransientError` | `_safe_rollback` | `raise AgentTransientError()` → **HTTP 503**, הלקוח מנסה שוב (`:807-809`) |
| `MetaApiError` (permanent) | `_safe_rollback` + `capture_exception(where="approve_budget_meta")` | `_BUDGET_UPDATE_FAILED_MSG` · `type=unavailable` (`:810-817`) |
| **הצלחה** | השורה נשארת → sync quiz | הודעת-אישור (`:819-827`) |

- `_safe_rollback` (`:237-243`) → `budget_agreement_service.delete_by_id` (`:97-105`), **best-effort** (כשל מחיקה → warning בלבד, לא מסתיר את שגיאת ה-Meta המקורית). **כלל 12ד:** אסור להשאיר רשומת "אושר" לפעולה שלא בוצעה.
- **מצבים סופיים:** *שורה קיימת* = אושר והוחל (השאלון מסונכרן); *אין שורה* = לא הוחל (מעולם לא נכתב, או נכתב-ונמחק). "נכשל" מיוצג ב**היעדר-שורה** + שגיאת-HTTP, לא ב-status מאוחסן.
- **מיפוי שגיאות:** `_guard` (`:215-234`): sub-error חולף → `AgentUnavailableError`(503); לא-מזוהה → `UnexpectedAgentError`(500).

### 5.4 אידמפוטנטיות — reserve-first, לא CAS

**המנגנון = reserve-first INSERT + rollback-on-failure** (מתועד ב-`budget_agreement_service.py:1-8`). **אין** CAS-על-status, **אין** search-before-write, **אין** UNIQUE/dedup.
- **איך double-apply נמנע בפועל:** הכתיבה ל-Meta היא **ערך מוחלט** (`daily_budget = agorot`, לא תוספת), ו-`agorot` מחושב מחדש דטרמיניסטית בכל קריאה → retry/double-click מתכנס לאותו מצב ad_set (אידמפוטנטי אצל Meta). `update_budget_to_daily` גם היא כתיבה-מוחלטת אידמפוטנטית.
- **העלות היחידה של retry:** שורת-audit `approved` **נוספת** — "תיעוד עודף, לא נזק" (`agent_orchestrator.py:822`). אין constraint שמונע שתי שורות מ-double-click; הבטיחות מגיעה מ-idempotency של ערך-מוחלט, לא ממפתח.

### 5.5 סטטוס אחרי approve — אין polling

**אין polling בזרימת התקציב.** אחרי ש-`runBudgetAction('approve')` מסתיים, אותו handler מרנדר סינכרונית את הבועה + הצ'יפים (`index.html:2681`). ה"עודכן" הוא `d.message` מהשרת. (כל ה-polling בקובץ — `_pollActionImages`/`_pollActionUntilSettled`/תשלום/job — שייך ליצירת-תמונות/דחיפת-קמפיין/תשלום, לא לתקציב.)
- כשל חולף (503) → `_agentErrMsg(err,true)` בתפיסה של `runBudgetAction` (`index.html:2675`) → "השירות אינו זמין כרגע, נסה שוב בעוד רגע."

---

## 6. מסלול ב' — "השאר את התקציב הקיים" (decline) + force-creative (תמצית)

> פירוט מלא: `docs/problem4-force-creative-plan.md`. כאן תמצית לשלמות.

> **⚠️ עודכן — force-creative עבר ל-propose→approve:** הקופי (3 וריאציות + תמונות eager) מוצג למשתמש לבחינה/
> עריכה/**אישור לפני** push (כמו בעיות 1-2), במקום generate+push מיידי. `force_creative_propose` (orchestrator,
> `push_status=NULL`) → ה-approve דרך ה-approve הגנרי (`execute_approval`, ענף `budget_mismatch` מקביל ל-
> `meta_rejection`) → `budget_creative_push_service` (`window=None` + record `creative_against_advice` **אחרי**
> push מוצלח + סגירת `done`; `push_failed` → `failed`). ה-`record` עבר מ-reserve-first ל-post-push (שומר
> "מתועד ⟺ נדחף"). התמצית שלהלן מתארת את המימוש **המקורי** (fire-and-forget). frontend: chip → `runForceCreativePropose`
> → `openActionView` (view-action). ראה גם ההערה בראש `problem4-force-creative-plan.md`.

```
צ'יפ 'השאר את התקציב הקיים'
  → runBudgetAction('decline') (index.html:2618)  → POST .../budget/decline (agent.py:364-374)
  → decline_budget (:886-908):
       record(budget_increase_declined, ...)   ← תיעוד ההגנה המשפטי (בלי Meta)
       message = _format_budget_declined_message (:831-840, ROADMAP חלק 6.2)
       chips = [נשאיר את המודעות כמו שהם, בכל זאת ננסה קריאייטיב חדש]
```
- **`נשאיר את המודעות כמו שהם`** (`keep_ads`) → **frontend בלבד** → חזרה לתפריט (הסירוב כבר תועד; `index.html:2619`).
- **`בכל זאת ננסה קריאייטיב חדש`** (`force_creative`) → `POST .../budget/force-creative` (`agent.py:386-397`) → `force_creative` (`:917-987`):
  1. `cpl_before` = CPL חי (fallback `market_cpl`) — ההגנה "ה-CPL שלך כבר היה תקין".
  2. **מיני-session** (`open_session("budget_mismatch", {cpl_before})`) — דפוס `rejection_fix`: פותחים סדרה אמיתית וסוגרים מיד ל-`done`, **בלי window/cron/מנוע-שלבים**.
  3. resume-check אידמפוטנטי → מסלול-ראשון: `record(creative_against_advice)` (reserve-first) → `generate_solution` (3 וריאציות, `eager_images=False`).
  4. `optimization_push_service.push(..., step_number=1, window_hours=None)` — `None` → אין `window_ends_at` → ה-cron לא בוחר → אין מחזור-מדידה.
  5. `set_session_status(done)` → הודעה `_FORCE_CREATIVE_DONE_MSG` (`:912-914`).
- **דגל אנטי-זיוף:** `CHAT.creativePending = chips.some(id==='force_creative')` (`index.html:2680`), נצרך אטומית (`index.html:2603`).
- **מנוע-שלבים:** `_STEP_PLANS["budget_mismatch"] = {1: _CREATIVE_REFRESH}` (`optimization_service.py:648-650`) — שלב יחיד (`creative_refresh`, זוויות pain/dream/urgency, unique); שלב 2 = `None` → terminal. קיים **רק** כדי לשרת את force-creative (האבחון הרגיל לא נוגע במנוע-השלבים).

---

## 7. הלולאה ה-reactive — הזיכרון הוא הטבלה (בלי cron)

בכל תלונה חוזרת ("מעט פניות"), `diagnose_problem_4` **לפני** האבחון (`:642-678`):
1. `budget_agreement_service.fetch_last(campaign_id)` (`:643`) — הרשומה האחרונה.
2. אם היא `creative_against_advice` → שולף CPL חי best-effort (`:668-675`).
3. `_format_reactive_reminder(last_agreement, current_cpl)` (`:854-883`) → אם לא-ריק, מקדים ל-`msg`.

**וריאנטים** (לפי `agreement_type`, `_fmt_agreement_date` = DD/MM/YYYY):
- `budget_increase_declined` → "כפי שבחרת ב-[תאריך], התקציב נשאר על ₪[Y]… כמות הלידים מוגבלת לכ-[expected_leads]… רוצה לשקול שוב?"
- `budget_increase_approved` → "ב-[תאריך] העלינו את התקציב ל-₪[W]…"
- `creative_against_advice` → "ב-[תאריך] התעקשת… העלות לליד הייתה ₪[cpl_before] לפני, ועכשיו ₪[cpl_now]. הבעיה לא הייתה בקריאייטיב אלא בפער התקציב."

> **אין cron בשום מקום בלולאה** (מאומת: `ROADMAP.md:14913-14925`, `problem4-force-creative-plan.md:489-505`). התיעוד = הזיכרון; כל חזרה = שליפה + אבחון-מחדש עם הנתונים העדכניים (אולי התקציב/עלות-השוק השתנו).

---

## 8. סכמת ה-DB — `budget_agreements`

**Migration:** `supabase/migrations/0103_budget_agreements.sql:9-46`.

| עמודה | טיפוס | הערות |
|-------|-------|-------|
| `id` | `uuid` | PK, `default gen_random_uuid()` |
| `campaign_id` | `uuid` | `not null references campaigns(id) on delete cascade` |
| `user_id` | `uuid` | `not null references auth.users(id) on delete cascade` |
| `agreement_type` | `text` | `not null CHECK IN ('budget_increase_approved','budget_increase_declined','creative_against_advice')` |
| `market_cpl` | `numeric` | הנתונים שהוצגו בזמן ההסכמה (X) |
| `daily_budget_at_time` | `numeric` | Y — התקציב היומי בזמן ההסכמה |
| `monthly_lead_goal_at_time` | `integer` | Z — היעד |
| `required_budget` | `numeric` | W — התקציב הדרוש |
| `expected_leads` | `integer` | הלידים שהתקציב מאפשר |
| `new_budget` | `numeric` | ל-`approved`: התקציב היומי החדש |
| `cpl_before` | `numeric` | ל-`creative_against_advice`: העלות לפני השינוי |
| `created_at` | `timestamptz` | `not null default now()` |

- **CHECK** יחיד על `agreement_type` (3 הערכים) — צד ה-DB של אותו `_VALID_TYPES` (`frozenset`) ב-service (`budget_agreement_service.py:23-25`).
- **אין עמודת `status`, אין UNIQUE, אין CAS.** ה-PK היחיד הוא `id`.
- **Index:** `idx_budget_agreements_campaign (campaign_id, created_at desc)` — שליפת ה-reactive = האחרונה (tiebreaker `id`).
- **RLS:** `enable row level security`; policy `select using (auth.uid() = user_id)`; `grant select ... to authenticated`. כתיבה דרך service_role בלבד (admin client — עוקף RLS).
- **Migration `0104`** (`0104_budget_mismatch_problem_type.sql`) **לא נוגע** בטבלה זו — הוא מרחיב את ה-CHECK של `optimization_sessions.problem_type` להוסיף `'budget_mismatch'` (למען המיני-session של force-creative).

**Service — `budget_agreement_service.py`:**
- `record(*, campaign_id, user_id, agreement_type, market_cpl=None, daily_budget_at_time=None, monthly_lead_goal_at_time=None, required_budget=None, expected_leads=None, new_budget=None, cpl_before=None) -> str` (`:56-94`) — INSERT דרך admin client, מחזיר את ה-`id`.
- `delete_by_id(agreement_id) -> None` (`:97-105`) — rollback.
- `fetch_last(campaign_id) -> dict | None` (`:108-123`) — `created_at DESC, id`.
- שגיאות: `BudgetAgreementUnavailableError(TransientError)`→503; `UnexpectedBudgetAgreementError`→500 (`:32-53`).

---

## 9. טבלת ה-endpoints (כולם `POST /me/agent/conversations/{conversation_id}/...`)

| Path | Router (`agent.py`) | Orchestrator | תפקיד |
|------|---------------------|--------------|-------|
| `/diagnose` (`few_leads`) | `:286-287` | `diagnose_problem_4` (`:634-690`) | אבחון + הצגת הפער + צ'יפים |
| `/budget/preview` | `:319-329` | `preview_budget_increase` (`:758-775`) | אישור-מפורש (בלי record/Meta) |
| `/budget/approve` | `:341-352` | `approve_budget` (`:778-827`) | **החלה: record → update_ad_set → sync quiz** |
| `/budget/decline` | `:364-374` | `decline_budget` (`:886-908`) | תיעוד סירוב (בלי Meta) |
| `/budget/force-creative` | `:386-397` | `force_creative` (`:917-987`) | ניסיון קריאייטיב חד-פעמי |

- **api.js:** `diagnose(...)` (`:231-241`), `budgetAction(conversationId, action)` (`:245-250`, בונה `.../budget/{action}`). *(הערה: docstring של `budgetAction` ב-`api.js:243-244` מיושן — כתוב "approve|decline", אך בפועל מועברים 4 ערכים: `preview`/`approve`/`decline`/`force-creative`.)*
- כל ה-endpoints: `Depends(get_current_user)`, `response_model=DiagnoseResponse`, `except AgentServiceError → _to_http_error`.

---

## 10. דיאגרמת רצף — מסלול "הגדל תקציב"

```
Frontend (index.html)        api.js            Router (agent.py)        Orchestrator                     Meta / DB
────────────────────         ──────            ─────────────────        ────────────                     ─────────
לחיצה "מעט פניות"
  runDiagnose ───────────► diagnose ─────────► POST /diagnose ────────► diagnose_problem_4
                                                                           ├─ fetch_last (reactive)  ◄──── budget_agreements (SELECT)
                                                                           ├─ market_cpl, daily, goal
                                                                           ├─ assess() → feasibility
   ◄───── budget_options {msg, [הגדל תקציב, השאר]} ◄──────────────────────┘

לחיצה "הגדל תקציב"
  runBudgetAction('preview') ► budgetAction ► POST /budget/preview ─────► preview_budget_increase
                                                                           └─ _assess_budget_increase
   ◄───── {msg "מ-₪Y ל-₪W... מאשר?", [מאשר, ביטול]} ◄──────────────────────┘
  CHAT.budgetConfirm=true (רק אם השרת החזיר confirm_raise)

לחיצה "מאשר" (budgetConfirmPending)
  runBudgetAction('approve') ► budgetAction ► POST /budget/approve ─────► approve_budget
                                                                           ├─ _assess_budget_increase (recompute)
                                                                           ├─ record(budget_increase_approved) ──► budget_agreements (INSERT)  [reserve-first]
                                                                           ├─ get_push_client
                                                                           ├─ update_ad_set(daily_budget=agorot) ─► Meta AdSet  (או no-op בסנדבוקס)
                                                                           │     כשל → _safe_rollback ──────────► budget_agreements (DELETE)
                                                                           └─ update_budget_to_daily ───────────► quiz_responses (UPDATE)
   ◄───── budget_options {"עדכנתי את התקציב ל-₪W...", [חזרה לתפריט]} ◄──────┘
```

---

## 11. יחידות, עיגול, ומלכודות (gotchas)

1. **אגורות מול ש"ח.** פנימית התקציב בש"ח; Meta מקבל **אגורות (×100)**. approve משתמש ב-`ceil` (לא `round`) כדי לא לרדת מתחת ל-required. צד היצירה משתמש ב-`round`.
2. **מקור התקציב לאבחון = השאלון** (`quiz.answers.budget`), **לא** Meta החי. אם הלקוח שינה תקציב ישירות ב-Meta — האבחון לא יידע (עד sync).
3. **ה-CPL אינו קלט לאבחון.** בעיה 4 מניחה קמפיין בריא ומשתמשת ב-`market_cpl` של הענף. CPL חי נשלף רק ל-reactive של `creative_against_advice`.
4. **תקציב lifetime → אין "הגדל" אוטומטי.** האבחון מציע keep-only + רמז ידני; approve דוחה lifetime (ב-`_assess_budget_increase`).
5. **`monthly_lead_goal = null` → הצ'יפ לא מופיע → הבעיה לא נגישה.** אין null-handling בתוך הזרימה.
6. **אין status/worker/cron בענף approve** — הכל סינכרוני; המצב מיוצג בקיום-שורה; ה-reactive נשען על שליפת-הרשומה.
7. **gating אנטי-זיוף:** פעולות שמזיזות כסף (`approve`, `force-creative`) מגודרות בדגלי `CHAT.budgetConfirm`/`CHAT.creativePending` שעולים **רק** כשהשרת החזיר את צ'יפ ה-id המתאים, ונצרכים אטומית בראש כל dispatch. טקסט מוקלד "מאשר" → `_freeText` (לא חיוב).
8. **`update_ad_set` ברמת ad_set, לא campaign.** מקור: `campaigns.meta_ad_set_id`.

---

## 12. הבחנה מבעיה 2 — ה-"value-turn" **אינו** כאן

ה-"value-turn" שייך ל**בעיה 2** (`low_quality_leads` / `filter_addon`), **לא** ל-`budget_mismatch`:
- בעיה 2 היא **תלת-שלבית**: (1) 2 צ'יפי-קטגוריה `[לא מהאזור הרצוי, אין תקציב מתאים]` → (2) **turn של ערך** (`value_question` — אזור/מחיר, בלי לפתוח session) → (3) ניתוח + פתיחת session (`diagnose_problem_2`, `agent_orchestrator.py:376-431`).
- בעיה 4, לעומת זאת, היא **חד-שלבית, חסרת-מצב**: בקשה→תשובה יחידה; ה"2 קטגוריות" המקבילות שלה הן פיצול `feasible/infeasible`; אין subcategory ואין value-turn ואין session (הלולאה היא reactive דרך `budget_agreements`).
- תיעוד בעיה 2: `docs/low-quality-leads-filter-redesign.md`, `docs/frontend-integration.md:57-58`.

---

## 13. מפת קבצים (אינדקס file:line)

**Frontend** — `app/web/index.html`
- `1901-1913` שדה יעד בשאלון · `5112-5117`/`5145` קריאה ל-quiz
- `2540` `_PROBLEM_PT` · `2588-2596` `_renderProblemMenu` (צ'יפ מותנה `:2593`)
- `2599-2624` `_chatDispatch` (ניתוב 6 ענפי-תקציב; `:2602-2603` צריכת-דגלים אטומית)
- `2626-2666` `runDiagnose` (case `budget_options` `:2654-2659`) · `2670-2682` `runBudgetAction` (`:2679` budgetConfirm, `:2680` creativePending)
- `2543-2557` מיפוי שגיאות עברית (`_agentErr`/`_agentErrMsg`)

**api.js** — `231-241` `diagnose` · `245-250` `budgetAction`

**Orchestrator** — `app/services/agent_orchestrator.py`
- `634-690` `diagnose_problem_4` · `584-631` `_phrase_budget_mismatch` · `854-883` `_format_reactive_reminder` · `843-851` `_fmt_agreement_date`
- `758-775` `preview_budget_increase` · `778-827` `approve_budget` · `720-755` `_assess_budget_increase` · `237-243` `_safe_rollback` · `215-234` `_guard`
- `886-908` `decline_budget` · `917-987` `force_creative` · `831-840` `_format_budget_declined_message` · `711-717` `_format_budget_approved_message`
- `525-549` קבועים (`_PROBLEM_BUDGET_MISMATCH`, `_BUDGET_TRIGGER`, צ'יפים, הודעות-degradation) · `562-575` `_normalize_daily_budget` · `552-559` `_budget_for_campaign`

**Services** — `budget_feasibility_service.py:24-62` (assess+dataclass) · `benchmark_service.py:101-112` (market_cpl) · `budget_agreement_service.py:23-123` (record/fetch_last/delete_by_id) · `quiz_service.py:324-349` (update_budget_to_daily) · `prompts_service.py:186-229` (build_agent_prompt)

**Meta seam** — `meta.py:664-672` (`update_ad_set`) · `meta_client.py:60`/`113-114` · `sandbox_meta_client.py:91-92` · `meta_client_factory.py:39-51`

**Router** — `agent.py:286-287` (diagnose) · `:319-397` (4 endpoints של budget)

**Prompts** — `app/prompts/agent/modules/budget_mismatch/{feasible,infeasible}.txt`

**Migrations** — `0102` (monthly_lead_goal) · `0103_budget_agreements.sql` · `0104_budget_mismatch_problem_type.sql`

**Models** — `app/models/agent.py` (`DiagnoseRequest`/`DiagnoseResponse`/`Chip`/`ConversationResponse`) · `app/models/benchmark.py:15-25` (INDUSTRY_KEYS)

**Sandbox** — `sandbox_service.py:296-297` (diagnose), `run_budget_action` (approve/decline/force-creative)

**מסמכים קשורים** — `docs/ROADMAP.md` (Session 7.6.5, חלקים 1–11) · `docs/problem4-force-creative-plan.md` (ענף force-creative) · `docs/sandbox-agent-prompts.md:1044-1047`
