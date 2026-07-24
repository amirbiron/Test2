# תכנית שינוי — בעיה 4 (`few_leads` / `budget_mismatch`): הסרת יעד-לידים, תקציב בקלט-משתמש, והתראה לאחר הגדלה

> מסמך תכנון בלבד. אינו נוגע בקוד — מיועד להעלאה לריפו ולהרצה דרך Claude Code.
> מקור-אמת של המצב הקיים: מסמך המיפוי המפורט של בעיה 4 (`docs/budget-mismatch-flow.md`, מיפוי file:line). מסמך זה מגדיר רק את הדלתא ביחס אליו, לפי אותם עוגני-קוד.
> **החלטות-מוצר שאושרו:** (1) מסירים את "העצירה החכמה" (המקרה שבו התקציב מספיק → הפניה לבעיה 1). (2) המשתמש **מקליד** את התקציב החדש (במקום שהשרת יחשב אותו מהיעד). ולידציה: אסור נמוך מהתקציב הנוכחי; תקרה של **+100₪** מעל התקציב היומי הנוכחי.

---

## 0. תמצית הדלתא

לפני: יעד-לידים (`monthly_lead_goal`) נאסף בשאלון, הצ'יפ "מעט פניות" מותנה בקיומו, האבחון משווה `expected_leads` מול היעד ומסווג `feasible`/`infeasible`, וה-approve מחשב את התקציב החדש מהיעד. אחרי: אין יעד כלל, הצ'יפ מוצג תמיד, האבחון מציג את צפי-הלידים בתקציב הנוכחי ומציע להוסיף תקציב בלי לשפוט היתכנות, המשתמש מקליד סכום, השרת מוודא אותו (>נוכחי, ≤נוכחי+100), מחיל אותו על Meta, ואז שולח מייל+וואטסאפ.

מהות השינוי: המסלול נעשה **פשוט יותר** (נעלמת כל המתמטיקה של `required_budget` ושל ה-split), אבל נוסף בו לראשונה **קלט-משתמש שמזיז כסף** — ולכן מרכז הכובד עובר לשכבת ולידציה שרתית, ולתוספת ערוץ-התראה. שלוש התכונות הקיימות שאסור לאבד: reserve-first (רישום לפני Meta), rollback בכשל Meta, ו-idempotency של ערך-מוחלט אצל Meta (`daily_budget = agorot`). כולן נשמרות; הן דווקא **מתחזקות**, כי עכשיו יש קלט-משתמש שצריך לאמת בשני שלבים (preview + approve).

---

## 1. הכרעות — מאושרות, ושלוש הכרעות-משנה שאני מסמן להחלטתך

| # | הכרעה | סטטוס |
|---|-------|-------|
| 1 | מסירים את "העצירה החכמה" — כל תלונה על מעט פניות מציגה צפי ומציעה הגדלה, בלי לבדוק אם התקציב כבר מספיק. | ✅ אושר |
| 2 | המשתמש מקליד את התקציב החדש. ולידציה: אסור נמוך מהנוכחי (→ הודעת "זה לא מסתדר" + שאלה חוזרת); תקרה +100₪ מעל הנוכחי. | ✅ אושר |
| 3 | **הצ'יפ "מעט פניות" נעשה בלתי-מותנה.** תוצאה ישירה של הסרת היעד: אי אפשר יותר לגדר אותו ב-`monthly_lead_goal != null`. | 🔸 החלטה-משנה — ברירת-המחדל שלי: תמיד מוצג. אם תרצה בעתיד לגדר אותו (למשל רק לקמפיין שרץ מספיק ימים) — נקודת-השינוי היא מקום אחד (`_renderProblemMenu`). |
| 4 | **תקציב מעל התקרה** (>נוכחי+100): שאלה חוזרת עם הסבר התקרה — **לא** קיצוץ שקט לתקרה. | 🔸 החלטה-משנה — ברירת-המחדל שלי: דחייה + שאלה חוזרת. לא מקצצים כסף בשקט. |
| 5 | **מיקום הכוכבית** (הדיסקליימר "דורש למידה חוזרת ואינו מבטיח כמות פניות"): על הודעת ההצלחה, אחרי ההגדלה — כפי שרצף הבקשה שלך מציג. | 🔸 החלטה-משנה — חלופה סבירה: להציג אותה כבר על מסך האישור ("מאשר?"), להסכמה-מדעת לפני שהכסף זז. אם תעדיף — זו שורה אחת להזזה. |

שלוש החלטות-המשנה קטנות ולא חוסמות; קבעתי ברירת-מחדל לכל אחת כדי שהמסמך יהיה שלם. תגיד אם משהו מהן לא מתאים.

---

## 2. שינוי 1 — הסרת יעד-הלידים מהשאלון

השדה `monthly_lead_goal` היה "השער" לכל הבעיה. מסירים אותו מהאיסוף לחלוטין. כל העוגנים מהמיפוי הקיים:

