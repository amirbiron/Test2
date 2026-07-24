# תכנית שינוי — ניטור יזום של "עלות לפנייה גבוהה" (Problem 1 / `high_cpl`) ברקע

> מסמך תכנון בלבד. אינו נוגע בקוד — מיועד להעלאה לריפו ולהרצה דרך Claude Code.
> **מקורות-אמת של המצב הקיים:** מסמך המיפוי המלא של בעיה 1 בצ'אט (`docs/high-cpl-flow.md`, מיפוי file:line) + `docs/ROADMAP.md` (Session 7.3.5 + 7.4א/ב/ג, Phase 8.1). מסמך זה מגדיר רק את הדלתא — הטריגר החדש — ונשען על כל מה שכבר בנוי.
> **החלטת-מוצר שאושרה:** דרך א' — הרקע רץ `diagnose → open_session → propose` (יוצר 3 קופי + 3 תמונות eager), **עוצר ב-`push_status=NULL`** (מוכן-לאישור), ושולח התראה. **אפס פעולה על Meta בלי אישור אנושי.** אין תקרת-סדרות-ליום.

---

## 0. העיקרון בשורה אחת

הניטור-היזום **אינו מנוע חדש** — הוא **טריגר שני** (cron יומי) שנכנס לאותה לוגיקת-אבחון שהצ'יפ נכנס אליה. במקום שהמשתמש ילחץ "עלות לפנייה גבוהה", ה-cron קורא לאותו גרעין-אבחון פר-קמפיין-פעיל. כל השערים הקיימים (CPL חי, Lock, benchmark, reserve-first, מכונות-המצב) עובדים לרקע **בלי שינוי**. ההבדל היחיד: אין `override`, ואין frontend שמריץ את `propose` — הרקע מריץ אותו בעצמו ואז שולח התראה במקום להחזיר `type=diagnosis`.

```
cron יומי (run_tick monotonic ב-runner.py)
  → לכל קמפיין פעיל של כל משתמש:
      run_high_cpl_scan(user_id, campaign_id)  [פונקציית-רקע חדשה]
        │
        ├─ שער 1 — CPL חי מ-Insights           →  אין נתונים / cpl=None  → skip שקט (בלי התראה)
        ├─ שער 2 — Lock (סדרה פתוחה בחלון)      →  נעול                   → skip שקט  ← זה "5 הימים"
        ├─ שער 3 — benchmark (classify_cpl)     →  amazing/average         → skip שקט  ← חוק-הברזל המובלע
        │                                          unknown                 → skip שקט (§4.3)
        └─ expensive:
             open_session("high_cpl")  →  generate_solution(eager_images=True)  →  push_status=NULL
             →  התראה (מייל+וואטסאפ) עם קישור ל-view-action  →  עוצר.
                המשתמש נכנס, רואה 3 קופי + 3 תמונות, לוחץ "אשר והעלה" / עורך / מרענן — כמו היום.
```

**מה מגיע בחינם (בלי שורת-קוד):** שער ה-Lock פותר את "5 הימים שאין צורך לבדוק"; חוק-הברזל אוכף את העצירה-החכמה מעצמו; reserve-first, creative-swap, חלון-המדידה, וכל מכונת-המצבים ממשיכים לעבוד — כי הרקע מגיע בדיוק לאותה נקודה (`push_status=NULL`) שהצ'אט מגיע אליה, ומשם המסלול זהה.

---

## 1. מה כבר קיים ומה נבנה

