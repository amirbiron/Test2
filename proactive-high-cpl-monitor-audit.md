# אודיט: הסוכן-היוזם ברקע (ניטור 8.1 / high_cpl) — מה הקוד *באמת* עושה

> מבוסס קריאה ישירה של הקוד (לא של התכנון). כל טענה עם `file:line`. סימונים: 🔴 = **דורש אימות / הנחה לא-מאומתת** · 🟡 = **פער-עיצוב מובלע** · ✅ = **תוקן** · ⚙️ = החלטה/סף ממשי.
>
> **תוכנית-האב:** `docs/proactive-high-cpl-monitor-plan.md`.

> ## ✅ עודכן אחרי הסקירה — 2 פערים נסגרו
> 1. **קמפייני-סנדבוקס לא נסרקים עוד** — `fetch_active_campaigns` מסנן `is_sandbox=false` (קמפיין-דמה נוצר `live`
>    בכוונה ל-QA, וה-cron היה מבזבז עליו LLM+תמונות אמיתיים ושולח התראה אמיתית). *(commit `0fe2594`)*
> 2. **learning-phase gate** — `created_at < now-120h`: קמפיין צעיר מ-5 ימים לא נסרק. השורש: ה-Lock (חלון-מדידה)
>    חל **רק על פעולות-סוכן** (`optimization_actions`), ו-3 המודעות המקוריות מה-wizard הן `ads` — הן לא יוצרות
>    Lock, וגם `campaign_push_service` לא פותח `optimization_session`; בלי ה-gate ה-cron "קפץ" על ליד-בודד ראשון
>    בתוך ה-learning phase של Meta. *(commit `0eed2f2`)*
>
> **הפתוחים** (§10, לא באגים — נקודות-החלטה לפני פרודקשן): restart→סריקה-חוזרת · סקייל-סדרתי · `unknown`-שקוף ·
> worker-count · אין-תזכורת · CPL-cache · תלות-Meta.

---

## 0. TL;DR — מה זה עושה בפועל

`cron` בתוך לולאת ה-worker (`runner.py:460`) שכל 24 שעות סורק **קמפיינים חיים** (מסונן: לא-סנדבוקס, בני ≥5 ימים), מריץ על כל אחד את אותו גרעין-אבחון של הצ'אט (`assess_high_cpl`), ו**רק** אם ה-CPL מסווג `expensive` (מעל סף-הענף) ו**אין Lock פעיל** — פותח סדרה, מייצר 3 קופי + 3 תמונות ועוצר ב-`push_status=NULL` ("מוצע, טרם אושר"), ושולח התראת `PROPOSAL_READY`. **אפס פעולה על Meta בלי אישור אנושי** — ה-approve/push זהים לצ'אט.

**מה הרקע *לא* נוגע בו:** `paused`/`draft`/`archived` · קמפיין-סנדבוקס · קמפיין צעיר מ-5 ימים · קמפיין בחלון-מדידה פעיל (Lock, אחרי push של הסוכן) · `amazing`/`average`/`unknown`/`no_data`.

---

## 1. תזמון — מתי זה *באמת* רץ

| מה | ערך בפועל | file:line |
|---|---|---|
| מרווח | `HIGH_CPL_SCAN_INTERVAL_SECONDS = 86400` (24h) | `runner.py:58` |
| שעון | `time.monotonic()` — **לא** wall-clock | `runner.py:447,460` |
| ריצה ראשונה | `last_high_cpl_scan = 0.0` → **רץ מיד** בהעלאת ה-worker | `runner.py:438,460-462` |
| מיקום בלולאה | סדרתי, בין `monitor_optimization` ל-`billing_daily` — **אותו thread** | `runner.py:457-465` |
| רשת-ביטחון | `_run_high_cpl_scan_tick` עוטף ב-`try/except` — לעולם לא מפיל את הלולאה | `runner.py:312-320` |