| מיקום | פעולה |
|-------|-------|
| `index.html:1901-1913` — שדה `#wiz-monthly-goal` בשלב התקציב (`#wiz-budget`) | להסיר את השדה מה-HTML של השאלון. |
| `index.html:5112-5117` — `_readMonthlyLeadGoal()` | למחוק את הפונקציה. |
| `index.html:5145` — אריזת `monthly_lead_goal` ב-`_buildQuizBody()` | להסיר את השורה. |
| `app/models/agent.py:41-42` — `ConversationResponse.monthly_lead_goal` | להסיר את השדה מהמודל. |
| `index.html:2581` — `CHAT.monthlyLeadGoal` ב-`_enterConversation` | להסיר את ההשמה. |

**עמודת ה-DB `campaigns.monthly_lead_goal` (migration `0102`) — נשארת, לא נוגעים בה.** היא nullable, תישאר NULL לקמפיינים חדשים, ואינה מזיקה. לכן שורש-הבעיה (התלות הלוגית ביעד) מוסר במלואו בלי migration הרסני. מחיקת העמודה עצמה אפשרית כניקוי עתידי, אבל היא לא נדרשת ולא שווה את הסיכון של migration הרסני עכשיו. (הערה: אין להסיר את `budget_agreements.monthly_lead_goal_at_time` — עמודת audit היסטורית, נשארת ותהיה NULL ברשומות חדשות; ראה §7.)

---

## 3. שינוי 2 — הצ'יפ "מעט פניות" בלתי-מותנה

`_renderProblemMenu()` (`index.html:2588-2596`) דחף את `'מעט פניות'` רק כאשר `CHAT.monthlyLeadGoal != null` (התנאי ב-`:2593`). מסירים את התנאי — הצ'יפ נכנס תמיד לצד `['עלות לפנייה גבוהה', 'פניות לא איכותיות']`. שאר הצ'יפים בתפריט (ככל שקיימים מעבר לשלושה אלה) — לא נוגעים בהם; משנים **רק** את הגדר על "מעט פניות". המיפוי `_PROBLEM_PT` (`:2540`, `'מעט פניות':'few_leads'`) נשאר כמות שהוא.

---

## 4. שינוי 3 — האבחון: הסרת feasible/infeasible, מסר-הצעה יחיד

`diagnose_problem_4(user_id, conversation_id)` (`agent_orchestrator.py:634-690`). המבנה החדש:

**קלטים שנשארים:** `campaign_id`, `industry`→`market_cpl` (`benchmark_service.market_cpl_for_industry`, ללא שינוי — עדיין `(amazing_max+expensive_above)/2`), `budget`→`daily_budget` (`_normalize_daily_budget`, ללא שינוי), `last_agreement` (ללולאה ה-reactive — §8).

**קלטים שמוסרים:** `goal` (`:647`), וקריאת `budget_feasibility_service.assess` (`:652`).

**החישוב החדש:** צפי-הלידים בלבד, בלי יעד להשוות אליו:
```
expected_leads = ⌊(daily_budget × 30) / market_cpl⌋
```
(floor — "לא מבטיחים יותר ממה שיש", כמו `int(expected)` הקיים.)

**התוצאות — משתיים היו שלוש, לשתיים:**

| תנאי | `type` | הודעה | צ'יפים |
|------|--------|-------|--------|
| אין `market_cpl` (ענף לא-מוכר/NULL) **או** `daily_budget is None` **או** אין `meta_ad_set_id` | `unavailable` | `_BUDGET_NO_DATA_MSG` (ללא LLM) | `[חזרה לתפריט]` |
| `budget.mode == "lifetime"` | `budget_options` | מסר-הצפי + `_BUDGET_LIFETIME_HINT` | `[השאר את התקציב הקיים, חזרה לתפריט]` — **בלי "כן"** (approve לא נוגע ב-lifetime) |
| רגיל (daily) | `budget_options` | מסר-הצפי (LLM, §6) | `[כן אני רוצה להגדיל את התקציב, לא אני רוצה להישאר עם התקציב הקיים]` |

**מה נמחק כאן:** ענף `feasibility.feasible` שהפנה לבעיה 1 (`high_cpl`) — נעלם לגמרי (החלטה 1). וכל ה-payload של היעד (`monthly_lead_goal`, `required_daily_budget_for_goal`) יורד מהניסוח.

**מה נשמר:** בדיקת `meta_ad_set_id` מוקדמת נכנסת כבר לאבחון — כי אם אין ad_set, אין טעם להציע הגדלה שתיכשל אחר-כך; עדיף `unavailable` מיד. (במיפוי הקיים בדיקת ה-ad_set הייתה רק ב-`_assess_budget_increase:729`; מקדימים אותה כדי לא להציע "כן" שיגיע לקיר.)

> **הבחנה חשובה מה-CPL החי:** בעיה 4 עדיין **לא** משתמשת ב-CPL בפועל של הקמפיין. היא מניחה קמפיין בריא ומשתמשת ב-`market_cpl` של הענף. ה-CPL החי נשלף רק בלולאה ה-reactive עבור רשומת `creative_against_advice` קודמת (`:668-675`), ללא שינוי.

### שירות `budget_feasibility_service` — לצמצם, לא למחוק

