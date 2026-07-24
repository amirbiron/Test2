# זרימת "עלות לפנייה גבוהה" (Problem 1 / `high_cpl`) — פרוטוקול האופטימיזציה המלא

> מסמך תפעולי מפורט מקצה-לקצה של הבעיה **הראשונה** (והמורכבת ביותר) בפרוטוקול-הסוכן: הלקוח מתלונן בצ'אט
> ש**עלות-הפנייה גבוהה**. המסמך מכסה **רק** את התלונה הזו: מה הסוכן עונה, מה חוקי-הברזל שלו, איך הוא מאבחן,
> מציע, מבצע, מודד, ומתקדם. ממפה עיצוב↔קוד: כל שלב עם הקובץ, הפונקציה, ה-endpoint, הנוסח, ומעברי-המצב.
>
> **מקורות-אמת:** `docs/ROADMAP.md` (Session 7.3.5 + 7.4א/ב/ג) = העיצוב; הקוד עצמו = המימוש.
> **מסמכים אחים:** `docs/budget-mismatch-flow.md` (בעיה 4), `docs/low-quality-leads-filter-redesign.md` (בעיה 2).

---

## 0. TL;DR — כל הזרימה במבט אחד

```
צ'יפ "עלות לפנייה גבוהה"  →  diagnose_problem_1(user, conv, override=false)
  │
  ├─ שער 1 — מטריקות חיות:  CPL = spend/leads מ-Meta Insights (חי, לא מהשאלון)
  │     אין נתונים / אין CPL  → type=unavailable (הודעה רכה, בלי צ'יפ)
  │
  ├─ שער 2 — Lock:  סדרת high_cpl פתוחה + action בחלון-מדידה חי (window_ends_at > now)
  │     נעול (ולא override)  → type=lock  ("אנחנו באמצע חלון הבדיקה…")  chips:[חזרה לתפריט]
  │
  ├─ שער 3 — חוק-הברזל (benchmark):  classify_cpl(industry, cpl) ∈ {amazing, average, expensive}
  │     amazing/average (ולא override)  → type=benchmark_stop  (template, בלי LLM! "המספרים שלך מעולים…")
  │                                        chips:[אני מקבל את ההמלצה / אני בכל זאת רוצה לשנות]
  │
  └─ אחרת (expensive / unknown / override)  →  Scenario B:
        open_session("high_cpl")  →  _phrase (LLM מנסח את המספרים)  →  type=diagnosis + session_id
        → הפרונטד קורא אוטומטית ל-propose
              propose → generate_solution: 3 וריאציות קופי (LLM) + תמונות eager
                        (הזוויות נקבעות דטרמיניסטית ע"י מנוע-השלבים, ה-LLM רק כותב — חוק 7)
              → הצעה בעמוד view-action (3 כרטיסים + תמונות + עריכה/רענון)
              → "✅ אשר והעלה לקמפיין"  →  execute_approval  →  push (creative-swap ל-Meta):
                    3 creatives+ads חדשים ACTIVE  +  השהיית 3 הישנים  (reserve-first + rollback)
                    → finalize: push_status=pushed + window_ends_at = now + 120 שעות
              → cron ניטור (שעתי) מודד אחרי 120ש':  CPL חדש < X?  (X = ה-CPL בתחילת הסדרה)
                    שיפור → חלון שני (120ש' נוספות) → שיפור שוב → סדרה done ✅ (SERIES_RESOLVED)
                    אין-שיפור → action done, השלב מתקדם (STEP_ADVANCED) בפעם הבאה שהלקוח חוזר
              → מנוע-השלבים (4 שלבים):  1 רענון-קופי → 2 social_proof → 3 authority → 4 שינוי-הצעה → סוף
```