🔴 **`monotonic` + `last=0.0` על restart:** בכל אתחול של ה-worker (deploy / קריסה / OOM) `last_high_cpl_scan` מתאפס ל-0 → ה-scan **רץ מיד שוב**, ללא קשר לזמן שעבר. אין persist של "מתי רץ לאחרונה" ב-DB. **המשמעות:** אם יש כמה deploys ביום, ה-scan רץ כמה פעמים ביום (לא "יומי"). ה-idempotency (§8) מונע כפילות-state וכפילות-התראה, אבל `generate_solution` (LLM + 3 תמונות) עלול לרוץ שוב אם החלון בין restart ליצירת-הצעה נתפס. **דורש אימות: תדירות deploy בפרודקשן.**

🔴 **חסימה סדרתית:** `_run_high_cpl_scan_tick` רץ inline בלולאת-ה-worker היחידה, **אחרי** `monitor` ו**לפני** `billing`. אם הסריקה איטית (הרבה קמפיינים × LLM × תמונות — §10), היא **מעכבת את כל שאר ה-ticks** (חיוב יומי, cleanup, reaper). **דורש אימות: כמה זמן לוקח tick מלא בסקייל צפוי.**

---

## 2. מה נסרק — `fetch_active_campaigns` (`campaign_service.py:522`) — ✅ 3 מסננים ב-DB

```sql
select id, user_id from campaigns
where status = 'live' and is_sandbox = false and created_at < now() - interval '120 hours'
order by id
```
- ⚙️ **`status='live'` בלבד** — `draft`/`paused`/`archived` לא מוציאים כסף (אין CPL) וה-push דורש live ממילא.
- ✅ **`is_sandbox = false`** — קמפיין-דמה נוצר `live` בכוונה (QA), אך ה-seam (`SandboxMetaClient`) מבודד את **הצ'אט**, לא את בורר-ה-cron. בלי הסינון ה-cron היה מבזבז LLM+תמונות ושולח התראה אמיתית על דמה. העמודה `NOT NULL DEFAULT FALSE` (0101) → `.eq(False)` בטוח.
- ✅ **`created_at < now-120h`** (`_MIN_CAMPAIGN_AGE_HOURS`) — **learning-phase gate**: קמפיין צעיר מ-5 ימים לא נסרק. `created_at` (יצירת draft) ≈ live (push תוך דקות); edge של draft-ישן-שעלה-late → מעט מוקדם, נדיר, low-severity.
- ⚙️ **כל המשתמשים** — `get_admin_client()`, חוצה-RLS. אין סינון per-user.

---

## 3. הגרעין הדטרמיניסטי — `assess_high_cpl` (`agent_orchestrator.py:122`)

3 שערים ברצף, **בלי override, בלי side-effect, בלי LLM** (`HighCplAssessment`, `:107`):

| שער | תנאי | outcome | file:line |
|---|---|---|---|
| 1 — מטריקות | `status.error` או `metrics is None` | `no_data` (reason=`no_metrics`) | `:133-139` |
| 1 — מטריקות | `cost_per_action.value is None` | `no_data` (reason=`no_cpl`) | `:140-147` |
| 2 — Lock | `check_lock(...).locked` | `locked` (**קודם ל-benchmark**) | `:151,155-156` |
| 3 — benchmark | `level ∈ {amazing, average}` | `benchmark_ok` | `:157-158` |
| 3 — benchmark | `level == expensive` | **`expensive`** | `:159-160` |
| 3 — benchmark | אחרת (ענף לא-מוכר / CPL לא-סופי) | `unknown` | `:161-162` |

⚙️ **ה-scan מגיב רק ל-`expensive`.** כל outcome אחר → `skip_{outcome}` שקט (`high_cpl_scan_service.py:75-78`).

### הספים — `classify_cpl` (`benchmark_service.py:85`)
```
cpl <= amazing_max      → amazing
cpl >  expensive_above  → expensive   ← הסף שמפעיל את הרקע
אחרת                    → average
industry None/לא-בטבלה  → unknown ;  CPL לא-סופי → unknown
```
- הספים פר-ענף מ-`benchmarks.json` (7 ענפים). דוגמה: `beauty` → expensive כש-`cpl > 50`; `realestate` → `cpl > 200`.
- ⚙️ ה-CPL עצמו = `ci.cpl` (`cost_per_action`) מ-Meta insights, **cache 5 דק'** (`get_campaign_status_for_agent:211,228`).