השירות כולו (`assess`, `BudgetFeasibility`, `_finite_positive` ב-`budget_feasibility_service.py:24-62`) נבנה כדי לחשב `feasible`/`required_budget`/`expected_leads`. בעולם החדש נשאר צורך אחד בלבד — `expected_leads`. השורש: להחליף את `assess` בפונקציה אחת שמחזירה `int | None`:
```
def expected_leads(daily_budget, market_cpl) -> int | None
```
היא שומרת על שער-התקינות (`_finite_positive` על שני הקלטים) שממנו נגזרת התנהגות ה-degradation (`None` → `unavailable`). ה-dataclass `BudgetFeasibility` ו-`required_budget`/`feasible` — נמחקים. זהו פתרון-שורש (הסרת הלוגיקה שהתייתרה), לא טלאי — לא משאירים dataclass עם שדה יחיד או שדות מתים.

---

## 5. שינוי 4 — תור חדש: בקשת התקציב מהמשתמש (חסר היום)

זהו החלק החדש היחיד שאין לו מקבילה במימוש הנוכחי. היום: "הגדל תקציב" → `preview` (השרת מחשב W) → `approve`. אחרי: "כן" → הסוכן **שואל** "לאיזה תקציב תרצה להעלות?" → המשתמש **מקליד** → ולידציה → `preview` ("מ-Y ל-W מאשר?") → `approve`.

מבנית זה מוסיף ל-בעיה 4 **turn של קלט** — בדיוק כמו ה-value-turn של בעיה 2. (במיפוי הקיים §12 נכתב שבעיה 4 "חסרת-מצב, בלי value-turn"; מעכשיו יש לה אחד. יש לעדכן את §12 של מסמך המיפוי — §12 כאן.) הדפוס לחיקוי: אותו מנגנון של דגל-frontend + צריכה-אטומית + אנטי-זיוף שכבר קיים ל-`budgetConfirm`/`creativePending`.

### 5.1 רצף התור

1. **צ'יפ "כן אני רוצה להגדיל את התקציב"** (id `raise_budget`) → `runBudgetAction('raise-intent')` → `POST .../budget/raise-intent` (endpoint חדש, ראה §10).
   - השרת מחזיר `type=budget_input` + הודעה: `_format_budget_ask_message(current_daily)` = **"התקציב היומי הנוכחי שלך בקמפיין הוא ₪{X}. לאיזה תקציב יומי תרצה להעלות?"** + צ'יפ `[ביטול]` (→ תפריט).
   - ה-endpoint הזה הוא **הודעה בלבד** — בלי record, בלי Meta — בדיוק כמו `preview` היום (`:758-775` "בלי record, בלי Meta"). הוא רק שולף את התקציב היומי הנוכחי (`_budget_for_campaign` → `_normalize_daily_budget`) ומנסח.
   - Frontend מציב `CHAT.awaitingBudgetInput = chips.some(c => c.id === 'awaiting_budget')` — הדגל עולה **רק** אם השרת החזיר בפועל את הצ'יפ-הסמן `awaiting_budget` (אנטי-זיוף; אותו דפוס כמו `:2679`). (אפשר גם לגזור אותו מ-`type === 'budget_input'`, אך צ'יפ-סמן עקבי עם השאר.)

2. **המשתמש מקליד מספר** → `_chatDispatch` (`:2599-2624`): הדגל `awaitingBudgetInput` **נצרך אטומית בראש ה-dispatch** (ליד `:2602-2603`, כמו `budgetConfirmPending`) → מפנה ל-`runBudgetTarget(text)` במקום ל-`_freeText`. כל ניתוב אחר מאפס את הדגל.
   - `_parseBudgetInput(text) -> number | null` (frontend, חדש): מחלץ מספר מהטקסט (תומך ב-`"90"`, `"90 ש"ח"`, `"₪90"`, `"90.5"`, פסיקים). ריק/לא-מספרי → `null`.
   - `null` → שאלה חוזרת מקומית (בלי סבב-שרת): הודעת "לא הצלחתי לקרוא את הסכום…" + השארת מצב הקלט פתוח.
   - מספר → `window.api.budgetPreviewTarget(CHAT.conv, num)` → `POST .../budget/preview` עם גוף `{target_daily: num}`.

3. **preview מחזיר אחת משתיים:**
   - **תקין** → `type=budget_options`, הודעה "מ-₪{Y} ל-₪{W} מאשר?" + `[מאשר (confirm_raise), ביטול (cancel_raise)]`. Frontend מציב `CHAT.budgetConfirm = chips.some(id==='confirm_raise')` (`:2679`, קיים) **וגם** `CHAT.pendingTarget = num` (חדש — הערך לשליחה ל-approve). W כאן = הקלט של המשתמש, לא חישוב.
   - **לא-תקין** (נמוך/גבוה מדי) → `type=budget_input`, שאלה חוזרת (§5.3), Frontend מחזיר את `awaitingBudgetInput` למצב פתוח.

4. **צ'יפ "מאשר"** → `budgetConfirmPending` נצרך אטומית → `runBudgetAction('approve', {target_daily: CHAT.pendingTarget})` → `POST .../budget/approve`. השרת **מאמת מחדש** את `target_daily` (server-authoritative — §5.2), לא סומך על הערך מהלקוח.

