# מסמך מימוש — PR6: force-creative (בעיה 4, ROADMAP חלק 7)

> **סטטוס:** ✅ **מומש** (ראה "הערות מימוש" בסוף המסמך לסטיות מהתוכנית).
> **בעיה:** 4 (`budget_mismatch` / "מעט פניות"). **תלוי ב:** PR1–PR5 (כבר מוזגו ל-`claude/eloquent-bardeen-9kGfT`).
> **ROADMAP:** Session 7.6.5, חלק 6–7. **branch:** `claude/eloquent-bardeen-9kGfT`.
> **מסמך-אב:** `/root/.claude/plans/goofy-plotting-hopcroft.md` (PR6 הוא ה-PR האחרון בתוכנית).

> ## ⚠️ עודכן — force-creative עבר ל-propose→approve (הצגת הקופי לאישור לפני push)
> המסמך שלהלן מתאר את המימוש **המקורי** (fire-and-forget: generate+push מיידי, בלי שהמשתמש רואה את הקופי).
> **שונה בהחלטת-מוצר:** force-creative כעת עובר **propose→approve** כמו בעיות 1-2 — הקופי (3 וריאציות +
> תמונות eager) מוצג למשתמש לבחינה/עריכה/**אישור** לפני push ל-Meta. הדלתא (מקור-אמת: הקוד):
> - **`force_creative` → `force_creative_propose`** (`agent_orchestrator`): פותח session +
>   `generate_solution(eager_images=True)` ומחזיר `ProposeResponse` (push_status=NULL). **בלי record/push/done.**
>   endpoint `budget/force-creative` → `ProposeResponse` (במקום `DiagnoseResponse`).
> - **ה-approve** זורם דרך ה-approve הגנרי (`POST /me/actions/{id}/approve` → `execute_approval`, ענף
>   `pt=="budget_mismatch"` מקביל ל-`meta_rejection`) → **`budget_creative_push_service`** (חדש, דק):
>   `push(window_hours=None)` → record `creative_against_advice` + סגירת `done`; `push_failed` → `failed`.
>   eager images → נתיב-ה-async (`claim_and_enqueue_images` → job branch `budget_mismatch`).
> - **ה-`record` עבר מ-reserve-first ל-אחרי-push-מוצלח** — שומר את האינווריאנט "מתועד ⟺ נדחף" (משתמש שנוטש
>   לפני אישור אינו יוצר record שקרי). `cpl_before`+`market_cpl` נשמרים ב-`starting_metric` ב-propose.
> - **frontend:** chip `force_creative` → `runForceCreativePropose` → `openActionView` (מיחזור view-action של בעיה 1).
> - **sandbox:** `run_budget_action("force-creative")` → propose (`eager_images=False`, תמונות inline); approve דרך `run_approve`.

---

## 0. TL;DR — מה ה-PR הזה עושה

הלקוח של קמפיין **בריא** (CPL תקין) מתלונן על מעט לידים. הסוכן הראה לו את האמת המתמטית (התקציב קטן מדי),
והלקוח **סירב** להעלות תקציב. בחלק 6 הצגנו לו הודעת-סירוב מורחבת עם שני צ'יפים. אם הלקוח **מתעקש** לנסות
קריאייטיב חדש נגד ההמלצה — חלק 7 (PR6) מבצע **ניסיון חד-פעמי**:

1. **מתעד** את ההתעקשות (`creative_against_advice` + `cpl_before`) — ההגנה המשפטית הקריטית ביותר, כי זה
   המסלול היחיד שבו הסוכן עושה משהו שהוא עצמו אמר שמזיק.
2. **מייצר 3 וריאציות** קריאייטיב דרך `solution_service` (אותו service של בעיות 1–2).
3. **מעלה אותן ל-Meta** דרך `optimization_push_service.push` — **בלי `optimization_session` חי, בלי
   `window_ends_at`, בלי מנוע-שלבים/cron**. ניסיון בודד (דפוס "מיני-session" של `rejection_fix`).
4. **מאשר ללקוח** שהווריאציות הועלו, עם תזכורת שלא צפוי שיפור.

**שורש טכני שה-PR הזה פותר:** `optimization_push_service.push` **מקבע** `window_hours=120` (שורה 475). כדי
לקבל "ניסיון בלי מחזור מדידה" צריך **לפרמטר** את `window_hours` (root, לא טלאי). ו-`generate_solution`
**זורק `StepsExhaustedError`** ל-`budget_mismatch` כי הוא חסר מ-`_STEP_PLANS` — צריך להוסיף שלב.

---

## 1. ההקשר מה-ROADMAP (מקור-אמת, ציטוט מילולי)

### חלק 6.2 — הודעת הסירוב (ROADMAP שורות 14875–14876)

> "הבנתי. היעד לכמות הלידים יישאר בהתאם לתקציב הנוכחי — כלומר כ-[expected_leads] לידים בחודש. אפשר טכנית
> לנסות שינויי קריאייטיב כדי לשפר מעט את העלות לליד, אבל אני חייב להיות כן: זה לא מומלץ כלל. הקמפיין שלך
> עובד תקין, ושינוי כרגע עלול דווקא לפגוע ביציבות שלו בלי סיבה מקצועית."
> `[הבנתי, משאיר כמו שזה / בכל זאת נסה קריאייטיב]`

### חלק 7 — התעקשות על קריאייטיב (ROADMAP שורות 14882–14891)

> ## חלק 7 — התעקשות על קריאייטיב (ניסיון חד-פעמי)
> הלקוח התעקש לנסות קריאייטיב נגד ההמלצה.
> 1. **תיעוד ההתעקשות (החלטה 8ג):** כתיבה ל-`budget_agreements` — סוג `creative_against_advice`, תאריך,
>    `cpl_before` (העלות לפני, לשליפה עתידית). **זה המסלול היחיד שבו הסוכן עושה משהו שהוא עצמו אמר שמזיק —
>    ולכן הכי צריך תיעוד.**
> 2. **יצירת 3 וריאציות:** קריאה ל-`solution_service` (מ-7.4ב — אותו service). 3 קופי/כותרות/זוויות.
> 3. **אישור + העלאה:** דרך `optimization_push_service` (מ-7.4ב — אותו דפוס אטומי). **אבל:** ללא פתיחת
>    סדרה (`optimization_session`), ללא `window_ends_at`, ללא מנוע שלבים. ניסיון בודד.
> 4. **אישור ללקוח:** "העליתי את הווריאציות החדשות. כפי שהסברתי, אני לא צופה שיפור משמעותי - אבל נקווה שאני טועה"

---

## 2. הכרעות עיצוב (החלטות מימוש — לאישור לפני קוד)

| # | נושא | הכרעה מומלצת | נימוק |
|---|------|-------------|-------|
| **D1** | ניסוח הצ'יפים | `נשאיר את המודעות כמו שהם` / `בכל זאת ננסה קריאייטיב חדש` | **המשתמש ביקש מפורשות** את הניסוח הזה (גובר על `[הבנתי, משאיר כמו שזה / בכל זאת נסה קריאייטיב]` שב-ROADMAP). |
| **D2** | "מיני-session" | פותחים `optimization_session` אמיתי עם `problem_type="budget_mismatch"`, ומיד סוגרים אותו ל-`done` עם `window_hours=None` | ה-ROADMAP כותב "ללא סדרה", אבל `solution_service.generate_solution` **דורש** `session_id`, ו-`push` דורש `session.status in (in_progress, success_monitoring)`. אין דרך לעקוף בלי שכפול קוד. הפתרון הוא בדיוק התקדים של `rejection_fix` (חלק 3): session שנסגר מיד, `window_ends_at=NULL` → ה-cron מסנן `IS NOT NULL` → אין מחזור מדידה. **זה ה-"ללא window/cron/שלבים" שה-ROADMAP התכוון אליו.** |
| **D3** | `window_hours` ב-`push` | **לפרמטר** `push(..., window_hours=optimization_service._WINDOW_HOURS)` ו-force_creative מעביר `None` | שורש, לא טלאי. הקורא היחיד הקיים (`optimization_approve_service`) לא מעביר → default 120 → אפס שינוי התנהגות. `finalize_push_action` כבר תומך ב-`None` (מחזיר `""`). חלופה (push ייעודי כמו rejection) = שכפול מיותר כי הסמנטיקה זהה ל-creative_refresh. |
| **D4** | סוג השלב | `_STEP_PLANS["budget_mismatch"] = {1: _CREATIVE_REFRESH}` (מיחזור הקבוע הקיים) | ה-action_type הוא `creative_refresh` (3 זוויות pain/dream/urgency, unique). **מיחזור הפרומפט הקיים** — אין צורך בפרומפט/action_type חדש. `step_plan(2,...)=None` → terminal אחרי שלב 1 (`advance_to_next_step` סוגר). |
| **D5** | מקור `cpl_before` | ה-CPL **החי בפועל** מ-`status.metrics["cost_per_action"]["value"]`; fallback ל-`market_cpl` אם אין מטריקה | ההגנה המשפטית החזקה ביותר: "ה-CPL שלך היה ₪32 — כבר מצוין; הקריאייטיב לא יכול לעזור". `market_cpl` הוא מרכז-טווח benchmark, לא העלות האמיתית של הקמפיין. בקמפיין "מעט פניות" יש בד"כ *מעט* לידים → CPL זמין. (קורא-מחקר המליץ market_cpl מטעמי זמינות; אנו מעדיפים אמת בפועל עם fallback.) |
| **D6** | אידמפוטנטיות | `open_session` ראשון (idempotent) → resume-check על action קיים → record+generate רק במסלול ראשון | מונע double-record / generate-מעל-pushing על double-click או transient-retry (כלל 9 + כלל 2). פירוט בסעיף 6. |
| **D7** | ניתוב `keep_ads` | צ'יפ "נשאיר את המודעות כמו שהם" = **frontend-only** → חזרה לתפריט (כמו "ביטול"). **לא** קורא ל-backend. | ה-`decline` כבר תועד בשלב הקודם (הוא מה שהפיק את ההודעה הזו). ניתוב חוזר ל-`decline` היה יוצר רשומת `declined` כפולה. |

---

## 3. ממצאי החקירה — גוצ'ות שחייבים לטפל בהן

1. **`generate_solution` זורק `StepsExhaustedError` ל-budget_mismatch.**
   `solution_service.generate_solution` (שורה 366–373) קורא `step_plan(step, problem_type)`; אם `None` →
   `raise StepsExhaustedError()`. `budget_mismatch` חסר מ-`_STEP_PLANS` (optimization_service.py:602–617).
   **חובה להוסיף `{1: _CREATIVE_REFRESH}`** אחרת הקריאה הראשונה נכשלת 409.

2. **`push` מקבע `window_hours=120` (שורה 474–475).** אין פרמטר. `finalize_push_action(window_hours=None)`
   כבר תומך (שורות 413–415: מדלג על `window_ends_at`; שורה 431: מחזיר `""` ולא `None`). **חובה לפרמטר את
   `push`** (D3). שים לב: שורה 477 `if window_ends_at is None:` — עם `None` מקבלים `""` (לא `None`) → ה-fallback
   לא רץ → מוחזר `window_ends_at=""`. ה-orchestrator מתעלם מהערך ממילא.

3. **`equality gate` ב-push (שורה 368): `len(variations) != len(live_old)` → 422.** ל-`creative_refresh`,
   `_normalize_variations` מייצר בדיוק 3 וריאציות (unique pain/dream/urgency). לכן הקמפיין חייב **3 מודעות
   חיות**. זו אותה הנחה כמו high_cpl/low_quality step 1 (לא חדשה ל-PR6). **לאמת בסנדבוקס** שקמפיין budget_mismatch
   נוצר עם 3 מודעות (אחרת gate ייכשל — ראה סעיף 8 בדיקה ידנית).

4. **אין `CHECK` על `optimization_actions.action_type`** (migration 0051:55 — `text not null` בלבד). לכן
   `creative_refresh` בסדרת budget_mismatch מתקבל ללא שינוי migration נוסף. **רק** ה-`problem_type` של
   `optimization_sessions` צריך הרחבה (migration 0104).

5. **`advance_to_next_step` סובלני ל-budget_mismatch.** (שורות 688–692) — אחרי שלב 1 מסתיים, קורא
   `step_plan(2, "budget_mismatch")=None` → מחזיר `None` (terminal). לא נדרש טיפול מיוחד.

6. **`open_session` אידמפוטנטי** (שורות 93–114, RPC + advisory lock + partial index) — קריאה חוזרת מחזירה
   את הסדרה הפתוחה הקיימת. בסיס ל-resume (D6).

7. **`finalize_push_action` לא נוגע ב-session status** (רק `optimization_actions`). **`push` גם לא** מעביר
   את הסדרה ל-`success_monitoring` (זה קורה ב-router של high_cpl). לכן force_creative חייב **לסגור מפורשות**
   את הסדרה ל-`done` (כמו `_close_rejection_session`), אחרת `has_open_session` נשאר True וחוסם.

---

## 4. עץ הזרימה (end-to-end)

```
diagnose_problem_4 (infeasible)
  └─ chips: [הגדל תקציב, השאר את התקציב הקיים]
        │
        ├─ "הגדל תקציב"  → preview → "מאשר"/"ביטול" → approve_budget   (PR3/5.1 — קיים)
        │
        └─ "השאר את התקציב הקיים" → decline_budget                     (PR4 — קיים; ההודעה+צ'יפים משתנים ב-PR6)
              │  ├─ record: budget_increase_declined  (קיים)
              │  └─ message: חלק 6.2 + chips: [נשאיר את המודעות כמו שהם, בכל זאת ננסה קריאייטיב חדש]  ← PR6
              │
              ├─ "נשאיר את המודעות כמו שהם"  → frontend בלבד → חזרה לתפריט  (D7 — אין backend)
              │
              └─ "בכל זאת ננסה קריאייטיב חדש" → force_creative()            ← לב ה-PR6
                    1. cpl_before = live CPL (fallback market_cpl)          (D5)
                    2. open_session("budget_mismatch")  [idempotent]        (D2)
                    3. resume-check (get_active_push_action)                (D6)
                       └─ first attempt: record(creative_against_advice) → generate_solution (3 וריאציות)
                    4. push(session, variations, step_number=1, window_hours=None)   (D3)
                    5. set_session_status(done)                             (סעיף 3.7)
                    6. message: "העליתי את הווריאציות החדשות..."
```

**לולאה reactive (קיימת מ-PR4):** `_format_reactive_reminder` כבר מטפל ב-`creative_against_advice`
(agent_orchestrator.py:783–788) — מציג "בפעם הקודמת ניסינו שינוי קריאייטיב... (העלות לפנייה אז הייתה ₪{cb})".
**אין שינוי נדרש שם** — רק לוודא ש-`cpl_before` נכתב נכון (D5).

---

## 5. השינויים — קובץ-אחר-קובץ

### חלק A — Migration + מנוע שלבים (תשתית)

#### A1. `supabase/migrations/0104_budget_mismatch_problem_type.sql` (חדש)

```sql
-- 0104_budget_mismatch_problem_type.sql — PR6 (force-creative): הוספת budget_mismatch ל-problem_type.
-- מרחיב את ה-CHECK של optimization_sessions.problem_type כדי לאפשר מיני-session לניסיון קריאייטיב חד-פעמי.
-- שלב 1 = creative_refresh; שלב 2+ = None (terminal). idempotent. מוחל ב-deploy.

alter table public.optimization_sessions drop constraint if exists opt_sessions_problem_type_check;
alter table public.optimization_sessions add constraint opt_sessions_problem_type_check
  check (problem_type in (
    'high_cpl', 'low_quality_leads', 'meta_rejection', 'budget_mismatch'
  ));
```

> **לאמת:** שם ה-constraint. ב-0051 ה-CHECK הוא inline (Postgres נותן שם אוטומטי). **לפני כתיבת ה-migration**
> יש לבדוק את השם בפועל ב-DB (`opt_sessions_problem_type_check` הוא הניחוש; ייתכן
> `optimization_sessions_problem_type_check`). דפוס ה-`DROP IF EXISTS` + `ADD` הוא idempotent (כמו ב-0024/0053).
> אם השם שונה — להתאים, ולהוסיף `drop ... if exists` לשני השמות לבטיחות.

#### A2. `app/services/optimization_service.py` — `_STEP_PLANS` (שורה ~616)

```python
_STEP_PLANS: dict[str, dict[int, dict]] = {
    "high_cpl": { ... },           # קיים
    "low_quality_leads": { ... },  # קיים
    "budget_mismatch": {           # ← PR6: ניסיון קריאייטיב חד-פעמי (חלק 7). שלב 1 בלבד → terminal.
        1: _CREATIVE_REFRESH,      # action_type=creative_refresh, angles=(pain,dream,urgency), unique=True
    },
}
```

> מיחזור `_CREATIVE_REFRESH` (D4) → אין action_type/פרומפט חדשים. `step_plan(2,"budget_mismatch")=None`.

### חלק B — פרמטור `window_hours` ב-push (השורש)

#### B1. `app/services/optimization_push_service.py` — `push` (שורות 322–328, 470–476)

```python
async def push(
    session: dict,
    user_id: str,
    variations: list[dict],
    *,
    step_number: int,
    window_hours: int | None = optimization_service._WINDOW_HOURS,  # ← PR6: None = ניסיון בלי מחזור מדידה
) -> dict:
    ...
    step = "finalize"
    window_ends_at = await optimization_service.finalize_push_action(
        action_id, window_hours=window_hours,   # ← היה: window_hours=optimization_service._WINDOW_HOURS
    )
```

> - default = `_WINDOW_HOURS` (120) → הקורא הקיים `optimization_approve_service.push(...)` ללא שינוי.
> - force_creative מעביר `window_hours=None` → `finalize` מחזיר `""` → שורה 477 `if window_ends_at is None`
>   = False → אין fallback → `window_ends_at=""` ב-dict המוחזר (נזנח).
> - **לעדכן את ה-docstring + ההערה בשורה 470–472** ("window=+120h לשני סוגי-הבעיה") לציין את מקרה ה-`None`
>   (force_creative — בלי מחזור מדידה, כמו rejection_fix).

### חלק C — orchestrator + צ'יפים + הודעת decline

#### C1. `app/services/agent_orchestrator.py` — imports (שורות 8–34)

```python
import dataclasses                                  # ל-asdict על SolutionVariation
from app.services import optimization_push_service  # ← חדש
from app.services import solution_service           # ← חדש
from app.core.transient import TransientError       # ← לסיווג transient (כלל 11)
# optimization_service, benchmark_service, budget_agreement_service, meta_client_factory — כבר מיובאים
```

> **לאמת:** האם `_guard` (שורות 192–211) צריך להוסיף את שגיאות `solution_service`/`optimization_push_service`
> ל-tuple שלו. כרגע `_guard` ממפה lock/optimization/quiz/budget_agreement. הקריאות ל-push/generate ב-force_creative
> **לא** עוטפות ב-`_guard` (יש להן טיפול-שגיאות ייעודי — ראה C3), אבל `open_session`/`set_session_status`/`record`
> כן (הן optimization/budget_agreement — כבר מכוסות).

#### C2. constants חדשים (ליד שורות 471–487)

```python
_CHIP_KEEP_ADS = ("keep_ads", "נשאיר את המודעות כמו שהם")                # D1, D7 — frontend-only
_CHIP_FORCE_CREATIVE = ("force_creative", "בכל זאת ננסה קריאייטיב חדש")  # D1 — → endpoint
_FORCE_CREATIVE_DONE_MSG = (
    "העליתי את הווריאציות החדשות. כפי שהסברתי, אני לא צופה שיפור משמעותי — אבל נקווה שאני טועה."
)
```

#### C3. החלפת `_format_budget_declined_message` (שורות 759–765) להודעת חלק 6.2

```python
def _format_budget_declined_message(expected_leads: int | None) -> str:
    """הודעת הסירוב (ROADMAP חלק 6.2) — מסבירה את המשמעות הכמותית + פותחת את אפשרות הקריאייטיב נגד המלצה.
    (PR6: הוחלפה הגרסה המצומצמת של ה-MVP; כעת מציעה את מסלול force-creative.)"""
    leads = agent_service._fmt_num(expected_leads) if expected_leads is not None else "המספר הנוכחי"
    return (
        f"הבנתי. היעד לכמות הלידים יישאר בהתאם לתקציב הנוכחי — כלומר כ-{leads} לידים בחודש. "
        "אפשר טכנית לנסות שינויי קריאייטיב כדי לשפר מעט את העלות לליד, אבל אני חייב להיות כן: "
        "זה לא מומלץ כלל. הקמפיין שלך עובד תקין, ושינוי כרגע עלול דווקא לפגוע ביציבות שלו בלי סיבה מקצועית."
    )
```

#### C4. `decline_budget` (שורה 814) — שינוי הצ'יפים המוחזרים

```python
# היה: return AgentDecision(type="budget_options", message=msg, chips=[_CHIP_BACK])
return AgentDecision(
    type="budget_options", message=msg, chips=[_CHIP_KEEP_ADS, _CHIP_FORCE_CREATIVE],
)
```

#### C5. הפונקציה החדשה `force_creative` (אחרי `decline_budget`)

```python
async def force_creative(user_id: str, conversation_id: str) -> AgentDecision:
    """מסלול ג' (בעיה 4, ROADMAP חלק 7): הלקוח התעקש על קריאייטיב נגד ההמלצה — ניסיון חד-פעמי.
    תיעוד creative_against_advice (reserve-first — ההגנה המשפטית הקריטית ביותר) → מיני-session (בלי
    window/cron/מנוע-שלבים, דפוס rejection_fix) → 3 וריאציות (solution_service) → push בלי window
    (window_hours=None) → סגירת הסדרה ל-done. transient → 503 (resume); permanent → rollback + הודעה."""
    campaign_id = await agent_service.get_owned_campaign_id(user_id, conversation_id)
    camp = await agent_service._campaign_for_chat(user_id, campaign_id)
    industry = (camp or {}).get("industry")
    market_cpl = benchmark_service.market_cpl_for_industry(industry)

    # cpl_before = ה-CPL החי בפועל (D5 — ההגנה החזקה: "ה-CPL שלך כבר היה תקין"); fallback ל-market_cpl
    status = await agent_service.get_status_for_conversation(user_id, conversation_id)
    cpl_before = None
    if status is not None and status.metrics is not None:
        cpl_before = status.metrics.get("cost_per_action", {}).get("value")
    if cpl_before is None:
        cpl_before = market_cpl

    # מיני-session idempotent (resume אם כבר פתוחה) — DB-only, אין Meta, אין record עדיין.
    session = await _guard(optimization_service.open_session(
        campaign_id, user_id, _PROBLEM_BUDGET_MISMATCH, {"cpl_before": cpl_before},
    ))
    session_id = str(session.get("id"))

    # resume-check (D6 — כלל 9/2): action קיים ⇒ record כבר נכתב (record→generate). אל תשכפל.
    existing = await _guard(optimization_service.get_active_push_action(session_id, 1))
    agreement_id: str | None = None
    if isinstance(existing, dict):
        ps = existing.get("push_status")
        if ps in ("pushing", "pushed"):
            variations = list(existing.get("variations") or [])  # push כבר התחיל — מקור-אמת מאוחסן
        else:
            solution = await solution_service.generate_solution(session_id, user_id, campaign_id)
            variations = [dataclasses.asdict(v) for v in solution.variations]
    else:
        # מסלול ראשון: record (reserve-first מול Meta — הצעד החיצוני הבלתי-הפיך הראשון הוא ב-push) → generate.
        agreement_id = await _guard(budget_agreement_service.record(
            campaign_id=campaign_id, user_id=user_id, agreement_type="creative_against_advice",
            market_cpl=market_cpl, cpl_before=cpl_before,
        ))
        solution = await solution_service.generate_solution(session_id, user_id, campaign_id)
        variations = [dataclasses.asdict(v) for v in solution.variations]

    # push בלי window (resumable; idempotent על double-submit דרך open_push_action CAS).
    try:
        await optimization_push_service.push(
            session, user_id, variations, step_number=1, window_hours=None,
        )
    except Exception as exc:
        if isinstance(exc, TransientError):
            # transient — push נשאר 'pushing' (resume ב-retry). record+session נשמרים. (כלל 11.)
            raise AgentTransientError() from exc
        # permanent — לא בוצע (push כבר עשה rollback ל-Meta entities). rollback התיעוד + סגירת session ל-failed.
        if agreement_id is not None:
            await _safe_rollback(agreement_id)
        await _guard(optimization_service.set_session_status(session_id, "failed"))
        capture_exception(
            exc, conversation_id=conversation_id, campaign_id=campaign_id,
            problem_type=_PROBLEM_BUDGET_MISMATCH, where="force_creative",
        )
        await agent_service.save_assistant_message(user_id, conversation_id, _BUDGET_UPDATE_FAILED_MSG)
        return AgentDecision(type="unavailable", message=_BUDGET_UPDATE_FAILED_MSG, chips=[_CHIP_BACK])

    # סגירת הסדרה ל-done (אין window/cron שיסגור; כמו rejection_fix). idempotent.
    await _guard(optimization_service.set_session_status(session_id, "done"))

    await agent_service.save_assistant_message(user_id, conversation_id, _FORCE_CREATIVE_DONE_MSG)
    return AgentDecision(type="budget_options", message=_FORCE_CREATIVE_DONE_MSG, chips=[_CHIP_BACK])
```

> **נקודות לאימות בזמן המימוש:**
> - **`get_active_push_action` — מה ה-filter?** האם הוא מחזיר action עם `push_status=NULL` (proposal שנוצר
>   ע"י `generate` לפני push)? אם הוא מחזיר רק `pushing`/`pushed` — ה-resume-check יפספס מצב "generate הסתיים,
>   push לא התחיל" → על retry יווצר record כפול (benign — תקדים `approve_budget`: "record נוסף ב-retry —
>   תיעוד עודף, לא נזק"). אם זה לא מקובל, להשתמש ב-`get_last_action(session_id)` (מחזיר את ה-action האחרון
>   ללא תלות ב-push_status) לבדיקת ה-resume.
> - **`generate_solution` מעל action קיים:** לוודא ש-`_persist_proposal` לא דורס action ב-`pushing`. ב-resume
>   אנחנו *לא* קוראים ל-generate כש-`ps in (pushing, pushed)` בדיוק כדי למנוע זאת.
> - **חתימות שגיאות `solution_service`:** ה-`except Exception + isinstance(TransientError)` (כלל 11) תופס את
>   ה-transient של *שני* ה-services בלי tuple ידני. `StepsExhaustedError` לא אמור לקרות (הוספנו את ה-step),
>   אך אם כן — הוא permanent → rollback + הודעה. **לאמת** ש-`*UnavailableError` של solution/push יורשים
>   `TransientError` (gate: `tests/test_transient_mixin.py`).

### חלק D — endpoint + sandbox

#### D1. `app/routers/agent.py` — endpoint חדש (אחרי שורה 375, ליד budget/decline)

```python
@router.post("/me/agent/conversations/{conversation_id}/budget/force-creative",
             response_model=DiagnoseResponse)
async def budget_force_creative(
    conversation_id: str, user: UserPublic = Depends(get_current_user),
) -> DiagnoseResponse:
    try:
        decision = await agent_orchestrator.force_creative(user.id, conversation_id)
    except AgentServiceError as exc:
        raise _to_http_error(exc)
    return _decision_to_response(decision)   # אותו wrapper כמו preview/approve/decline
```

> להשתמש באותו דפוס-wrapper של 3 ה-endpoints הקיימים (budget/preview|approve|decline, שורות 311–375).
> `AgentTransientError` → 503, `AgentUnavailableError` → 503, וכו' דרך `_to_http_error` הקיים.

#### D2. `app/services/sandbox_service.py` — `run_budget_action` (שורה ~315)

```python
elif action == "force-creative":
    decision = await agent_orchestrator.force_creative(user_id, conversation_id)
```

> `app/routers/admin/sandbox.py` (POST `/campaigns/{id}/budget/{action}`) **לא משתנה** — הוא גנרי (כל
> string ל-`{action}`). `app/web/api.js` `budgetAction()` גם גנרי — **לא משתנה**.

### חלק E — frontend routing

#### E1. `app/web/index.html` — `_chatDispatch` (שורות 2527–2530)

```javascript
// שינוי label של צ'יפ הסירוב + הוספת ניתוב force-creative:
if(text==='נשאיר את המודעות כמו שהם'){ _renderProblemMenu(); return; }          // D7 — frontend-only (חזרה לתפריט)
if(text==='בכל זאת ננסה קריאייטיב חדש'){ runBudgetAction('force-creative'); return; }  // ← חדש
```

> **שים לב:** ה-label של "השאר את התקציב הקיים" (שמפעיל `decline`) **נשאר** — הוא הצ'יפ שמופיע ב-diagnose
> (לפני decline). הצ'יפ החדש "נשאיר את המודעות כמו שהם" מופיע **אחרי** decline (בהודעת חלק 6.2). שניהם
> מנותבים: "השאר את התקציב הקיים" → `decline`; "נשאיר את המודעות כמו שהם" → תפריט.

#### E2. `app/templates/admin/sandbox.html` — `onChip` (שורות ~162–165)

```javascript
if (value === "keep_ads") { renderProblemMenu(); return; }                 // D7
if (value === "force_creative") { runBudgetAction("force-creative"); return; }  // ← חדש
```

> בסנדבוקס ה-onChip מנתב לפי **chip id** (לא label). להוסיף את שני ה-id-ים. (ה-`keep_budget`→`decline`
> הקיים נשאר — הוא צ'יפ ה-diagnose.)

---

## 6. טיפול בכשלים ואידמפוטנטיות (החלק הקריטי)

| מצב | התנהגות | נימוק |
|-----|---------|-------|
| **double-click / transient retry** | `open_session` resume (idempotent) → resume-check מוצא action קיים → **דילוג** על record+generate → push resume (CAS) | מונע double-record, double-generate, ו-double-spend ב-Meta. (D6, כלל 9.) |
| **transient ב-push (503)** | `raise AgentTransientError` — **בלי** rollback. push נשאר `pushing`, session+record נשמרים → retry מ-resume | כלל 11 + כלל 12ב (transient = retry, לא "אבוד"). |
| **transient ב-generate** | אותו `except` (isinstance TransientError) → 503. record כבר נכתב (reserve-first), session פתוחה → retry | reserve-first שומר את ההגנה המשפטית גם אם generate מתעכב. |
| **permanent ב-push/generate** | `_safe_rollback(agreement_id)` + `set_session_status(failed)` + `capture_exception` + הודעה גנרית | push כבר עשה rollback ל-Meta entities שלו (לא בוצע) → התיעוד `creative_against_advice` כוזב → מוחק (כלל 12ד). session→failed כדי ש-`has_open_session` ישוחרר. |
| **micro-race (2 מסלולים-ראשונים בו-זמנית)** | benign double-record; advisory-lock ב-`open_session` + CAS ב-`open_push_action` מונעים double-**push** | רשומות `creative_against_advice` כפולות = תיעוד עודף (לא נזק); `_format_reactive_reminder` משתמש ב-`fetch_last` (האחרונה). תקדים: `approve_budget`. |

**עקרון מנחה:** ה-record הוא reserve-first (לפני הצעד החיצוני הבלתי-הפיך הראשון, שהוא ב-`push`). אם push לא
בוצע (permanent) — מוחקים את ה-record (אסור להשאיר תיעוד "ניסינו קריאייטיב" אם לא נוצר קריאייטיב, אחרת
ה-reactive reminder ישקר). אם push בוצע (pushed) — ה-record נשאר.

---

## 7. טסטים (`tests/test_agent_diagnose.py` + נלווים)

הרחבת helper קיים `_patch_p4`/`_patch_approve` ל-`_patch_force_creative` (mock ל-`solution_service.generate_solution`,
`optimization_push_service.push`, `optimization_service.open_session`/`get_active_push_action`/`set_session_status`,
`budget_agreement_service.record`/`delete_by_id`, `get_status_for_conversation`).

1. **happy path:** `force_creative` → `record(creative_against_advice, cpl_before=<live>)` נקרא **לפני** push;
   `open_session("budget_mismatch")`; `generate_solution` מחזיר 3 וריאציות; `push` נקרא עם `window_hours=None`
   ו-`step_number=1`; `set_session_status(done)`; ההודעה = `_FORCE_CREATIVE_DONE_MSG`; chips=[back].
2. **cpl_before = live CPL** כשיש מטריקה; **= market_cpl** כש-`status.metrics is None` (D5 fallback).
3. **decline message** = חלק 6.2 המלא; chips=`[keep_ads, force_creative]` (לא `[back]`).
4. **transient push** (`PushUnavailableError`) → `AgentTransientError`; **אין** `delete_by_id`; **אין**
   `set_session_status(failed)` (session נשאר פתוח ל-resume).
5. **permanent push** (`PushValidationError`/`UnexpectedPushError`) → `_safe_rollback` (delete_by_id) +
   `set_session_status(failed)` + הודעת `_BUDGET_UPDATE_FAILED_MSG`.
6. **resume (action קיים `pushing`)** → **לא** קורא ל-`record` ו-**לא** ל-`generate_solution`; משתמש ב-variations
   המאוחסנות; push resume.
7. **window=None דרך push:** טסט יחידה ל-`optimization_push_service.push(..., window_hours=None)` → `finalize_push_action`
   נקרא עם `window_hours=None` → התוצאה `window_ends_at=""` (לא session פעיל/cron). וגם: default (ללא הפרמטר)
   → `_WINDOW_HOURS` (רגרסיה לקורא הקיים).
8. **`_STEP_PLANS["budget_mismatch"][1]`** קיים ו-`step_plan(1,"budget_mismatch")==_CREATIVE_REFRESH`;
   `step_plan(2,"budget_mismatch") is None` (terminal).
9. **reactive** (קיים, לוודא רגרסיה): `fetch_last`=`creative_against_advice` → `_format_reactive_reminder`
   מציג `cpl_before`.
10. **sandbox** (`tests/test_sandbox.py`): `run_budget_action("force-creative")` מנתב ל-`force_creative`.
11. **migration smoke** (אם יש דפוס בריפו): `problem_type="budget_mismatch"` מתקבל ל-`optimization_sessions`.
12. **`test_transient_mixin.py`** (CI gate) — לוודא שאינו נשבר (לא הוספנו `*UnavailableError` חדש).
13. **`import app.main`** עובר; `test_web.py` — צ'יפ "בכל זאת ננסה קריאייטיב חדש" מנותב.

---

## 8. אימות (Verification)

1. `python -m pytest -q` ירוק + `python -c "import app.main"`.
2. **ידני בסנדבוקס:** קמפיין budget_mismatch (יעד גבוה) → "מעט פניות" → diagnose (infeasible) →
   "השאר את התקציב הקיים" → decline (הודעת 6.2 + 2 צ'יפים) → "בכל זאת ננסה קריאייטיב חדש" →
   force_creative → לוודא: (א) רשומת `creative_against_advice` + `cpl_before` נכתבה; (ב) 3 וריאציות
   הועלו ב-`SandboxMetaClient` (no-op) **בלי** `optimization_session` חי / `window_ends_at`; (ג) הסדרה
   `done`; (ד) diagnose שוב → reactive מציג "בפעם הקודמת ניסינו קריאייטיב".
3. **equality gate (ממצא 3):** לוודא שקמפיין הסנדבוקס נוצר עם **3 מודעות חיות** (אחרת push → 422). אם
   `create_sandbox_campaign` יוצר מספר שונה — לתעד/להתאים.
4. **migration 0104:** `budget_mismatch` מתקבל; idempotent (הרצה כפולה לא נכשלת); שם constraint נכון.
5. **רגרסיה:** high_cpl/low_quality push עדיין עם window 120 (default לא השתנה).

---

## 9. מסמכים לעדכון (חלק מ-PR6)

- **`docs/ROADMAP.md`** — סימון ✅ לחלק 7 ב-Session 7.6.5 (חלק 6 כבר נוגע ב-PR4/5; להשלים את 7).
- **`docs/sandbox-agent-prompts.md`** — מסלול force-creative במודול budget_mismatch (אם רלוונטי).
- **`docs/frontend-integration.md`** — צ'יפ force-creative + keep_ads במסך בעיה 4.
- **`docs/backend-gaps.md`** — לסגור פער אם נרשם ל-force-creative.
- **`docs/SETUP_CHECKLIST.md`** — **לא צפוי env חדש** (לאמת: force_creative לא מוסיף תלות חיצונית).
- **המסמך הזה** — לעדכן סטטוס ל-"מומש" + commit hash בסוף ה-PR.

---

## 10. סדר מימוש מומלץ (כל צעד ירוק לפני הבא)

1. **A** (migration 0104 + `_STEP_PLANS`) → טסט `step_plan` ירוק.
2. **B** (פרמטור `window_hours` ב-push) → טסט יחידה window=None + רגרסיה default.
3. **C** (orchestrator + צ'יפים + decline message) → טסטי force_creative (happy/transient/permanent/resume).
4. **D** (endpoint + sandbox) → טסט sandbox routing.
5. **E** (frontend) → טסט web.
6. **F+G** (טסטים מלאים + docs) → חבילה ירוקה → commit + push.

**Commit (עברית):** `feat(agent): בעיה 4 — force-creative (ניסיון קריאייטיב חד-פעמי נגד המלצה, חלק 7)`
עם trailers נדרשים.

---

## 11. רשימת קבצים (סיכום)

| קובץ | שינוי |
|------|-------|
| `supabase/migrations/0104_budget_mismatch_problem_type.sql` | **חדש** — CHECK + budget_mismatch |
| `app/services/optimization_service.py` | `_STEP_PLANS["budget_mismatch"]={1:_CREATIVE_REFRESH}` |
| `app/services/optimization_push_service.py` | `push(..., window_hours=_WINDOW_HOURS)` param + docstring |
| `app/services/agent_orchestrator.py` | imports; `_CHIP_KEEP_ADS`/`_CHIP_FORCE_CREATIVE`/`_FORCE_CREATIVE_DONE_MSG`; `_format_budget_declined_message` (6.2); `decline_budget` chips; **`force_creative`** |
| `app/routers/agent.py` | endpoint `budget/force-creative` |
| `app/services/sandbox_service.py` | `run_budget_action` ↦ `"force-creative"` |
| `app/web/index.html` | `_chatDispatch` — keep_ads (תפריט) + force-creative |
| `app/templates/admin/sandbox.html` | `onChip` — `keep_ads` + `force_creative` |
| `tests/test_agent_diagnose.py` (+ `test_sandbox.py`, `test_web.py`, יחידה ל-push) | טסטים (סעיף 7) |
| `docs/ROADMAP.md`, `frontend-integration.md`, `sandbox-agent-prompts.md`, וזה | docs |
| **ללא שינוי** | `app/routers/admin/sandbox.py`, `app/web/api.js` (גנריים) |



::: note

> ### מוסיף את הבלוק הזה, אולי זה חוסר במסמך

המנגנון שסוגר את הכל. **לא cron — תגובתי.**

כשהלקוח **חוזר ומתלונן שוב** ("מעט פניות") — הקוד, לפני האבחון:
1. **שולף את התיעוד האחרון** מ-`budget_agreements` לקמפיין הזה.
2. **מציג אותו ללקוח** לפי הסוג:
   - `budget_increase_declined` → "כפי שבחרת ב-[תאריך], התקציב נשאר על ₪[Y], ולכן כמות הלידים מוגבלת לכ-[expected_leads] בחודש. רוצה לשקול שוב להעלות תקציב?"
   - `creative_against_advice` → "ב-[תאריך] התעקשת לנסות שינוי קריאייטיב נגד המלצתי. כפי שהזהרתי, העלות לליד הייתה ₪[cpl_before] לפני, ועכשיו היא ₪[cpl_now]. הבעיה לא הייתה בקריאייטיב אלא בפער התקציב."
   - `budget_increase_approved` → "ב-[תאריך] העלינו את התקציב ל-₪[W]. אם עדיין מעט לידים, בוא נבדוק שוב את המספרים."
3. **חוזר לאבחון (חלק 3)** — מחשב מחדש את ההיתכנות עם הנתונים הנוכחיים (אולי התקציב השתנה, אולי עלות השוק השתנתה).

> **התיעוד הוא הזיכרון.** הלולאה לא צריכה cron או מעקב — בכל פעם שהלקוח חוזר, הקוד שולף את ההיסטוריה ומאבחן מחדש. זה reactive, פשוט, ועמיד.

:::

---

## 12. הערות מימוש (סטיות מהתוכנית המקורית)

המימוש בוצע לפי A→B→C→D→E→F→G. שתי תוספות מעבר לתוכנית המקורית, ושינוי-ניסוח אחד:

1. **לולאת reactive הועשרה** (מבטל את "אין שינוי נדרש" שבסעיף 4) — לפי הבלוק שהוסף בסוף המסמך (סעיף 11).
   `_format_reactive_reminder(last_agreement, current_cpl=None)` כעת מציג **תאריך** ההחלטה (`created_at` →
   `_fmt_agreement_date`, DD/MM/YYYY, best-effort), את ערכי-המצב שתועדו (Y/W/expected_leads), וב-
   `creative_against_advice` גם **השוואת cpl_before מול cpl_now**. ה-`cpl_now` נשלף ב-`diagnose_problem_4`
   **רק** כשהרשומה האחרונה היא `creative_against_advice` (שליפת status חיה, best-effort — `except
   AgentServiceError` → נשמט בלי לשבור את האבחון). קבצים: `agent_orchestrator.py` (`_fmt_agreement_date`
   חדש, `_format_reactive_reminder`, `diagnose_problem_4`).

2. **gating ל-force-creative בפרונטד** (מעבר לתוכנית — בעקבות ממצא Cursor High על `מאשר`): הצ'יפ "בכל זאת
   ננסה קריאייטיב חדש" יוצר מודעות (= spend), ולכן מנותב דרך דגל `CHAT.creativePending` שעולה רק כשהשרת
   החזיר את צ'יפ `force_creative` (אחרי decline), ונצרך בראש כל dispatch — בדיוק כמו `CHAT.budgetConfirm`
   של `מאשר`. טקסט מוקלד/מודלף → `_freeText` (לא spend). (sandbox.html בטוח — ניתוב לפי chip-id.)

3. **ניסוח `_FORCE_CREATIVE_DONE_MSG`** עודכן לפי עריכת המשתמש: "…אבל נקווה שאני טועה." (במקום "הקמפיין כבר
   היה תקין").

**resume-check (D6):** `get_active_push_action` מסנן `push_status IN (pushing, push_rollback_pending,
pushed)` (אומת בקוד) — לכן action פעיל ⇒ push התחיל ⇒ record כבר נכתב. record נכתב **רק** כשאין action
פעיל; אחרת resume עם ה-variations המאוחסנות (בלי record/generate כפולים).

**אימות:** `python -m pytest -q` → **2402 passed** (11 טסטים חדשים: force_creative happy/transient/permanent/
resume/fallback, decline-chips, step_plan, push-window_hours-param, reactive date+cpl_now ×2, sandbox
force-creative). `import app.main` ✓.