### ⚙️ האינטראקציה Lock↔cron (למה הרקע לא "מרעיש")
- **שער 2 (`check_lock`)** נעול כל עוד יש action עם `window_ends_at > now` (`lock_service.py:73,102`). חלון-המדידה = 120h (`_WINDOW_HOURS`), נקבע ב-`finalize` של ה-push.
- **המשמעות:** ברגע שהסוכן דוחף שינוי (approve+push) → 120h Lock → ה-cron מקבל `outcome=locked` ו**מדלג 5 ימים**. זו הסיבה שהרקע לא מציע חוזר-ונשנה על אותו קמפיין.
- **החולשה שתוקנה (§2):** ה-Lock חל **רק על `optimization_actions`** (פעולות-סוכן) — לא על 3 המודעות המקוריות מה-wizard (שהן `ads`). לכן קמפיין חדש **אין לו Lock**, וה-learning-phase gate (`created_at`) נחוץ כדי לא לקפוץ עליו מוקדם.

---

## 4. הזרימה בפועל — `run_high_cpl_scan` (`high_cpl_scan_service.py:62`)

צעד-צעד, עם ה-outcome/skip המדויק בכל ענף:

1. **status** (`:73`): `get_campaign_status_for_agent(campaign_id, user_id)` — ⚙️ סדר-פרמטרים (campaign קודם). conversation-free.
2. **assess** (`:74`): `assess_high_cpl(status)`. `outcome != "expensive"` → `return skip_{outcome}` (`:75-78`). **לא נפתחת סדרה, לא נשלחת התראה.**
3. **open_session** (`:81`): אידמפוטנטי — `partial index uq_open_session_per_problem` מבטיח ≤1 סדרה פתוחה → resume. `no id` → `skip_no_session` (`:85-87`).
4. **peek לשלב** (read-only, **לפני** כל side-effect):
   - `advance_to_next_step` + `step_plan` (טהורים). `plan is None` → **`skip_exhausted`** (מוצו 5 השלבים) (`:90-95`).
   - `action_type == offer_change` → **`skip_offer_change`** (`:96-100`) — שלב 4 דורש בחירת-הצעה אנושית; דילוג **לפני** יצירה (אחרת `set_awaiting_offer` היה חוטף את ההודעה הבאה בצ'אט).
5. **pending-guard** (`:104`): `get_pending_proposal_action(session_id, step)` — כבר הוצע (`push_status=NULL`)? → `_notify_proposal_ready` (idempotent) + **`already_proposed`** (`:105-109`). **בלי regen.**
6. **active-push guard** (`:117`): `get_active_push_action` (`pushing`/`push_rollback_pending`/`pushed`)? → **`already_pushed`** (`:118-120`) — **בלי notify** (המשתמש כבר אישר; התראה תטעה).
7. **generate_solution** (`:124`, `eager_images=True`) → 3 קופי (LLM) + 3 תמונות eager, `push_status=NULL`:
   - `StepsExhaustedError` → `skip_exhausted` (race; `:127-130`).
   - `MissingQuizError` → `skip_no_quiz` (קמפיין live בלי שאלון; `:131-134`).
   - מחזיר `OfferRecommendation` (race לשלב 4) → `skip_offer_recommendation`, **בלי notify** (`:136-139`).
   - אין `action_id` → `skip_no_action` (`:141-144`).
8. **notify** (`:145`): `_notify_proposal_ready` → **`proposed`**.

---

## 5. איפה הרקע *מתנהג אחרת* מהצ'אט

| היבט | צ'אט (`diagnose_problem_1`, `:313`) | רקע (`run_high_cpl_scan`) |
|---|---|---|
| טריגר | conversation (המשתמש) | cron, conversation-free |
| `outcome=expensive` | מנסח **אבחון טקסטואלי** (LLM) + `session_id`; המשתמש לוחץ *propose* בנפרד | **מדלג על האבחון** — ישר `generate_solution` + התראה |
| `outcome=unknown` | **מטופל** — נכנס ל"תרחיש ב'" (`:358`) ומאבחן | 🟡 **מדלג** (`skip_unknown`) |
| `benchmark_ok` (amazing/average) | `benchmark_stop` — chips *accept*/*override* | `skip_benchmark_ok` |
| override | **יש** (המשתמש עוקף את `benchmark_stop`) | **אין** — הרקע לעולם לא נוגע ב-amazing/average |
| קמפיין צעיר (<5 ימים) | מטופל (אין age-gate בצ'אט) | ✅ **מדלג** (learning-phase gate ב-`fetch_active_campaigns`) |
| קמפיין-סנדבוקס | מטופל (QA ידני) | ✅ **מדלג** (`is_sandbox` filter) |
| `no_data` | 2 הודעות `unavailable` נפרדות | `skip_no_data` שקט |

🟡 **הפער שנותר:** קמפיין עם **industry שאינו בטבלת ה-benchmarks** (או CPL לא-סופי) מסווג `unknown` → **הרקע לעולם לא ינטר אותו**, בעוד שבצ'אט הוא כן מטופל (§10 #3). קמפיינים כאלה "שקופים" לסוכן-היוזם.

---

## 6. מה קורה בכשל — טקסונומיה מדויקת

### בתוך `run_high_cpl_scan` (פר-קמפיין)
- `StepsExhaustedError` / `MissingQuizError` → **נבלעים כאן** ל-`skip_*` (מצבי-רקע לגיטימיים; `:127-134`).
- `TransientError` (Meta/DB/LLM חולף, כולל `AgentTransientError`) → **מתפשט** ל-tick.
- כל שגיאה אחרת (permanent/unknown) → **מתפשט** ל-tick.

### בתוך `run_high_cpl_scan_tick` (בידוד פר-קמפיין) — `high_cpl_scan_service.py:150`
- `TransientError` → `summary["transient"]++`, **דלג** (ה-tick הבא = מחר ינסה שוב; **לא** Sentry) (`:168-170`).
- `Exception` (unknown) → `capture_exception` + `summary["errors"]++`, **דלג** (`:171-174`).
- ⚙️ **לעולם לא זורק** — כשל בקמפיין אחד לא עוצר את הסריקה של השאר.

### התראה — `_notify_proposal_ready` (`:34`)
- **best-effort:** `except Exception` → warning + `capture_exception`; ה-action כבר קיים ומחכה בעמוד גם אם ההתראה נכשלה (`:55-59`).

🟡 **`unknown → terminal` בקוד-רקע (כלל Cron 11):** בניגוד ל-handlers שמסמנים entity ל-terminal על unknown, כאן unknown פשוט **מדולג** והקמפיין ייסרק שוב מחר. **מכוון** ונכון (הישות = קמפיין live תקין, לא job שצריך terminal-state) — לתשומת-לב.

---

## 7. שני תהליכים בו-זמנית — race / concurrency

**הנחת-הבסיס:** worker יחיד → ה-tick סדרתי, אין 2 ריצות-רקע במקביל.

🔴 **דורש אימות: כמה worker-instances רצים?** אם >1 (scale-out) — 2 ticks יכולים לסרוק את אותו קמפיין במקביל. ההגנות מחזיקות גם אז:
- `open_session` אידמפוטנטי (`partial index` → resume).
- `pending-guard` + `active-push guard` + `save_proposed_action` (ON CONFLICT פר-`(session,step)`) → action **אחד** לכל היותר.
- החולשה: **חלון בין ה-guard ל-`generate_solution`** — אם שתי ריצות עוברות את ה-guard לפני שאחת שמרה, **שתיהן** ירוצו `generate_solution` (2× LLM + 2× 3 תמונות); ה-`save` idempotent → action אחד. **תוצאה: בזבוז מחשוב, לא כפילות-state ולא כפילות-Meta.**

**רקע + צ'אט בו-זמנית:** אותו סיפור — resume + guards + `save` idempotent מונעים כפילות; לכל היותר `generate_solution` כפול. `_notify_proposal_ready` idempotent (§8) → לא 2 מיילים.

---

## 8. ההתראה — idempotency + מכסה (`send_agent_notification`, `notification_service.py:139`)

- **תמיד מייל** (רצפת-ביטחון) + **מותנה WhatsApp** — רק אם `agent_whatsapp_enabled` + יש `agent_alert_phone` + מתחת ל-`_AGENT_ALERTS_QUOTA` (`:149-150`).
- הכל ב-**RPC אטומי אחד** `create_agent_notification`: email-first + `FOR UPDATE` לספירת-מכסה race-safe + **`ON CONFLICT` 4-col** (`user+type+anchor+channel`) ל-idempotency (`:151-152`).
- ה-anchor = `action:{action_id}` (`high_cpl_scan_service.py:48`).
- ⚙️ סריקה יומית חוזרת על אותה הצעה (`already_proposed` → notify שוב) **לא כופלת התראה** — ה-`ON CONFLICT` בולע. המשתמש מקבל התראה **פעם אחת** לכל action.
- 🟡 **אין תזכורות חוזרות:** הצעה לא-מאושרת יושבת ב-`push_status=NULL` ללא הגבלת-זמן; התראה פעם-אחת בלבד. אין eskalation. **דורש החלטה: תזכורת? TTL להצעה?** (§10 #4)
- `PROPOSAL_READY` **נספר במכסה** של 30/חודש (migration 0113).

---

## 9. סיכום החלטות/ספים ממשיים (⚙️)

| החלטה | ערך בפועל | מקור |
|---|---|---|
| מרווח סריקה | 24h (monotonic, רץ-מיד-ב-boot) | `runner.py:58,438` |
| אוכלוסיית סריקה | `live` **+ לא-sandbox + בן ≥5 ימים**, כל המשתמשים | `campaign_service.py:522-540` |
| סף פעולה | `cpl > expensive_above` (פר-ענף) **בלבד** | `benchmark_service.py:96` |
| Lock | סדרה פתוחה + action עם `window_ends_at > now` (120h) — **רק פעולות-סוכן** | `lock_service.py:72-119` |
| eager images | **כן** (3 תמונות עם הקופי) | `high_cpl_scan_service.py:124-126` |
| עצירה | `push_status=NULL` — **אפס Meta בלי אישור** | `high_cpl_scan_service.py:5,122` |
| התראה | מייל תמיד + WhatsApp מותנה, idempotent פר-action, נספרת במכסה | `notification_service.py:149-152` |

---

## 10. 🔴 הנחות פתוחות / דורש-אימות (לפני פרודקשן)

1. **restart → סריקה מיידית חוזרת** (`last=0.0`, §1). deploy תכוף = ריצות חוזרות; `generate_solution` עלול לבזבז LLM+תמונות בחלון שלפני יצירת-ההצעה. *אימות: תדירות deploy; שקילת persist של `last_run` ב-DB.*
2. **סקייל / חסימה סדרתית** (§1). ה-tick רץ inline, סדרתי, `generate_solution` פר-קמפיין ברצף. N קמפיינים expensive = N × (LLM + 3 תמונות) חוסמים את שאר ה-ticks. *אימות: כמה `live`+`expensive` צפויים; זמן-tick worst-case; תקרה/batching?*
3. **`unknown` שקוף לרקע** (§5). industry לא-בטבלה → אף פעם לא מנוטר ברקע (בצ'אט כן). *החלטה מודעת?*
4. **אין תזכורת/eskalation** (§8). הצעה לא-מאושרת יושבת ללא הגבלת-זמן, התראה פעם-אחת. *החלטה: תזכורת? TTL?*
5. **worker-count** (§7). ה-race-safety מוכחת גם ל-multi-worker (idempotency), אבל הבזבוז בחלון-ה-guard גדל. *אימות: מספר instances.*
6. **CPL מ-cache 5 דק'** (§3). ה-scan מחליט על CPL בן עד 5 דקות. *סביר, לתשומת-לב.*
7. **תלות ב-Meta לכל קמפיין** (`get_campaign_status_for_agent:227-228`). כל קמפיין = קריאת insights (או cache). Meta חולף → `AgentTransientError` → הקמפיין נדלג ל-tick הבא. *אימות: התנהגות בעומס Meta / rate-limit.*

**נסגרו:** ✅ סינון קמפייני-סנדבוקס (`0fe2594`) · ✅ learning-phase gate לקמפיין צעיר (`0eed2f2`).

---

*אין `TODO`/`FIXME` מסומנים בקוד. הפריטים הפתוחים הם הנחות **מובלעות** שחולצו מהתנהגות הקוד — לא באגים, אלא נקודות-החלטה שדורשות אישור מודע לפני פרודקשן.*