### 5.2 ולידציה שרתית — נקודת הכובד החדשה

הכלל מ-spec §7 ("נתון שמגיע מהמשתמש לעולם אינו ה-authority לפעולת admin") נשמר: התקציב שהמשתמש הקליד עובר ולידציה **בשרת**, בשני שלבים נפרדים (preview **וגם** approve). ה-frontend מוודא רק לנוחות; השרת הוא השער.

הולידציה יושבת ב-`_assess_budget_increase(...)` (`:720-755`), שמשתנה מהותית — במקום לחשב W מהיעד, הוא **מקבל** `target_daily` ומוודא אותו:

```
current_daily = _normalize_daily_budget(quiz.answers.budget)   # הנוכחי, מהשאלון
cap = current_daily + 100

שערים (מוחזרים כ-(False, AgentDecision)):
  • market_cpl is None / current_daily is None / אין ad_set_id  → unavailable (degradation)
  • budget.mode == "lifetime"                                   → הודעת-lifetime (approve לא נוגע)
  • target_daily לא finite/positive                            → budget_input: שאלה-חוזרת "לא הצלחתי לקרוא…"
  • target_daily <= current_daily                              → budget_input: שאלה-חוזרת "זה לא מסתדר" (§5.3)
  • target_daily > cap                                         → budget_input: שאלה-חוזרת "עד ₪{cap}" (§5.3)

הצלחה:
  new_daily = target_daily
  agorot    = ceil(new_daily × 100 − 1e-9)      # עיגול כלפי מעלה, כמו הקיים (:750)
  return (True, {new_daily, agorot, ad_set_id, market_cpl, current_daily, ...})
```

הערך `market_cpl` נשמר בתוצאה כדי לחשב `expected_leads` לרשומת ה-audit (§7). שים לב: ה-"feasible → כבר מספיק → הפניה לבעיה 1" (`:feasibility.feasible` הקיים) **נמחק** — אין יותר מושג feasible.

**למה גם ב-approve וגם ב-preview:** בין preview ל-approve התקציב הנוכחי יכול (תיאורטית) להשתנות, והלקוח יכול לשלוח `pendingTarget` שונה מזה שהוצג. approve שולף מחדש את הנוכחי ומוודא מחדש את `target_daily` מולו + מול התקרה. אותה פונקציה (`_assess_budget_increase`) משרתת את שניהם — DRY, ושרת-אוטוריטטיבי בשתי הנקודות.

### 5.3 נוסחי השאלות-החוזרות (סטטי, לא LLM)

בדיוק כמו `preview`/`approve`/`decline` שהם היום סטטיים (`_format_*`, `:711-717`, `:831-840`), גם התור החדש טרנזקציוני → הודעות סטטיות דטרמיניסטיות:

- **בקשת התקציב** (`_format_budget_ask_message`): "התקציב היומי הנוכחי שלך בקמפיין הוא ₪{X}. לאיזה תקציב יומי תרצה להעלות?"
- **נמוך מהנוכחי** (הנוסח שלך, `_format_budget_too_low_message`): "ביקשת להגדיל את התקציב היומי בגלל שכמות הלידים נמוכה לטענתך, אבל בחרת סכום נמוך מהתקציב הנוכחי — זה לא מסתדר. אשאל שוב: התקציב היומי שלך כרגע הוא ₪{X}, לאיזה תקציב יומי תרצה להעלות?"
- **מעל התקרה** (`_format_budget_too_high_message`): "כדי לשמור על יציבות הקמפיין אפשר להעלות את התקציב היומי עד ₪{cap} בכל פעם. לאיזה תקציב יומי תרצה להעלות?"
- **לא-מספרי** (`_format_budget_unparseable_message`): "לא הצלחתי לקרוא את הסכום. אנא הקלד מספר בלבד (למשל 90). לאיזה תקציב יומי תרצה להעלות?"

כל אחת מהן מוחזרת עם `type=budget_input` + צ'יפ-סמן `awaiting_budget` (+ `[ביטול]`), כדי שה-frontend ידע להשאיר את מצב הקלט פתוח.

---

## 6. שינוי 5 — ניסוח ה-LLM: קובץ פרומפט אחד במקום שניים

`_phrase_budget_mismatch(...)` (`:584-631`) ושני קבצי המודול קורסים לאחד:

- **מוחקים:** `app/prompts/agent/modules/budget_mismatch/feasible.txt`.
- **מחליפים:** `.../infeasible.txt` → קובץ יחיד `.../offer.txt` (מסר-ההצעה). `state_key` נעשה קבוע `"offer"` (במקום `"feasible"/"infeasible"` מ-`:659`), ו-`build_agent_prompt(mode="chip", issue_type="budget_mismatch", state_key="offer", ...)` תמיד טוען את הקובץ הזה.

**ה-payload החדש** ל-`[SYSTEM_PAYLOAD]` (השדות שיורדים: `monthly_lead_goal`, `required_daily_budget_for_goal`):

| מפתח | ערך |
|------|-----|
| `service_name` | שם השירות |
| `market_average_cost_per_lead` | `market_cpl` |
| `current_daily_budget` | `daily_budget` |
| `leads_the_current_budget_allows_per_month` | `expected_leads` |