| רכיב | סטטוס | תפקיד בניטור |
|------|-------|--------------|
| `diagnose_problem_1` (3 שערים + Scenario B) | ✅ קיים | הגרעין שהרקע מזין (§3) |
| `benchmark_service.classify_cpl` | ✅ קיים | חוק-הברזל — הרקע פועל רק על `expensive` |
| `lock_service.check_lock` | ✅ קיים | פתרון "5 הימים" — הרקע נעצר בחלון-מדידה חי |
| `optimization_service.open_session` (אידמפוטנטי) | ✅ קיים | פתיחת סדרה race-safe |
| `solution_service.generate_solution` (eager images) | ✅ קיים | 3 קופי + 3 תמונות **לפני** אישור |
| `optimization_monitor_service` (cron שעתי, חלון 120ש') | ✅ קיים | מודד ומתקדם — **לא נוגעים בו** |
| `send_agent_notification` (מייל תמיד + וואטסאפ מותנה, 30/חודש) | ✅ קיים | ערוץ ההתראה (§5) |
| `run_tick` monotonic ב-`runner.py` (דפוס 6.1 / 7.4ב-3) | ✅ קיים | דפוס ה-tick לחיקוי |
| **`run_high_cpl_scan(user_id, campaign_id)`** | 🔨 חדש | פונקציית-הרקע (§3–§4) |
| **`daily_high_cpl_tick`** ב-`runner.py` | 🔨 חדש | ה-cron היומי (§2) |
| **סוג-אירוע `PROPOSAL_READY`** בעדכוני-הסוכן | 🔨 חדש | ההתראה על action מוכן-לאישור (§5) |

עיקר העבודה מרוכז בפונקציה אחת (`run_high_cpl_scan`) שמחברת חלקים קיימים, ב-tick יומי שקורא לה, ובסוג-אירוע-התראה אחד. אין לוגיקת-אבחון חדשה, אין מכונת-מצבים חדשה, אין מסלול-הצגה חדש.

---

## 2. הטריגר — cron יומי ב-`runner.py`

אותו דפוס בדיוק כמו הניטור השעתי (7.4ב-3, `run_tick` ב-`:64,298`) ורענון-הטוקן (6.1) — tick מבוסס-שעון monotonic ב-`runner.py`, בלי scheduler חיצוני ובלי טבלת-state נפרדת.

- **תדירות:** אחת ל-24 שעות (tick monotonic; אם ה-worker היה כבוי, הבדיקה תרוץ בהפעלה הבאה — אותה סמנטיקה של שאר ה-ticks).
- **מה הוא עושה:** שולף את כל הקמפיינים ה**פעילים** (status חי ב-Meta — `live`/`active`, לא `draft`/`paused`/`archived`) של כל המשתמשים, ולכל אחד קורא `run_high_cpl_scan(user_id, campaign_id)`.
- **בידוד כשלים:** כשל בקמפיין אחד לא מפיל את הסריקה של השאר (try/except פר-קמפיין + `capture_exception`), בדיוק כמו ש-`run_tick` הקיים מבודד כשל פר-action.
- **בחירת הקמפיינים:** להעדיף שאילתה שמסננת כבר ב-DB לקמפיינים פעילים (כמו `fetch_due_action_windows` ב-`optimization_service.py:816` שמסנן ב-SQL), במקום למשוך הכל ולסנן ב-Python. אם אין helper קיים לשליפת "קמפיינים פעילים פר-כל-המשתמשים", זה ה-query היחיד החדש שנדרש.

> **סדר הסריקה (מאושר):** פר-קמפיין, בלי תקרה. לכל קמפיין עם CPL גבוה נפתחת סדרה עצמאית; ה-Lock מונע כפילות פר-קמפיין. לקוח עם 2–3 קמפיינים בעייתיים יקבל 2–3 התראות ו-actions — זה תקין, כל סדרה עצמאית. (אם יתברר כהצפה — תקרת "N סדרות-רקע ליום ללקוח" מתווספת בשורה אחת; לא נדרש עכשיו.)

---

## 3. הליבה — `run_high_cpl_scan(user_id, campaign_id)`

פונקציית-הרקע החדשה. היא מריצה את **אותה** רצף-ההכרעות של `diagnose_problem_1`, אבל בלי frontend ובלי `override`, וממשיכה בעצמה ל-`propose` במקום להחזיר `type=diagnosis`.

### 3.1 השאלה המבנית: לחלץ גרעין משותף, לא לשכפל

`diagnose_problem_1` (`agent_orchestrator.py:246-308`) מחזיר `AgentDecision` שמיועד לצ'אט (הודעת-LLM מנוסחת, צ'יפים, `type`). הרקע **לא צריך** את הניסוח ואת הצ'יפים — הוא צריך את ההכרעה (skip / expensive→המשך) ואת ה-`cpl`/`session`. שכפול הלוגיקה = טלאי. לכן, פתרון-השורש:

