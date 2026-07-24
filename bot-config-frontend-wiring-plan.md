# תוכנית מימוש: חיווט הגדרת-הבוט לפרונטד (וויזרד + עמוד-הגדרות בדשבורד)

> **סטטוס:** טיוטה לאישור (טרם מומש). **ענף:** `claude/sandbox-3-image-renderer-qyk2rn`
> **מטרה:** לחבר את הגדרת-הבוט מה-UI אל ה-backend הקיים (`PUT /bot/config`), בשני מקומות:
> (א) **שלב-הבוט בוויזרד** (onboarding חד-פעמי); (ב) **עמוד-הגדרות חדש בדשבורד** (עריכה מתמשכת).
> ה-backend כבר מלא ועובד (`bot.py` + `bot_config_service.py` + RPC `upsert_bot_config`) — זו עבודת-**frontend**.

---

## 0. החלטות פתוחות — לאישור (פערי-שדות)

ה-UI הנוכחי **לא** אוסף את כל מה שה-backend צריך (`BotConfigInput`). שלושה פערים אמיתיים:

| # | שדה חסר | מקור אפשרי | המלצה |
|---|---|---|---|
| **1** | **`service_label`** (1–80, "הטיפול"/"הקורס" שבפתיחה) | שדה-קלט חדש קצר / `WIZ.data.serviceName` / default | **שדה חדש קצר** ("איך נקרא לשירות בהודעה? למשל 'הטיפול'") — זה בדיוק מה שמופיע בפתיחה |
| **2** | **`closing_cta`** (1–30, משפט-סיום הפתיחה) | שדה-קלט חדש / default | **שדה חדש קצר** עם default "מעוניין?" |
| **3** | **טלפון בעל-העסק ל-`human_handoff`** (E.164) | שדה-טלפון חדש / טלפון VIP / מספר ה-WhatsApp line | **שדה-טלפון חדש** תחת אופציית "human" — זה המספר שאליו תגיע התראת-הליד, חייב מפורש (היום ה-UI מבטיח "יישלח לוואטסאפ שלך" בלי לדעת לאן) |
| **4** | **תיאום-תור native** (`bot_schedule_appointment`) | אופציה שלישית בפעולת-הסיום (דורש יומן Google מחובר) | **v1: רק Calendly + human** (כמו ה-UI הקיים); native = הרחבה עתידית |

`business_name` כבר זמין מ-`WIZ.data.bizName` (נאסף בשלב wiz-business). `default_appointment_duration_minutes`
נשאר default=60 (רלוונטי רק ל-native scheduling).

---

## 1. רקע: הפער הנוכחי

- **`wizCollectBot()` מיושן** (`index.html:2779`): מחזיר `{openMsg, closing, calendarUrl, questions}` — קורא
  `#bot-open-msg` ש**כבר לא קיים** (ב-5.4 הוחלף ל-3 שדות-template), ולא אוסף `service_label`/`closing_cta`/טלפון.
- **אין קריאת-API:** השלב רק שומר ל-`WIZ.data.bot` (מסומן "לחיבור API עתידי"). grep ל-`bot/config` ב-`app/web/`
  מחזיר ריק → **הנתונים נאבדים** כשסוגרים את המסך.
- **אין עמוד-עריכה בדשבורד** — בעל-העסק לא יכול לשנות שאלות/פעולת-סיום אחרי ה-onboarding.

## 2. שתי המשימות

- **משימה A — חיווט הוויזרד:** בשלב `wiz-bot`, לאסוף נכון → `api.saveBotConfig` → המשך ליצירת-קמפיין.
- **משימה B — עמוד-הגדרות בדשבורד:** `view-bot` חדש (כמו `view-business`) שטוען (`GET /bot/config`) ושומר
  (`PUT /bot/config`) — לעריכה בכל עת.

שתיהן חולקות את אותו backend ואת אותם 2 methods חדשים ב-`api.js`.

## 3. `api.js` — 2 methods חדשים (דפוס קיים)

```js
/* getBotConfig: GET /bot/config → BotConfigResponse, או null אם טרם הוגדר (404). (כמו getBusinessProfile.) */
async function getBotConfig() {
  const res = await apiFetch('/bot/config', { method: 'GET' });
  if (res.status === 404) return null;
  return jsonOrThrow(res);
}
/* saveBotConfig: PUT /bot/config (Premium) → BotConfigResponse. זורק ApiError(status):
   402 לא-Premium · 422 ולידציה · 412 יומן לא-מחובר · 503. */
async function saveBotConfig(body) {
  const res = await apiFetch('/bot/config', { method: 'PUT', body: JSON.stringify(body) });
  return jsonOrThrow(res);
}
```
להוסיף ל-`window.api = {...}` (שורה 506).