פרמטרי ה-LLM ללא שינוי (`temperature=0.5`, `max_tokens=600`, `json_mode=False`; rate-limit→503, אחר→500).

**נוסח היעד ש-`offer.txt` מכוון אליו** (לפי הבקשה): "עם תקציב יומי של ₪{current_daily_budget} ועם עלות פנייה ממוצעת בשוק של כ-₪{market_average_cost_per_lead}, הצפי החודשי הממוצע הוא כ-{expected_leads} לידים. האם תרצה להוסיף תקציב למודעות?"

**כתיבת הפרומפט — הקשר מורחב, לא תמצות.** לפי כלל-הפרומפטים: הקובץ צריך להסביר ל-LLM את **מלוא ההקשר** של הרגע — שהלקוח התלונן על מעט פניות, שהמערכת כבר חישבה דטרמיניסטית את הצפי (חוק 7 — ה-LLM לא מעריך היתכנות, רק מנסח), שהמטרה היא להציג את המספרים בשקיפות ולהציע הגדלה בלי ללחוץ ובלי להבטיח תוצאה, ושהטון בגובה-העיניים ובעברית מקצועית ללא מונחים באנגלית (spec §"חוקי שפה"). ההסבר הרחב הזה נותן ל-LLM תמונה שלמה של למה הוא מנסח את מה שהוא מנסח, וזה נקלט הרבה יותר טוב מהוראה מתומצתת. הערכים המספריים מגיעים דרך `[SYSTEM_PAYLOAD]` (המודול עצמו בטקסט חופשי, בלי `{}` — כמו הקיים).

**המסרים הטרנזקציוניים נשארים סטטיים** — רק שני שינויים בהם:
- `preview` ("מ-₪Y ל-₪W… מאשר?", `:768-771`) — ללא שינוי מבני; W עכשיו = הקלט של המשתמש.
- `_format_budget_approved_message` (`:711-717`) — מוסיפים את הכוכבית ומרככים את ההבטחה. ראה §7.3.

---

## 7. שינוי 6 — התאמת התיעוד (`budget_agreements`) — בלי migration

טבלת `budget_agreements` (`0103`) נשארת כמות שהיא — אין status, אין UNIQUE, שלושת ה-`agreement_type` ב-CHECK ללא שינוי. משתנים רק הארגומנטים ל-`record(...)` (`budget_agreement_service.py:56-94`) ומקור-החישוב של כמה שדות:

| שדה | `budget_increase_approved` (חדש) | `budget_increase_declined` (חדש) |
|-----|----------------------------------|----------------------------------|
| `daily_budget_at_time` | הנוכחי (Y, לפני ההגדלה) | הנוכחי (Y) |
| `new_budget` | הקלט של המשתמש (W) | NULL |
| `expected_leads` | `expected_leads(new_daily, market_cpl)` — צפי תחת התקציב **החדש** | `expected_leads(current, market_cpl)` — תחת הנוכחי |
| `market_cpl` | market_cpl | market_cpl |
| `monthly_lead_goal_at_time` | **NULL** (אין יעד) | **NULL** |
| `required_budget` | NULL (המושג "דרוש-ליעד" בוטל) | NULL |
| `cpl_before` | NULL | NULL |

`creative_against_advice` — ללא שינוי (`cpl_before` = CPL חי).

### 7.1 reserve-first + rollback — ללא שינוי

רצף ה-approve נשאר: **record לפני Meta** (`:793-798`) → `update_ad_set(ad_set_id, {"daily_budget": agorot})` (`:800-802`) → כשל → `_safe_rollback(agreement_id)` (מחיקת השורה, best-effort, `:803-817`) → הצלחה → sync שאלון (`:823`, `update_budget_to_daily`, fail-loud). מיפוי הכשלים (`MetaClientNotConnectedError`/`MetaTransientError`→503/`MetaApiError`→degradation) ללא שינוי. "מצב סופי" מיוצג בקיום-שורה, לא ב-status. הכל סינכרוני, בלי worker/cron בענף approve.

### 7.2 idempotency — ללא שינוי בהיגיון

`daily_budget = agorot` הוא ערך-מוחלט מחושב-מחדש דטרמיניסטית → double-click/retry מתכנס לאותו מצב ad_set. double-click מוסיף שורת-audit עודפת בלבד ("תיעוד עודף, לא נזק"). **הבדל אחד חדש:** ההתראה (§8) **לא** יכולה להיות "עודפת" בלי נזק — מייל/וואטסאפ כפול זו הטרדה. לכן ל-התראה יש idempotency נפרד וחזק יותר (ראה §8.3).

### 7.3 הודעת ההצלחה + הכוכבית

`_format_budget_approved_message(new_daily)` (`:711-717`) — מרככים את ההבטחה הקיימת ("אתה אמור לראות עלייה בכמות הפניות") ומוסיפים דיסקליימר, שגם מתיישב עם spec §"לעולם אל תבטיח תוצאות":

