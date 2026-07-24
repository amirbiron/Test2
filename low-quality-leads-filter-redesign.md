# Redesign בעיה 2 (low_quality_leads) — `filter_addon`

> **סטטוס:** ✅ מומש (PR1–PR4, branch `claude/eloquent-bardeen-9kGfT`). מבוסס על Session 7.5 (ROADMAP).

## Context — מה ולמה

בעיה 2 (`low_quality_leads`) ייצרה **3 קופי חדשים מאפס** (LLM) לכל קטגוריה. עבור `wrong_area`/`no_budget` זו
טעות-תכנון: הבעיה אינה שהקופי גרוע — אלא ש**חסר בו פרט מסנן** (אזור שירות / טווח מחיר). רינדור-מאפס זורק נכס
עובד ומכניס מסר לא-בדוק כדי להוסיף משפט אחד. **הפתרון: עריכת 3 המודעות החיות + משפט מסנן** (edit-in-place),
עקבי עם "מניעת שינויים מיותרים" (#3 ב-`core.txt`) מעל "שיפור כמות פניות" (#5). חוק 7 לכל האורך.

## ההכרעות

| נושא | הכרעה |
|------|-------|
| קטגוריות | רק `wrong_area` + `no_budget` (`dont_understand`/`standards` בוטלו מה-chips ומה-Literal; נשארות ב-i18n לקריאה בטוחה של סדרות באמצע). |
| שיטה | עריכת הקופי הקיים + משפט מסנן (לא רינדור). |
| מנוע-שלבים | `low_quality` → `{1: filter_addon, 2: screening_question}` (שלבי angle_change נפלו). |
| `filter_value` | ב-`optimization_sessions.starting_metric` ליד `subcategory` (jsonb, בלי migration). |
| `wrong_area` ארצי | שאלת-אזור פתוחה. `no_budget` | טקסט-מחיר חופשי (בלי פירסור, בלי מטבע — ₪ מובלע). |

## ה-crux הטכני — זיווג זוויות (קריטי)

ה-`push` מזווג `variation[i] ↔ live_old[i]` **פוזיציונית, אחרי מיון לפי `_angle_rank`** (לא לפי התאמת-זווית).
ל-`creative_refresh` הזוויות שונות (pain/dream/urgency מול emotional/...) → זיווג שרירותי-אך-יציב. ל-`filter_addon`
עורכים **מודעה ספציפית** וצריכים שהקופי הערוך יחזור ל**תמונה הנכונה**, לכן:

`_FILTER_ADDON.angles = ("emotional", "pain_solution", "result_success")` (== `_OLD_AD_ANGLE_ORDER` של
`optimization_push_service`) ⟹ שני המיונים זהים ⟹ `variation[i].angle == live_old[i].angle` ⟹ כל קופי ערוך
נוחת על מודעת-המקור שלו. **אפס שינוי ל-push/normalize/gate/sync.** `_parse_variations(bind_angles=...)` אוכף את
הזיווג הפוזיציוני (וגם backstop מול prompt-injection מהקופי הקיים).

## הזרימה

```
diagnose_problem_2 (תלת-שלבי):
  שלב 1 (subcategory=None) → 2 chips
  שלב 2 (subcategory בלי value) → value_question (אזור מ-quiz.location / שאלה; או מחיר) — בלי סדרה
  שלב 3 (subcategory+value) → open_session(starting_metric={subcategory, filter_value}) → ניתוח
propose → _generate_filtering → ענף filter_addon:
  fetch_ads_with_copy → 3 מודעות חיות (סינון live + angle ב-_OLD_AD_ANGLE_ORDER + copy; מיון)
  → prompt filter_addon (edit-in-place; untrusted: filter_value+source_block) → _run_copy_llm(bind_angles)
  → push (זיווג פוזיציוני) → window 120ש' → QUALITY_FOLLOWUP → "השתפר?" → לא → שלב 2 (screening)
```

## קבצים

| קובץ | שינוי |
|------|-------|
| `app/models/agent.py` | `Subcategory`→2; `DiagnoseRequest.value` (+validator); `DiagnoseResponse.type`+`value_question` |
| `app/services/agent_orchestrator.py` | `_LOW_QUALITY_CHIPS`→2; `diagnose_problem_2` תלת-שלבי; `_default_area_from_quiz`; צ'יפים/הודעות value |
| `app/services/optimization_service.py` | `ACTION_FILTER_ADDON`+`_FILTER_ADDON`; `_STEP_PLANS[low_quality]={1:filter_addon,2:screening}` |
| `app/services/optimization_push_service.py` | `_CHANGE_DESCRIPTIONS[filter_addon]` |
| `app/services/ads_service.py` | `fetch_ads_with_copy` (headline/body — `fetch_ads_for_push` בלעדיהם) |
| `app/services/solution_service.py` | ענף `_generate_filter_addon`+`_filter_value_of`+`_filter_addon_source_block`; `_parse_variations`/`_run_copy_llm` +`bind_angles`; `_filtering_step_label` |
| `app/services/agent_service.py` | `FilterAddonAdsError` (fail-fast → 409) |
| `app/routers/agent.py` | `diagnose` מעביר `value`; מיפוי `FilterAddonAdsError` |
| `app/prompts/agent/filter_addon.txt` | prompt edit-in-place (חדש) |
| `app/routers/agent.py`/`app/web/api.js`/`app/web/index.html` | חיווט ה-value-turn (frontend) |
| `app/models/sandbox.py`/`app/services/sandbox_service.py`/`app/routers/admin/sandbox.py`/`templates/admin/sandbox.html` | `value` ב-סנדבוקס |

## טיפול בכשלים

- **count≠3 חי** (מודעה נדחתה/הושהתה) → `FilterAddonAdsError` (409, fail-fast ביצירה — עדיף על equality-gate
  גנרי ב-push). **חסר filter_value** (סדרה לפני ה-redesign) → אותו error. סימטריה (כלל 12) wrong_area↔no_budget.
- **prompt-injection** מהקופי הקיים → `untrusted` (PR16 wrap+sanitize) + `bind_angles` backstop ב-parse.

## Verification

1. `python -m pytest -q` ירוק + `python -c "import app.main"`.
2. **crux:** filter_addon עם 3 מודעות → `_parse_variations(bind_angles)` אוכף זיווג; push מצמיד כל variation
   למודעת-המקור (תמונה+angle משומרים).
3. **ידני בסנדבוקס:** low_quality → 2 chips → wrong_area → value_question (ברירת-מחדל/שאלה) → ערך → propose →
   3 וריאציות **שמשמרות מסר+זווית** + משפט מסנן → approve. `no_budget` → טקסט-מחיר.
4. **רגרסיה:** high_cpl/meta_rejection ללא שינוי; screening עבר לשלב 2.
