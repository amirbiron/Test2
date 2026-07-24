# תוכנית מימוש: תמונות חדשות ברענון-קריאייטיב (אופטימיזציה, פרודקשן)

> **סטטוס:** טיוטה — **ההכרעות העיקריות סוכמו עם הבעלים** (סעיף 0). **ענף:** `claude/sandbox-3-image-renderer-qyk2rn`
> **⚠️ זה הופך החלטת-MVP מתועדת.** היום רענון-קריאייטיב מחליף **רק טקסט** וממחזר את התמונה הישנה     # עדכון קטן: התיעוד הזה הוא מעיקרו טעות, גיא לא אמר דבר כזה מעולם (אמיר)
> (`docs/ROADMAP.md` ~12976/13151 — בחירת גיא: "רק טקסט ב-MVP, זול ופחות מפריע לאלגוריתם"). השינוי מבקש
> את האופציה (ב) שנדחתה. דורש תיעוד ההיפוך ב-ROADMAP/spec.
> **נכתב אחרי:** workflow חקירה (4 חוקרים) → סינתזת-תכנון → ביקורת-יריב (3 עדשות) שאומת מול הקוד.
> **מחליף/בולע** את `docs/sandbox-image-renderer-plan.md` — תצוגת-הסאנדבוקס מסופקת כאן כתוצר-לוואי (סעיף 9).
>
> **הערת-עלות (החלטת בעלים):** העלות זניחה (גרושים), ואנו עוקבים אחריה ב-OpenAI. **בדיקות מקיפות ומוצר
> ללא-באגים גוברים על חיסכון בשקלים בודדים.** לכן — הוסרה כל מורכבות שנועדה לחסוך עלות (quota מלאכותי,
> תמונת-דמה בסאנדבוקס); נשמרה ואף חוזקה **כל** עבודת-הנכונוּת (idempotency, סימטריה, טסטים).

---

## 0. ההכרעות (סוכמו עם הבעלים)