> "מצוין — עדכנתי את התקציב היומי ל-₪{W}. התשלום למטא ייגבה מעתה לפי התקציב החדש.
> \* כל העלאת תקציב או שינוי מהותי בקמפיין דורש למידה חוזרת של המערכת ואינו מבטיח כמות פניות מסוימת."

(מיקום הכוכבית — החלטה-משנה 5. אם תעדיף אותה על מסך האישור, מעבירים את השורה ל-`preview`.)

### 7.4 הלולאה ה-reactive — כמעט ללא שינוי

`_format_reactive_reminder` (`:854-883`) קורא תאריך + `new_budget` (ל-approved) ו-`expected_leads`/Y (ל-declined). אף אחד מהם לא קרא את היעד — לכן הווריאנטים ממשיכים לעבוד כמות שהם. ודא רק שהנוסחים לא מזכירים "יעד" בשום מקום (אם כן — להסיר). הזיכרון נשאר הטבלה, בלי cron.

---

## 8. שינוי 7 — התראה לאחר הגדלה (מייל + וואטסאפ) — חדש

אחרי approve מוצלח (שורה נשמרה, שאלון סונכרן) שולחים ללקוח הודעה שהתקציב הוגדל כבקשתו. זו **תוספת ערוץ קיים, לא אינטגרציה חדשה** — מתחברת לתשתית ההתראות שכבר בנויה.

### 8.1 לאיזה ערוץ זה שייך

הבקשה שלך קשרה במפורש את הוואטסאפ ל"ערוץ עדכוני ווטסאפ" — כלומר ערוץ **עדכוני-הסוכן לבעלים** (spec §"התראות — שני ערוצים נפרדים"), ולא ערוץ התראות-המערכת. לכן:
- **מייל** → ל-`subscriptions.agent_alert_email` (המקור הבלעדי, בלי fallback למייל-ההרשמה; ריק → דילוג).
- **וואטסאפ** → ל-`subscriptions.agent_alert_phone`, **רק אם** `agent_whatsapp_enabled = true` ומספר קיים — בדיוק סמנטיקת ערוץ עדכוני-הסוכן.
- נספר מול `agent_alerts_quota` (חינם, 30/חודש; מעבר למכסה → וואטסאפ נעצר, **מייל ממשיך**).
- זהו **סוג-אירוע חמישי** על ארבעת הקיימים (step_advanced/series_resolved/ad_rejected/quality_followup): `budget_increased`.

### 8.2 תלות קריטית שצריך לאמת בריפו (ל-CC)

ערוץ עדכוני-הסוכן לוואטסאפ מתועד כ**רדום מאחורי gate ה-WABA** (template `agent_update_owner` טרם אושר, `whatsapp_production_ready` כבוי), ו-Phase 8.3 (השליחה היזומה לוואטסאפ) מסומן 🔮 (טרם התחיל). לכן **המימוש בפועל של שולח עדכוני-הסוכן אולי עדיין לא קיים** — בעוד ששולח התראות-המערכת (`send_notification`, Resend, `sent_notifications`) קיים ועובד מ-Phase 4.6.

**הוראה ל-CC:** בדוק בריפו אם קיים שולח לעדכוני-הסוכן (רישום template `agent_update_owner` + החלטת `agent_alerts_quota`). לפי הממצא:
- **אם קיים** → הוסף אליו את הסוג `budget_increased` (מייל + וואטסאפ, ה-וואטסאפ מגודר ב-`whatsapp_production_ready`).
- **אם לא קיים** → מְמַשׁ את **רגל המייל** דרך `send_notification` הקיים (סוג `budget_increased`, נמען = `agent_alert_email`), ובנֵה **רגל וואטסאפ מגודרת** מאחורי `whatsapp_production_ready` + `agent_whatsapp_enabled`, בדיוק בדפוס הרדום-עד-WABA. כך המייל עובד היום, והוואטסאפ יידלק אוטומטית כשה-WABA יאושר — בלי לשנות קוד.

בשני המקרים: **המייל עובד מיד; הוואטסאפ יישלח בפועל רק אחרי אישור ה-WABA.** זה מצב-הביניים המתוכנן, לא באג.

### 8.3 מנגנון + idempotency

- **מתי:** אחרי הצלחת approve + sync (בסוף `approve_budget`, אחרי `:823`).
- **איך:** enqueue של job `send_notification` (לא סינכרוני — כדי לא לתלות את הצלחת approve בהצלחת המייל; אותו דפוס כמו טריגרי 4.6 מ-3.4/2.6.1). גוף: `{type:'budget_increased', campaign_id, user_id, old_daily, new_daily, service_name}`.
- **best-effort:** כשל ב-enqueue → warning/Sentry, **לא** מפיל את approve (התקציב כבר הוגדל; ההתראה משנית, כמו שאר ההתראות).
- **idempotency:** מפתח ב-`sent_notifications` = `budget_increased:{campaign_id}:{agorot}`. שני approve לאותו יעד (double-click) → התראה אחת. הגדלה מאוחרת ליעד אחר (agorot אחר) → התראה חדשה. (מפתח על היעד, לא על `agreement_id`, כי double-click יוצר שתי שורות audit אך צריך התראה אחת — §7.2.)
- **תוכן:** "התקציב היומי בקמפיין {service_name} הוגדל מ-₪{old_daily} ל-₪{new_daily} כבקשתך." + אותו דיסקליימר של הכוכבית.
- **סנדבוקס:** לא שולחים התראה כלל (approve בסנדבוקס לא עושה enqueue) — מקביל ל-no-op של Meta (§9).

