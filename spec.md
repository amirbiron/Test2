# Campaign AI — מפרט טכני (spec.md)

> מקור האמת היחיד לפרויקט. כל החלטת מוצר/ארכיטקטורה מתועדת כאן.
> Claude Code עובד לפי המסמך הזה. בכל סתירה בין שיחה למסמך — המסמך גובר.
> עדכון אחרון: יוני 2026

---

## 1. מה המערכת עושה

Campaign AI היא מערכת SaaS לעסקים קטנים שמבצעת אוטומציה מלאה של קמפיינים
ממומנים בפייסבוק. במקום לשכור סוכנות שיווק, בעל העסק מחבר את חשבון הפייסבוק
שלו, עונה על שאלון קצר, וה-AI מייצר ומעלה לאוויר מודעות שמביאות לידים.

**הסוכן הוא מנהל קמפיינים, לא מחולל.** הוא לא רק מקים מודעות — הוא מנטר אותן
ברקע לאורך כל חיי הקמפיין, מזהה בעיות, ומבצע אופטימיזציות (חלקן אוטונומיות).

**עיקרון מרכזי:** כל קמפיין מקדם **שירות אחד ספציפי** של העסק (למשל
"שיעורי נהיגה"). כשליד נכנס, המערכת יודעת בדיוק על איזה שירות מדובר — וזה מה
שמוזרק להודעת הבוט.

---

## 2. הסטאק הטכני