| # | נושא | הכרעה |
|---|---|---|
| **1** | ארכיטקטורה — מתי מייצרים | ✅ **A3** — יצירה אסינכרונית (worker) לפני ה-push (מהיר, דטרמיניסטי, בלי לתקוע request) |
| **2** | סקופ — אילו action types | ✅ **כל סוגי ה-3-קופי: `creative_refresh` + `angle_change` + `offer_change`** (לא רק רענון) |
| **3** | דחייה מ-Meta | ✅ **רענון מלא מחדש — 3 קופי + 3 תמונות** (במקום תיקון-בודד על תמונה ישנה) |
| **4** | צ'קבוקס דינמי | ✅ **דגל `include_images` בכל הזרימה** + צ'קבוקס בסאנדבוקס. כבוי → recycle הקיים (וגם מתג-חירום) |
| **5** | Quota מלאכותי | ✅ **אין** — הלולאה חסומה מעצמה (מספר שלבים קבוע). gate "קמפיין משלם" אוטומטי ממילא |
| **6** | סאנדבוקס | ✅ **תמונות אמיתיות** (בדיקה אמינה מקצה-לקצה), לא placeholder |
| **7** | מתג-חירום גלובלי | ✅ flag ב-`app_settings` לכפיית recycle בכל המערכת בלי deploy (גיבוי ל-#4) |

**שתי נקודות פתוחות-קטנות בלבד:** (א) האם להציג את הצ'קבוקס גם ב-approve הפרודקשן (מול הלקוח) או רק
בסאנדבוקס — ברירת-מחדל מוצעת: בפרודקשן הדגל=ON ל-3 הסוגים, צ'קבוקס-לקוח = אופציה עתידית; (ב) ברירת-המחדל של
הצ'קבוקס בסאנדבוקס (מוצע: מסומן ✔️, כך שבדיקה רגילה כוללת תמונות; מבטלים כדי לבדוק דברים אחרים מהר).

---

## 1. רקע: מה היום, מה מבקשים

**היום (`optimization_push_service.push`, create loop, שורות 434-450):** מאשרים 3 וריאציות קופי (טקסט) →
לכל וריאציה `i`: `old = live_old[i]` → `_download_image(old.image_url)` → `upload_ad_image` → `create_ad_creative`
(**תמונה ישנה ממוחזרת + טקסט חדש**) → `create_ad`. בסוף: PAUSE ל-3 הישנות + חלון-מדידה 120ש'.

**מבקשים:** שכל וריאציית קופי תקבל **תמונה חדשה** מ-`image_concept` שלה (כמו ה-wizard), במקום מיחזור — **אך
ניתן-לכיבוי דינמי** דרך דגל `include_images` (סעיף 4).

**ההחלטה שמתהפכת:** text-only נבחר ל-MVP במכוון — זול, ו**החלפת תמונה משבשת את אלגוריתם-המסירה של Meta יותר
מהחלפת טקסט**. ההיפוך לגיטימי; מתועד ב-ROADMAP (סעיף 7), והדגל הדינמי + מתג-החירום נותנים שליטה.

## 2. הארכיטקטורה — A3 (יצירה אסינכרונית לפני push)

נשקלו 3 מיקומים. **ההכרעה היא לפי latency ו-דטרמיניזם — לא עלות:**

| אופציה | מתי | בעיה עיקרית |
|---|---|---|
| **A1 — propose-time** | ב-`generate_solution` | קופי מיוצר-מחדש בכל propose (temp 0.8) → תמונות לא תואמות לקופי שאושר; ה-`save_proposed_action` דורס. **נדחה.** |
| **A2 — push-sync** | בתוך ה-create loop, ב-request של approve | יצירה איטית (×`_MAX_RETRIES=3`, `timeout=60`) → **כמה דקות** תוקעות request אחד; timeout באמצע → resume עם קלט לא-דטרמיניסטי |
| **A3 — pre-step אסינכרוני** ✅ | worker job, ב-approve; push מוריד מ-path שמור | מורכבות נוספת (handler + polling), אך ה-push נשאר **צרכן טהור** של תמונות שכבר נוצרו → דטרמיניסטי כמו היום |

**למה A3:** יצירת-תמונה איטית; להריץ אותה ברקע (לא לחסום את ה-request) זה גם UX טוב למשתמש וגם שומר על ה-push
**דטרמיניסטי** (הוא רק מוריד תמונה קבועה, בדיוק כמו שהיום מוריד את התמונה הישנה). זה **בדיוק דפוס ה-wizard**
(`handle_generate_ad_images`).

> **עדכון — "תמונות עם הקופי" (שחזור חזון ה-spec §159/216: 3 קופי + 3 תמונות יחד).** לבקשת המוצר, התמונות
> נוצרות עכשיו **eager בזמן ה-propose** (job נפרד `optimization_image_generate`, generate-only) כדי שיוצגו
> לצד הקופי ב-`view-action` ל**בחינה/רענון לפני** approve. זה **הופך** את דחיית "A1 — propose-time" שלמעלה:
> החשש ("קופי מיוצר-מחדש → תמונה לא-תואמת") מנוטרל — המשתמש רואה, יכול לרענן-תמונה (endpoint חדש
> `POST /me/actions/{id}/variations/{i}/regenerate-image`), ורענון-קופי משאיר את התמונה (reserve-first
> idempotent). ה-approve נשאר **צרכן** (`generate_for_action` מדלג על slots מלאים → push). מסלול
> ה-A3-אחרי-approve נשמר כ-**fallback** (approve בלי תמונות מוכנות → יצירה+push). מתג-החירום הגלובלי
> (`optimization_images_forced_off`) עדיין גובר אבסולוטית. **גם הוויזרד** עבר ל-auto (תמונות עם הקופי, בלי
> כפתור "צור תמונות"), **וגם הסנדבוקס** מציג את התמונות שנוצרו ב-propose (inline — אין worker).

## 3. הצ'קבוקס הדינמי — דגל `include_images`

- **Backend:** פרמטר `include_images: bool` על בקשת ה-approve (chat + פאנל + סאנדבוקס).
- `include_images=True` **ו**-`action_type ∈ {creative_refresh, angle_change, offer_change}` → מסלול היצירה (A3).
- `include_images=False` → **המסלול הקיים** (recycle של התמונה הישנה, text-only). זה גם מצב "בדוק דברים אחרים
  מהר" וגם מתג-חירום per-approval.
- **סאנדבוקס:** צ'קבוקס "כלול יצירת תמונות" (ברירת-מחדל ✔️) מחובר לדגל.
- **פרודקשן:** ברירת-מחדל ON ל-3 הסוגים; **מתג-חירום גלובלי** (`app_settings`) יכול לכפות recycle בכל המערכת.

## 4. סקופ — אילו action types מקבלים תמונות חדשות

| Type | תמונות חדשות? | נימוק |
|---|---|---|
| **`creative_refresh`** | **כן** (כשהדגל ON) | 3 זוויות ייחודיות → 3 תמונות |
| **`angle_change`** | **כן** (כשהדגל ON) | זווית חדשה ראויה לוויזואל חדש |
| **`offer_change`** | **כן** (כשהדגל ON) | הצעה חדשה → קונספט חדש |
| **`screening`** | לא | עובר ב-`lead_form_push_service` (טופס+כותרות על creative קיים) |
| **`rejection_fix`** | **משתנה — ראו סעיף 7** | היום תיקון-בודד; ההחלטה: **רענון מלא 3+3** |

מימוש: gate על `action_type ∈ {creative_refresh, angle_change, offer_change}` **וגם** `include_images`.
ה-create loop משותף לשלושתם — אותו תיקון מכסה את כולם. screening נשאר recycle (מתועד, כלל 12).

## 5. Idempotency / resume — אבן-הפינה (אפס double-spend, אפס מודעה כפולה)

> זו עבודת-**נכונוּת** (לא עלות) — נשמרת במלואה. כפילות מודעה/creative ב-Meta = באג חמור, לא "גרוש".

**האינווריאנט:** ה-`image_path` של slot `i` נשמר ל-`generated_image_paths[i]` **לפני** כל צעד בלתי-הפיך שצורך אותו.

1. **עמודה חדשה** `generated_image_paths text[]` על `optimization_actions` (migration; הטבלה נכתבת ע"י
   `service_role` → עוקף RLS+GRANT). `update_action_state` גנרי → אין שינוי חתימה.
2. **A3:** ה-handler שומר `generated_image_paths[i]` **מיד אחרי `storage_service.upload`** (לפני
   `create_signed_url`). retry → אם מלא, דלג. ה-push לעולם לא מייצר — רק `_resolve_generated_image(path)`
   (חתימה-מחדש + download). path קבוע → אותם bytes בכל resume → idempotent מול ה-guards הקיימים.
3. **חלון-שארית מתועד (יושר):** crash **בין** החזרת gpt-image-2 ל-`upload` → יצירה חוזרת. חלון-קריאה-יחידה
   שיורי (כמו ה-wizard). מתועד ב-ROADMAP; אסור לטעון "אין מסלול שמייצר מחדש".
4. **חתימה-מחדש בכל download** — לעולם לא להוריד מ-signed URL שנלכד ביצירה (TTL 7 ימים → 403 → retry אינסופי).
5. **זיווג פוזיציוני מפורש:** ה-handler וה-push עוברים על אותו `action.variations` הגולמי באותו סדר (ה-push
   משתמש בסדר הגולמי השמור, שורות 398-401 — **לא** ב-sorted). אסור ל-handler למיין. טסט-רגרסיה + assert על angle.

## 6. שינויי קוד פר-קובץ (כולל תיקוני הביקורת)

### 6.1 — 4 פרומפטי ה-propose (`app/prompts/agent/{solution_copy,angle_change_copy,offer_copy,filtering_copy}.txt`)
הוספת שדה-פלט `image_concept` (אנגלית, כמו `phase3/copy_generation.txt`) לצד `angle/headline/body`. `cta` נשאר בחוץ.

### 6.2 — `SolutionVariation` (`solution_service.py:70`)
`image_concept: str | None = None` (חדש, optional → תאימות-לאחור: variations שמורים, rejection, מסלולים שלא פולטים).

### 6.3 — `_parse_variations` (`solution_service.py:123`) — קריאה **optional** (לא חובה! אחרת שובר data ישן).

### 6.4 — שמירת image_concept ב-**3 אתרי-שחזור** (סימטריה, כלל 12)
(1) `optimization_push_service._normalize_variations:215`; (2) **`optimization_views.py:_normalize_variations:55`**
(נורמלייזר הפאנל — אחרת approve-פאנל נופל בשקט ל-recycle); (3) `handlers.py:handle_meta_rejection:624` (לתיעוד).

### 6.5 — עמודת `generated_image_paths text[]` (migration + מודל) — סעיף 5.

### 6.6 — helper Storage משותף `image_service.store_generated_image(user_id, key, bytes) -> (path, url)`
(חילוץ מ-`_generate_and_store`; reuse ל-wizard ולאופטימיזציה).

### 6.7 — worker handler `handle_optimization_refresh_images` (A3)
פר slot של `action.variations` הגולמי, idempotent (כלל 9): מלא → דלג; אחרת בנה
`CopyVariant(angle="", headline, body, cta="", image_concept=(v.get("image_concept") or ""))` *(angle=""; `or ""`
מונע "None" מילולי)*, פתור context admin-side (`fetch_quiz_for_campaign`+`get_or_extract_business_context`+`answers`),
`generate_image_bytes_for_variant` → `upload` → **persist path מיד** → המשך. שגיאות: transient→raise;
permanent-per-item→סמן slot; no-key (misconfig)→skip+alert edge-triggered (לא terminal).

### 6.8 — `optimization_push_service.push` create loop (434-450) — gated
```python
if action_type in _IMAGE_GEN_ACTIONS and include_images and variation.get("image_concept"):
    if not (i < len(gen_paths) and isinstance(gen_paths[i], str) and gen_paths[i]):
        raise PushUnavailableError()              # לא מוכן → נשאר pushing → resume (לא IndexError/יצירה)
    image_bytes = await _resolve_generated_image(gen_paths[i])
else:
    old = live_old[i]
    image_bytes = await _download_image(await _resolve_image_url(old))   # FIX re-sign (סעיף 8)
```

### 6.9 — `ads_service.sync_ads_after_swap` — שמירת `image_path` החדש + signed URL טרי על שורות ה-ads החדשות.

### 6.10 — approve flow (חוזה אסינכרוני) — **2 נקודות-כניסה** (chat `agent.py:454` + פאנל `optimization_views.py:144`)
השינוי בליבה המשותפת `execute_approval` (או בשתיהן סימטרית). הרצף: [gate "קמפיין משלם"] → **claim** (`open_push_action`
NULL→pushing, חוסם double-submit) → **enqueue** image-gen → job handle (polling). push (צרכן) רץ כש-paths מלא;
כשל-יצירה permanent → terminal `push_failed` reason `image_gen_failed` (**לא** נשאר pushing → אחרת reconciler מתריע שווא).
`include_images=False` → דילוג על enqueue, ה-push הסינכרוני הקיים רץ כרגיל.

## 7. דחייה מ-Meta → רענון מלא 3+3 (החלטת בעלים) + הערות-מסירה

> **✅ מומש כ-אפשרות B (כירורגי 1+1), לא 3+3 — עדכון-הכרעה (PR-R).** בחקירת-המימוש התגלה ש**הסטטוס המקומי
> של מודעה נדחית נשאר `'live'`** (לא נכתב 'rejected') — כך שאין סיבוך טכני. אך הוצגה תופעת-לוואי: 3+3 מחליף
> **גם את 2 המודעות התקינות** (מאפס פאזת-למידה + עלות 3 תמונות). **הבעלים בחר B (כירורגי):** תמונה חדשה (A3)
> למודעה הנדחית **בלבד**, 2 התקינות נשמרות — מטפל במוטיבציה ("לא למחזר תמונה ישנה בתיקון") בלי הקולטרל. מומש
> דרך `push_rejection_fix` הקיים (single-ad) + gate מקור-תמונה + `claim_and_enqueue_rejection_images` + ניתוב
> ב-handler המאוחד; ללא rewrite, ללא step_plan/window-threading. הסעיף למטה (3+3) נשמר כתיעוד-ההחלטה-המקורית.

**ההחלטה המקורית (הוחלפה ב-B):** דחיית מודעה (במיוחד כשמקורה בתמונה החדשה) → **לייצר מחדש 3 קופי + 3 תמונות**
(רענון מלא), במקום התיקון-הבודד של היום.

- **שינוי לזרימת הדחייה:** היום `generate_rejection_fix` מחזיר **וריאציה אחת** (טקסט) על התמונה הישנה. עכשיו:
  דחייה → רענון מלא (3 וריאציות + 3 תמונות), כמו creative_refresh. זה **שינוי מהותי** למסלול הדחייה (1→3) —
  לפרט בעיצוב: granularity הטריגר (פר-מודעה-שנדחתה מול פר-סט), וכיצד משתלב במנוע-השלבים.
- **למה זה טוב:** קופי חדש מטפל בדחייה שמקורה בטקסט, תמונות חדשות בדחייה שמקורה בתמונה — מכסה כל סיבה. (אין
  עוד "fallback לתמונה ישנה" — בוטל לבקשת הבעלים.)
- **נקודה אחת לתיעוד:** כל סבב מייצר תוכן **חדש** (לא אותו נכס) → אין לולאת אותו-נכס; אך תמונת-AI חדשה *יכולה*
  להידחות שוב → סבב נוסף. נדיר. (לשקול תקרת-סבבים-לדחייה אם יתברר כבעיה.)

**הערות-מסירה (לתעד ב-ROADMAP, לא לחסום):** מודעות חדשות מאפסות פאזת-למידה בכל מקרה; מיחזור התמונה שמר "עוגן"
מוכח — 3 תמונות חדשות הן מהלך גדול יותר על קמפיין שכבר מתפקד גרוע. **הדגל הדינמי (#4) + מתג-החירום הגלובלי (#7)**
נותנים שליטה מלאה לחזור ל-recycle מיידית.

## 8. סימטריה (כלל 12) — מה זז יחד

- **2 נורמלייזרים:** push + פאנל (6.4).
- **2 נקודות approve:** chat + פאנל (6.10).
- **3 אתרי-recycle עם באג ה-7-ימים:** `optimization_push:437`, `lead_form_push:192`, `rejection_push:214` — כולם
  `_download_image(raw image_url)` בלי re-sign. **תיקון-שורש ל-3 באותו PR** (`_resolve_image_url` משותף, transient
  על Storage-glitch) או carve-out מתועד ב-`backend-gaps.md`. אסור לתקן 1 מ-3.
- **3 אתרי-שחזור-variation:** 6.4.

## 9. סאנדבוקס — תמונות אמיתיות + צ'קבוקס (תוצר-הלוואי)

1. **תמונות אמיתיות:** הסאנדבוקס מייצר תמונות OpenAI אמיתיות (בדיקה אמינה מקצה-לקצה) — **בלי** placeholder.
   (ה-seam על Meta נשאר no-op; OpenAI/Storage אמיתיים — וזה רצוי לבדיקה.)
2. **צ'קבוקס** "כלול יצירת תמונות" (ברירת-מחדל ✔️) → דגל `include_images`. מבטלים כדי לבדוק זרימות אחרות מהר.
3. **הרצת ה-handler ישירות:** כמו ש-`advance_time` קורא ל-`run_tick` ישירות, הסאנדבוקס יקרא ללוגיקת
   `handle_optimization_refresh_images` ישירות (לא דרך התור) ואז push — מריץ את **אותו קוד** של פרודקשן (כלל 12).
4. **פער-context:** `create_sandbox_campaign` לא כותב `business_name`/`audience` → backfill ב-create
   (`business_name`=service_name, audience גנרי) + degrade-graceful בבונה-הפרומפט.
5. **הזכייה:** זו בדיוק תצוגת-ה-preview של המסמך המקורי — מסופקת כתוצר-לוואי של הבדיקה האמיתית.

## 10. Edge cases

- **angle vocabulary:** `CopyVariant.angle` = `Literal['emotional',...]`, אך וריאציות נושאות זוויות-פרוטוקול
  (`pain/dream/...`). `_build_image_prompt` angle-agnostic → לבנות עם `angle=""` (לא להזריק זווית-פרוטוקול ל-Literal).
- **image_concept=None → "None":** `_wrap_untrusted(str(None))` → "None" מילולי. guard `or ""`; falsy ל-action שמייצר
  → fallback recycle ל-slot (לא לייצר על פרומפט-"None").
- **no-key (כלל 10/Cron 11):** misconfig גלובלי → skip + alert edge-triggered, לא terminal פר-slot.

## 11. טסטים (נכונוּת — הליבה; "בדיקות קריטיות, מוצר ללא-באגים")

- `_parse_variations`: image_concept קיים→נשמר; חסר→None **בלי raise** (רגרסיה ל-data ישן/rejection).
- 2 הנורמלייזרים (push + פאנל): image_concept נשמר; approve-פאנל של 3-הסוגים מעביר אותו.
- **resume-idempotency (אבן-פינה):** crash אחרי persist-path לפני creative → resume מוריד אותו path, **בלי
  gpt-image-2 שני, בלי creative/ad כפול**.
- handler idempotency: path מלא → דילוג.
- **דגל `include_images`:** OFF → recycle (אפס gpt-image-2); ON → יצירה. פר 3 הסוגים.
- gate action_type: 3 הסוגים מייצרים; screening/rejection-ישן → recycle.
- סאנדבוקס: צ'קבוקס OFF → recycle; ON → תמונה אמיתית (בטסט: seam של ה-image-gen ל-bytes דמה); business_name/
  audience חסרים → לא קורס.
- `_resolve_image_url` ב-3 האתרים: image_url נושן + image_path תקף → re-sign; Storage-fail → 503/transient (לא rollback).
- דחייה → רענון 3+3 (לא תיקון-בודד); הזרימה החדשה.
- `tests/test_transient_mixin.py` (CI): כל `*UnavailableError` חדש יורש `TransientError`.

## 12. מסמכים לעדכון (אותו PR)

- `docs/ROADMAP.md` — תיעוד **היפוך** החלטת text-only (~12976/13151) + הערות-מסירה + חלון-השארית; ✅.
- `docs/spec.md` — התנהגות התמונה ב-3 סוגי-הרענון + הדגל הדינמי + שינוי זרימת-הדחייה (החלטות-מוצר).
- `docs/SETUP_CHECKLIST.md` — אין env חדש (OpenAI/Storage קיימים); מתג-החירום ב-`app_settings`.
- `docs/frontend-integration.md` — approve אסינכרוני (A3 polling) + צ'קבוקס ל-**שני** המסכים (chat + פאנל) + הסאנדבוקס.
- `docs/backend-gaps.md` — carve-out אם תיקון-3-האתרים לא נסגר כולו.
- `docs/sandbox-image-renderer-plan.md` — superseded ע"י מסמך זה.

## 13. סדר ביצוע

1. **חוזה/סכמה (אפס שינוי-התנהגות):** image_concept ל-4 פרומפטים + `SolutionVariation` + `_parse_variations`
   (optional) + 2 הנורמלייזרים. טסטים. *(תאימות-לאחור.)*
2. **Migration + מודל:** `generated_image_paths text[]`.
3. **helper Storage** `store_generated_image` + refactor ה-wizard.
4. **תיקון-שורש 7-ימים** ב-3 אתרי-recycle (`_resolve_image_url`). *(עומד בפ"ע.)*
5. **handler** `handle_optimization_refresh_images` (context admin + reserve-first; סאנדבוקס מריץ ישירות).
6. **create loop** gated (3 סוגים + `include_images`) + guard ה-path. **טסט resume-idempotency.**
7. **`ads_service` sync** — image_path + URL טרי.
8. **approve** (2 נקודות/execute_approval): gate → claim → enqueue → polling + דגל `include_images`; push gated.
9. **דחייה → 3+3** (שינוי `generate_rejection_fix`/זרימת הדחייה).
10. **סאנדבוקס:** צ'קבוקס + תמונות אמיתיות + context backfill; **מתג-חירום גלובלי** ב-`app_settings`.
11. **מסמכים** + ✅.

---

## נספח: ממצאי ביקורת-היריב (severity, אומת מול הקוד) — עבודת-נכונוּת

**HIGH:** (1) gate "images-ready" + guard ה-path (אחרת IndexError/יצירה-בתוך-request). (2) זיווג פוזיציוני
handler↔push — סדר-גולמי זהה (פין בקוד+טסט). (3) crash אחרי OpenAI לפני persist = double-spend → reserve-first
מיד אחרי upload + תיעוד חלון-שארית. (4) **נורמלייזר שני בפאנל** מפשיט image_concept. (5) **2 נקודות approve**.
(6) **לולאת-דחייה** — נפתר ע"י החלטת 3+3 (סעיף 7).
**MEDIUM:** finalize/reconciler false-alert (permanent-gen→terminal); image_concept=None→"None"; no-key→degradation
(לא terminal); **3 אתרי-recycle** יחד; short-circuit/זיהוי-סאנדבוקס דרך fetch_is_sandbox (transient→propagate);
angle vocabulary (angle=""); latency worst-case ~דקות (מחזק A3).
**LOW:** ads sync threading של image_path; transient-mapping ב-port; אתר-שחזור-variation שלישי לתיעוד.

> **הערה:** ה-quota המלאכותי (HIGH בביקורת המקורית) — **בוטל** בהחלטת הבעלים (סעיף 0#5); הבלימה הטבעית היא
> מספר-השלבים-הקבוע בפרוטוקול. ה-placeholder בסאנדבוקס (MEDIUM) — **בוטל**; הסאנדבוקס מייצר תמונות אמיתיות (#6).