---

## 9. שינוי 8 — parity בסנדבוקס

הסנדבוקס מריץ מימושים מקבילים שחייבים לשקף את הזרימה החדשה:
- `sandbox_service.py:296-297` (diagnose) — לשקף את האבחון החדש (בלי feasible/infeasible, מסר-הצעה יחיד).
- `run_budget_action` בסנדבוקס — להוסיף את `raise-intent`, ולהתאים `preview`/`approve` לקבלת `target_daily` + אותה ולידציה (>נוכחי, ≤נוכחי+100).
- עדכון ה-Meta נשאר **no-op מלוגג** (`sandbox_meta_client.py:91-92`); רישום ההסכמה וסנכרון השאלון כן קורים (כמו היום).
- **התראה:** לא נשלחת בסנדבוקס.
- `docs/sandbox-agent-prompts.md` — עדכון הנוסח החדש אם מוגדרים שם פרומפטים לבעיה 4.

---

## 10. שינוי 9 — מודלים, `api.js`, ו-wiring ב-frontend

### 10.1 Endpoints (כולם `POST /me/agent/conversations/{conversation_id}/...`)

| Path | Orchestrator | שינוי |
|------|--------------|-------|
| `/diagnose` (`few_leads`) | `diagnose_problem_4` (`:634-690`) | שוכתב — §4 |
| `/budget/raise-intent` | `raise_budget_intent` (**חדש**) | חדש — שואל את התקציב (§5.1) |
| `/budget/preview` | `preview_budget_increase` (`:758-775`) | מקבל `{target_daily}`, מוודא, מחזיר preview או שאלה-חוזרת (§5) |
| `/budget/approve` | `approve_budget` (`:778-827`) | מקבל `{target_daily}`, מוודא-מחדש, record→update_ad_set→sync→**enqueue התראה** (§7–§8) |
| `/budget/decline` | `decline_budget` (`:886-908`) | ללא שינוי |
| `/budget/force-creative` | `force_creative` (`:917-987`) | ללא שינוי (עצמאי מהיעד — נשאר תחת ה-decline) |

### 10.2 `app/models/agent.py`

- להסיר `ConversationResponse.monthly_lead_goal` (`:41-42`).
- גוף בקשה ל-preview/approve: להוסיף `target_daily: float` (Pydantic מוודא מספר חיובי-סופי בלבד; טווח >נוכחי ו-≤נוכחי+100 נבדק בשרת מול ה-DB ב-`_assess_budget_increase`, לא כ-bound סטטי).
- `DiagnoseResponse`/`Chip` — ללא שינוי מבני; הסוג `budget_input` מצטרף לערכי ה-`type` הקיימים (`budget_options`/`unavailable`).
- `DiagnoseRequest.problem_type` (Literal, `:114`) — ללא שינוי (`few_leads` נשאר).

### 10.3 `api.js`

- להכליל את `budgetAction(conversationId, action, body=null)` (`:245-250`) כך שיצרף גוף אופציונלי — לתמיכה ב-`{target_daily}`.
- (אופציונלי, לקריאוּת) עטיפות: `budgetPreviewTarget(conv, target)` ו-`budgetApprove(conv, target)`. לתקן את ה-docstring המיושן (`:243-244`, "approve|decline") לרשימה המלאה: `raise-intent`/`preview`/`approve`/`decline`/`force-creative`.

### 10.4 `index.html`

- `_renderProblemMenu` (`:2588-2596`) — הצ'יפ בלתי-מותנה (§3).
- `_chatDispatch` (`:2599-2624`) — להוסיף ענף `awaitingBudgetInput` (צריכה-אטומית בראש, ליד `:2602-2603`) → `runBudgetTarget(text)`.
- `runBudgetTarget(text)` (**חדש**) — `_parseBudgetInput` → `null`? שאלה-חוזרת מקומית : `api` preview עם `{target_daily}`.
- `_parseBudgetInput(text)` (**חדש**) — חילוץ מספר (§5.1).
- `runBudgetAction` (`:2670-2682`) — לתמוך ב-action `raise-intent`; בטיפול ב-preview התקין להציב `CHAT.pendingTarget` (חדש) לצד `CHAT.budgetConfirm` (`:2679`); "כן" → `raise-intent`.
- דגלי-frontend: `awaitingBudgetInput` ו-`pendingTarget` (חדשים) נוהגים כמו `budgetConfirm`/`creativePending` — נצרכים אטומית, ועולים רק כשהשרת החזיר את הצ'יפ-הסמן המתאים (`awaiting_budget`/`confirm_raise`). טקסט מוקלד "מאשר" בלי דגל → `_freeText` (לא חיוב).

---

## 11. שינוי 10 — עדכון מסמכים