**עקרונות-הליבה (החוקים של הסוכן):**
1. **חוק 7 — הכל דטרמיניסטי, ה-LLM רק מנסח.** הקוד מחשב את הסיווג (benchmark), מחליט אם לעצור, ובוחר את הזווית של כל שלב. ה-LLM **לא** מעריך אם העלות גבוהה ולא בוחר פעולה — הוא מקבל payload מוכן ומנסח בעברית.
2. **חוק-הברזל (benchmark-stop).** אם ה-CPL כבר טוב (amazing/average) — הסוכן **מסרב לשנות** ומסביר שזה הימור מסוכן. **לא פותח סדרה, לא קורא ל-LLM** (template קבוע).
3. **Lock — סדרה אחת בחלון.** בזמן חלון-מדידה חי (120ש') אי-אפשר להתחיל שינוי חדש לאותה בעיה — "שווה לחכות לתוצאה".
4. **חלון-מדידה 120 שעות (5 ימים).** כל שינוי נמדד מול ה-CPL ההתחלתי; שיפור = `cpl < X` **מוחלט** (בלי מרווח).
5. **reserve-first + rollback.** יוצרים את המודעות החדשות ב-Meta, ורק **בסוף** משהים את הישנות; כשל → מחיקת החדשות + החזרת הישנות (או escalate אם ה-rollback עצמו נכשל).
6. **הזמן = הזיכרון.** אין דחיפה יזומה של הלקוח; ה-cron מודד ומתקדם, והלקוח חוזר לצ'אט לפי ההתראה.

---

## 1. כניסה — הצ'יפ "עלות לפנייה גבוהה"

| שלב | קוד |
|-----|-----|
| מיפוי label→problem_type | `index.html:2549` — `_PROBLEM_PT = {'עלות לפנייה גבוהה':'high_cpl', ...}` |
| תפריט הבעיות | `index.html:2600-2601` — `probs=['עלות לפנייה גבוהה','פניות לא איכותיות','מעט פניות']` |
| לחיצה | `_chatDispatch` (`:2631-2632`) → `CHAT.problemType='high_cpl'` → `runDiagnose()` (`:2636`) |
| api.js | `diagnose(conversationId, problemType, opts)` (`api.js:231-241`) |
| Endpoint | `POST /me/agent/conversations/{id}/diagnose`, גוף `{problem_type:'high_cpl', override?}` |
| ניתוב שרת | `routers/agent.py:266-291` → `else: agent_orchestrator.diagnose_problem_1(user.id, conversation_id, body.override)` |

> תווית ה-review/action שונה מהצ'יפ: `_PROBLEM_HE['high_cpl']='עלות לליד גבוהה'` (`index.html:3693`) מול צ'יפ-הכניסה `'עלות לפנייה גבוהה'`.

---

## 2. האבחון — `diagnose_problem_1` (3 שערים + Scenario B)

**פונקציה:** `diagnose_problem_1(user_id, conversation_id, override=False) -> AgentDecision` — `agent_orchestrator.py:246-308`. סדר דטרמיניסטי (חוק 7): מטריקות → Lock → benchmark → היסטוריה → ניסוח.

צ'יפים קבועים (`:65-67`): `_CHIP_BACK=("back_to_menu","חזרה לתפריט")`, `_CHIP_ACCEPT=("accept_recommendation","אני מקבל את ההמלצה")`, `_CHIP_OVERRIDE=("override_change","אני בכל זאת רוצה לשנות")`.

### 2.1 שער 1 — מטריקות חיות (CPL מ-Meta) — `:252-266`
`status = agent_service.get_status_for_conversation(...)`. ה-CPL **החי** נשלף מ-Meta Insights דרך `agent_service.get_campaign_status_for_agent` (`:225-257`) → `meta_client_factory.get_insights_client` → `get_campaign_insights` → `ci.cpl` (= `spend/leads`, `None` כשאין לידים). נחשף כ-`metrics["cost_per_action"]["value"]`.
- `status.error` או `metrics is None` → **`type=unavailable`**: "אין לי כרגע גישה לנתוני הביצועים של הקמפיין, אז לא אוכל לאבחן עכשיו. נסה שוב מאוחר יותר." (בלי צ'יפ; לא נשמר כהודעה — soft-fail).
- `cpl is None` → **`type=unavailable`**: "עדיין אין מספיק נתוני פניות בקמפיין כדי לאבחן את העלות לפנייה." (בלי צ'יפ).

> **קריטי:** ה-CPL הוא **חי מ-Meta**, לא מהשאלון (בשונה מבעיה 4 שמשתמשת בתקציב-השאלון). זה **המדד עצמו** שהבעיה מטפלת בו.

### 2.2 שער 2 — Lock (סדרה פתוחה בחלון-מדידה) — `:268-277`
`lock = lock_service.check_lock(campaign_id, "high_cpl")` (`lock_service.py:72-120`): **נעול ⇔** קיימת סדרת high_cpl פתוחה (`status ∈ {in_progress, success_monitoring}`) **וגם** action עם `window_ends_at > now`.
- נעול **ולא** override → **`type=lock`**: template `modules/high_cpl/lock` + `_lock_suffix` ("נשארו עוד כ-{hours} שעות עד שאוכל לראות אם זה עבד."). chips `[חזרה לתפריט]`. הנוסח:
  > כבר ביצענו שינוי על {service_name} ואנחנו באמצע חלון הבדיקה.{remaining} שינוי נוסף עכשיו עלול לקלקל את המדידה — שווה לחכות לתוצאה לפני צעד נוסף.
- **פר-`problem_type`:** סדרת high_cpl פתוחה נועלת רק high_cpl; סדרת low_quality עצמאית (לא נועלת). אכיפה: partial unique index `uq_open_session_per_problem`.

### 2.3 שער 3 — חוק-הברזל (benchmark) — `:279-287`
`benchmark = benchmark_service.classify_cpl(status.industry, cpl)` (`benchmark_service.py:85-98`) — מודל דו-סף חסר-חורים:
```python
if industry not-known or cpl not-finite: level="unknown"
elif cpl <= amazing_max:                  level="amazing"
elif cpl >  expensive_above:              level="expensive"
else:                                     level="average"     # (amazing_max, expensive_above]
```
- **`not override` וגם `level ∈ {amazing, average}`** → **`type=benchmark_stop`** — **חוק-הברזל**: template `modules/high_cpl/benchmark_stop` (**בלי LLM** — נוסח קבוע), **בלי לפתוח סדרה**. chips `[אני מקבל את ההמלצה / אני בכל זאת רוצה לשנות]`. הנוסח:
  > אני מבין את השאיפה לשפר ולהגיע לטופ, אבל כמנהל הקמפיינים שלך חובתי המקצועית להגיד לך שהמספרים הנוכחיים שלך מעולים ביחס לממוצע בשוק לענף שלך. שינוי אגרסיבי עכשיו הוא הימור מסוכן — הוא עלול לבלבל את האלגוריתם, לפגוע ביציבות ולהעלות את עלות הפנייה משמעותית. אני ממליץ בחום לא לגעת כרגע ולתת לקמפיין להמשיך לעבוד.

**טבלת ה-benchmark** (`app/data/benchmarks.json`; שמות עבריים מ-`agent_i18n.INDUSTRY_HE`):

| ענף | עברית | `amazing` (≤) | `average` | `expensive` (>) |
|-----|-------|:-------------:|:---------:|:---------------:|
| `beauty` | קוסמטיקה וטיפולים | ₪25 | ₪25–50 | ₪50 |
| `courses` | בתי ספר וקורסים | ₪50 | ₪50–100 | ₪100 |
| `fitness` | כושר וספורט | ₪30 | ₪30–60 | ₪60 |
| `local_services` | שירותים לבית ולעסק | ₪40 | ₪40–85 | ₪85 |
| `professional` | עו"ד, רו"ח ופיננסים | ₪90 | ₪90–160 | ₪160 |
| `realestate` | נדל"ן ותיווך | ₪100 | ₪100–200 | ₪200 |
| `b2b_hr` | הייטק, B2B וגיוס | ₪120 | ₪120–250 | ₪250 |

`unknown` = ענף חסר/לא-מוכר או CPL לא-סופי → מדלג על חוק-הברזל (fail-safe: `load_benchmarks` כושל → `{}` → הכל unknown, הסוכן לא קורס).

### 2.4 מסלול-העקיפה (override) — הלקוח מתעקש
- לחיצת **"אני בכל זאת רוצה לשנות"** (`override_change`) → `runDiagnose({override:true})` (`index.html:2622`) → diagnose חוזר עם `override=true` → **עוקף את שער ה-Lock (`:270`) ואת חוק-הברזל (`:282`)** ונופל ל-Scenario B.
- לחיצת **"אני מקבל את ההמלצה"** (`accept_recommendation`) → **frontend בלבד** (`:2623`): "מצוין — נשאיר את הקמפיין כמו שהוא. 👍" + חזרה לתפריט. אין קריאת-שרת.
- ב-override עם level amazing/average → Scenario B רץ אבל `state_key=optimize_not_high` (§2.5) כדי שה-LLM **לא** יטען שהעלות גבוהה (מניעת סתירה payload↔instruction).

### 2.5 Scenario B — פתיחת סדרה + ניסוח LLM — `:289-308`
```
last_session = optimization_service.get_active_or_last_session(campaign_id, "high_cpl")
mixed        = optimization_service.build_mixed_analysis(last_session)      # X→Y→Z אם יש היסטוריה
has_open     = last_session.status ∈ {in_progress, success_monitoring}
session      = optimization_service.open_session(campaign_id, user_id, "high_cpl", {"cpl": cpl})   # idempotent (resume)
msg          = _phrase(..., state_key=_high_cpl_state_key(level, mixed, has_open), ...)             # LLM
→ AgentDecision(type="diagnosis", message=msg, chips=[], session_id=session_id)
```
זהו הענף היחיד ש**פותח סדרת-אופטימיזציה** וקורא ל-LLM. הפרונטד עובר אוטומטית לחצי-הפתרון (`runPropose`).

**בחירת `state_key`** — `_high_cpl_state_key(level, mixed, has_open)` (`:108-126`), קובץ `modules/high_cpl/{state_key}.txt`:

| תנאי | state_key | תוכן ההנחיה |
|------|-----------|-------------|
| `level != expensive` (override/unknown) | `optimize_not_high` | משקף את הסיווג בדיוק; "אסור לטעון שהעלות גבוהה" |
| `expensive` + יש `mixed` (היסטוריית X→Y→Z) | `expensive_mixed` | 3 נקודות-מדידה (לפני→אחרי→עכשיו) |
| `expensive` + סדרה פתוחה (בלי תוצאה) | `expensive_repeat_open` | "אל תאמר שזו פנייה ראשונה"; בלי מספרי לפני/אחרי |
| `expensive` + פנייה ראשונה | `expensive_first` | "זו הפנייה הראשונה, אין שינוי קודם" |

**המשתנים ל-`[SYSTEM_PAYLOAD]`** (`_build_chip_payload`, `:129-160`; enums באנגלית, ה-LLM מתרגם): `service_name`, `market_classification` (=`EXPENSIVE`/`AMAZING`/…), `current_cost_per_lead` (ה-CPL החי), `industry` (עברית), `complaint_cycle` (`FIRST`/`REPEATED`), `cpl_range_for_the_current_classification` (מ-`format_range_he`), ובהיסטוריה גם `cost_before_change` (X) ו-`cost_immediately_after_change` (Y). *(Z=current לא מוצג — כפילות עם ה-CPL החי.)*

פרמטרי ה-LLM: `_phrase` (`:169-212`) → `build_agent_prompt(mode="chip", issue_type="high_cpl", state_key, system_state)`; `openai_agent_model`, temperature 0.5, max_tokens 600, json_mode=False. rate-limit→503, אחר→500.

**עץ-האבחון המלא:**
```
diagnose_problem_1
├─ אין מטריקות / אין CPL ──────────────→ unavailable        chips:[]
├─ lock.locked ולא override ───────────→ lock               chips:[חזרה לתפריט]
├─ level∈{amazing,average} ולא override → benchmark_stop     chips:[מקבל / בכל זאת לשנות]   ← חוק-הברזל
└─ expensive / unknown / override ─────→ diagnosis + session → runPropose()
```

---

## 3. מנוע-השלבים — `_STEP_PLANS['high_cpl']` (4 שלבים)

`optimization_service.py:630-651`. ה-action_types הם מקור-אמת יחיד (`:607-615`); `_VARIATIONS_PER_STEP=3` (תמיד 3, כמספר המודעות החיות).

| שלב | `action_type` | זוויות (`angles`) | `unique` | מה עושה | פרומפט |
|:---:|---------------|-------------------|:--------:|---------|--------|
| **1** | `creative_refresh` | `pain, dream, urgency` | ✓ | רענון-קופי — 3 זוויות **שונות**, כל הסט הקנוני | `solution_copy` |
| **2** | `angle_change` | `social_proof` | ✗ | זווית **אחת** (הוכחה-חברתית) × 3 | `angle_change_copy` |
| **3** | `angle_change` | `authority` | ✗ | זווית **אחת** (סמכות) × 3 | `angle_change_copy` |
| **4** | `offer_change` (STAGE_C) | `pain, dream, urgency` | ✓ | 3 זוויות סביב **הצעה חדשה** (דו-שלבי §8) | `offer_copy` |
| **5+** | — | — | — | `step_plan(5)=None` → terminal → `StepsExhaustedError` (409) | — |

`step_plan(step, problem_type)` (`:654-658`) — lookup טהור (`problem_type` חובה). `advance_to_next_step(session)` (`:704-727`) — **פונקציה טהורה של ה-action האחרון** (לא סומכת על `current_step` שיכול לפגר): אין actions→1; אחרון פעיל→אותו שלב (resume); `done`→step+1 אם קיים אחרת `None` (terminal). הזוויות נבחרות **דטרמיניסטית ע"י הקוד**, ה-LLM רק כותב (חוק 7).

---

## 4. propose — 3 וריאציות קופי (LLM) + תמונות eager

`solution_service.generate_solution(session_id, user_id, campaign_id, chosen_offer=None, *, eager_images=True)` — `solution_service.py:485-575`:
1. `advance_to_next_step` → `step`; `step_plan(step, "high_cpl")` → `None`→`StepsExhaustedError` (409).
2. שולף quiz (`brand_tone`, `answers.offer`) + `BusinessContext`.
3. **STAGE_C ג'-1:** `offer_change` בלי `chosen_offer` → `_recommend_offer` (LLM, פרומפט `stage_c`) → `OfferRecommendation` (בלי וריאציות/כפתור-אישור); `set_awaiting_offer(True)`.
4. בחירת פרומפט לפי action_type; `_run_copy_llm` (temp 0.8, max 2000, json_mode) + retry.
5. `_parse_variations` — בדיוק **3** וריאציות, headline/body לא-ריקים; `unique=True`→כל הזוויות שונות + הסט מלא; `unique=False`→זווית אחת ×3. פולט `SolutionVariation{angle, angle_he, headline, body, image_concept?}`.
6. `_persist_proposal` → `save_proposed_action` (**`push_status=NULL`** — "הוצע, טרם אושר") + `action_id`.
7. **תמונות eager** (`:573-574`): `_maybe_enqueue_propose_images` — רק ל-IMAGE_GEN_ACTION_TYPES וכשמתג-החירום כבוי; `include_images=True, image_gen_settled=False`, enqueue job `optimization_image_generate`. best-effort (כשל לא מפיל את ה-propose). מייצר 3 תמונות **לפני** האישור כדי שיוצגו ב-view-action.

**Endpoint:** `POST /me/agent/conversations/{id}/propose` (`agent.py:437-500`, `ProposeResponse{action_id, variations, step_label, step_number, recommendation, screening_question}`). `step_number` הוא **החוזה הקושר** — הלקוח מחזיר אותו ב-approve.

---

## 5. approve → push — creative-swap ל-Meta

**Endpoints:** צ'אט `POST /me/agent/conversations/{id}/approve` (`agent.py:503-575`) ו-פאנל `POST /me/actions/{id}/approve` (`optimization_views.py:199-227`) — שניהם → `optimization_approve_service.execute_approval` (`:49-144`).

**`execute_approval`:** (1) `advance_to_next_step`→`current_step`; (2) **binding**: `current_step != step_number`→409 (מגן מפני cron/approve-מקביל שהזיז שלב); (3) `action_type` מ-`step_plan`; (4) ניתוב: לבעיה 1 עם `include_images=True` → **מסלול async** `claim_and_enqueue_images` → status `generating` (worker מייצר תמונות ואז דוחף); ה-`push` הסינכרוני הוא fallback ל-recycle/kill-switch.

**`optimization_push_service.push(session, user_id, variations, *, step_number, window_hours=120)`** (`:414-619`) — **creative-swap**:
- **שערים (לפני כל קריאת-Meta):** step_plan תקף · session פעיל · `_normalize_variations` · resume-peek (action `pushed`→sync מקומי, 0 קריאות) · campaign live+owned · Meta client · live-old-ads · **שער-שוויון** `len(variations)==len(live_old)` אחרת 422.
- **claim/resume:** `open_push_action` (advisory-lock + CAS-lease). ה-`variations` המאוחסנות הן מקור-האמת ל-resume.
- **reserve-first (save-as-you-go):** פר-וריאציה — `upload_ad_image`→`create_ad_creative` (name=headline, image ממוחזר או חדש)→`create_ad` (ACTIVE), שמירת ה-id מיד אחרי כל create.
- **השהיית הישנים (D1 — אחרון!):** רק אחרי שכל החדשים חיים — `update_ad(old, PAUSED)` פר-מודעה. כך אף רגע אין קמפיין בלי מודעות חיות.
- **finalize:** `finalize_push_action` → CAS `pushing→pushed` + `window_ends_at = now + 120h`. (זו **הנקודה היחידה** שקובעת את החלון.)
- **תוצאה נטו ב-Meta:** 3 AdCreatives + 3 Ads חדשים (ACTIVE, ממחזרים image-hash ישן), 3 Ads ישנים PAUSED. התקציב נשמר.

**כשל (`_handle_failure`):** `finalize` → **לעולם לא rollback** (ה-swap כבר בוצע; 503 ל-resume). transient→503 (נשאר `pushing`, resume). permanent→`rollback_optimization_push` (מחיקת חדשים + הפעלת ישנים) → `push_failed`; אם ה-rollback **עצמו** נכשל (D2 — אולי אין מודעות חיות) → `push_rollback_pending` + `capture_exception` + `escalate_session`.

הודעת ההצלחה (`pushed`): "…אמדוד את התוצאה בעוד 120 שעות." (`agent.py`).

---

## 6. חלון-המדידה (120ש') + ניטור — `optimization_monitor_service`

**cron שעתי** (`runner.py:54,298` → `run_tick`), **דטרמיניסטי, בלי LLM**. `_WINDOW_HOURS=120`.

- **בחירת עבודה:** `fetch_due_action_windows` (`optimization_service.py:816`) — `status ∈ {in_progress, success_monitoring}` ו-`push_status='pushed'` ו-`window_ends_at <= now`.
- **מדידת CPL חדש:** `_measure_high_cpl` (`:256`) — `get_insights_client → cpl`. `CampaignNotLiveError`→action `done` (`closed_not_live`, terminal); transient→`skipped`; אחר→`escalate`.
- **סף-השיפור:** `X = starting_metric.cpl`; **`improved ⇔ cpl < X` מוחלט** (בלי אחוז/מרווח — "אפילו ₪1"). `cpl is None` (אפס לידים) או `X is None` → **regressed** (אין מדידה אמינה = אין שיפור).

**עץ-ההחלטה — `_apply_window` (`:152`), שתי חלונות לפי `action.status`:**

| חלון | `action.status` | תוצאה | פעולה |
|------|-----------------|-------|-------|
| **1** | `in_progress` | **improved** (`cpl<X`) | CAS→`success_monitoring` + חלון שני (now+120h); `session→success_monitoring`; outcome=`improved` |
| **1** | `in_progress` | **not improved** | CAS→`done`; outcome=`closed_no_improve` (`current_step` **לא** מתקדם — יתקדם כשהלקוח יחזור) |
| **2** | `success_monitoring` | **improved** (vs X, **לא** Y) | CAS→`done`; `session→done`; outcome=`closed_done` ✅ |
| **2** | `success_monitoring` | **regressed** | CAS→`done`; `session→in_progress` (נפתח לשלב הבא); outcome=`regressed` |

ה-**CAS על ה-action הוא נקודת-ה-commit**; `set_session_status` הוא projection best-effort אחריו.

**התראות לבעל-העסק — `_notify_for_outcome` (`:206`)** (best-effort, דרך `send_agent_notification` — מייל תמיד + וואטסאפ מותנה, כלל 30/חודש):

| outcome | NotificationType | anchor | תנאי |
|---------|------------------|--------|------|
| `closed_done` | **`SERIES_RESOLVED`** | `session:{id}` | תמיד |
| `closed_no_improve` / `regressed` | **`STEP_ADVANCED`** | `action:{id}` | **רק אם** `advance_to_next_step != None` (יש שלב הבא) |
| `improved` / `skipped` / `closed_not_live` | — | — | אין התראה |

---

## 7. הלולאה ה-reactive — חזרה, resume, סיום

- **חלון חי → Lock חוסם** (§2.2): תלונה חוזרת באמצע 120ש' → `type=lock`, "שווה לחכות" (אלא override).
- **חלון חלף → resume:** `open_session` **אידמפוטנטי** — מחזיר את הסדרה הפתוחה הקיימת (partial unique index מבטיח ≤1). ה-state_key משקף חזרה/היסטוריה (`expensive_repeat_open`/`expensive_mixed`).
- **סיום בשלב 4 (אין-שיפור):** ה-action → `done`, אבל **הסדרה נשארת `in_progress`** (לא נסגרת ל-done/failed בנתיב הזה). היא מפסיקה להיות אקטיבית רק כי `advance_to_next_step` מחזיר `None`. הסיום צץ כשהלקוח חוזר: `propose` → `advance→None` → `StepsExhaustedError` → **409 "השלמת את כל שלבי הרענון הזמינים לקמפיין זה"**. ⚠️ *מלכודת מתועדת: שורת-הסדרה תלויה ב-`in_progress` לנצח בנתיב הזה — לא נזק (advance סוגר), אבל שווה מודעות.*

---

## 8. STAGE_C — שלב 4 (`offer_change`, דו-שלבי)

שלב 4 שונה: הוא מציע **הצעה חדשה** לפני יצירת הקופי.
- **ג'-1 (המלצת-הצעה):** propose בלי `chosen_offer` → `_recommend_offer` (LLM, `stage_c`) → `OfferRecommendation` (בלי כפתור-אישור); `set_awaiting_offer(True)`. הפרונטד מציג את ההמלצה + `[חזרה לתפריט]`.
- **ג'-2 (יצירה):** propose עם `chosen_offer` → 3 וריאציות (`offer_copy`) → אישור → push רגיל; אחרי push מוצלח `finish_offer`.
- דגלי-סדרה נפרדים מ-status: `awaiting_offer`/`offer_generating`/`offer_generation_started_at` (migrations 0056/0058/0060) עם claim בעל TTL 5 דק'.

---

## 9. Frontend — מקצה-לקצה

1. **בועת-אבחון** — `runDiagnose` (`index.html:2636`), `switch(d.type)`:
   - `diagnosis` (`:2655`): `CHAT.sessionId=d.session_id`; מציג את `d.message`; ואז `runPropose()` **אוטומטית**.
   - `benchmark_stop` (`:2659`): הודעה + `[אני מקבל את ההמלצה / אני בכל זאת רוצה לשנות]`.
   - `lock`/`unavailable`/default (`:2670`): הודעה + `[חזרה לתפריט]`.
2. **כרטיס-הצעה** — `runPropose` (`:2716`) → `_renderProposalReply` (`:2725`): **לא** מרנדר את 3 הווריאציות בצ'אט, אלא בועת-סיכום + כפתור:
   > "הכנתי {n} וריאציות קופי{label}. לאישור, עריכה או רענון — פתח את העמוד:" + כפתור **"📝 פתח את הצעת הקופי"** → `openActionView(action_id)`.
3. **עמוד ההצעה** (`view-action`, `:1623`) — `openActionView`/`_renderAction` (`:3695-3714`), `GET /me/actions/{id}`, לפי `approval_state`:
   - **`pending` (מצב 1 — אישור):** 3 כרטיסי "וריאציה i" (כותרת+גוף+תמונה), `✏️ ערוך` פר-כרטיס, כפתור **"✅ אשר והעלה לקמפיין"** + אופציונלי **"🔄 רענן קופי (3 חדשות)"**.
   - **`awaiting_feedback` (מצב 2 — low_quality בלבד):** "איך הקמפיין מתקדם? … האם איכות הפניות השתפרה?" + `כן, השתפר`/`לא השתפר`.
   - **`done` (מצב 3 — high_cpl):** "הוריאציות אושרו ועלו לקמפיין. נמדוד את התוצאה ונעדכן אותך." + טבלת CPL **לפני/אחרי/עכשיו** (`starting`/`post_change`/`current`).
4. **תמונות + polling:** `_actionCardImageHtml` (`:3750`, ready/⏳ בייצור/🎨 צור); `_pollActionImages` (`:3790`, כל 3ש', עד 30 ניסיונות). **אישור:** `_approveAction` (`:3810`) → `POST /me/actions/{id}/approve` → `_pollActionUntilSettled` (`:3829`, עד ~2 דק' עד `in_progress`→settled).
5. **התקדמות שלבים ב-high_cpl:** הלקוח **לא** מתקדם ידנית. אחרי אישור, ה-action `in_progress`→`done`. ההתקדמות מונעת-**cron**: `STEP_ADVANCED`→מייל→הלקוח חוזר לצ'אט→resume→propose מייצר את שלב-הבא. terminal (שלב 4)→409.

---

## 10. מכונות-המצב

**`optimization_sessions.status`** (CHECK, `0051:15-16`): `in_progress → success_monitoring → done | failed | escalated`.
```
[אין] --open_session--> in_progress(step=1)
in_progress --(שיפור, חלון 1)--> success_monitoring --(שיפור, חלון 2)--> done
success_monitoring --(regressed, חלון 2)--> in_progress   (נפתח לשלב הבא)
in_progress --(אין-שיפור, חלון 1)--> (הסדרה נשארת in_progress; ה-action→done)
any --rollback D2 / stuck--> escalated
```
**`optimization_actions.status`**: `in_progress → success_monitoring → done | escalated` (נפרד מ-push_status).
**`optimization_actions.push_status`** (`0053:21-24`): `NULL(הוצע) → pushing → pushed | push_failed | push_rollback_pending`. ה-INSERT הוא ה-claim (אין `draft`); approve תופס `NULL→pushing` (CAS + approved_at + lease).

---

## 11. יחידות / קבועים / מלכודות
1. **CPL חי מ-Meta** (`spend/leads`), לא מהשאלון — זה המדד עצמו. אפס לידים → `cpl=None` → unavailable (באבחון) / regressed (בניטור).
2. **חלון 120 שעות** (5 ימים, לא 96/4), נקבע **רק** ב-`finalize_push_action`. cron שעתי.
3. **שיפור = `cpl < X` מוחלט** מול ה-CPL ההתחלתי (X), בשני החלונות (לא מול Y).
4. **חוק-הברזל בלי LLM** — amazing/average → template קבוע, בלי סדרה. רק expensive/override פותח סדרה.
5. **reserve-first:** יוצרים חדשים לפני השהיית ישנים; אף רגע בלי מודעות חיות. rollback D2 → escalate.
6. **שער-שוויון:** `#variations == #live_old_ads` (בד"כ 3) — אחרת 422.
7. **binding step_number:** approve חייב לתאום את השלב שהשרת חישב מחדש (409 אחרת) — מגן מפני cron/מקביליות.
8. **הסדרה לא נסגרת ל-done בנתיב אין-שיפור** (§7) — נשארת in_progress; advance מחזיר None.

---

## 12. Endpoints + api.js

| method | endpoint | api.js |
|--------|----------|--------|
| diagnose | `POST /me/agent/conversations/{id}/diagnose` | `:231` |
| propose | `POST /me/agent/conversations/{id}/propose` | `:256` |
| approve (צ'אט) | `POST /me/agent/conversations/{id}/approve` | — |
| getAction | `GET /me/actions/{id}` | `:151` |
| approveAction (פאנל) | `POST /me/actions/{id}/approve` | `:164` |
| editAction | `PATCH /me/actions/{id}` | `:171` |
| refreshAction | `POST /me/actions/{id}/refresh` | `:183` |
| regenerateActionImage | `POST /me/actions/{id}/variations/{i}/regenerate-image` | `:191` |
| submitActionFeedback (low_quality) | `POST /me/actions/{id}/feedback` | `:202` |

---

## 13. מפת קבצים (file:line)

**Orchestrator** — `agent_orchestrator.py`: `diagnose_problem_1` (`:246-308`), `_high_cpl_state_key` (`:108-126`), `_build_chip_payload` (`:129-160`), `_phrase` (`:169-212`), `_guard` (`:215-234`), צ'יפים/קבועים (`:51-67`).
**Benchmark** — `benchmark_service.py:85-98` (`classify_cpl`), `:101-112` (`market_cpl_for_industry`, בעיה 4), `:120-138` (`format_range_he`) · `app/data/benchmarks.json` · `app/models/benchmark.py` · `app/services/agent_i18n.py`.
**Lock** — `lock_service.py:72-120` (`check_lock`).
**מנוע-שלבים + lifecycle** — `optimization_service.py`: `_STEP_PLANS` (`:630-651`), `step_plan` (`:654`), `advance_to_next_step` (`:704`), `open_session` (`:93`), `finalize_push_action` (`:410`), `get_active_push_action` (`:276`), `fetch_due_action_windows` (`:816`), `_WINDOW_HOURS=120` (`:220`).
**Solution** — `solution_service.py:485-575` (`generate_solution`), `_parse_variations` (`:130`), `StepsExhaustedError` (`:513`).
**Push** — `optimization_push_service.py:414-619` (`push`), `claim_and_enqueue_images` (`:352`), `rollback_optimization_push` (`:180`), `_handle_failure` (`:681`).
**Approve** — `optimization_approve_service.py:49-144` (`execute_approval`).
**ניטור** — `optimization_monitor_service.py`: `run_tick` (`:64`), `_measure_high_cpl` (`:256`), `_apply_window` (`:152`), `_notify_for_outcome` (`:206`).
**Router** — `agent.py:266-291` (diagnose), `:437-500` (propose), `:503-575` (approve) · `optimization_views.py:199-227` (panel approve).
**Models** — `app/models/agent.py`: `DiagnoseRequest/Response`+`Chip` (`:93-147`), `ProposeResponse` (`:173-191`), `ApproveRequest/Response` (`:203-223`).
**Prompts** — `app/prompts/agent/core.txt` + `modules/high_cpl/{benchmark_stop,lock,expensive_first,expensive_mixed,expensive_repeat_open,optimize_not_high}.txt` (`lock`/`benchmark_stop` = template בלי LLM).
**Frontend** — `index.html`: `_PROBLEM_PT` (`:2549`), `runDiagnose` (`:2636`), `runPropose`/`_renderProposalReply` (`:2716-2736`), `openActionView`/`_renderAction` (`:3695-3744`), תמונות+polling (`:3750-3840`) · `api.js` (טבלה §12).
**Migrations** — `0051` (טבלאות+index+CHECK), `0052` (`open_optimization_session` RPC), `0053` (push_status), `0079`/`0093` (`open_optimization_push_action`/`save_proposed_action`), `0056`/`0058`/`0060` (STAGE_C offer flags).

---

## 14. טריגר-רקע — ניטור יזום (Session 8.1)

בנוסף לצ'יפ שהמשתמש לוחץ, קיים **טריגר שני** לאותה זרימה: **cron יומי** שסורק את כל הקמפיינים החיים ומריץ את
**אותו** גרעין-אבחון פר-קמפיין — בלי צ'אט, בלי `override`, ובלי frontend שמריץ `propose` (הרקע מריץ אותו בעצמו,
ושולח התראה במקום להחזיר `type=diagnosis`). מקור-תכנון: `docs/proactive-high-cpl-monitor-plan.md`.

**הגרעין המשותף — `assess_high_cpl(status)` (`agent_orchestrator.py`):** חולץ מ-`diagnose_problem_1` כדי
שהצ'אט והרקע יחלקו את **אותה** הכרעה דטרמיניסטית (מטריקות → Lock → benchmark) ולא ידרפטו. מחזיר
`HighCplAssessment{outcome ∈ (no_data / locked / benchmark_ok / unknown / expensive), cpl, industry, status,
lock, benchmark, no_data_reason}` — **בלי מדיניות override** (זו נשארת ב-caller). הצ'אט קורא
`_guard(assess_high_cpl(status))` ומנסח לפי ה-outcome (+override); הרקע קורא ופועל (רק `expensive`).

**`run_high_cpl_scan(user_id, campaign_id)` (`high_cpl_scan_service.py`):**
```
status = get_campaign_status_for_agent(campaign_id, user_id)  # conversation-free; ⚠️ סדר (campaign_id, user_id)
a = assess_high_cpl(status)
a.outcome != "expensive"  → skip שקט (no_data / locked / benchmark_ok / unknown)
expensive → open_session (אידמפוטנטי) → peek (advance_to_next_step + step_plan, read-only):
              None / מוצו-שלבים    → skip
              offer_change (שלב 4) → skip (דורש בחירת-הצעה אנושית; בלי לקרוא generate_solution → בלי set_awaiting_offer hijack)
            → get_pending_proposal_action (push_status IS NULL): כבר הוצע? → notify idempotent **בלי regen**
            → generate_solution(eager_images=True) → 3 קופי + 3 תמונות, push_status=NULL
            → PROPOSAL_READY (anchor=action:{id} → /action/{id}) → עוצר.
```

**מה מגיע בחינם:** ה-Lock (חלון 120ש') הוא מנגנון ה"5 ימים שאין צורך לבדוק"; חוק-הברזל אוכף את העצירה-החכמה
(הרקע לא נוגע ב-amazing/average); מכאן ואילך approve→push→ניטור **זהים לצ'אט** — הרקע מגיע בדיוק לאותה נקודה
(`push_status=NULL`), ו-`execute_approval` / `GET /me/actions/{id}` תלויי-action/session (לא conversation).

**הבדלים מהצ'אט:** אין conversation; אין `override` (הרקע שמרן — `unknown` → skip, לא פותח סדרה); ההתראה
`PROPOSAL_READY` במקום `type=diagnosis`. **מחוץ לסקופ:** בעיה 3 (דחייה) = webhook; בעיה 2/4 = לא מדידות-לזיהוי.

**ההתראה `PROPOSAL_READY`** (`notification_service` enum+frozenset · `handlers` dispatch+`_AGENT_WA_SUMMARY` ·
`email_templates` template+deep-link · migration `0113` CHECK+RPC-quota): מצטרפת לערוץ עדכוני-הסוכן (מייל תמיד
+ וואטסאפ מותנה, מכסה 30/חו', רדום עד WABA). anchor `action:{id}` → `/action/{id}` + idempotency
(UNIQUE user+type+anchor+channel → סריקה יומית חוזרת על אותו action לא כופלת התראה; זה ה"Lock" של חלון-טרום-האישור).

**ה-tick (`daily_high_cpl_tick` ב-`runner.py`):** יומי (monotonic, כרענון-הטוקן `86400`), בידוד-כשל פר-קמפיין
(`TransientError`→retry הבא; unknown→`capture_exception`+skip; **לעולם לא קורס**). שליפה:
`campaign_service.fetch_active_campaigns` (`status='live'`, admin, cross-user).

**מפת קבצים (Session 8.1):** `agent_orchestrator.assess_high_cpl`+`HighCplAssessment` · `high_cpl_scan_service`
(`run_high_cpl_scan` / `run_high_cpl_scan_tick` / `_notify_proposal_ready`) · `optimization_service.get_pending_proposal_action`
· `campaign_service.fetch_active_campaigns` · `notification_service.PROPOSAL_READY` · `email_templates` (template+deep-link)
· `worker/handlers` (dispatch+WA summary) · `worker/runner` (constant+wrapper+tick) · `supabase/migrations/0113_proposal_ready_notification.sql`.