**לחלץ את גרעין-ההכרעה הדטרמיניסטי** משלושת השערים לפונקציה משותפת, למשל:
```
_assess_high_cpl(user_id, campaign_id) -> HighCplAssessment
    # מריץ: get_status → CPL חי → check_lock → classify_cpl
    # מחזיר outcome ∈ {no_data, locked, benchmark_ok, unknown, expensive}
    #        + cpl, industry, status, campaign_id
```
`diagnose_problem_1` (הצ'אט) קורא לגרעין הזה ואז **מנסח** לפי ה-outcome (LLM/template + צ'יפים). `run_high_cpl_scan` (הרקע) קורא לאותו גרעין ואז **פועל** לפי ה-outcome (skip / open+propose+notify). כך ההכרעה חיה במקום אחד ושני הטריגרים חולקים אותה — בדיוק כמו שהצ'אט של בעיה 4 והסנדבוקס חולקים את `_assess_budget_increase`.

> אם חילוץ מלא של הגרעין נראה גדול מדי כרפקטור בטוח, אלטרנטיבה זולה יותר: `run_high_cpl_scan` קורא ישירות לאותם שלושה שירותים (`agent_service.get_status_for_conversation`/`get_campaign_status_for_agent`, `lock_service.check_lock`, `benchmark_service.classify_cpl`) באותו סדר. זה עדיין **לא** משכפל לוגיקה (השירותים הם המקור), רק את סדר-הקריאה. CC יחליט לפי כמה `diagnose_problem_1` קשור ל-conversation. **העדפה: חילוץ גרעין**, כי הוא מונע drift עתידי בין הצ'אט לרקע.

### 3.2 הבדל מבני מהצ'אט: אין conversation