> **⚠️ caveat 412:** ב-`jsonOrThrow` (שורה 79) `detail = j.detail || j.message`. ב-412 ה-`detail` הוא **dict**
> (`{message, code, action_required}`) → `ApiError.message` הופך ל-"[object Object]". **פתרון:** ה-UI מסתעף
> על `err.status === 412` ומציג הודעה קבועה ("חבר יומן Google תחילה") — לא מסתמך על `err.message`. (רלוונטי רק
> אם נוסיף native scheduling — סעיף 0#4.)

## 4. מיפוי נתונים: UI → `BotConfigInput`

| שדה backend | מקור ב-UI |
|---|---|
| `business_name` | `WIZ.data.bizName` (וויזרד) / שדה בעמוד-הגדרות |
| `service_label` | **שדה חדש** (0#1) |
| `closing_cta` | **שדה חדש** (0#2) |
| `fallback_action` | radio: `calendar` → `"calendly_link"`; `human` → `"human_handoff"` |
| `fallback_value` | `calendly_link` → `#bot-calendar-url`; `human_handoff` → **שדה-טלפון חדש** (0#3) |
| `questions` | `[{question_text: i.value}]` מ-`#bot-questions-list .bot-q-input` (לא-ריקות) |
| `default_appointment_duration_minutes` | default 60 (לא נחשף ב-v1) |

## 5. משימה A — חיווט הוויזרד

### 5.1 שינויי UI (שלב `wiz-bot`, ~שורה 2041)
- להוסיף **שדה `service_label`** קצר (0#1) ו-**שדה `closing_cta`** קצר (0#2) בעמודת-ההגדרות.
- תחת אופציית "human" (שורה 2074-2077) — להוסיף **שדה-טלפון** (`#bot-handoff-phone`, placeholder `+9725...`).
- להסיר את ההתייחסות ל-`bot-open-msg` (מיושן).

### 5.2 שכתוב `wizCollectBot()` (שורה 2779) → `BotConfigInput`
```js
function wizCollectBot(){
  const closing = document.querySelector('input[name="bot-closing"]:checked')?.value || 'calendar';
  const action = closing === 'human' ? 'human_handoff' : 'calendly_link';
  return {
    business_name: (WIZ.data.bizName||'').trim(),
    service_label: (document.getElementById('bot-service-label')?.value||'').trim(),
    closing_cta:   (document.getElementById('bot-closing-cta')?.value||'').trim(),
    fallback_action: action,
    fallback_value: action === 'calendly_link'
      ? (document.getElementById('bot-calendar-url')?.value||'').trim()
      : (document.getElementById('bot-handoff-phone')?.value||'').trim(),
    questions: Array.from(document.querySelectorAll('#bot-questions-list .bot-q-input'))
      .map(i=>i.value.trim()).filter(Boolean).map(t=>({question_text:t})),
  };
}
```

### 5.3 נקודת-החיבור + שמירה (שורה 4481-4484)
היום: `WIZ.data.bot=wizCollectBot(); wizStartLoading();`. החדש — **ולידציה צד-לקוח (5.4) → `saveBotConfig` →
המשך**. כשל = להישאר בשלב עם הודעת-שגיאה (לא להמשיך ליצירת-קמפיין):
```js
} else if(WIZ.step==='wiz-bot'){
  const cfg = wizCollectBot();
  const err = validateBotConfig(cfg);            // 5.4 — ולידציה מקומית
  if(err){ showBotWizError(err); return; }
  try{
    await window.api.saveBotConfig(cfg);
  }catch(e){ showBotWizError(mapBotConfigError(e)); return; }  // 7 — מיפוי שגיאות
  WIZ.data.bot = cfg;
  wizStartLoading();
}
```
*(הופך את ה-handler ל-async אם אינו; הכפתור מושבת בזמן השמירה — כמו `ob-finish`.)*

## 6. משימה B — עמוד-הגדרות בדשבורד (`view-bot`)

חיקוי מלא של `view-business` (`index.html:1524`, `saveBusinessSettings`, `loadBusinessSettings`,
`showView`). בעל-העסק יכול לערוך **בכל עת** (אין נעילת-קמפיין — הבוט עצמאי מקמפיינים).

### 6.1 ניווט (~שורה 1126)
כפתור-ניווט חדש, **Premium בלבד** (כמו `nav-business`, `display:none` + מוצג כש-`APP.plan==='premium'`):
```html
<button class="ni" id="nav-bot" data-view="bot" style="display:none">
  <span class="ni-ico">🤖</span><span><div ...>סוכן הוואטסאפ</div><div ...>שאלות, פתיחה ופעולת-סיום</div></span>
</button>
```

### 6.2 ה-view (כמו `view-business`)
`<div id="view-bot">` עם: topbar + עורך-שאלות (אותו `bot-questions-list`/`botAddQuestion`, reuse) + שדות
`business_name`/`service_label`/`closing_cta` + בורר-פעולת-סיום (Calendly URL / טלפון-handoff) + `#bot-set-msg`
inline + כפתור-שמירה.

### 6.3 `showView` (שורה 5529) + טעינה
- להוסיף `'bot'` למערך ה-views.
- `if(v==='bot')loadBotSettings();` (שורה ~5536).
- `loadBotSettings()` — `GET /bot/config`; אם `null` (404) → טופס ריק; אחרת prefill את כל השדות + השאלות.
- `saveBotSettings()` — ולידציה (5.4) → `api.saveBotConfig` → `#bot-set-msg` ירוק/אדום (כמו `saveBusinessSettings`).
- export ב-`window` (שורה 5549).

## 7. ולידציות + מיפוי שגיאות (חוויית-משתמש)

### 7.1 ולידציה צד-לקוח (`validateBotConfig`, fail-fast לפני הקריאה)
מראה את הולידציה בשרת → הודעת עברית ידידותית:
- `business_name` 1–50 · `service_label` 1–80 · `closing_cta` 1–30 (לא-ריקים, באורך).
- כל שאלה 5–500 תווים; עד 5 שאלות; 0 מותר (הבוט פותח ועובר ישר לפעולת-סיום).
- `calendly_link` → URL לא-ריק שמתחיל ב-`https://`.
- `human_handoff` → טלפון תואם `^\+\d{9,15}$` (E.164).

### 7.2 מיפוי שגיאות שרת (`mapBotConfigError(err)`)
| status | משמעות | הודעה למשתמש |
|---|---|---|
| **402** | לא-Premium | "הגדרת הסוכן זמינה במסלול Premium." |
| **422** | ולידציה / fallback_value | להציג את `err.message` (השרת מחזיר עברית ידידותית, למשל "קישור ה-Calendly חייב להתחיל ב-https://") |
| **412** | יומן Google לא-מחובר (רק native scheduling) | "כדי לאפשר תיאום-תור יש לחבר יומן Google תחילה." + קישור-חיבור (לא `err.message` — caveat §3) |
| **503** | תקלה חולפת | "השירות אינו זמין כעת, נסה שוב בעוד רגע." |
| אחר/5xx | לא-צפוי | "אירעה שגיאה בשמירת ההגדרות, נסה שוב." |

## 8. Edge cases

- **412 dict detail** — סעיף 3 (UI מסתעף על status, לא על message).
- **Premium gating** — `nav-bot` ושלב-הוויזרד מוצגים רק ל-Premium; ה-backend אוכף 402 ממילא (defense-in-depth).
- **0 שאלות** — חוקי (הבוט עובר ישר לפעולת-סיום) — אסור שהולידציה המקומית תחסום.
- **handoff בלי טלפון** — היום הפער; השדה החדש (0#3) פותר. בלעדיו השרת מחזיר 422 ברור.
- **Calendly מול native** — v1: "calendar" = `calendly_link` (URL). native (`bot_schedule_appointment`) = הרחבה
  (דורש יומן מחובר → 412/422).
- **idempotency** — `PUT` הוא upsert מלא; שמירה חוזרת מ-הדשבורד פשוט דורסת — תקין.

## 9. טסטים

- **backend** — כבר מכוסה (`bot_config_service`). אין שינוי-שרת.
- **frontend (ידני/e2e):** וויזרד Premium → מילוי → שמירה → לוודא `GET /bot/config` מחזיר את הערכים; וריאציות:
  Calendly עם URL לא-https → הודעה; human בלי טלפון/טלפון פסול → הודעה; לא-Premium → 402; עמוד-הדשבורד טוען
  ושומר; עריכה חוזרת דורסת.

## 10. מסמכים לעדכון (אותו PR)

- `docs/frontend-integration.md` — לעדכן את שורת ה-wizard (שלב-הבוט מחווט ל-`PUT /bot/config`) + שורה חדשה
  לעמוד-ההגדרות (`view-bot` → `GET/PUT /bot/config`).
- `docs/backend-gaps.md` — לסגור את הפער "הגדרת-בוט לא מחוברת מה-frontend" (אם רשום שם).
- `docs/spec.md`/`prototype/frontend-changes.md` — אם נוספים שדות-UI (service_label/closing_cta/handoff phone).

## 11. סדר ביצוע

1. `api.js` — `getBotConfig` + `saveBotConfig` (+ export).
2. **משימה A:** שדות-UI חדשים בשלב `wiz-bot` → שכתוב `wizCollectBot` → `validateBotConfig`/`mapBotConfigError` →
   חיבור השמירה בנקודת-המעבר.
3. **משימה B:** `nav-bot` (Premium) + `view-bot` (חיקוי `view-business`) → `loadBotSettings`/`saveBotSettings` →
   `showView`.
4. ולידציות + הודעות-שגיאה (משותף ל-A ו-B).
5. מסמכים.

---

## נספח: עוגני-קוד (אומת מול הריפו)

- `api.js`: דפוס `apiFetch`+`jsonOrThrow`+`ApiError` (שורות 49-84); `getBusinessProfile` null-on-404 (459-464);
  `window.api={...}` (506).
- וויזרד: `wiz-bot` HTML (2041-2099), `wizCollectBot` מיושן (2779), `botAddQuestion` (2788), נקודת-המעבר (4481-4485).
- דשבורד: ניווט `data-view` (1120-1127), `view-business` כמודל (1524-1544), `saveBusinessSettings` (5501-5526),
  `_applyBizEditLock` (5494), `showView` (5528-5538), exports (5549).
- backend (ללא שינוי): `routers/bot.py` (GET/PUT `/bot/config`, מיפוי 402/422/412/503), `bot_config_service.upsert_config`
  + `_validate_fallback`, מודל `BotConfigInput` (`models/bot.py`).