- `docs/ROADMAP.md` (Session 7.6.5, חלקים 1–11) — לשכתב: אין איסוף יעד, צ'יפ בלתי-מותנה, תקציב בקלט-משתמש + ולידציה +100₪, סוג-התראה `budget_increased`. `expected_leads` נשאר; `required_budget`/`feasible` יורדים.
- מסמך המיפוי של בעיה 4 (`docs/problem4-*`, המקור ל-file:line כאן) — לעדכן במלואו למצב החדש, כולל §12 שלו (בעיה 4 **כן** מקבלת turn של קלט מעכשיו) ו-§1/§3 (אין יעד; שתי תוצאות אבחון במקום שלוש).
- `docs/spec.md` — לוודא שאין התייחסות ל-`monthly_lead_goal` בשלב התקציב של ה-flow; להוסיף את `budget_increased` לרשימת סוגי עדכוני-הסוכן.
- אם מוגדרת בעיה 4 ב-`docs/sandbox-agent-prompts.md` — לעדכן נוסח.

---

## 12. מה נמחק / מה נשאר — טבלת-על

| רכיב | גורל |
|------|------|
| `monthly_lead_goal` בשאלון (frontend + מודל) | **נמחק** |
| `campaigns.monthly_lead_goal` (עמודת DB) | נשאר (nullable, NULL) — לא migration |
| גדר הצ'יפ על קיום-יעד | **נמחק** (צ'יפ תמיד מוצג) |
| ענף `feasible` → הפניה לבעיה 1 | **נמחק** (החלטה 1) |
| `budget_feasibility_service.assess`/`BudgetFeasibility`/`required_budget`/`feasible` | **נמחק** → `expected_leads(daily, market_cpl)` בלבד |
| `budget_mismatch/feasible.txt` | **נמחק** |
| `budget_mismatch/infeasible.txt` | מוחלף ב-`offer.txt` (state_key יחיד `"offer"`) |
| payload היעד (`monthly_lead_goal`, `required_daily_budget_for_goal`) | **נמחק** מהניסוח |
| חישוב W מהיעד ב-approve | **נמחק** → W = קלט המשתמש, מאומת בשרת |
| טיפול lifetime | **נשאר** (approve לא מעלה lifetime) |
| `expected_leads`, `market_cpl_for_industry`, `_normalize_daily_budget` | **נשאר** ללא שינוי |
| reserve-first + rollback + idempotency ערך-מוחלט + sync שאלון | **נשאר** ללא שינוי |
| `budget_agreements` (סכמה) | **נשאר** — בלי migration; משתנים רק ארגומנטי `record` |
| לולאה reactive (`_format_reactive_reminder`) | **נשאר** כמעט ללא שינוי |
| `force_creative` + `decline` | **נשאר** ללא שינוי |
| turn קלט-תקציב + ולידציה שרתית | **חדש** (§5) |
| `/budget/raise-intent` | **חדש** |
| התראה `budget_increased` (מייל+וואטסאפ) | **חדש** (§8) |

---

## 13. סדר ביצוע מומלץ ל-CC

1. **הסרת היעד** (§2) + **צ'יפ בלתי-מותנה** (§3) — שינוי מבודד, לא שובר כלום אחר.
2. **צמצום `budget_feasibility_service`** ל-`expected_leads` (§4) — לפני שכתוב האבחון, כי האבחון תלוי בו.
3. **שכתוב `diagnose_problem_4`** + **`offer.txt`** (§4, §6) — שתי התוצאות, מסר-ההצעה.
4. **תור הקלט**: `/budget/raise-intent` + הודעות סטטיות + ולידציה ב-`_assess_budget_increase` + preview/approve מקבלים `target_daily` (§5).
5. **frontend wiring**: `_parseBudgetInput`, `runBudgetTarget`, דגלים, `_chatDispatch`, `api.js` (§10).
6. **תיעוד ה-audit**: ארגומנטי `record` + כוכבית בהודעת ההצלחה (§7).
7. **ההתראה** `budget_increased` — קודם בדיקת-ריפו של שולח עדכוני-הסוכן, ואז מייל (ודאי) + וואטסאפ מגודר (§8).
8. **parity סנדבוקס** (§9) + **עדכון מסמכים** (§11).

בכל שלב: פתרון-שורש, לא טלאי — לא להשאיר dataclass מרוקן, לא לגדר את הצ'יפ על שדה מת, ולא לשלוח התראה בלי idempotency.

---

## 14. נקודות פתוחות שדורשות ממך החלטה לפני הרצה

1. **החלטות-המשנה 3–5** (§1): צ'יפ בלתי-מותנה / תקרה = דחייה-ושאלה-חוזרת / מיקום הכוכבית על ההצלחה. אישרתי ברירות-מחדל — תגיד אם משהו משתנה.
2. **תלות ה-WABA** (§8.2): הוואטסאפ יישלח בפועל רק אחרי אישור ה-template והדלקת `whatsapp_production_ready`. המייל עובד מיד. לוודא שזה מקובל כמצב-ביניים.
3. **מחיקת עמודת `monthly_lead_goal`** (§2): ברירת-המחדל — להשאיר (לא-הרסני). אם תרצה ניקוי מלא, זה migration נפרד בהמשך.