`diagnose_problem_1` מקבל `conversation_id` (הצ'אט תמיד בהקשר שיחה). לרקע **אין שיחה** — הוא פועל על קמפיין, לא על conversation. שתי השלכות:
- שליפת המטריקות בצ'אט עוברת דרך `get_status_for_conversation`. הרקע צריך את הגרסה מבוססת-הקמפיין: `get_campaign_status_for_agent(user_id, campaign_id)` (`agent_service.py:225-257`) — היא כבר קיימת ומקבלת `campaign_id` ישירות. הרקע קורא לה, בלי conversation.
- `open_session(campaign_id, user_id, "high_cpl", {"cpl": cpl})` — כבר לא-תלוי-conversation (מקבל `campaign_id`). ללא שינוי.

> **החלטת-משנה שאני מסמן:** לאיזו שיחה מקושר ה-action שהרקע יוצר? ה-view-action וה-`execute_approval` הקיימים נשענים על action מקושר-סדרה (`optimization_actions` → `optimization_sessions` → campaign), **לא** בהכרח על conversation. אם משהו בשרשרת ה-approve דורש `conversation_id` — צריך לספק אחד. **ברירת-המחדל שלי:** אם קיימת שיחת-סוכן פעילה לקמפיין, לקשר אליה; אחרת ליצור/לא-לקשר (תלוי אם ה-approve באמת דורש conversation). CC יבדוק אם `execute_approval` / `GET /me/actions/{id}` תלויים ב-conversation, ולפי זה יכריע. אם הם תלויי-קמפיין בלבד — אין כאן בעיה כלל.

### 3.3 הרצף המלא של `run_high_cpl_scan`

```
1. assess = _assess_high_cpl(user_id, campaign_id)     # §3.1
2. אם assess.outcome ∈ {no_data, locked, benchmark_ok, unknown}:
      log(outcome) ; return                             # skip שקט, בלי התראה, בלי סדרה
3. # outcome == expensive:
   session = open_session(campaign_id, user_id, "high_cpl", {"cpl": assess.cpl})   # אידמפוטנטי (resume)
   result  = generate_solution(session.id, user_id, campaign_id, eager_images=True) # 3 קופי + תמונות eager
      # generate_solution כבר: advance_to_next_step → step_plan → _run_copy_llm →
      #   _parse_variations (3) → save_proposed_action (push_status=NULL) → enqueue images
   4. אם generate_solution זרק StepsExhaustedError (409 — הקמפיין מיצה את כל השלבים):
        log ; (אופציונלי: התראה "מיצינו את שלבי הרענון") ; return    # §4.4
   5. שליחת התראה PROPOSAL_READY (§5) עם action_id/session_id ל-deep-link.
   6. return.  # הרקע עוצר כאן. push_status=NULL. הכל מחכה ללחיצת המשתמש.
```

**קריטי — הרקע עוצר לפני `execute_approval`.** הוא **לא** קורא ל-approve, לא ל-push, לא נוגע ב-Meta. ה-action נשאר `push_status=NULL` = "הוצע, טרם אושר". הלחיצה "אשר והעלה" (בעמוד view-action) היא שתתפוס `NULL→pushing` ותריץ את ה-creative-swap — בדיוק כמו היום, מאותו `execute_approval`, בלי הבחנה בין action שנוצר בצ'אט לאחד שנוצר ברקע.

---

## 4. מה מגיע בחינם, ומה ההבדלים המדויקים מהצ'אט

### 4.1 Lock = פתרון "5 הימים" (בלי לוגיקה מיוחדת)
אמרת "ב-5 הימים שאחרי שינוי אין צורך לבדוק בכלל". הרקע **לא** צריך תנאי-זמן מיוחד. שער ה-Lock (`check_lock`, §2.2 במיפוי) כבר מחזיר "נעול" כשקיימת סדרת high_cpl פתוחה עם `window_ends_at > now`. הרקע ייתקל בזה ויֵצא (skip), בדיוק כמו שהצ'אט מחזיר `type=lock`. הנעילה של 120 השעות **היא** מנגנון ה"5 ימים" — היא כבר בנויה, וחלה על הרקע והצ'אט כאחד. זו הסיבה שההבחנה שלך היא נגזרת ישירה מחוק-הברזל, לא כלל נפרד.

### 4.2 חוק-הברזל = העצירה-החכמה, מובלעת
בצ'אט, לקוח שמתלונן על CPL תקין (amazing/average) מקבל `benchmark_stop` ("המספרים שלך מעולים, אל תיגע") ויכול לעקוף עם `override`. ברקע **אין override ואין תלונה** — הרקע פשוט לא פועל על amazing/average (skip). כך העצירה-החכמה נאכפת מעצמה: הרקע נוגע **רק** ב-`expensive`. אין צורך לממש את `benchmark_stop` ברקע — פשוט לא ממשיכים.

### 4.3 `unknown` (ענף לא-מוכר / CPL לא-סופי) → skip, לא פעולה
בצ'אט, `unknown` **פותח** סדרה (נופל ל-Scenario B עם `state_key=optimize_not_high`), כי המשתמש ביקש במפורש. ברקע אין בקשה מפורשת, ואין benchmark להשוות אליו — לכן **הרקע לא פועל על `unknown`** (skip). זה הבדל מכוון מהצ'אט: הרקע שמרן יותר כי הוא פועל בלי אדם. פעולה יזומה על קמפיין שאי-אפשר לסווג את ה-CPL שלו = ניחוש; עדיף לא לגעת. (בצ'אט זה בסדר — שם אדם החליט.)

### 4.4 מיצוי-שלבים (שלב 4 → terminal)
אם קמפיין כבר עבר את כל 4 השלבים, `generate_solution` (דרך `advance_to_next_step`) זורק `StepsExhaustedError` (409). ברקע זה **לא שגיאה** אלא מצב-לגיטימי: "אין עוד מה לרענן אוטומטית". הרקע תופס את זה, מדלג, ואופציונלית שולח התראה חד-פעמית ("מיצינו את שלבי הרענון לקמפיין הזה — כדאי לבחון אותו ידנית"). **החלטת-משנה:** להתריע על מיצוי או לדלג בשקט? ברירת-המחדל שלי: **דילוג שקט** (בלי להטריד), כי הצ'אט כבר מתריע דרך `STEP_ADVANCED` כשמגיעים לשלב האחרון. אם תרצה התראת-מיצוי — שורה אחת.

### 4.5 טבלת ההבדלים המרוכזת

| היבט | צ'אט (`diagnose_problem_1`) | רקע (`run_high_cpl_scan`) |
|------|-----------------------------|----------------------------|
| טריגר | לחיצת צ'יפ | cron יומי |
| conversation | תמיד יש | אין (§3.2) |
| `override` | קיים (הלקוח מתעקש) | **אין** — לא נוגעים ב-amazing/average |
| `unknown` | פותח סדרה (בקשה מפורשת) | **skip** (§4.3) |
| מי מריץ `propose` | frontend (אוטומטית) | הרקע בעצמו |
| מה קורה אחרי propose | `type=diagnosis` ל-frontend | התראה PROPOSAL_READY (§5) |
| נקודת-עצירה | `push_status=NULL`, המשתמש לוחץ | **זהה** — `push_status=NULL`, המשתמש לוחץ |
| approve → push → Meta | `execute_approval` (לחיצה) | **זהה** — אותו `execute_approval`, אותה לחיצה |
| Lock / benchmark / reserve-first / חלון-מדידה | כמו במיפוי | **זהה, בלי שינוי** |

השורה התחתונה: משתי העמודות, כל מה שאחרי `push_status=NULL` **זהה לחלוטין**. הרקע רק מזין את אותה מכונה מטריגר אחר ועוצר באותה נקודה.

---

## 5. ההתראה — סוג-אירוע `PROPOSAL_READY` בערוץ עדכוני-הסוכן

זו **תוספת סוג-אירוע** לערוץ הקיים, לא ערוץ חדש. מצטרפת ל-`SERIES_RESOLVED`/`STEP_ADVANCED` שכבר קיימים ב-`optimization_monitor_service._notify_for_outcome` (§6 במיפוי), ונשלחת דרך אותו `send_agent_notification` (מייל תמיד ל-`agent_alert_email`; וואטסאפ מותנה ב-`agent_whatsapp_enabled` + מספר; כלל 30/חודש; רדום מאחורי gate ה-WABA עד אישור template).

- **מתי:** בסוף `run_high_cpl_scan`, אחרי `generate_solution` מוצלח (3 קופי + תמונות eager מוכנים), על ה-action עם `push_status=NULL`.
- **anchor / deep-link:** `action:{action_id}` (כמו `STEP_ADVANCED`) → הקישור בעמוד מוביל ישירות ל-view-action עם ה-action המוכן-לאישור. המשתמש רואה 3 קופי + 3 תמונות + כפתורי "אשר והעלה" / "ערוך קופי" / "רענן קופי" / "רענן תמונה" — כולם כבר קיימים בעמוד.
- **תוכן (מייל+וואטסאפ):** בטון של הסוכן, בעברית, בגובה-העיניים — "זיהיתי שעלות הפנייה בקמפיין {service_name} גבוהה (₪{cpl}). הכנתי 3 גרסאות קופי חדשות עם תמונות — הן מחכות לאישורך. להצגה ואישור: {link}". (הניסוח דרך אותו מנגנון template של עדכוני-הסוכן; אם הוא מבוסס-LLM — פרומפט מורחב שמסביר את מלוא ההקשר, לא תמצית.)
- **idempotency:** מפתח ב-`sent_notifications` שקושר ל-`action_id` (למשל `proposal_ready:{action_id}`). כך אם `run_high_cpl_scan` ירוץ שוב על אותו קמפיין **בזמן שהסדרה עדיין פתוחה** — לא תישלח התראה כפולה. (בפועל ה-Lock כבר חוסם ריצה חוזרת בחלון, אבל בין propose לאישור אין עדיין `window_ends_at`, אז ה-idempotency על ה-action הוא ההגנה הישירה. ראה §7.)
- **best-effort:** כשל בהתראה לא מבטל את ה-action שכבר נוצר (בדיוק כמו `_notify_for_outcome` הקיים שהוא best-effort). ה-action מחכה בעמוד גם אם המייל נכשל.

> **תלות ה-WABA (זהה לכל עדכוני-הסוכן):** המייל עובד מיד (Resend, Phase 4.6). הוואטסאפ יישלח בפועל רק אחרי אישור ה-template והדלקת `whatsapp_production_ready`. מצב-ביניים מתוכנן, לא באג.

---

## 6. מה שבמפורש **לא** נכנס לסריקה הזו

### 6.1 בעיה 3 (מודעה נדחתה) — מסלול webhook נפרד, לא cron
אמרת "חוץ מלבדוק אם מודעה נדחתה". דחיית-מודעה **אינה** בדיקה תקופתית — היא אירוע-דחיפה מ-Meta (webhook על שינוי status → `rejected`). ה-spec כבר מגדיר את זה (enum `rejected` הפיך מ-`live` + "webhook להאזנה לשינויי status"), ובעיה 3 בצ'אט (`meta_rejection`) היא מסלול משלה. **הוראה מפורשת ל-CC:** אל תתחוב זיהוי-דחייה לתוך ה-cron היומי של `run_high_cpl_scan`. אם ניטור-דחייה-יזום רצוי, הוא מסמך-תכנון נפרד סביב webhook, לא סביב הסריקה הזו.

### 6.2 בעיות 2 ו-4 — לא ניתנות לזיהוי-יזום
- **בעיה 2 (לידים לא איכותיים):** אין אות מספרי לאיכות-ליד; המדד היחיד הוא תשובת-הבעלים במייל-המעקב. הרקע לא יכול לזהות אותה. מחוץ לסקופ.
- **בעיה 4 (מעט פניות):** מבוססת-תלונה (ציפיית-לקוח מול צפי-תקציב), ובלי יעד אין מול מה להשוות. מחוץ לסקופ.

הסריקה היזומה מטפלת ב-`high_cpl` **בלבד**. זו לא הגבלה זמנית — זו נגזרת מכך ש-CPL היא הבעיה היח��דה שהיא מדידה מספרית טהורה שהסוכן יכול לזהות לבד.

---

## 7. אינטראקציה עם הצ'אט — כבר פתורה ע"י ה-Lock

מה קורה אם הרקע פתח סדרה בלילה והלקוח נכנס לצ'אט בבוקר ומתלונן על אותה בעיה? **חוק ה-Lock מטפל בזה בלי קוד נוסף:**
- הרקע פתח סדרת high_cpl (`open_session`, אידמפוטנטי, partial unique index `uq_open_session_per_problem` מבטיח ≤1 פר קמפיין+בעיה).
- הלקוח לוחץ "עלות לפנייה גבוהה" בצ'אט → `diagnose_problem_1` → אם ה-action כבר ב-window חי → `type=lock`. אם עדיין `push_status=NULL` (טרם אושר, אין window) → `open_session` **מחזיר את אותה סדרה** (resume), ו-`propose` יראה את ה-action הקיים. אין כפילות — הצ'אט והרקע חולקים את אותה סדרה ואותו action.

**נקודה שדורשת אימות מ-CC:** בחלון שבין propose-של-הרקע לאישור (כש-`push_status=NULL` ואין עדיין `window_ends_at`), האם `diagnose_problem_1`/`propose` בצ'אט מזהים נכון את ה-action הקיים ולא יוצרים כפול? הזרימה הקיימת אמורה לטפל בזה (`open_session` אידמפוטנטי + `save_proposed_action` שאמור לזהות action פתוח), אבל CC צריך לוודא שאין race שבו הרקע והצ'אט יוצרים שני actions באותה סדרה. אם ה-`save_proposed_action` הקיים לא מגן על זה — זו הנקודה היחידה שאולי דורשת חיזוק (CAS/unique על action-פתוח-פר-סדרה), והיא ממילא באג-פוטנציאלי קיים גם בלי הרקע (שני טאבים בצ'אט). פתרון-שורש: לוודא idempotency ברמת ה-action-הפתוח, לא לטלא ברמת הרקע.

---

## 8. סדר ביצוע מומלץ ל-CC

1. **חילוץ גרעין-האבחון** (`_assess_high_cpl`, §3.1) מ-`diagnose_problem_1`, ווידוא שהצ'אט ממשיך לעבוד דרכו (רגרסיה: הצ'אט לא משתנה בהתנהגות).
2. **`run_high_cpl_scan`** (§3.3) — הרצת הגרעין, skip על כל non-expensive, `open_session`+`generate_solution(eager_images=True)`, עצירה ב-`push_status=NULL`.
3. **סוג-אירוע `PROPOSAL_READY`** (§5) + idempotency על `action_id`.
4. **`daily_high_cpl_tick`** ב-`runner.py` (§2) — שליפת קמפיינים פעילים + לולאה מבודדת-כשל.
5. **אימות אינטראקציית צ'אט↔רקע** (§7) — לוודא שאין action כפול בחלון שלפני האישור; לחזק idempotency ברמת ה-action אם צריך.
6. **עדכון מסמכים** — `docs/ROADMAP.md` (Phase 8.1 עובר מ-🔮 ל-🔨/✅ — "ניטור יזום ל-high_cpl בלבד, דרך א', עוצר לאישור"); מסמך המיפוי של בעיה 1 (הוספת סעיף "טריגר-רקע" שמצביע על הגרעין המשותף).

בכל שלב: פתרון-שורש. הגרעין המשותף (לא שכפול `diagnose_problem_1`); הרקע מזין את המסלול הקיים (לא מסלול-הצגה חדש); idempotency ברמת ה-action (לא טלאי ברמת ה-cron).

---

## 9. נקודות פתוחות שדורשות החלטה / אימות-ריפו לפני הרצה

1. **חילוץ גרעין מלא מול קריאה-ישירה-לשירותים** (§3.1) — העדפתי חילוץ גרעין; CC יכריע לפי כמה `diagnose_problem_1` קשור ל-conversation. שתי הדרכים לא-משכפלות-לוגיקה.
2. **קישור ה-action לשיחה** (§3.2) — תלוי אם `execute_approval`/`GET /me/actions/{id}` דורשים `conversation_id` או שהם תלויי-קמפיין בלבד. CC לבדוק. אם תלויי-קמפיין — אין בעיה.
3. **התראת-מיצוי-שלבים** (§4.4) — ברירת-מחדל: דילוג שקט. אם תרצה התראה על "מיצינו שלבים" — שורה אחת.
4. **race של action-כפול לפני אישור** (§7) — CC לאמת שהזרימה הקיימת מגנה; אם לא, חיזוק idempotency ברמת ה-action (באג קיים ממילא, לא ייחודי לרקע).
5. **תלות ה-WABA** (§5) — הוואטסאפ יישלח רק אחרי אישור template + `whatsapp_production_ready`. המייל עובד מיד. לוודא שזה מקובל כמצב-ביניים.
6. **סדר-סריקה בלי תקרה** (§2) — מאושר. תקרה תיתכן בעתיד אם יתברר כהצפה (לא צפוי — נדיר שמשתמש מריץ מעל 2–3 קמפיינים במקביל).