- **Backend:** Python / FastAPI על Render
- **DB:** PostgreSQL על Supabase (עם RLS דלוק על כל טבלה)
- **אימות:** Supabase Auth
- **אינטגרציות:**
  - Meta Graph / Marketing API (OAuth, יצירת קמפיינים, טפסי ליד)
  - OpenAI — ChatGPT API (קופי), gpt-image-2 (תמונות)
  - WhatsApp: **Meta Cloud API** (WABA יחיד שלנו, קווים ייעודיים פר לקוח) — לבוט Premium
  - ספק סליקה אוטומטית: **פלאקארד** (PSP ישראלי, רץ על שב"א). עטוף ב-`integrations/pelecard.py`;
    שכבת abstraction אגנוסטית ב-`integrations/billing.py` כך שהחלפת ספק = שינוי במקום אחד.
- **לידים והודעות בוט** נכנסים דרך webhooks חיצוניים (לא מהמשתמש המחובר)
- **עיבוד אסינכרוני:** טבלת `jobs` ב-Postgres + worker (polling). העומס משלב
  שניים: **נקודתי** (יצירת קמפיין, יצירת תמונות — נכנס לתור לפי פעולת משתמש)
  ו**מתמשך** (ניטור ואופטימיזציה שוטפים על כל קמפיין פעיל, לאורך כל חייו).
  הניטור המתמשך מיושם כ-**cron מתוזמן שמכניס job ניטור לכל קמפיין פעיל במרווח
  קבוע** — מעובד ע"י אותו worker, אותו דפוס כמו cron רענון הטוקן (Phase 6).
  **עדיין בלי Redis/Celery** — התשתית הקיימת (jobs + worker + cron) מספיקה
  לנפח הזה.
- **שני שירותים ב-Render** מאותו repo: web service (FastAPI) + background
  worker (מושך jobs).
- **סביבת פיתוח:** נייד בלבד — Claude Code Web (אין טרמינל מקומי)
- **בדיקות:** סביבת סימולציה עם נתונים מומצאים בודקת את ה-flow המלא ואת
  השכבה הדטרמיניסטית בלי תלות במטא. השכבה החיצונית (Meta/OpenAI) נבדקת עם
  mocks. פלט LLM נבדק לפורמט בלבד (לא לאיכות). e2e — מעט, על מסלולים
  קריטיים. בייצור (לסוכן אוטונומי) — לוג מלא + התראות + מתג עצירה.

---

## 2א. ארכיטקטורה ומבנה הקוד (חוק — לבנות לפי זה מ-Session 0.1)

הקוד **מודולרי, מופרד לשכבות**. router לעולם לא מדבר ישירות עם שירות
חיצוני — הוא קורא ל-service, וה-service קורא ל-integration. כל אינטגרציה
חיצונית עטופה במודול משלה, כך שהחלפת ספק = שינוי במקום אחד בלבד.

```
app/
├── main.py              # הרכבת האפליקציה בלבד. אין לוגיקה.
├── config.py            # env vars (pydantic-settings). אין ערכים בקוד.
├── routers/             # endpoints בלבד — קלט/פלט HTTP. דקים.
│   ├── auth.py
│   ├── campaigns.py
│   ├── ads.py
│   └── webhooks.py      # מבודד. משתמש ב-admin client בלבד.
├── services/            # הלוגיקה העסקית. כאן קורה הכל.
│   ├── auth_service.py          # Supabase Auth: signup/login/refresh/logout (0.5.1)
│   ├── fb_service.py            # Meta OAuth + טוקן מוצפן ב-fb_connections (1.2); מחזיק admin client
│   ├── meta_service.py
│   ├── openai_service.py
│   ├── bot_service.py
│   ├── agent_service.py        # הסוכן הפנימי: צ'אט בדשבורד + פרוטוקול אופטימיזציה
│   └── subscription_service.py   # get_user_subscription; עדכון tier/מכסה (admin client)
├── integrations/        # עטיפות לשירותים חיצוניים. מבודדות.
│   ├── meta.py          # כל הקריאות ל-Meta API
│   ├── openai.py        # ChatGPT + gpt-image-2
│   ├── meta_whatsapp.py # Meta Cloud API (WhatsApp — בוט Premium)
│   └── billing.py       # ספק סליקה (אגנוסטי)
├── worker/              # עיבוד אסינכרוני
│   ├── runner.py        # ה-worker: polling על טבלת jobs, מריץ ומטפל בכשל
│   └── handlers.py      # מה כל סוג job עושה (create_ads, push_campaign...)
├── models/              # סכמות Pydantic (ולידציה)
├── db/
│   ├── client.py        # get_user_client / get_admin_client
│   ├── queries.py       # גישה ל-Supabase
│   └── migrate.py       # מריץ supabase/migrations/*.sql ב-deploy (ראה למטה)
└── core/
    ├── security.py      # אימות חתימת webhook, JWT
    └── deps.py          # FastAPI dependencies
```

**Migrations:** סכמת ה-DB מוגדרת ב-`supabase/migrations/*.sql` (מקור אמת, ממוספר).
מוחלים **אוטומטית ב-deploy** דרך `app/db/migrate.py` (`preDeployCommand` ב-render.yaml),
עם tracking ב-`schema_migrations` ו-`pg_advisory_lock`. דורש חיבור Postgres **ישיר**
(`SUPABASE_DB_URL`, session-mode) — כי PostgREST לא מריץ DDL. כשל ב-migration חוסם את
ה-deploy (fail-closed). הקבצים idempotent. (סוד ה-DB URL הוא migration-only, לא בזמן ריצה.)
**כל migration שמוסיף טבלה חייב גם `GRANT` מתאים** ל-`authenticated`/`anon` לפי מדיניות
ה-RLS — PostgREST בודק הרשאת טבלה **לפני** RLS, ו-Supabase כבר לא חושף אוטומטית טבלאות
public חדשות ל-Data API. RLS לבד → "permission denied". מדיניות SELECT-only → `grant select`.

### חוקי השכבות (לא לעבור)
1. **routers דקים** — מקבלים בקשה, קוראים ל-service, מחזירים תשובה. אפס
   לוגיקה עסקית, אפס קריאות ישירות ל-API חיצוני או ל-DB.
2. **services** — הלוגיקה. קוראים ל-integrations ול-db. לא יודעים על HTTP.
3. **integrations** — רק התקשורת עם השירות החיצוני. אם נחליף את gpt-image-2
   במודל אחר — נוגעים רק ב-`integrations/openai.py`.
   בידוד ה-integrations מאפשר **מצב sandbox פנימי**: `integrations/meta.py`
   תומך במקור-נתונים מוזרק (ידני) במקום Insights אמיתי, וב"ביצוע מזויף"
   שרושם פעולות ל-`optimization_actions` בלי לקרוא ל-Meta API. הסוכן,
   הפרוטוקול וה-DB זהים לייצור — רק הקלט/פלט החיצוניים מוחלפים. כלי פנימי
   בלבד, לא חשוף ללקוח.
4. **כל קריאה ל-subscription** עוברת דרך `subscription_service` — מקום
   אחד, כדי שאם מודל המנוי ישתנה, יש נקודת שינוי אחת.
5. **admin client מבודד** — `supabase_admin` (service_role, עוקף RLS) מותר רק
   במודול ה-webhooks המבודד (`routers/webhooks.py`) וב-services ייעודיים לפעולות
   privileged (`fb_service`, `subscription_service`, הקצאת `whatsapp_lines`).
   אסור לייבא אותו ל-router רגיל מול משתמש או לפזר אותו ב-services אחרים. ראה §7.3.
6. **אטומיות ביצירת קמפיין Meta** — יצירת קמפיין היא רב-שלבית
   (Campaign → Ad Set → 3 Ads). אם שלב נכשל באמצע (למשל מודעה 3 נדחתה),
   אסור להשאיר קמפיין "חצי-בנוי" ב-Meta. צריך state machine שמתעד באיזה
   שלב נמצאים, ו-cleanup/rollback בכשל. ה-status ב-`campaigns` וב-`ads`
   משקף את המצב האמיתי בכל רגע. זה החלק הכי מסוכן — טעות כאן עולה ביוקר.
   **הגנות על פעולות אוטונומיות:** פעולה אוטונומית של הסוכן מול Meta API
   (שינוי קהל / תקציב-בין-סטים / כיבוי מודעה) חייבת: (א) **תקרת שינוי** —
   גבול על כמה ובאיזו תדירות; (ב) **לוג מלא** — כל פעולה מתועדת ב-
   `optimization_actions` עם סיבה + לפני/אחרי (זה מה שמוצג בדשבורד "תהליכי
   שיפור"). באג בפעולה אוטונומית שורף כסף אמיתי של לקוח — הסיכון הגבוה
   ביותר במערכת.
   **ה-status של מודעה אינו טרמינלי ב-`live`:** מטא בודקת מודעות **אחרי**
   שעלו, ותוכן AI נדחה לא מעט. ה-enum כולל `rejected` כמצב הפיך מ-`live`.
   הטיפול בדחייה (webhook להאזנה לשינויי status + תגובה) מתוכנן אך לא נכלל
   ב-MVP — אבל ה-enum מוכן לו מהיום הראשון, כדי שלא יהיה טלאי.
7. **הפרדת LLM מלוגיקה דטרמיניסטית בסוכן.** הסוכן משלב שני סוגי פעולות,
   ואסור לערבב:
   - **LLM (OpenAI):** ניהול שיחה (צ'אט בדשבורד, שיחת הבוט) ויצירת קופי.
     דברים שדורשים ניסוח וגמישות, בלי "תשובה אחת נכונה".
   - **קוד דטרמיניסטי שקורא מה-DB:** הפרוטוקול — איזה שלב אופטימיזציה הבא,
     מתי הלקוח ננעל, מתי חלון ה-5 ימים (120h) נפתח. **אסור** ש-LLM "יחליט" אם
     עברו 5 ימים או איזה שלב נוסה — זה נתון, לא שיקול דעת.
   - **Meta Insights:** כל מספר שהסוכן מצטט (Spend, Leads, CPL, CTR) נשלף
     מ-Insights, **לא** מומצא ע"י המודל.
   - **ביצוע פעולות אוטונומיות:** קוד דטרמיניסטי שקורא ל-Meta API (עם
     הגנות — ראה חוק 6), מופעל ע"י הפרוטוקול.

### למה זה חשוב
- כל commit נוגע במודול אחד → קל לסקור מהנייד.
- החלפת ספק / מודל = שינוי מבודד, לא שבירה ברחבי הקוד.
- ownership מלא של ה-stack, פתרונות שורשיים ולא טלאים.

---

## 3. גבולות ה-MVP

### נכלל ב-MVP
- חיבור Meta OAuth + שליפת נכסים
- שני סוגי קמפיין: לידים (Meta Lead Form) ו-WhatsApp (Click-to-Chat)
- שאלון עומק → יצירת 3 מודעות (3 קופי + 3 תמונות) → העלאה אוטומטית ל-Meta
- בוט WhatsApp לסינון וחימום לידים (Premium) — **כן נכלל**, כי הוא חצי
  ממודל ההכנסה
- מנויים ומכסות לידים (subscriptions)
- **סליקה אוטומטית** (פלאקארד). המוצר Self-Service — תהליך התשלום חייב להיות
  אוטומטי, בלי גבייה ידנית. גבייה ידנית פוגעת באמון ובמודל ה-Self-Service.
  אינטגרציה כעטיפה מבודדת ב-`integrations/pelecard.py` + שכבת abstraction
  אגנוסטית ב-`integrations/billing.py` — אם יוחלף ספק בעתיד, שינוי מקום אחד.
- **סוכן AI פנימי בדשבורד** — צ'אט מומחה-שיווק שעונה ללקוח, מנתח ביצועים
  (מ-Insights) ופותר תקלות. חלק מהימור הליבה של "מנהל קמפיינים, לא מחולל".
- **שליפת מטריקות מ-Meta Insights** — Spend, Leads, CPL, CTR. זה המינימום
  שהסוכן צריך כדי לנטח ולקבל החלטות.

### נדחה בכוונה — אל תבנה את זה
- **תוספת VIP** — upsell עתידי. יש דגל `vip_addon` בסכמה, בלי לוגיקה.
- **דשבורד אנליטיקס *מתקדם*** — גרפים, חתכים, היסטוריה ארוכה. אחרי שהליבה
  מוכחת. (שליפת ארבעת השדות עצמה — **לא** נדחית, ראה "נכלל ב-MVP".)
- **יצירת וידאו** — מופיע במודל העלות של גיא ("3 סרטוני AI") אבל **לא בסקופ
  ה-MVP**. המוצר מייצר תמונות (3 פוסטים גרפיים), לא וידאו. אם ייכנס —
  החלטה עתידית.
- **חשבוניות מס אוטומטיות** — אינטגרציה ל-Green Invoice/iCount. ב-MVP (5–10
  לקוחות ראשונים) הפקה ידנית. אינטגרציה אוטומטית בפאזה נפרדת אחרי שיהיו לקוחות
  אמיתיים. **חובה לפני סקיילינג** (חוק חשבונית מס בישראל), לא לפני בטא סגורה.
- **מסך ניהול אמצעי תשלום בדשבורד** — בפרוטוטייפ אין; לקוח שרוצה להחליף כרטיס
  מבטל ונרשם מחדש.

**הנמקת ה-MVP:** הימור הליבה — AI **מנהל** קמפיינים שמביאים לידים, בלי
סוכנות (לא רק מחולל מודעות). זה מה שצריך להוכיח מול 5-10 לקוחות בטא ראשונים.
הבוט בודק הימור שני ("AI מסנן לידים טוב"). VIP / אנליטיקס-מתקדם / וידאו לא
נחוצים כדי לדעת אם יש כאן עסק.

---

## 4. ה-User Flow (סדר המשתמש)

0. **הרשמה/כניסה** — signup/login. נוצרת subscription `pending` (trigger).
1. **חיבור פייסבוק** — Meta OAuth, טוקן מוצפן.
2. **בחירת סוג קמפיין** — לידים / וואטסאפ.
3. **בחירת נכס פרסומי** — דף + חשבון מודעות (ובוואטסאפ גם מספר עסקי).
4. **בחירת חבילה** — לידים: `basic`/`premium` + מכסה; וואטסאפ: `whatsapp`.
   מעדכן את ה-subscription מ-`pending` לסוג שנבחר (דרך `subscription_service`).
5. **כניסה לדשבורד.**

--- הקמת קמפיין ראשון (רצף מודרך עם הסוכן) ---

6. **תשלום + trial** — בלחיצה על "הקם קמפיין": 7 ימי התנסות + הזנת אמצעי
   תשלום אצל ספק הסליקה. חיוב ראשון רק ביום ה-8. בלי אמצעי תשלום — אין
   הקמת קמפיין. שער להקמת הקמפיין.
7. **הסוכן פותח** — מסביר מה נבחר ומה המטרה.
8. **שאלון עומק** — סוג עסק, קהל, **תקציב** (יומי או כולל; השהיית שבת אופציונלית),
   **מיקום** (רדיוס פר-אזור: 5/10/20/30/50 ק"מ או עיר-בלבד; ריק → ארצי),
   **טון מותג** (אחד מ-4: `luxury` / `friendly_young` / `professional_direct` /
   `bold_dynamic` — מוזרק ליצירת הקופי). פירוט מיפוי ל-Ad Set ב-§9.
9. **גיל + מגדר** — טרגוט הקהל (רמת Ad Set).
10. **טופס ליד** (מותנה — קמפיין לידים).
11. **הגדרת בוט** (מותנה — Premium) ← **כאן, לא בהתחלה.**
12. **אישור הלקוח.**
13. **הסוכן בונה 3 קופי + 3 תמונות** → אישור הלקוח **פר-מודעה** ב-UI, ודחיפה ל-Meta
    ל-A/B. הדחיפה **אטומית**: 3 האישורים נאספים ונשלחים כ-push **אחד**; כשל באמצע →
    rollback מלא, אין קמפיין חצי-בנוי (§2א חוק 6).

> **ToS / אישור תנאי שימוש:** checkbox **חובה** בבחירת חבילה (שלב 4), לפני המעבר
> הלאה. נשמר ב-`subscriptions` (`tos_version` + `tos_accepted_at` — §6).

> **הערה על סדר הבנייה:** סדר המשתמש ≠ סדר הפיתוח. בונים לפי תלות, לא לפי
> ה-flow. ראה ROADMAP.

---

## 5. מודל המנויים והתמחור

החבילה היא **פר-חשבון** (לא פר-קמפיין). לקוח אחד = מנוי אחד, שתחתיו כמה
קמפיינים. שלושה סוגי מנוי:

- **`basic` / `premium`** — קמפיין לידים. ההבדל: האם כלול בוט WhatsApp.
  מכסת לידים `500` / `1000`.
- **`whatsapp`** — קמפיין וואטסאפ (A-Z, ₪397). **בלי מכסת לידים**
  (`lead_quota = null`, "ללא הגבלה"). הסוכן מנהל את הקמפיין ושולח התראות
  לבעלים; אין בוט-לידים בנתיב.

לקוח הוא **או-או**: או מנוי לידים (`basic`/`premium`) או מנוי וואטסאפ
(`whatsapp`), לא שניהם בו-זמנית. לקוח שרוצה לעבור בין הסוגים — מסיים חבילה
אחת ומתחיל אחרת. כך `tier` מקודד גם את סוג הקמפיין ברמת החשבון.

המכסה נמדדת לפי **לידים** בחלון החיוב, לא לפי מספר קמפיינים. לקוח Basic 500
יכול להריץ 2-3 קמפיינים במקביל כל עוד סך הלידים בחלון < 500.

### תמחור — דו-צירי (`tier × lead_quota`) + תוספת VIP
המחיר החודשי נגזר מ-**שני צירים**: ה-tier וכמות הלידים (`derive_monthly_price_agorot(tier, lead_quota,
vip_owner_alerts)` ב-`app/integrations/billing.py`; אגורות). תוספת **VIP** (`vip_owner_alerts` — התראות
WhatsApp לבעלים על לידים חדשים) מתווספת למחיר הבסיס, **מאוחדת** לאותו חיוב חודשי (חשבונית אחת) ותלויה
בכמות הלידים. זכאות ל-VIP: `basic`/`premium` בלבד (ל-`whatsapp` אין `lead_quota`).

| | 500 לידים | 1000 לידים |
|---|---|---|
| Basic | ₪397 | ₪497 |
| Premium | ₪597 | ₪897 |
| **תוספת VIP** | +₪250 | +₪350 |

`whatsapp` במחיר קבוע ₪397 (בלי מכסה, בלי VIP). שילוב `(tier, lead_quota)` לא-מוכר → שגיאה (fail-loud,
בלי ניחוש מחיר). הדגל `vip_owner_alerts` נשמר ב-`subscriptions`; ההפעלה/ביטול משפיעים **מהמחזור הבא**
(בלי proration). מימוש: `docs/VIP_ADDON_PLAN.md`.

חלון "החודש" לכל המכסות הוא **מחזור-הקמפיין** — `[campaigns.cycle_start_at, cycle_end_at)` (פר-קמפיין).
**Phase C:** כל המכסות (לידים 1a / agent-chat 1b / agent-alerts 1d / creative C3b) עברו per-campaign, כך
שקמפיין נוסף מקבל מכסה **נוספת** (§119 anti-abuse — לא מכסה משותפת). המקור-ההיסטורי היה anniversary פר-user
(`current_period_start`+1mo); `current_period_start` נשאר על `subscriptions` ל-`has_paid_access` grace בלבד,
לא לספירת-מכסות.

**אכיפת מכסה:** כשלקוח עובר את מכסת הלידים — **הליד עדיין נקלט ונספר** (לא
חוסמים קליטה — זה אכזרי ומאבד דאטה). אבל **הבוט מפסיק להגיב** ללידים חדשים
מעבר למכסה, ומוצגת ללקוח הצעת שדרוג. כך הליד לא אובד, אבל הלקוח לא מקבל
שירות-בוט מעבר למה ששילם.

---

## 6. סכמת ה-DB (מיושמת ב-Supabase)

‏28 טבלאות (מיושמות לאורך פאזות 0–7). כל טבלה תלויה במשתמש דרך `user_id`, עם RLS. הטבלה למטה כוללת את כל הטבלאות — ליבה דומיינית + תשתית (jobs), סליקה, יומן Google, וסוכן/אופטימיזציה שנוספו תוך כדי הפאזות.

| טבלה | תיאור |
|------|-------|
| `fb_connections` | חיבור Meta — טוקן מוצפן (Vault) + זהות. אחת למשתמש. |
| `campaigns` | הלב. סוג, status, `service_name`, נכסי Meta פר-קמפיין. |
| `quiz_responses` | תשובות שאלון (JSONB גמיש) — כולל **טון מותג**. טבלה נפרדת בכוונה. |
| `ads` | 3 המודעות. קופי + תמונה + meta_ad_id + `status` (`pending`/`generating`/`rejected`/`live`/`failed_push`). (`pushing`/`active` הם statuses של `campaigns`, **לא** של `ads`.) |
| `lead_form_fields` | שדות טופס הליד (קמפיין lead בלבד). |
| `bot_configs` | הגדרת בוט (Premium). הודעת פתיחה, פעולת סיום. |
| `bot_questions` | שאלות סינון (עד 5, עם סדר). |
| `leads` | רשומת-אירוע פנייה (מי, מתי, איזה שירות). **לא** מחזיקה מצב שיחה. כל פנייה = רשומה נפרדת. |
| `bot_conversations` | השיחה המתמשכת מול מספר טלפון (`contact_key`): `state`, `current_question`, `answers` + היסטוריה. אדם אחד = שיחה אחת פעילה. |
| `bot_messages` | היסטוריית הודעות הבוט מול הליד (append-only). |
| `webhook_events` | idempotency — מונע עיבוד כפול של webhooks. |
| `whatsapp_lines` | קו וואטסאפ ייעודי שהוקצה ללקוח Premium. `phone_number_id`, credentials של הספק, סטטוס הקצאה. תלוי ב-`user_id`, RLS. |
| `agent_conversations` | היסטוריית הצ'אט בין הלקוח לסוכן הפנימי בדשבורד. **נפרד לחלוטין מ-`bot_conversations`** (זה הבוט מול הלידים — שתי יישויות שונות). תלוי ב-`user_id`, RLS. |
| `optimization_sessions` | סדרת אופטימיזציה פר (קמפיין, בעיה). `problem_type`, `status`, `loop_count`, `starting_metric` (jsonb), `current_step`. סדרה פתוחה אחת פר (קמפיין, בעיה) — partial unique index (חוק ה-Lock). תלוי ב-`user_id`, RLS. |
| `optimization_actions` | פעולה בודדת בתוך סדרה. `action_type`, `step_number`, `status`, מטריקות לפני/אחרי (מ-Insights), `window_ends_at` (נעילת **5 ימים / 120h**), `requires_approval`/`approved_at`, `screening_question` (בעיה 2 שלב 4), `feedback_requested_at`/`feedback_response` (בעיה 2 — מייל-מעקב "השתפר?" 120ש' + תשובת הבעלים: `improved`=סוגר סדרה / `not_improved`=שלב הבא; `pushed`+requested+response NULL = `awaiting_feedback` = מצב 2, **תשובת-הבעלים = המדד היחיד** ל-low_quality). `push_status=NULL` ⟺ `approved_at=NULL` = "מוצע, טרם אושר" (נשמר ב-propose; ה-approve תופס → `pushing`). תלוי ב-`user_id`, RLS. |
| `jobs` | תור עיבוד אסינכרוני (Postgres queue) — `enqueue` → claim (`FOR UPDATE SKIP LOCKED`) → retry. הבסיס לכל פעולה אסינכרונית. |
| `pending_billing_sessions` | session זמני בזמן הזנת אמצעי תשלום (reserve-first). |
| `billing_profiles` | פרטי לקוח להפקת חשבונית (שם/ת"ז/טלפון/שם-עסק). שינוי תקף מכאן והלאה בלבד. |
| `billing_charge_attempts` | ניסיונות חיוב — idempotency על החיוב (UNIQUE + CAS), מונע חיוב כפול. |
| `billing_invoices` | חשבוניות מס/קבלה (Green Invoice). לקוח אחד = הרבה חשבוניות. |
| `sent_notifications` | idempotency של התראות מערכת (מייל). |
| `app_settings` | דגלים גלובליים (key→jsonb), admin-only — soft-disable (`whatsapp_production_ready`, `appointment_scheduling_enabled`). |
| `agent_messages` | הודעות הצ'אט בין הלקוח לסוכן (append-only, תחת `agent_conversations`). |
| `google_calendar_credentials` | טוקני Google Calendar מוצפנים (Vault) לתיאום תורים. |
| `working_hours` | שעות פעילות בעל העסק (לחישוב slots פנויים). |
| `special_days` | ימים מיוחדים (חופשות/חריגים) — גוברים על `working_hours`. |
| `appointments` | תורים שנקבעו דרך הבוט (reserve-first, UNIQUE partial נגד double-booking). |
| `subscriptions` | המנוי. ראה פירוט למטה. אחד למשתמש, נוצר ב-trigger ב-signup. |

### פירוט `subscriptions` (מעודכן ל-2.6)

- `tier` (`pending`/`basic`/`premium`/`whatsapp`) + `lead_quota` (nullable — `null` ב-`pending` וב-`whatsapp`, `500`/`1000` ב-`basic`/`premium`) + `status` (`trial`/`active`/`past_due`/`expired`/`canceled`) + `trial_ends_at`.
  - `status` ו-`tier` הם `TEXT` עם `CHECK` constraint (רשימת ערכים מפורשת) — **לא** `VARCHAR(N)` ולא Postgres enum: הוולידציה ברמת ה-DB (INSERT שגוי נכשל מוקדם), והרחבת ערכים = `DROP`+`ADD CONSTRAINT` (ראה 2.6.1 שלב 5 להוספת `past_due`).

- **סליקה (פלאקארד):**
  - `billing_customer_id` (text, nullable) — מזהה הלקוח אצל הספק.
  - `billing_provider` (text, nullable) — `'pelecard'` כברירת מחדל, enum מוכן להחלפה עתידית.
  - `billing_token` (text, nullable) — **רפרנס לטוקן ב-Vault**, לא הטוקן עצמו (אותו דפוס כמו `encrypted_token` ב-`fb_connections`).
  - `last_billing_error` (text, nullable) — שגיאת חיוב אחרונה (לבדיקת תמיכה).
  - `last_billing_error_at` (timestamptz, nullable).
  - `vip_owner_alerts` (boolean, default `false`) — דגל תוספת ה-VIP (התראות WhatsApp לבעלים). נכלל במחיר החודשי המאוחד (ראה §5 תמחור).
  - `vip_alert_phone` (text, nullable) — טלפון (E.164) ליעד התראות ה-VIP; חובה (נאכף ב-service) כש-`vip_owner_alerts=true`, אחרת `null`.
  - `agent_alert_email` (text, nullable) — יעד המייל של עדכוני-הסוכן (per-account, 0110). נאסף בשאלון (חובה שם); **המקור הבלעדי** — בלי fallback למייל-ההרשמה (ריק → המייל לא נשלח, skip).
  - `agent_alert_phone` (text, nullable) — יעד הוואטסאפ של עדכוני-הסוכן (E.164, 0110). **נפרד** מ-`vip_alert_phone`. אופציונלי (ריק → אין וואטסאפ, מייל בלבד). נאסף בשאלון ונערך בטוגל שב"החבילה שלי".

- `current_period_start` (timestamptz) — עוגן מחזור החיוב. נמלא בזמן הזנת אמצעי תשלום ב-flow שלב 6, לא בהרשמה. כל המכסות נספרות מ-`current_period_start` ועד +1 חודש (anniversary), לא לפי חודש קלנדרי. מתעדכן בכל חיוב חודשי מוצלח.

- **שלושה מוני שימוש נפרדים** (כל אחד נספר בחלון ה-anniversary, אסור לבלבל ביניהם):
  - `lead_quota` — מכסת לידים (`500`/`1000`, או `null` ב-`whatsapp`).
  - `agent_chat_quota` — שיחות ייעוץ של הלקוח עם הסוכן בצ'אט הדשבורד. `// ממתין למספרים מגיא`
  - `agent_alerts_quota` — עדכוני-סוכן לוואטסאפ של הבעלים (4 הסוגים: step_advanced/series_resolved/ad_rejected/quality_followup; ה-"0/15" בפרוטוטייפ). **חינם, מכסה 30/חודש**; מעבר למכסה — וואטסאפ נעצר, **מייל ממשיך** (ל-`agent_alert_email`). היעד **per-account** (0110): מייל ל-`agent_alert_email` (חובה בשאלון, המקור הבלעדי — בלי fallback להרשמה), וואטסאפ ל-`agent_alert_phone` (אופציונלי, **נפרד** מ-VIP); נאסף בשאלון ונערך ב"החבילה שלי". opt-in לוואטסאפ = `agent_whatsapp_enabled` + מספר קיים. רדום מאחורי gate ה-WABA עד אישור template `agent_update_owner` ב-Meta.
- **ספירה (מאיפה כל מונה נספר):** לידים מ-`leads` בחלון `[current_period_start, +1 month)`; שיחות-צ'אט מ-`agent_conversations` באותו חלון; התראות מטבלת ההתראות (תיווצר ב-Phase 4.6) באותו חלון. שלושת המונים — ספירה-בחלון, בלי reset.

- **ToS:** `tos_version` (text) + `tos_accepted_at` (timestamptz) — נכתבים ב-checkbox של שלב 4 (Phase 2.5). גרסה מתועדת כדי לדעת מי אישר איזו גרסה (תאימות משפטית §10).

- **נעילת trial = מצב מחושב:** הגבלת הגישה ב-frontend נגזרת מ-`status` + `trial_ends_at` (למשל `status='trial' AND trial_ends_at < now()` = פג; `status='past_due'` = יש בעיית חיוב, להציג התראה).

### מעברי `status` (מכונת מצבים)

```
                       signup
                          │
                          ▼
                   ┌──────────────┐
                   │   pending    │ ← signup, אין tier
                   └──────┬───────┘
                          │ בחירת חבילה (2.5)
                          ▼
                   ┌──────────────┐
                   │   pending    │ ← tier נבחר, אין billing
                   └──────┬───────┘
                          │ הזנת אמצעי תשלום (2.6)
                          ▼
                   ┌──────────────┐
                   │    trial     │ ← billing_token קיים, 7 ימים
                   └──────┬───────┘
                          │ יום 8 — חיוב ראשון
                  ┌───────┴───────┐
                  ▼ הצלחה         ▼ כשל
            ┌──────────┐    ┌──────────┐
            │  active  │    │ past_due │
            └────┬─────┘    └────┬─────┘
       חיוב חודשי │              │ retry (3×)
       ┌──────────┴───┐         │
       ▼ הצלחה   ▼ כשל         │ הכל נכשל
   נשאר active  past_due ──────┼──→ expired
                                │
   ביטול בכל שלב ──────────────┴──→ canceled
```

**מעברים חוקיים בלבד** (נאכף ב-`subscription_service`):
- `pending → pending` (בחירת חבילה)
- `pending → trial` (הזנת billing)
- `trial → active` (חיוב יום 8 הצליח)
- `trial → past_due` (חיוב יום 8 נכשל)
- `active → past_due` (חיוב חודשי נכשל)
- `past_due → active` (retry הצליח)
- `past_due → expired` (כל ה-retries נכשלו)
- `* → canceled` (ביטול מהמשתמש, מכל מצב)
- `expired/canceled → trial` **אסור** — לקוח שחזר = signup חדש.

### החלטות מבנה שכבר התקבלו (אל תשנה בלי דיון)

1. **כל פנייה = ליד נפרד.** לא מאחדים לפי טלפון. יש `contact_key`
   (טלפון מנורמל) שמכין איחוד עתידי בעלות אפס, אבל לא unique.
2. **שאלון = JSONB גמיש**, לא עמודות קבועות. השאלון עוד ישתנה.
3. **plan פר-חשבון** (ב-subscriptions), לא פר-קמפיין.
4. **subscription נוצר ב-signup, לא בבחירת חבילה.** trigger על `auth.users` יוצר שורה עם `tier=pending`, `lead_quota=null`. בחירת החבילה (שלב 4 ב-flow) רק *מעדכנת* את הרשומה הקיימת. כך אין רגע שבו משתמש קיים בלי מנוי.
`tier=pending` הוא מצב מפורש — לא `null` ולא ברירת מחדל ל-`basic`, כדי שמשתמש לפני בחירה לא ייספר בטעות כלקוח basic.

---

## 7. עקרונות אבטחה (קריטי — נאכף ברמת ה-DB ובקוד)

1. **RLS דלוק על כל טבלה.** מדיניות: `auth.uid() = user_id`.
2. **Composite FK** בכל טבלת בת — אוכף ש-`user_id` תואם לבעלים האמיתי, כדי
   שכל כתיבת service_role (webhook או פעולת admin שעוקפת RLS) לא תדליף נתונים
   בין לקוחות.
3. **שני Supabase clients נפרדים:**
   - `supabase_user` — anon key + JWT של המשתמש. **ברירת המחדל** לכל endpoint
     רגיל; RLS אוכף בעלות.
   - `supabase_admin` — service_role (עוקף RLS). מבודד למודולים ייעודיים, מותר
     **רק** בשני ההקשרים הבאים — לעולם לא ב-router רגיל מול משתמש, לעולם לא
     נחשף ל-frontend:
     - (א) **webhook handlers** — המודול המבודד `routers/webhooks.py` (הכותב
       אינו המשתמש המחובר; חתימה מאומתת + idempotency).
     - (ב) **פעולות privileged ש-state שלהן server-authoritative** ואסור שיהיו
       בשליטת המשתמש — דרך **service ייעודי פר-תחום** שמחזיק את ה-client (ה-router
       קורא ל-service, לא נוגע ב-admin): `fb_service` (טוקן Meta מוצפן
       ל-`fb_connections` — Phase 1.2), `subscription_service` (עדכון `tier`/מכסה
       — 2.5, שדות סליקה — 2.6), הקצאת `whatsapp_lines` (Phase 5.0).
   - **העיקרון:** ה-client הרגיש מבודד למודול אחד פר-תחום; routers רגילים קוראים
     ל-service ולא נוגעים ב-admin client — כך באג ב-route לא יכול לעקוף RLS.
     נתון שמגיע מהמשתמש לעולם אינו ה-authority לפעולת admin (`tier` נקבע ע"י
     הסליקה/הלוגיקה, לא מ-`request.body`).
   - יצירת ה-subscription בזמן signup היא מנגנון server-authoritative **נפרד** —
     trigger `SECURITY DEFINER` ברמת ה-DB (ראה 0.5.2), לא admin client.
4. **טוקן Meta מוצפן** דרך Supabase Vault — לא בעמודת plaintext.
5. **אימות חתימת webhook** לפני עיבוד:
   - Meta (לידים **וגם** WhatsApp Cloud API): `X-Hub-Signature-256` מול App Secret.
   - פלאקארד / Green Invoice: לא חותמים HMAC → אימות חוזר server-to-server חובה (§9).
   בלי זה כל אחד יכול להזריק לידים/הודעות מזויפים.
6. **idempotency:** לפני עיבוד webhook — `insert into webhook_events
   ... on conflict do nothing`. אם כבר עובד — צא.
7. **token refresh:** טוקן Meta long-lived תקף ~60 יום. cron מרענן ב-50%
   מהחיים, אחרת הקמפיין "מת בשקט" ביום 61.

---

## 7א. מודל הטוקנים של אימות המשתמש (Session 0.5.1)

אימות המשתמש דרך Supabase Auth, עם הפרדה בין שני טוקנים — זהו **לא** טוקן
ה-Meta (סעיף 7.4), אלא ה-session של המשתמש מול האפליקציה:

- **access token (JWT, קצר-חיים):** מוחזר בגוף התשובה של `signup`/`login`/
  `refresh`. ה-client שולח אותו בכל בקשה מוגנת ב-`Authorization: Bearer`.
  `get_current_user` מאמת אותו מול Supabase (`auth.get_user`) — בלי אימות
  חתימה עצמי.
- **refresh token (ארוך-חיים, רגיש):** נשמר אך ורק ב-**httpOnly cookie**
  (`Secure`, `SameSite=Lax`, `Path=/auth`). לעולם לא בגוף התשובה ולא
  ב-localStorage — httpOnly מונע גישה מ-JavaScript (הגנת XSS), SameSite=Lax
  מקטין CSRF על ה-POST של הרענון.
- **`POST /auth/refresh`:** קורא את ה-refresh token מה-cookie, מבקש מ-Supabase
  access token חדש. Supabase **מסובב** refresh tokens (כל שימוש מנפיק חדש
  ופוסל את הישן) — לכן בכל רענון **נשמר ה-refresh token החדש ב-cookie**
  (אחרת המשתמש מתנתק אחרי הרענון הראשון).
- **`POST /auth/logout`:** מבטל את ה-session ב-Supabase ומנקה את ה-cookie.
  מבטל **בלי לסובב** (`set_session`+`sign_out`, לא `refresh_session`) — סיבוב
  היה שובר את ההנחה "token לא תקף = session מת". "אין מה לבטל" נקבע לפי **שני**
  האסימונים יחד (refresh cookie **וגם** Bearer), לא לפי refresh בלבד — Bearer תקף
  בלי refresh = session חי שצריך לבטל (כלל 2: אל תסיק state מאינדיקטור עקיף):
  - אין refresh וגם אין Bearer תקף → 204, בלי קריאה ל-Supabase (באמת אין מה לבטל).
  - יש Bearer תקף (גם בלי refresh cookie) → מבטלים דרכו.
  - יש refresh אבל Bearer חסר/פג → 401; הלקוח מרענן (`/auth/refresh`) ואז logout.
  - כשל זמני/לא-מזוהה בביטול → שגיאה (לא 204), וה-cookie נשאר ל-retry.
- **מגבלה ידועה (JWT):** logout מבטל את ה-session בצד השרת (ה-refresh נפסל, אי
  אפשר לחדש), אבל access token שכבר נופק הוא JWT — תקף עד הפקיעה ולא ניתן לביטול
  מיידי בלי token-blocklist. זה מובנה ל-JWT; ה-SDK של Supabase מתעד זאת מפורשות
  וממליץ על TTL קצר ל-access token. לכן ה-access קצר-חיים וה-refresh הוא הרגיש.
- מיפוי שגיאות מרוכז (`classify_auth_error`, מקום אחד): 401/טוקן-פג = terminal;
  429/5xx/רשת = transient; לא-מזוהה = 500 (לא ניחוש). על transient משמרים state
  ומאפשרים retry; על terminal מנקים.
- **CORS:** כשייבנה frontend בדומיין נפרד — להגדיר CORS עם credentials ולוודא
  שה-cookie same-site מול דומיין ה-API.

**שתי זהויות-פייסבוק נפרדות (אסור לבלבל):**
- **התחברות (login) דרך פייסבוק** — Session 1.3: Supabase Auth Facebook Provider,
  scopes `email`+`public_profile` **בלבד**. אופציית login לזהות, לצד email/סיסמה;
  הזהות עדיין מנוהלת ע"י Supabase Auth (אותו `user_id`, אותם access/refresh tokens).
- **חיבור חשבון המודעות** — Phase 1 (`fb_connections`): OAuth משלנו עם `ads_management`
  וכו' (סעיף 9). יושב *על* המשתמש, **לא** מהווה/מחליף את הזהות.

ה-`user_id` לעולם אינו מגיע מטוקן ה-Meta של חשבון המודעות — login-via-FB ו-
connect-ad-account הם שני OAuth שונים, scopes שונים, תכלית שונה.

**מימוש 1.3 (backend-mediated PKCE):** `/auth/oauth/facebook/start` מייצר PKCE אצלנו
(`core/security`) ומפנה ל-`{SUPABASE_URL}/auth/v1/authorize` (ה-verifier נשמר ב-cookie חתום
httpOnly); `/auth/oauth/facebook/callback` מחליף `code`→session (`exchange_code_for_session`
עם `code_verifier` מפורש), שותל את ה-refresh ב-httpOnly cookie כמו login רגיל, ומפנה ל-frontend
**בלי טוקן ב-URL** (ה-frontend מאתחל access דרך `POST /auth/refresh`). למה backend ולא
supabase-js ישיר: כדי שה-refresh יישב ב-httpOnly cookie (§7א), לא ב-localStorage.

**מניעת takeover (דפוס קריטי 1):** ב-Supabase Dashboard — **email confirmation דלוק** +
**auto-link כבוי**; email מ-FB שכבר קיים כחשבון סיסמה → **409** ("האימייל הזה רשום…"), לא מיזוג
שקט. **קונפיג נדרש:** Facebook provider מופעל ב-Supabase; `OAUTH_REDIRECT_URI` רשום ב-Redirect
URLs; ב-Meta App — `https://<ref>.supabase.co/auth/v1/callback` ב-Valid OAuth Redirect URIs
(בנוסף ל-callback של חיבור חשבון המודעות מ-1.1/1.2).

---

## 8. סדר הבנייה (לפי תלות)

1. **שלד FastAPI** — מבנה, שני clients, `/health` שבודק חיבור ל-DB.
2. **חיבור ל-Render** — push ל-main מתפרס. לוודא שהצינור עובד מקצה לקצה.
3. **Meta OAuth** (flow שלב 1) — חיבור FB, שמירת טוקן ב-Vault.
4. **שליפת נכסים** (שלבים 2-3).
5. **שאלון + יצירת מודעות** (שלבים 8-13) — ה"קסם" שמוכיח ערך.
6. **בוט WhatsApp** (שלב 11) — הכי מורכב והכי שביר. אחרון בכוונה.

> ה-MVP המינימלי שמוכיח את ההימור: חבר פייסבוק → ענה על שאלון → קבל 3
> מודעות שעולות לאוויר. הבוט והסוכן האוטונומי הם הרחבות על גביו.
> סדר הסשנים המלא והמעודכן — ב-ROADMAP.

---

## 9. אינטגרציות — נקודות חשובות

### Meta
- **תקשורת (Session 3.4):** כל קריאות ה-Ads/assets/identity עוברות דרך `facebook-business`
  SDK. **חריג יחיד:** OAuth token exchange (`exchange_code_for_token`/
  `exchange_long_lived_token`) נשאר httpx — ה-SDK לא מכסה את endpoints ה-auth. הנמקה
  מלאה: `docs/decisions/0001-meta-sdk.md`. אסור httpx ל-Meta בכל מקום אחר.
- הרשאות OAuth: `ads_management`, `ads_read`, `pages_show_list`, `pages_read_engagement`,
  `business_management`, `whatsapp_business_management`. הבקשה נשלחת עם `auth_type=rerequest` (מאלץ מסך בחירת-דפים בכל חיבור).
  (`pages_manage_ads` הוסר — Meta דוחה אותו כ-"Invalid Scopes" וחוסם את דיאלוג ה-OAuth.
  `pages_show_list` נדרש כדי ש-`/me/accounts` יחזיר את דפי המשתמש; בלעדיו Meta מחזירה רשימה ריקה
  והמשתמש רואה "לא נמצא עמוד עסקי". `pages_read_engagement` הוא ל-engagement של דף, לא לרשימה.
  `business_management` קריטי לדפים תחת Business Portfolio — בלעדיו `/me/accounts` ריק גם כשהדף אושר;
  `list_pages` עושה fallback ל-`/me/businesses`→owned/client pages שדורש אותו.
  `whatsapp_business_management` (נוסף ב-2.1.5) — לשליפת מספרי ה-WhatsApp Business של המשתמש
  (`list_user_whatsapp_numbers`: `/me/businesses`→owned/client WABAs→phone_numbers). Advanced Access דורש
  App Review ל-production — ראה SETUP_CHECKLIST.)
- **`GET /me/meta-assets` — שדות לא-רגישים בלבד (כלל 6):** id + שם לבחירה. **לדפים** גם `picture` (URL ציבורי
  לתמונת-הפרופיל — שדה לא-רגיש שמוחזר מ-`pages_show_list`, להצגה ויזואלית במסך בחירת-הדף; `is_silhouette` =
  תמונת-ברירת-מחדל → מושמט, ה-frontend מציג אווטאר-אות). **לעולם לא** page access_token. ל-**חשבונות-מודעות
  אין** תמונת-פרופיל ב-Meta — שם אווטאר-אות/אייקון, לא picture.
- הקמת קמפיין: דחיפת Campaign + Ad Set (תקציב + טרגוט) + 3 מודעות נפרדות.
- **תקציב ה-Ad Set:** מצב **יומי** (daily budget) או **כולל** (lifetime budget),
  לבחירת הלקוח. **השהיית שבת** אופציונלית → day-parting (תזמון שעות) ב-Ad Set —
  המודעות לא רצות בשבת.
- **טרגוט ה-Ad Set** כולל: מיקום גיאוגרפי (מהשאלון — **רדיוס פר-אזור**: 5/10/20/30/50
  ק"מ או עיר-בלבד; רשימה ריקה → טרגוט ארצי) + **טווח גיל (min/max) +
  מגדר (נשים/גברים/הכל)** שהלקוח בוחר בהקמת הקמפיין. הקהלים המפורטים
  (interests/lookalike/remarketing) נשארים רחבים/אוטומטיים — לא נבחרים ע"י
  הלקוח. גיל+מגדר ניתנים לכוונון אוטונומי ע"י הסוכן (חוק 7 / פעולת אופטימיזציה).
- **Meta Insights:** endpoint ששולף Spend, Leads, CPL, CTR פר-קמפיין. מזין
  את הסוכן (צ'אט, ניתוח, פרוטוקול אופטימיזציה). הסוכן מצטט רק מספרים מכאן —
  לא מומצאים.
- **טיפול בדחיית מודעה:** כשמטא דוחה מודעה (אחרי שעלתה), הסוכן מנסח גרסה
  מתוקנת אוטומטית (LLM), אבל **מציג אותה ללקוח לאישור** לפני העלאה — תוכן
  חדש דורש אישור. זה החיבור בין enum הדחייה (`rejected`) לקטגוריות הפעולה.
- **עמוד-אינטראקציה (אישור/תצוגה):** מיילי-הסוכן מקשרים ל-entity ספציפי
  (`/action/{id}` או `/session/{id}`); ה-מצב נגזר מ-status. action שממתין-לאישור
  (כולל תיקון-דחייה) → **מצב אישור** (וריאציות + "אשר והעלה"); action/סדרה שהושלם
  → **מצב תצוגה** (מה הסוכן עשה + תוצאה). ה-אישור משותף עם הצ'אט — push-core אחד
  (`execute_approval`), 2 endpoints דקים, ה-idempotency/CAS-lease נשמר בשניהם.

### OpenAI
- **קופי:** ChatGPT API (`gpt-5.2`) — 3 גרסאות לפי זווית: `emotional` (רגשי) / `pain_solution` (כאב→פתרון) / `result_success` (תוצאה והצלחה).
- **תמונות:** **gpt-image-2** (לא DALL·E 3 — הוסר מה-API במאי 2026).
  ברירת מחדל: איכות **Medium** ביחס 1:1. אפשר להעלות ל-High פר-מודעה.
- **עריכת תמונות:** מעבר ליצירה מאפס (gpt-image-2), המערכת תומכת ב**עריכה/
  שיפור של תמונה שהועלתה** (Vision/image editing). שתי יכולות נפרדות:
  רענון = יצירה חוזרת; עריכה = שיפור תמונה קיימת.
- **שינוי תוצר קיים בגלריה (`revise`, Stage 1a):** `images.edit` עם **הוראת המשתמש** כ-prompt על
  התמונה השמורה של asset קיים (reuse של נתיב ה-`upgrade`, שם ה-prompt קבוע), ושמירת תוצר
  `asset_type='revised'` מקושר למקור (`revised_from_asset_id`). מכסה **פר-asset** (שונה משלוש
  המכסות השטוחות יצירה/שדרוג/פרסום): עד 5 שינויים ישירים לכל תמונה במחזור החיוב (ספירה לפי
  `revised_from_asset_id`); גם תוצר-שינוי הוא asset → ניתן לעריכה חוזרת עם 5 משלו. נחשף ב-
  `GET /me/creative/quota` (`revise_quota` + `revise_used`). endpoint: `POST /me/creative/{asset_id}/revise`.

### WhatsApp — שני מודלי מספר
- **קמפיין וואטסאפ:** הליד פונה ישירות למספר העסקי **של הלקוח**. המערכת
  מאמתת שהמספר תקין (flow), אבל **לא** בנתיב ההודעה.
- **לידים + Premium:** המערכת **מקצה מספר ייעודי משלה** דרך **Meta Cloud API**
  (WABA יחיד שלנו, קו ייעודי פר לקוח, System User Token גלובלי). הבוט יושב על
  המספר הזה ומאזין להודעות נכנסות. זה המספר שעליו רץ הבוט. (ההחלטה המקורית הייתה
  360dialog — הוחלף ל-Meta Cloud API במימוש Phase 5; `integrations/meta_whatsapp.py`.)

הקצאת המספר ל-Premium מנוהלת ב-`whatsapp_lines`. בבטא — provisioning ידני ע"י
אדמין (דורש WABA + אישור מטא, לא מיידי). self-serve מאוחר יותר.

> **שליפת מספרי וואטסאפ עסקיים (`/me/whatsapp-business-numbers`) — מומשה (Session 2.1.5).**
> ה-scope `whatsapp_business_management` נוסף ל-OAuth (ברשימה למעלה). השליפה דרך user-token
> (SDK): `/me/businesses`→owned/client WABAs→phone_numbers (`meta.list_user_whatsapp_numbers`).
> ב-onboarding מסלול וואטסאפ, אישור המספר האמיתי מוצג ב-**step-3** (אחרי חיבור Meta — שם קיים
> הטוקן; ב-step-2 אין). אין מספר / scope טרם אושר (permission→`MetaUnexpectedError`) →
> degrade-to-empty ב-`meta_service` → ה-frontend מפנה למדריך ההקמה (כמו `(#100)` ב-`list_pages`).

> **קמפיין WhatsApp (CTWA) — יצירה+דחיפה (מודל A):** יצירת-הקמפיין מאמתת שה-`whatsapp_phone_number_id` הוא
> אחד ממספרי-**המשתמש עצמו** (`campaign_service._verify_asset_ownership` מול `get_whatsapp_business_numbers` —
> IDOR, **לא** מול קו-פלטפורמה). ה-push בונה מודעת Click-to-WhatsApp (`campaign_push_service`): objective
> `OUTCOME_ENGAGEMENT`, ad set `destination_type=WHATSAPP`, creative CTA `WHATSAPP_MESSAGE`; הניתוב דרך המספר
> המחובר ל-Facebook Page של הלקוח (ה-`phone_number_id` = אימות/הצגה בלבד). הקמת ה-Meta לקמפיין WhatsApp:
> `docs/deployment/whatsapp-campaign-setup.md`.

**הבוט (Premium):** מאזין לליד נכנס → הודעת פתיחה תוך 30 שניות (עם
`service_name`) → שאלות סינון → פעולת סיום. מצב השיחה נשמר ב-`bot_conversations`
לפי `contact_key` (לא ב-`leads` — ליד הוא אירוע, שיחה היא מתמשכת פר-אדם).
זו מכונת המצבים.

### פלאקארד (סליקה)

- **משטח API:** Gateway21 (`gateway21.pelecard.biz` בפרודקשן, `gateway20.pelecard.biz/sandbox` בפיתוח). Match API החדש קיים אבל תיעוד פומבי חלקי — לאמת מול תמיכת פלאקארד לפני מעבר.
- **אימות:** שלשה (`terminal`, `user`, `password`) — server-side בלבד, env vars, לעולם לא ב-frontend.
- **`ActionType`:** `J2` ליצירת טוקן בזמן trial (אימות בלי חיוב), `J4` לחיוב חודשי על טוקן שמור (`IsToken=true`).
- **יחידות סכום:** אגורות (פי 100 משקלים). `Total: 9900` = ₪99.00. מקור באגים נפוץ — לוודא בכל קריאה.
- **טוקניזציה:** ב-J2 הראשוני שולחים `CreateToken=true`. פלאקארד מחזירה טוקן בתשובת `GetTransaction`. שמירה ב-Vault.
- **Webhooks (IPN):** `ServerSideGoodFeedbackURL` + `ServerSideErrorFeedbackURL` → `/webhooks/billing/pelecard`. **פלאקארד לא חותמת IPN ב-HMAC.** אימות חובה: השוואת `ConfirmationKey` + קריאה חוזרת ל-`PaymentGW/GetTransaction`. Idempotency: `PelecardTransactionId` (לא `ParamX` — `ParamX` יכול להיות זהה בשתי מסירות של אותו IPN).
- **Recurring:** פלאקארד לא מציעה recurring native. אנחנו יוזמים כל חיוב על הטוקן השמור (cron).
- **קריאה חובה לפני קוד:** `docs/integrations/pelecard/SKILL.md`.

### הסוכן — פרוטוקול האופטימיזציה
לינארי, כל שלב נבחן **5 ימים (120h)** לפני שמסיקים אם הצליח. השלבים **המומשים**
(`optimization_service._STEP_PLANS`): **(1) `creative_refresh`** — 3 זוויות
ייחודיות (pain / dream / urgency) → **(2) `angle_change` (social_proof)** →
**(3) `angle_change` (authority)**. שלב **`offer_change`** (שינוי הצעה שיווקית,
≥4), **`עדכון קהלים`** ו**`מעבר לוואטסאפ`** — מתוכננים אך **טרם מומשו** (כרגע
מיצוי שלב 3 = terminal). הקוד (דטרמיניסטי) קובע איזה שלב הבא ומה ננעל; ה-LLM
מנסח את ההודעה ללקוח. נעילת 5 הימים אוכפת `optimization_actions.window_ends_at`
— אם הלקוח מבקש פעולה נוספת בתוך החלון, מציגים תזכורת (לא חוסמים).

**תמונות חדשות ברענון (A3):** 3 סוגי-הרענון (`creative_refresh`/`angle_change`/`offer_change`) מקבלים
**תמונה חדשה פר-וריאציה** מ-`image_concept` (gpt-image, worker אסינכרוני; ה-push צרכן-טהור של התמונה
השמורה). **"תמונות עם הקופי" (שחזור החזון §159/216):** התמונות נוצרות **eager כבר בזמן ה-propose** (job
`optimization_image_generate`) ומוצגות ב-`view-action` לצד הקופי — המשתמש בוחן, מרענן-תמונה
(`POST /me/actions/{id}/variations/{i}/regenerate-image`), עורך/מרענן-קופי (התמונה נשמרת) — **לפני** approve.
ה-approve צורך את התמונות המוכנות (fallback: יצירה+push אם חסרות). **הוויזרד** מייצר תמונות אוטומטית עם
הקופי (בלי כפתור). דגל `include_images` דינמי (פרודקשן תמיד-ON; סאנדבוקס צ'קבוקס) + מתג-חירום גלובלי
(`app_settings.optimization_images_forced_off`) שכופה חזרה ל-recycle בלי deploy. `screening`/`filter_addon`
ממחזרים את התמונה הישנה (re-sign מ-`image_path`). **דחיית-Meta (`rejection_fix`):** תיקון-הדחייה מייצר תמונה
**חדשה** למודעה הנדחית **בלבד** (אפשרות B — לא 3+3; 2 המודעות התקינות לא נגעות), במקום מיחזור התמונה הישנה;
recycle כשהדגל כבוי. idempotency: ה-paths נשמרים reserve-first ב-`optimization_actions.generated_image_paths`
לפני כל צעד בלתי-הפיך.

**שתי קטגוריות פעולה:**
- **דורש אישור לקוח:** תוכן חדש (מודעות, תמונות, קופי) + הגדלת תקציב כולל.
- **אוטונומי (הסוכן מבצע לבד, מתעד בדשבורד):** שינוי גילאים, מגדר, אזורי
  מיקוד, פתיחת רימרקטינג, קהל דומה, **העברת תקציב בין סטים קיימים** (לא
  הגדלת הכולל), כיבוי מודעות חלשות.

### התראות — שני ערוצים נפרדים
- **התראות מערכת** — אירועים תפעוליים: קמפיין עלה לאוויר, נגמר ה-trial
  והתבצע חיוב, 80% מהמכסה. מייל/וואטסאפ.
- **התראות הסוכן לבעלים** — הסוכן יוזם ושולח לוואטסאפ של בעל העסק: תובנות
  ועדכוני קמפיין ("זיהיתי עלות לליד גבוהה, הרחבתי את הקהל") **וגם בקשות
  אישור** (מודעה/קופי לאישור לפני עלייה). מכסה: `agent_alerts_quota`. שונה
  גם מבוט-הלידים (מדבר עם הלידים) וגם מצ'אט-הסוכן (הבעלים יוזם).
  **מימוש:** היעד **per-account** נאסף בשאלון (מייל `agent_alert_email` חובה + מספר
  `agent_alert_phone` אופציונלי, נפרד מ-VIP) ונערך ב"החבילה שלי"; חינם, מכסה 30/חודש;
  template `agent_update_owner` גנרי (תקציר פר-type + deep-link). **מייל נשלח ל-`agent_alert_email`**
  (המקור הבלעדי — בלי fallback להרשמה; ריק → skip); הוואטסאפ תוספת אופציונלית. רדום מאחורי gate ה-WABA
  עד אישור Meta.

---

## 10. תאימות משפטית (לא לדחות)

המערכת מנהלת **מאגר מידע אישי על אזרחים ישראלים** (לידים: שם, טלפון).
חלים חוקי הגנת הפרטיות + תיקון 13 (אוגוסט 2025):
- בסיס חוקי לעיבוד, מדיניות פרטיות אמיתית (לא placeholder).
- אבטחת מידע, ואולי רישום מאגר.
- המערכת גם נושאת טוקני גישה לחשבונות פרסום של לקוחות — רגיש במיוחד.

---

## 11. שותפות

מוצר משותף 50/50 (אמיר — ארכיטקטורה ופיתוח; גיא — מוצר ועסקי).
החלטות מוצר/מודל הכנסה דורשות אישור משותף. ה-repo, Supabase ו-Render
תחת ארגון/חשבון משותף.
