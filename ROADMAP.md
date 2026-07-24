# Campaign AI — תוכנית פיתוח (ROADMAP.md)

> מסמך דינמי. מתעדכן אחרי כל סשן (מסמנים ✅ מה הושלם).
> משלים את spec.md (ה*מה*) — כאן ה*סדר ומתי*.
> חיווט הפרונטד ל-backend מנוהל בנפרד ב-`docs/frontend-integration.md` (חתך per-screen), לא כ-phase כאן.
>

---

## Phase 0 · תשתית

### Session 0.1 — שלד FastAPI מודולרי ✅
- [x] **מבנה התיקיות המלא לפי spec.md סעיף 2א** (routers/ services/
      integrations/ models/ db/ core/) — תיקיות ריקות עם `__init__.py`,
      מוכנות לאכלוס בסשנים הבאים
- [x] `requirements.txt`, `.gitignore`, `.env.example`, `render.yaml`
- [x] `config.py` טוען env vars (לא ערכים בקוד)
- [x] `db/client.py`: `get_user_client(jwt)` + `get_admin_client()` מבודד
- [x] `routers/health.py`: `GET /health` — ok + בודק חיבור ל-Supabase
- [x] `main.py`: מרכיב את האפליקציה ורושם את ה-router. אפס לוגיקה.

**Done:** השרת עולה ו-`/health` מחזיר 200. מבנה התיקיות תואם בדיוק
לסעיף 2א ב-spec.
**לא לעשות:** שום לוגיקה עסקית. שום endpoint מעבר ל-health. לא לדחוס הכל
ל-main.py — לבנות לפי השכבות.

### Session 0.2 — חיבור ל-Render
- [ ] חיבור ה-repo ל-Render (web service, plan `starter` — נדרש ל-`preDeployCommand`)
- [ ] הזנת env vars אמיתיים ב-Render: 3 מפתחות Supabase + **`SUPABASE_DB_URL`**
      (session-mode pooler / IPv4 — ל-migrations; ראה .env.example)
- [ ] deploy ראשון — ה-`preDeployCommand` מריץ את ה-migrations אוטומטית
      (`app/db/migrate.py`) לפני שהגרסה עולה; כשל בו חוסם את ה-deploy

**Done:** `/health` מחזיר 200 **מה-URL של Render** (לא רק מקומי). הצינור
repo → Render → Supabase עובד מקצה לקצה, וטבלת `subscriptions` (0.5.2) נוצרה
אוטומטית ב-deploy (אימות ה-trigger/RLS שנדחה מ-0.5.2 קורה כאן).
**לא לעשות:** כלום מעבר לזה. זה אבן דרך, לא פיצ'ר.

---

## Phase 0.5 · הרשמה ו-subscription

> ה-invariant שכל ה-phases הבאים נשענים עליו: **משתמש קיים ⟺ יש לו רשומת subscription**. Phase 1 (OAuth) כבר מניח `user_id` קיים — לכן זה חייב לבוא לפניו.

### Session 0.5.1 — Auth (Supabase Auth) ✅
- [x] signup + login דרך Supabase Auth (email+password)
- [x] `core/deps.py`: `get_current_user` — מוציא ומאמת את ה-user מתוך ה-JWT (Bearer)
- [x] `routers/auth.py` דק — מסתמך על Supabase Auth, בלי לוגיקת אימות עצמית
- [x] endpoint מוגן לבדיקה (`GET /me`) שמחזיר את ה-user רק עם JWT תקין
- [x] **שכבת טוקנים מלאה:** access token בגוף; refresh token ב-httpOnly+Secure+SameSite=Lax cookie (Path=/auth) — ראה spec.md סעיף 7א
- [x] `POST /auth/refresh` עם **rotation** — שומר את ה-refresh token החדש בכל רענון
- [x] `POST /auth/logout` — ביטול session ב-Supabase + ניקוי ה-cookie

**Done:** משתמש נרשם, מתחבר, ומקבל JWT. `GET /me` מחזיר 401 בלי טוקן ו-200 עם טוקן תקין. רענון מאריך session בלי login מחדש.
**לא לעשות:** endpoints/UI לאימות אימייל ולשחזור סיסמה (Supabase מטפל), לוגיקת אימות עצמית. התחברות דרך פייסבוק עוברת ל-Session 1.3 (scopes `email`+`public_profile` בלבד, נפרדת מחיבור חשבון המודעות). **חיבור חשבון המודעות (Phase 1, `ads_management`) לעולם אינו הזהות** — הוא יושב *על* המשתמש; ה-`user_id` תמיד מ-Supabase Auth. (ראה spec §7א.)

**עדכון (1.3 — דרישת Dashboard, לא קוד):** ב-Supabase Auth יש **להפעיל email confirmation** **ולכבות auto-link** של OAuth identities — מניעת OAuth password takeover (דפוס קריטי 1): התחברות FB עם email שכבר רשום כחשבון סיסמה לא משתלטת עליו (1.3 מחזיר 409). זו הגדרת-Dashboard בלבד; הקוד כבר תומך — `sign_up` מחזיר `AuthTokens` בלי טוקנים כשאין session. **המשמעות:** signup ב-email/password לא יחזיר JWT מיד; המשתמש מאשר מייל ואז מתחבר (ה-Done לעיל תקף ל-flow ה-login; signup דורש אישור מקדים). (היה קודם "email confirmation כבוי" — בוטל לטובת מניעת takeover.)

### Session 0.5.2 — יצירת subscription אוטומטית ✅

- [x] ודא/השלם את טבלת `subscriptions` לפי spec §6: `status` (`trial`/`active`/`expired`/`canceled`), `tier` (`pending`/`basic`/`premium`/`whatsapp`), `lead_quota` (nullable), `trial_ends_at`, `current_period_start`, `billing_customer_id`+`billing_provider` (placeholder), מוני שימוש (placeholder)
- [x] **DB trigger** על `auth.users`: משתמש חדש → נוצרת אוטומטית שורת subscription עם `status=trial`, `tier=pending`, `lead_quota=null`
- [x] **RLS:** המשתמש קורא **רק** את השורה שלו (`auth.uid() = user_id`). אין insert/update ישיר מהמשתמש — יצירה דרך ה-trigger, עדכון דרך service בלבד
- [x] `services/subscription_service.py`: `get_user_subscription` (כפי שמופיע בסעיף 2א ב-spec) + `GET /me/subscription`

**Done:** משתמש חדש נרשם (0.5.1) ← מיד יש לו שורת subscription ב-`trial`/`pending`, בלי שום קוד אפליקציה שיצר אותה. RLS עובד — משתמש לא רואה מנוי של אחר.
**מצב מימוש:** SQL (`supabase/migrations/0001_subscriptions.sql`) + service + endpoint + טסטים (mocked) — הושלמו. ה-trigger וה-RLS הם ברמת Postgres → אימות e2e מול Supabase נדחה (כמו ב-0.5.1, אחרי 0.2). צ'קליסט אימות ידני ב-PR.
**לא לעשות:** בחירת חבילה (Phase 2.5 — כאן רק `pending`), סליקה (Phase 2.6), אכיפת מכסה (Phase 4.2). שדות הסליקה/מונים קיימים בסכמה אבל ריקים.
**זהירות:** ה-trigger חייב להיות `SECURITY DEFINER` עם `search_path` מפורש — אחרת זה חור אבטחה ידוע ב-Supabase/Postgres. זה החלק היחיד פה שדורש דיוק.

**עדכון — הרחבת התכנון (0.5.2):**

**מה ה-trigger קובע מול מה שנשאר NULL:**
- ה-trigger קובע בלבד: `status=trial`, `tier=pending`, `lead_quota=null`.
- נשארים **NULL** ונמלאים מאוחר יותר: `trial_ends_at` + `current_period_start`
  (Phase 2.6, ברגע הזנת הכרטיס — **לא** בהרשמה), `billing_customer_id` +
  `billing_provider` (2.6), `agent_chat_quota` + `agent_alerts_quota`
  (2.5, ממתינים לגיא).
- `user_id`: `UNIQUE NOT NULL` + FK ל-`auth.users` עם `ON DELETE CASCADE`
  (מנוי אחד למשתמש, נאכף ב-DB).

> **trial_ends_at לא נקבע ב-trigger.** לפי מודל גיא, 7 ימי ה-trial מתחילים
> ברגע שהלקוח מזין כרטיס בהקמת הקמפיין (flow שלב 6), לא בהרשמה. לכן
> `trial_ends_at` ו-`current_period_start` נקבעים ב-Phase 2.6. בהרשמה — NULL.

**ה-trigger (החלק היחיד שדורש דיוק):**
- `AFTER INSERT` על `auth.users`, `SECURITY DEFINER` עם `search_path` מפורש.
- כשל ב-trigger → **כל ההרשמה מתבטלת** (rollback). זה רצוי — אוכף את
  ה-invariant "משתמש ⟺ מנוי". המשמעות: באג ב-trigger חוסם את כל ההרשמות,
  אז הוא חייב להיות פשוט ונכון.

**RLS — רק קריאה:**
- מדיניות **SELECT** אחת: `auth.uid() = user_id`.
- **אין** מדיניות INSERT/UPDATE/DELETE למשתמש כלל — פיזית לא יכול לגעת
  בטבלה דרך ה-API. יצירה רק דרך ה-trigger (SECURITY DEFINER עוקף RLS).
  מונע לקוח שמשנה לעצמו `tier=premium` דרך Supabase ישירות.

**שירות + endpoint לבדיקה:**
- [ ] `subscription_service.get_user_subscription` (דרך user client, RLS אוכף)
- [ ] `GET /me/subscription` דק שקורא לו — מחזיר את שורת המנוי של המשתמש.
      אותו דפוס כמו `GET /me` מ-0.5.1, מאפשר אימות מהנייד שה-trigger וה-RLS עובדים.

**מוסכמה לתעד:** `status=trial` + `trial_ends_at=null` = "נרשם, זכאי ל-trial,
השעון עוד לא רץ". ה-enum לא מורחב בשביל זה.

**Done (מורחב):**
- משתמש נרשם → שורה ב-`trial`/`pending`, שאר השדות NULL.
- `GET /me/subscription` עם הטוקן מחזיר את השורה; טוקן של אחר — רק את שלו (RLS).
- ניסיון UPDATE ישיר דרך ה-API — נדחה.
- ה-trigger מאומת כ-SECURITY DEFINER עם search_path מפורש.
- הכל נבדק בלי Meta/OpenAI — שכבה דטרמיניסטית טהורה.

**לא לעשות (חידוד):** לא בחירת חבילה (2.5), לא סליקה (2.6), לא
`trial_ends_at`/`current_period_start` (2.6), לא אכיפת מכסה (4.2). מוני הסוכן —
העמודות נוצרות ריקות, המספרים מאוחר יותר.

---

## Phase 1 · Meta OAuth (flow שלב 1)

### Session 1.1 — מבנה החיבור + הפניית OAuth ✅
- [x] endpoint שמתחיל OAuth: בונה את ה-redirect ל-Meta עם ההרשאות
      הנכונות (`ads_management`, `ads_read`, `pages_show_list`, `pages_read_engagement`, `business_management`)
- [x] state parameter למניעת CSRF

**מצב מימוש:** `GET /auth/meta/start` (routers/fb.py) + state חתום ב-cookie.
signer משותף ב-`core/security.py` (HMAC, stdlib), בניית URL ב-`integrations/meta.py`,
אורקסטרציה ב-`fb_service.py`. Meta env אופציונליים (degradation → 503 אם לא מוגדר).
login אמיתי מול Meta → 1.2 / אחרי 0.2.
**עודכן ב-1.2:** ה-endpoint כעת **מאומת** ומחזיר JSON `{authorize_url}` (במקום 302),
וה-`user_id` מוטמע ב-state החתום — כי ל-callback (ניווט-דפדפן) אין Bearer לזיהוי המשתמש.
ה-URL נושא רק את ה-nonce (ה-user_id לא נחשף ל-Meta).

**Done:** קריאה ל-endpoint מחזירה redirect תקין ל-Meta (אפשר לבדוק שה-URL
נבנה נכון, גם בלי להשלים את הלוגין).
**לא לעשות:** עדיין לא לטפל ב-callback. רק את ההפניה.

**עדכון — הרחבת התכנון (1.1):**

**שמירת state (CSRF):** ב-httpOnly + Secure cookie חתום, `SameSite=Lax`, `Path=/auth/meta`, TTL של 5 דקות. אותו דפוס כמו ה-refresh token cookie מ-0.5.1 — מנגנון חתימה משותף ב-`core/security.py`. בלי טבלת `oauth_states` נפרדת — חוסך מיגרציה, רשומות, ו-cleanup.

**endpoint:** `GET /auth/meta/start` — בונה URL ל-Meta עם App ID, הרשאות (`ads_management`, `ads_read`, `pages_show_list`, `pages_read_engagement`, `business_management`), `auth_type=rerequest`, redirect URI, ו-state. שותל את ה-state ב-cookie ומחזיר 302. אין קריאה חיצונית — רק בניית URL.

**Env vars:** `META_APP_ID`, `META_APP_SECRET`, `META_REDIRECT_URI`. אין ערכים בקוד.

**Done (מורחב):** קריאה ל-`/auth/meta/start` מחזירה 302 ל-URL תקין של Meta + cookie state חתום. אפשר לבדוק את ה-URL ידנית בלי להשלים login.

**לא לעשות:** אין endpoint ניתוק, אין SELECT-קודם, אין endpoint מצב חיבור (זה ב-1.2).

### Session 1.2 — Callback + שמירת טוקן ב-Vault
- [x] endpoint callback שמקבל את הקוד מ-Meta, מחליף ל-access token
- [x] שמירת הטוקן **מוצפן דרך Vault** (לא plaintext) ב-`fb_connections`
- [x] שמירת `token_expires_at`

**Done:** משתמש משלים OAuth אמיתי, והטוקן נשמר מוצפן ב-DB. בדיקה: השורה
קיימת ב-`fb_connections`, והטוקן לא קריא כ-plaintext.
**לא לעשות:** עדיין לא לשלוף נכסים. רק לשמור את החיבור.
**זהירות (CLAUDE.md):** external-sdk patterns — טיפול בשגיאות OAuth,
טוקן שפג.

**עדכון — הרחבת התכנון (1.2):**

**ה-callback:** `GET /auth/meta/callback?code=...&state=...`. ה-router דק ומאמת state בלבד; כל הלוגיקה (כולל ה-admin client) ב-`fb_service`. שלבים: (1) קריאת ה-state מה-cookie והשוואה ל-state שב-query — אם לא תואם, 403 ומחיקת ה-cookie. (2) החלפת ה-code לטוקן short-lived מול Meta. (3) **החלפה מיידית ל-long-lived** (~60 יום) — בלי זה ה-cron של Phase 6 לא יעזור, הטוקן יפוג אחרי שעתיים. (4) שליפת `meta_user_id` (קריאה ל-`/me` של Meta). (5) UPSERT ל-`fb_connections` דרך `fb_service`. (6) מחיקת ה-state cookie. (7) redirect לדף ההצלחה ב-frontend.

**מבנה `fb_connections`:** `user_id` (FK ל-`auth.users` עם `ON DELETE CASCADE` + UNIQUE — אחד למשתמש), `meta_user_id`, `encrypted_token` (Vault reference, לא plaintext), `token_expires_at`, `refresh_failed_at` (nullable, ל-Phase 6), `refresh_error` (nullable, ל-Phase 6), `created_at`, `updated_at`. RLS: SELECT בלבד למשתמש (`auth.uid() = user_id`); אין INSERT/UPDATE/DELETE — כתיבה רק דרך **`fb_service`** (ה-service הייעודי שמחזיק את ה-admin client), לפי spec §7.3(ב): פעולה privileged server-authoritative שאסור שתהיה בשליטת המשתמש (לא webhook). ה-router לא נוגע ב-admin client — קורא ל-`fb_service`.

**UPSERT + מודאל אישור:** `fb_service` עושה UPSERT ב-`user_id` UNIQUE — דריסה פשוטה אם יש כבר חיבור. אכיפת האישור-לפני-דריסה היא ב-frontend (יטופל כשייבנה), לא ב-backend. נימוק: המשתמש שמגיע ל-OAuth ממילא רואה לאן הוא מופנה במטא ויכול לבטל שם — backend enforcement יוסיף סיבוך בלי להגדיל ביטחון.

**endpoint מצב חיבור (חדש):** `GET /me/fb-connection` — מחזיר 200 עם `meta_user_id`, `token_expires_at`, `created_at` (**בלי** ה-token עצמו), או 404 אם אין. נדרש כדי שה-frontend ידע אם להציג את מודאל האישור לפני OAuth חוזר. RLS מבטיח שמשתמש רואה רק את שלו.

**הצפנת הטוקן:** Supabase Vault. הטוקן לעולם לא נשמר plaintext, לעולם לא חוזר ל-frontend. שליפה לקוד שצריך אותו (Phase 2 ואילך) — דרך `meta_service`, שקורא ל-Vault.

**Done (מורחב):** משתמש משלים OAuth אמיתי על חשבון admin של אפליקציית Meta ב-Development → שורה ב-`fb_connections` עם טוקן long-lived מוצפן. בדיקה: השורה קיימת, הטוקן לא קריא כ-plaintext ב-SELECT רגיל. `GET /me/fb-connection` עם הטוקן של המשתמש מחזיר את הפרטים (בלי הטוקן). חיבור חוזר עם חשבון אחר → דורס את הקודם (UPSERT). state לא תואם → 403.

**לא לעשות:** עדיין לא לשלוף נכסים (Phase 2). אין endpoint ניתוק (לא נדרש כי UPSERT פותר חיבור חוזר). אין לוגיקת רענון טוקן (Phase 6) — רק העמודות בסכמה.

**מצב מימוש (1.2):** `GET /auth/meta/callback` (בלי auth — הזהות מגיעה מה-user_id החתום ב-state) + `GET /me/fb-connection` (מאומת, RLS). זרימה ב-`fb_service.complete_oauth`: אימות state → `exchange_code_for_token` → `exchange_long_lived_token` → `get_meta_user_id` → UPSERT דרך admin client. הטוקן מוצפן ב-**Vault** דרך RPC `upsert_fb_connection` (SECURITY DEFINER, service_role-only) במיגרציה `0002_fb_connections.sql`. סיווג שגיאות Meta מרוכז ב-`integrations/meta.classify_meta_error` (transient/permanent/unknown — 2.1 ישתמש בו שוב); סיווג PostgREST חולץ ל-`app/db/errors.py` המשותף. GRANT עמודתי על `fb_connections` (בלי `encrypted_token`). `FRONTEND_URL` חדש ליעד ה-redirect. 143 טסטים עוברים. אימות OAuth/Vault אמיתי מול Supabase + Meta → אחרי 0.2 (דורש Vault מופעל).

**זהירות (CLAUDE.md):** external-sdk patterns — טיפול בכשלי OAuth, code שכבר נוצל, state שפג, Meta שמחזירה שגיאה. כל אלה צריכים שגיאה מובהקת ל-frontend + ניקוי ה-cookie.

### Session 1.3 — התחברות דרך פייסבוק (Facebook Login)

- [x] הפעלת **Facebook Provider** ב-Supabase Auth — **אותו** Meta App, אבל scopes
      `email`+`public_profile` **בלבד** (נפרד לחלוטין מ-`ads_management` של חיבור חשבון המודעות, 1.1/1.2).
- [x] כפתור "התחבר עם פייסבוק" כאופציית login לצד email/סיסמה. Supabase מחזיר את אותם
      access/refresh tokens — שכבת הטוקנים מ-0.5.1 לא משתנה, וה-trigger יוצר subscription כרגיל.

**Done:** משתמש נרשם/מתחבר דרך פייסבוק → נוצר אותו `user_id` ושורת subscription (`trial`/`pending`),
בדיוק כמו email/סיסמה. הזהות נשארת של Supabase Auth.
**לא לעשות:** לא לערבב עם חיבור חשבון המודעות (1.1/1.2) — שני OAuth שונים, scopes שונים.
**Magic Link ✅ (מומש):** כניסה-בלי-סיסמה **לרשומים-בלבד** — `POST /auth/magic-link` (pre-check `email_exists`
+ `should_create_user=false` → לא רושם חדשים; לא-קיים → 404 "הירשם תחילה") → מייל עם token_hash → `GET /auth/magic-link/verify`
(verify_otp server-side → refresh ב-httpOnly cookie → `/login/success`, כמו חזרת FB). דורש התאמת email-template (SETUP_CHECKLIST).
**אבטחה (CLAUDE.md דפוס קריטי 1 — OAuth password takeover) — הוכרע (1.3):** Supabase ממזג זהות FB
לחשבון email/סיסמה קיים אוטומטית על email **מאומת-ע"י-הספק**, ואין toggle שמכבה זאת. **מקובל** כי
פייסבוק מחזיר email רק אם אומת מול התיבה — תוקף בלי גישה לתיבת-הקורבן לא יגיע למיזוג (וגישה לתיבה =
takeover ממילא דרך "שכחתי סיסמה"). אימות-פייסבוק הוא שכבת-ההגנה; ה-409 הוא הגנת-עומק. אימות לפני go-live:
`docs/deployment/facebook-login-verification.md`.

**מצב מימוש (1.3):** Backend-mediated PKCE — `GET /auth/oauth/facebook/start` (302 ל-Supabase
authorize + PKCE verifier חתום ב-cookie) ו-`GET /auth/oauth/facebook/callback` (מחליף code→session,
שותל refresh ב-httpOnly cookie, ואז redirect ל-`{FRONTEND_URL}/login/success` **בלי טוקן ב-URL** —
ה-frontend מאתחל access דרך `POST /auth/refresh` הקיים). את ה-PKCE מייצרים אצלנו
(`core/security.generate_pkce_pair`) כי ל-supabase-py ברירת מחדל `flow_type="pkce"` שדורסת
code_challenge מוזרק; לכן ה-URL נבנה ידנית (`auth_service.oauth_authorize_url`) וה-exchange מקבל
`code_verifier` מפורש (`exchange_oauth_code`). **מגן takeover (דפוס קריטי 1):** email קיים → **409**
("האימייל הזה רשום…") דרך מסווג מרכזי `classify_oauth_callback_error` + זיהוי `email_exists` ב-exchange.
176 טסטים עוברים.
**דרישות Dashboard (לא קוד, לאחר 0.2):** הפעלת Facebook provider + **App ID/Secret**; רישום `OAUTH_REDIRECT_URI`
ב-Redirect URLs; **הפעלת email confirmation** (signup-סיסמה; OAuth פטור — משתמש FB חדש נכנס מיד); **auto-link
מקובל** (אין toggle לכבותו — אימות-פייסבוק הוא הגדר, לא הגדרת-Dashboard); ב-Meta App להוסיף ל-Valid OAuth
Redirect URIs את `https://<ref>.supabase.co/auth/v1/callback` (לצד ה-callback של 1.1/1.2). **frontend מחווט (1.3):**
כפתור "חיבור מהיר באמצעות פייסבוק" → `startFacebookLogin()`→`/auth/oauth/facebook/start`; חזרה דרך
`_handleOAuthLoginRedirect()` (`/login/success`→refresh+ניתוב, `/login?error=`→הודעה). **go-live מותנה ב-runbook:**
`docs/deployment/facebook-login-verification.md` (כולל מבחן-תקיפה אדוורסרי T2b).

---

## Phase 2 · שליפת נכסים (flow שלבים 2-3)

### Session 2.1 — שליפת דפים וחשבונות מודעות ✅
- [x] endpoint ששולף מ-Meta API את הדפים וחשבונות המודעות של המשתמש
- [x] מחזיר רשימה ל-frontend לבחירה

**Done:** משתמש מחובר מקבל את רשימת הנכסים האמיתית שלו מ-Meta.
**לא לעשות:** עדיין לא לשמור בחירה. רק שליפה והצגה.

**עדכון — הרחבת התכנון (2.1):**

**היקף Session 2.1 (נקבע בתכנון):** בונים **רק** את `GET /me/meta-assets` (דפים + חשבונות מודעות — מסלול הלידים, שהוא ליבת ה-MVP).

> **דחייה מודעת → מומש ב-Session 2.1.5:** `/me/whatsapp-business-numbers` נבנה (ה-scope `whatsapp_business_management` נוסף ל-§9). השליפה דרך user-token SDK (`/me/businesses`→owned/client WABAs→phone_numbers, `meta.list_user_whatsapp_numbers`), עם degrade-to-empty על permission/scope-חסר (כמו `(#100)` ב-`list_pages`) → frontend מפנה למדריך הקמה. אישור המספר ב-onboarding step-3 (אחרי חיבור Meta — שם קיים הטוקן). הערה היסטורית: בתכנון 2.1 זה נדחה בכוונה (לא טלאי endpoint שמחזיר 401 על scope חסר) ובוצע כיחידה נפרדת.

**`GET /me/meta-assets`** — מחזיר דפים (`/me/accounts`) + חשבונות מודעות (`/me/adaccounts`). נקרא במסלול לידים **ובמסלול וואטסאפ** (שניהם צריכים את הנכסים). מחזיר **רק שדות לא-רגישים** — לעולם לא את ה-page access_token (סוד; כלל 6). **לדפים** נוסף `picture` (URL ציבורי לתמונת-הפרופיל — לבחירה ויזואלית במסך בחירת-הדף; `is_silhouette` מושמט → אווטאר-אות). **חשבונות-מודעות בלי `picture`** (אין תמונת-פרופיל ב-Meta).

**טוקן Meta — שליפה ופענוח (תוקן בתכנון 2.1):** הנתיב המקורי שתוכנן כאן ("`meta_service` שולף את ה-encrypted token דרך ה-user client + RLS, מפענח מ-Vault") **אינו אפשרי** עם מודל האבטחה שנבנה ב-1.2: ה-GRANT העמודתי **אינו** חושף את `encrypted_token` ל-`authenticated`, ו-Vault נגיש ל-`service_role` בלבד (וטוב שכך — הקשחה נכונה). הנתיב השורשי: `meta_service` קורא ל-`fb_service.get_decrypted_token(user_id)`, שמפענח דרך **RPC ייעודי** (`get_fb_token`, `SECURITY DEFINER`, `search_path=''`, service_role-only — אותו דפוס בדיוק כמו `upsert_fb_connection` מ-1.2). ה-`user_id` מגיע מה-JWT המאומת (server-authoritative, §7.3-ב), לא מגוף הבקשה. הטוקן עצמו לא חוזר ל-frontend — רק תוצאות הקריאה ל-Meta.

**אין חיבור Meta עדיין:** אם למשתמש אין שורת `fb_connections` → שגיאת דומיין `FbConnectionMissingError` → ה-router מחזיר **409 מתויג** (`code: "fb_not_connected"`; ה-frontend מפנה ל-OAuth של Phase 1). לא 500.

**בלי caching:** כל קריאה — ל-Meta. המשתמש בוחר נכס פעם אחת בחיי הקמפיין, זו לא קריאה חוזרת. אופטימיזציה לא נדרשת ב-MVP; קל להוסיף אחר כך אם יתברר שצריך.

**שגיאה ספציפית לטוקן פג / הרשאה נשללת:** מטא מחזירה 401/403 → `integrations/meta` מסווג כ-`MetaPermanentError` (הסיווג המרכזי הקיים מ-1.2) → ה-router מחזיר **409 מתויג** (`code: "fb_token_invalid"`), לא שגיאה גנרית. **למה 409 ולא 401:** ה-session של האפליקציה תקין — רק חיבור ה-Meta נשבר; 401 היה מתנגש ב-interceptors של ה-frontend שמטפלים ב-401 כ"session פג" (logout/refresh מיותר). 409 + `code` מבדיל "לחבר מחדש את Meta" מ"להתחבר מחדש לאפליקציה". ה-frontend מפנה ל-OAuth של Phase 1. נשען על המסווג המרכזי `classify_meta_error`, מקום אחד.

**Done:** משתמש מחובר קורא ל-`/me/meta-assets` ומקבל את הדפים וחשבונות המודעות האמיתיים שלו. אין חיבור → 409 (`fb_not_connected`). טוקן פג → 409 מתויג (`fb_token_invalid`), לא 500. (שליפת מספרי וואטסאפ — נדחתה ל-Session 2.1.5/Phase 5, ראה למעלה.)

**לא לעשות (במסגרת 2.1):** עדיין לא לשמור בחירה (זה ב-2.2). אין caching. (`/me/whatsapp-business-numbers` + ה-scope `whatsapp_business_management` + לוגיקת "אין WhatsApp Business → מדריך הקמה" — **מומשו בנפרד ב-Session 2.1.5**, ראה ה-blockquote למעלה.)

**מצב מימוש (2.1.5 — שליפת מספר WhatsApp Business):** `GET /me/whatsapp-business-numbers` (מאומת) ב-`routers/fb.py` → `meta_service.get_whatsapp_business_numbers` → `meta.list_user_whatsapp_numbers` (SDK traversal: `/me/businesses`→owned/client WABAs→`phone_numbers`; שמות ה-SDK באיות `whats_app`). scope `whatsapp_business_management` נוסף ל-`OAUTH_SCOPES`. מיפוי שגיאות: transient→503, permanent→409 (`fb_token_invalid`), permission/scope-חסר (`MetaUnexpectedError`)→**degrade-to-empty**→frontend מציג מדריך הקמה (כמו `(#100)` ב-`list_pages`; האסימטריה מול list_pages — שם בולעים גם transient/permanent כי יש primary, פה אין — מוצדקת ומתועדת). frontend: אישור המספר ב-onboarding **step-3** (אחרי חיבור Meta), `loadWhatsAppNumber`/`confirmWANumberAt`/`_to_whatsapp_numbers`. טסטים: `tests/test_meta_assets.py` (integration+service+router, 62 עוברים). **אימות אמיתי מול Meta — אחרי deploy** (scope מאושר ב-App Review + WABA אמיתי; ל-Dev אדמין/tester מיידי).

**מצב מימוש (2.1):** `GET /me/meta-assets` (מאומת) ב-`routers/fb.py` → `meta_service.get_meta_assets` → `integrations/meta.list_pages`/`list_ad_accounts`. פענוח הטוקן דרך `fb_service.get_decrypted_token` (admin client) הקורא ל-RPC `get_fb_token` (SECURITY DEFINER, search_path ריק, service_role-only) במיגרציה `0003_fb_token_rpc.sql`. **ה-RPC תלת-מצבי** (`returns table(connection_exists, token)`) כדי לא לבלבל "לא מחובר" עם "מחובר אך הטוקן חסר ב-Vault": אין שורה → 409 `fb_not_connected`; שורה+טוקן → הצלחה; שורה בלי טוקן → `FbTokenUnavailableError` → 500 (אנומליית integrity, לא ניחוש). מודלים ב-`models/meta_assets.py` — **בלי שדות רגישים** (page access_token לא נשלף כלל; כלל 6). מיפוי שגיאות נשען על `classify_meta_error` הקיים מ-1.2: 409 מתויג (`fb_not_connected`/`fb_token_invalid`) / 503 / 500. 216 טסטים עוברים (32 בקובץ `test_meta_assets.py`). **אימות אמיתי מול Meta + Vault (פענוח `get_fb_token`) → אחרי 0.2** (כמו 1.2/1.3 — דורש Vault מופעל וטוקן long-lived אמיתי).

**זהירות (CLAUDE.md):** external-sdk patterns — Meta API rate limits, timeout, תשובה לא תקינה, טוקן פג. כל אלה צריכים סיווג שגיאה מובהק (נשען על `classify_meta_error` הקיים).

### Session 2.2 — שמירת בחירת נכס בקמפיין
- [x] יצירת campaign טיוטה עם הנכס הנבחר (`fb_ad_account_id`,
      `fb_page_id`, ובמסלול whatsapp גם `whatsapp_phone_number_id`)
- [x] שמירת `type` (`service_name` עובר ל-Phase 3.1 — ראה הרחבה למטה)

**Done:** קמפיין טיוטה נוצר ב-DB עם הנכסים הנכונים, שייך למשתמש (RLS עובד).
**לא לעשות:** עדיין לא יצירת קמפיין אמיתי ב-Meta.

**עדכון — הרחבת התכנון (2.2):**

**endpoint:** `POST /campaigns` — יוצר רשומת קמפיין חדשה עם הנכסים שנבחרו.

**מבנה הקמפיין בעת יצירה:**
- `user_id` (FK, RLS), `status='draft'` (התחלתי), `type` (`lead` או `whatsapp`).
- **נכסי Meta:** `fb_ad_account_id`, `fb_page_id`, ובמסלול וואטסאפ גם `whatsapp_phone_number_id`.
- `service_name` **נשאר null בשלב הזה** — מתמלא ב-Phase 3.1 (האשף, שלב 0/2 — שם השירות יוצא ממטרת הקמפיין/ההצעה השיווקית; לפי הפרוטוטייפ סעיף 7.2). זה הפרדה נכונה: 2.2 = *איפה* (איזה נכס); 3.1 = *מה* (איזה שירות).

**enum status של campaigns:** `draft`/`pending_review`/`pushing`/`live`/`paused`/`archived`. כאן אנחנו רק עם `draft`. שאר הערכים נכנסים בהמשך (3.4 וייתר ה-Phases). ה-enum מוגדר מהיום הראשון כדי שלא יהיה טלאי בעתיד.

**RLS:** SELECT/INSERT/UPDATE/DELETE למשתמש על קמפיינים שלו (`auth.uid() = user_id`). שונה מ-`fb_connections` (שם רק SELECT) — קמפיין הוא יישות שהמשתמש יוצר ומנהל בעצמו. אין צורך ב-admin client כאן; קוד 7.3(ב) לא חל.

**ולידציה ב-service:**
- `type='lead'` → חייב `fb_ad_account_id` ו-`fb_page_id`. `whatsapp_phone_number_id` חייב להיות null.
- `type='whatsapp'` → חייב גם `whatsapp_phone_number_id`. בלעדיו → 400.
- ה-`fb_ad_account_id`/`fb_page_id`/`whatsapp_phone_number_id` חייבים להיות מהנכסים *שהמשתמש באמת מחזיק* (לא ערך שרירותי). בדיקה: לפני שמירה, קריאת אימות מול Meta — האם המשתמש באמת בעלים של הנכסים האלה. שכבת הגנה נגד שליחת ערכים מזויפים מה-frontend.

**Done:** קמפיין draft נוצר ב-DB עם הנכסים הנכונים, שייך למשתמש (RLS עובד), ולידציה מונעת קמפיין עם נכסים שאינם של המשתמש או חוסר התאמה בין type לנכסים.

**לא לעשות:** עדיין לא יצירת קמפיין אמיתי ב-Meta (Phase 3.4). עדיין לא `service_name` (Phase 3.1). אין שאלון, אין מודעות, אין דחיפה.

**זהירות:** הקמפיין-draft יכול להיתקע במצב הזה לנצח אם המשתמש נטש באמצע. ב-MVP — בסדר, נשאר ב-DB. אם בעתיד יהיה צורך — cleanup ל-drafts ישנים. לא עכשיו.

**מצב מימוש (2.2):** `POST /campaigns` (מאומת) ב-`routers/campaigns.py` → `campaign_service.create_campaign` → אימות בעלות דרך `meta_service.get_meta_assets` → INSERT דרך user client (RLS). מיגרציה `0004_campaigns.sql`: טבלה עם CHECK constraints (type ∈ lead/whatsapp, status ∈ 6 ערכים, עקביות type↔whatsapp_phone_number_id), RLS עם CRUD מלא למשתמש, GRANT ל-authenticated, אינדקס על user_id. ולידציית type: lead → חייב page+account, אסור phone; whatsapp → חייב גם phone. אימות בעלות: שולף נכסים מ-Meta ומוודא שה-IDs שנשלחו קיימים ברשימה — הגנה נגד ערכים מזויפים מה-frontend. אימות whatsapp_phone_number_id מול Meta נדחה ל-Phase 5 (ה-scope עוד לא קיים). מיפוי שגיאות: 400 (ולידציה/בעלות), 409 מתויג (fb_not_connected/fb_token_invalid), 503 (חולף), 500 (לא מזוהה). מודלים ב-`models/campaign.py`. 244 טסטים עוברים (28 חדשים בקובץ `test_campaigns.py`).

---

## Phase 2.5 · בחירת חבילה

### Session 2.5 — עדכון חבילה ושמירת ToS

ה-Phase שמעדכן את ה-subscription מ-`tier=pending` לחבילה נבחרת, שומר את המכסה ואת אישור ה-ToS באותה עסקה. חוסם את Phase 4 (אכיפת מכסה צריכה `lead_quota` לא-null) ואת Phase 5 (הבוט הוא Premium-only — צריך `tier` ידוע).

- [x] endpoint `PATCH /me/subscription` — מקבל `{tier, volume?, tos_version}`
- [x] ולידציה ב-router: `tier` אחד מ-`basic`/`premium`/`whatsapp`; `volume` חובה ב-basic/premium ובאחד מ-[500, 1000]; ב-whatsapp לא נשלח (או מתעלמים ממנו); `tos_version` חובה (string לא ריק)
- [x] `subscription_service.update_tier(user_id, tier, volume, tos_version)` — מחזיק את ה-admin client (לפי §7.3(ב)):
  - שולף את ה-subscription הנוכחי
  - אם `tier != 'pending'` → raise `SubscriptionAlreadySelectedError` (router מחזיר 409 Conflict + הודעה: "חבילה כבר נבחרה. ליצירת קשר לשינוי.")
  - **`lead_quota` נגזר ב-service** מ-tier+volume: `basic`/`premium` → 500 או 1000; `whatsapp` → null. **לא** מתקבל מ-`request.body`.
  - UPDATE על subscriptions: `tier`, `lead_quota`, `tos_version`, `tos_accepted_at=now()` — באותה עסקה
  - `current_period_start` נשאר null (יסונכרן עם ספק הסליקה ב-Phase 2.6)
- [x] `subscription_service.get_user_subscription(user_id)` — קיים מ-0.5.2, **להרחיב**: מחזיר גם `is_locked` (computed): `status IN ('canceled', 'expired') AND trial_ends_at IS NOT NULL AND trial_ends_at < now()`. ב-Phase זה תמיד `false` בפועל, אבל הקוד נכנס פעם אחת.

**Done:** משתמש בוחר חבילה ב-UI → קריאת `PATCH /me/subscription` → ה-subscription עובר מ-`pending` לסוג שנבחר עם המכסה הנכונה ועם ToS שמור. `get_user_subscription` מחזיר עכשיו גם `is_locked`. ניסיון לעדכן שוב (לאחר שכבר נבחרה חבילה) → 409.

**לא לעשות:**
- אין סליקה (Phase 2.6)
- אין אכיפת מכסה (Phase 4.2)
- אין מעברים שדרוג/downgrade/החלפת תחום — רק מ-pending. עתידי, ממתין להכרעת גיא.
- אין מילוי `current_period_start` (Phase 2.6)

**זהירות:**
- `tier`, `lead_quota`, `tos_accepted_at` הם **server-authoritative**. ה-service לא מקבל אותם ישירות מ-`request.body`. `lead_quota` נגזר מ-tier+volume; `tos_accepted_at` תמיד `now()` של השרת.
- ה-admin client רק ב-`subscription_service`, לעולם לא ב-router. ה-router קורא ל-service.
- **בדיקות חייבות לכלול:** ניסיון לעדכן שוב מ-tier לא-pending → 409; ניסיון לשלוח volume לא חוקי (200/750) → 400; ניסיון לשלוח `tos_version` ריק → 400; ניסיון לעדכן user_id אחר → RLS חוסם (אבל ה-service ממילא מקבל user_id מ-`get_current_user`, לא מ-body).

**מצב מימוש (2.5):** `PATCH /me/subscription` (מאומת) ב-`routers/subscription.py` → `subscription_service.update_tier` (admin client, §7.3-ב). מיגרציה `0005_subscription_tos.sql`: הוספת `tos_version` + `tos_accepted_at` (nullable; ה-SELECT policy + GRANT מ-0001 כבר חלים, כתיבה רק דרך admin). ולידציה ב-`UpdateTierRequest` (Pydantic, שכבת ה-router): `tier` ∈ basic/premium/whatsapp, `volume` ∈ 500/1000 (חובה ב-basic/premium דרך model_validator), `tos_version` לא-ריק (strip). **server-authoritative:** `lead_quota` נגזר ב-service מ-tier+volume (`_derive_lead_quota`), `tos_accepted_at`=now() של השרת — **לא** מ-body. **CAS אטומי (כלל 2):** `UPDATE ... WHERE user_id=? AND tier='pending'`; rowcount=0 → בירור (אין מנוי→404 / כבר נבחר→409), לא SELECT-ואז-UPDATE. `is_locked` מומש כ-`computed_field` ב-`SubscriptionResponse` (נגזר מ-status+trial_ends_at, נכלל ב-serialization של GET ושל PATCH; fail-open ל-naive datetime). מיפוי שגיאות: 409 (already-selected), 404, 422 (ולידציה), 503 (חולף), 500 (לא-מזוהה). הערה: ה-ROADMAP כתב "400" לוולידציה אך Pydantic מחזיר **422** (תקן FastAPI) — אותה משמעות (בקשה שגויה). 293 טסטים עוברים (40 חדשים ב-`test_subscription.py`). **אימות אמיתי מול Supabase (CAS, RLS, trigger) → אחרי 0.2** (כמו שאר ה-Phases).

---

## Phase 2.6 · סליקה אוטומטית

> תלוי ב-2.5 (חבילה נבחרה → יודעים על מה לחייב). חוסם את הקמת הקמפיין בפועל (flow שלב 6 — תשלום), אבל **לא** את אבן הדרך של 3.4 (אפשר לבדוק את הליבה עם לקוחות בטא שעוברים ידנית). אפשר לבנות במקביל ל-Phase 3.

### Session 2.6.1 — אינטגרציית פלאקארד (Pelecard)

**עדכון — הרחבת התכנון (2.6.1):**

**ספק הסליקה: פלאקארד.** PSP ישראלי
נממש Stripe **אחרי ה-MVP**

**חשבוניות מס:** פלאקארד מטפלת **בכסף בלבד**, לא במסמך המס. חשבונית מס/קבלה דורשת אינטגרציה נפרדת (Green Invoice / iCount). נממש בפאזות הבאות.

**קריאה מקדימה של כל הקבצים הבאים חובה לפני קוד:** `docs/integrations/pelecard/SKILL.md` + `docs/integrations/pelecard/references/*`. בלי זה — באגים שעולים כסף אמיתי בפרודקשן (J2 vs J4, אגורות vs ש"ח, dedupe על `PelecardTransactionId` ולא `ParamX`, IPN לא חתום ב-HMAC).

**Match API החדש vs Gateway21 הישן (החלטה לפני קוד):** פלאקארד מציעה שני משטחי API — Gateway21 הוותיק (`gateway21.pelecard.biz`, מתועד היטב ב-Postman) ו-Match API החדש (`match-api.pelecard.biz`, REST מודרני אבל תיעוד פומבי חלקי). **לפני שמתחילים לקודד — אמיר/גיא יפנו לתמיכת פלאקארד לאמת על איזה משטח לבנות.** ברירת המחדל לפתיחה: Gateway21 (מתועד יותר, פחות סיכון). אם תמיכת פלאקארד ממליצה על Match — מעבירים. שינוי בין השניים = שינוי במודול אחד (`integrations/pelecard.py`), לא בקוד שמעליו.

#### שלבי הביצוע

- [x] **0. אישור ספק** — קבלת `terminal`/`user`/`password` ל-sandbox ול-production מפלאקארד. הגדרת תיבת פלאקארד שעובדת מול חשבון Gateway20 sandbox. בלי זה — אי אפשר להתחיל.

- [x] **1. תיעוד פלאקארד בריפו** — העתקת `docs/integrations/pelecard/` (SKILL.md + references/ + scripts/validate_pelecard_response.py). תוספת ל-CLAUDE.md שמפנה לקריאה לפני נגיעה ב-`billing.py`.

- [x] **2. `integrations/pelecard.py`** — עטיפה מבודדת לפלאקארד. ה-API החיצוני שמודולים אחרים רואים הוא **אגנוסטי** (`create_customer`, `start_trial_subscription`, `charge_now`, `cancel_subscription`, `verify_transaction`). הפנימיות (terminal/user/password, agorot, Gateway21 vs Match) נשארות כאן ולא דולפות החוצה. env vars: `PELECARD_TERMINAL`, `PELECARD_USER`, `PELECARD_PASSWORD`, `PELECARD_HOST` (`gateway20.pelecard.biz/sandbox` בפיתוח, `gateway21.pelecard.biz` בפרודקשן). ראה spec §2א חוק 3.

- [x] **3. `integrations/billing.py`** — שכבת abstraction דקה מעל `pelecard.py`. ה-service קורא ל-`billing.*`, לא ל-`pelecard.*`. אם נחליף ספק בעתיד — מחליפים רק את ה-implementation מתחת.

- [x] **4. הרחבת סכמת `subscriptions`** (migration ייעודי). השדות:
  - `billing_customer_id` (text, nullable) — מזהה הלקוח אצל פלאקארד.
  - `billing_provider` (text, nullable) — `'pelecard'` עם enum מוכן להחלפה.
  - `billing_token` (text, nullable) — **רפרנס לטוקן ב-Vault**, לא הטוקן עצמו. (פלאקארד מנפיקה טוקן שמייצג את הכרטיס; שמירת plaintext = חור אבטחה כמו עם Meta token ב-1.2.)
  - `trial_ends_at` (timestamptz, nullable) — נמלא בשלב 6 בזמן הזנת הכרטיס (לא ב-signup).
  - `current_period_start` (timestamptz, nullable) — נמלא יחד עם `trial_ends_at`.
  - `last_billing_error` (text, nullable) — שגיאת חיוב אחרונה.
  - `last_billing_error_at` (timestamptz, nullable).

- [x] **5. הרחבת ערכי `status`** — הוספת `past_due`. ⚠️ `status` הוא `TEXT` + `CHECK` (לא Postgres enum) כבר מ-0.5.2 (`0001_subscriptions.sql`), אז ההרחבה היא `ALTER TABLE public.subscriptions DROP CONSTRAINT subscriptions_status_check, ADD CONSTRAINT subscriptions_status_check CHECK (status IN ('trial','active','past_due','expired','canceled'))` — **לא** `ALTER TYPE`. שלבי מעבר: `trial` → `active` (חיוב יום 8 הצליח) → `past_due` (חיוב חודשי נכשל, עוד מנסים) → `expired` (נכשל סופית) או `canceled` (משתמש ביטל). ראה spec §6 לפירוט.

- [x] **6. `subscription_service.start_trial_with_billing`** — endpoint `POST /me/subscription/start-billing`. מותנה ב-`tier != 'pending'` (משתמש בחר חבילה ב-2.5) וב-`billing_customer_id IS NULL` (אין עוד מנוי פעיל). **מותנה גם ב-`billing_profiles` קיים עם `full_name`** (ראה 6.5–6.7) — אם חסר, מחזיר `400` עם רשימת שדות חסרים; הקליינט מציג טופס, שולח `PATCH /me/billing-profile`, ורק אז חוזר לכאן. זרימה: (א) פתיחת iframe של פלאקארד (`pelecard.create_iframe_session`) עם `ActionType=J2` (אימות כרטיס בלבד, לא חיוב) + `CreateToken=true`. (ב) מחזיר ל-frontend `{iframe_url, confirmation_key}`. ה-frontend מציג את ה-iframe. (ג) שמירת `confirmation_key` ב-טבלת עזר (`pending_billing_sessions`, פר user_id, TTL 30 דק׳) — לפי דפוס ה-state cookie של 1.1, אבל server-side כי ה-IPN לא נושא את ה-cookie.

- [x] **6.5. טבלה חדשה `billing_profiles`** (migration ייעודי). הפרופיל מאחסן את פרטי הלקוח שדרושים להפקת חשבונית (2.6.2). הטבלה ניתנת לעריכה בכל עת ממסך "החבילה שלי", אבל **שינוי תקף לחשבוניות מכאן והלאה בלבד** — חשבוניות ישנות לא נכתבות מחדש (גם דרישה חוקית).
  - `id` (uuid, PK)
  - `user_id` (FK ל-`auth.users`, UNIQUE, ON DELETE CASCADE) — מנוי אחד = פרופיל אחד
  - `full_name` (text, **NOT NULL**) — חובה. נדרש לחשבונית
  - `tax_id` (text, nullable) — ת"ז או ח.פ. אופציונלי. **נדרש לקיזוז מע"מ של הלקוח** — אם חסר, החשבונית מונפקת בלעדיו אבל הלקוח לא יוכל לקזז
  - `phone` (text, nullable) — שימושי לתמיכה ול-VIP alerts (Phase 5)
  - `business_name` (text, nullable) — אם הלקוח רוצה שהחשבונית תהיה על שם החברה ולא על שמו האישי
  - `created_at`, `updated_at` (timestamptz)

  RLS: SELECT למשתמש על שלו (`auth.uid() = user_id`). INSERT/UPDATE רק דרך service (admin client, spec §7.3(ב)) — לא דרך הקליינט, כי הוולידציה חייבת לעבור דרך service.

- [x] **6.6. הרחבת `subscription_service.start_trial_with_billing`** — לפני פתיחת iframe של פלאקארד, בדיקה ש-`billing_profiles` קיים למשתמש עם `full_name`. אם לא — `400` עם רשימת השדות החסרים. הקליינט מציג טופס מילוי, שולח `PATCH /me/billing-profile` (endpoint חדש ב-6.7), ואז מנסה שוב.

- [x] **6.7. endpoints לניהול הפרופיל:**
  - `GET /me/billing-profile` — מחזיר את הפרופיל של המשתמש (או 404 אם אין).
  - `PATCH /me/billing-profile` — יוצר או מעדכן. ולידציה: `full_name` חייב, השאר אופציונלי. אם `tax_id` מסופק — בדיקת פורמט בסיסית (9 ספרות לת"ז, 9 ספרות לח.פ.).

**עדכון לטופס הזנת אמצעי תשלום (frontend):**
המסך של "הזנת אמצעי תשלום" (flow שלב 6 בפרוטוטייפ 7.1) **מורחב לכלול את הפרופיל בראש המסך**, לפני iframe של פלאקארד:
- שדה **שם מלא** — חובה
- שדה **ת.ז / ח.פ.** — אופציונלי, עם טקסט מעליו: **"דרוש לקיזוז מע"מ. אופציונלי — תוכל למלא בהמשך, אבל לא תוכל לקזז מע"מ על חשבוניות שיופקו עד אז."**
- שדה **טלפון** — אופציונלי
- שדה **שם עסק** — אופציונלי, עם טקסט: "אם תרצה שהחשבונית תהיה על שם החברה."
- ואז ה-iframe של פלאקארד למטה.

זרימה: הקליינט שולח `PATCH /me/billing-profile` עם הפרטים, מקבל 200, ורק אז קורא ל-`POST /me/subscription/start-billing` שפותח את ה-iframe.

- [x] **7. webhook לקבלת אישור הכרטיס (IPN של פלאקארד)** — `POST /webhooks/billing/pelecard` במודול המבודד `routers/webhooks.py` (spec §7.3(א)). **אימות חובה לפני כל פעולה:** (א) השוואת `ConfirmationKey` ל-`confirmation_key` השמור. (ב) קריאה חוזרת ל-`pelecard.verify_transaction(transaction_id)` — פלאקארד לא חותמת IPN ב-HMAC, אימות חוזר server-to-server הוא **חובה**. (ג) בדיקת `DebitTotal == 0` (J2, לא חויב). (ד) idempotency דרך `webhook_events` עם `key=PelecardTransactionId` (לא `ParamX` — ראה SKILL.md). אחרי אימות מוצלח: שמירת הטוקן ב-Vault עם רפרנס ב-`billing_token`; שמירת `billing_customer_id`, `billing_provider='pelecard'`; הגדרת `trial_ends_at = now() + 7 days`, `current_period_start = now()`; מחיקת ה-`pending_billing_session`.

- [x] **8. cron יומי לחיוב ראשון (יום 8)** — job מתוזמן ב-worker שרץ פעם ביום: שולף subscriptions עם `status='trial'` ו-`trial_ends_at < now()`, מחייב דרך `pelecard.charge_token` (J4, ₪לפי tier × 100 = אגורות), ומעדכן `status='active'`. כשל חיוב → `status='past_due'`, `last_billing_error`. **לא בונים מנוע חיובים שלם** — פלאקארד לא מציעה recurring native (ראה תת-החלטה למטה), אנחנו רק יוזמים חיוב על טוקן שמור.

- [x] **9. cron חודשי לחיובים חוזרים** — אותו job, שולף subscriptions עם `status='active'` ו-`current_period_start + 1 month < now()`, מחייב, מעדכן `current_period_start = current_period_start + 1 month`. כשל → `past_due` + retry policy (3 ניסיונות ב-3 ימים, אחרי זה → `expired`).

- [x] **10. endpoint ביטול** — `POST /me/subscription/cancel`. מעדכן `status='canceled'`, **לא** מבטל את הטוקן בפלאקארד (הלקוח ממשיך לקבל גישה עד `current_period_start + 1 month`, לפי תנאי השימוש בפרוטוטייפ 7.8). cron יומי אחר מנקה טוקנים של מנויים שעבר תאריך הגישה שלהם.

- [x] **11. helper `has_active_billing`** ב-`subscription_service` — נגזר: `trial_ends_at IS NOT NULL AND status IN ('trial', 'active', 'past_due')`. **לא נאכף ב-2.6** — Phase 3 ישתמש בו כדי לחסום הקמת קמפיין בלי billing פעיל. מקום אחד, dependable.

**Done:** משתמש שבחר חבילה ב-2.5 → לוחץ "אשר אמצעי תשלום" → iframe של פלאקארד נפתח → מזין כרטיס → ה-webhook מקבל אישור → המנוי עובר ל-`status='trial'` עם `trial_ends_at = +7d` ו-`billing_token` שמור. ביום 8 ה-cron מחייב ועובר ל-`active`. ביטול בכל שלב מעביר ל-`canceled` ושומר גישה עד סוף ה-period.

**Done (תוספת — Billing Profiles):**
- משתמש שנכנס לטופס תשלום בלי פרופיל — מקבל שדות למילוי. שולח → נשמר ב-`billing_profiles`. רק אז ה-iframe נפתח.
- ניסיון לקרוא ל-`/start-billing` בלי `full_name` בפרופיל → `400` עם שגיאה ברורה.
- RLS: ניסיון לקרוא פרופיל של משתמש אחר → 0 שורות.

**לא לעשות:**
- חשבוניות אוטומטיות (אינטגרציית Green Invoice — Session 2.6.2 נפרד).
- אכיפת `has_active_billing` כשער להקמת קמפיין (Phase 3).
- מסך ניהול אמצעי תשלום בדשבורד (Phase 4.6 / מאוחר).
- עדכון/החלפת כרטיס. ב-MVP: ביטול + רישום מחדש.

**זהירות (CLAUDE.md):**
- **external-sdk patterns** — פלאקארד דוחה J2 שגוי כ-J4, אגורות vs ש"ח (פי 100!), קודי שגיאה לפי acquirer, sandbox vs production hosts.
- **webhook patterns + idempotency** — `webhook_events` עם `PelecardTransactionId`, **לא** `ParamX`. אימות חוזר חובה (`PaymentGW/GetTransaction`).
- **state-machine patterns** — `trial → active → past_due → expired` או `canceled`. מעברים לא חוקיים אסורים (למשל `expired → active` בלי לעבור דרך billing חדש).
- **secrets-handling** — `PELECARD_PASSWORD` לעולם לא ב-frontend, לעולם לא בלוגים. הטוקן ב-Vault, לא ב-`billing_token` plaintext.

#### תת-החלטות שננעלו במהלך התכנון

1. **פלאקארד לא מציעה recurring native** — אנחנו יוזמים כל חיוב מהצד שלנו על הטוקן השמור (J4 על `IsToken=true`). זה לא חיסרון — זה דפוס נפוץ אצל PSPs ישראליים, ונותן לנו שליטה מלאה על דחיות חוזרות.

2. **trial 7 ימים — מי סופר?** אנחנו, לא פלאקארד. `trial_ends_at` ב-DB + cron יומי שמחייב כשעובר. פלאקארד רק שומרת את הטוקן וסולקת חיובים שאנחנו יוזמים.

3. **`ActionType` ב-trial:** `J2` (אימות בלבד, לא חיוב). ביום 8 קוראים ל-`J4` על אותו טוקן. הסיבה: לא רוצים לחייב ₪1 וזיכוי — חוויית משתמש גרועה ועלות סליקה אמיתית.

4. **`Currency=1`** קבוע — ש"ח בלבד ב-MVP. הקוד מקבל את זה כפרמטר אבל מוגדר default ל-1; אם יבוא יום וירצו $, שינוי מקומי ב-config.

5. **טוקן ב-Vault, לא בעמודה רגילה** — אותו דפוס בדיוק כמו טוקן Meta ב-1.2 (encrypted_token). עקבי, חוק אבטחה אחיד.

6. **`webhook_events` לכל IPN** — הטבלה כבר קיימת (spec §6). מוסיפים `event_type='pelecard_ipn'` ועובדים עם אותו דפוס idempotency מ-Phase 4.1 (Meta leads).

7. **`pending_billing_sessions` — טבלה זמנית חדשה?** כן. `user_id`, `confirmation_key`, `pelecard_session_id`, `created_at`, `expires_at` (= `created_at + 30min`). cron יומי מנקה. RLS: SELECT למשתמש על שלו בלבד. INSERT/DELETE רק דרך service.

8. **בלי "החלפת כרטיס" ב-MVP** — אם הלקוח רוצה כרטיס אחר: cancel + re-subscribe. מסך החלפת כרטיס דורש token deletion + re-creation והרבה edge cases (חיוב באמצע מחזור) — נדחה.

#### תלויות לפני 2.6.1

- ✅ Session 0.2 חייב להיות done (אחרת אי אפשר לקבל IPN מבחוץ).
- ✅ Session 2.5 חייב להיות done (`tier != 'pending'` הוא precondition).
- 🔲 פרטי גישה של פלאקארד (sandbox + production) — באחריות גיא.
- 🔲 שיחה עם תמיכת פלאקארד על Gateway21 vs Match API — באחריות אמיר.

#### שאלה פתוחה לגיא

1. כשחיוב חודשי נכשל (כרטיס פג, אין מסגרת), כמה פעמים לנסות לפני שהמנוי עובר ל-expired?

### Session 2.6.2 — הפקת חשבוניות מס אוטומטית (Green Invoice) ✅

**עדכון — הרחבת התכנון (2.6.2):**

**ספק החשבוניות: Green Invoice (Morning).** ה-PSP (פלאקארד) מטפל בכסף בלבד — חשבונית מס/קבלה (סוג 320) דורשת ספק נפרד. Green Invoice מנפיק חשבונית חתומה דיגיטלית, שולח ללקוח, ושומר עותק. אינטגרציה דרך REST API + webhooks.

**דרישת מסלול:** Green Invoice **Extra ומעלה** — נדרש גם ל-API וגם ל-webhooks. **תשתית, לפני קוד.** באחריות גיא.

**סוג המסמך הקבוע:** 320 (חשבונית מס/קבלה). תשלום מיידי דרך כרטיס אשראי (סוג תשלום 3), במטבע ILS. מתאים גם ל-`basic`/`premium`/`whatsapp`.

**שע"מ (Tax Authority allocation number) — לא ב-MVP.** הסף החוקי כיום ₪5,000 נטו לחשבונית B2B. כל המסלולים שלנו (₪397–₪797) מתחת לסף. **אבל** — חבילות שנתיות עתידיות עשויות לעבור את הסף; ההרחבה לחיבור גיוון אישית עם תיעוד דרך `israeli-e-invoice` תיכנס בפאזה נפרדת, לא ב-MVP.

**קריאה מקדימה חובה לפני קוד:** הסקיל של `green-invoice` (ב-`docs/integrations/green-invoice/SKILL.md` + references), בדומה לדפוס של פלאקארד. כלל חובה מוגדר ב-CLAUDE.md.

#### תלויות לפני 2.6.2

- ✅ Session 2.6.1 חייב להיות done (חיוב חודשי מוצלח הוא ה-trigger).
- 🔲 מסלול Green Invoice Extra — באחריות גיא.
- 🔲 פרטי גישה ל-API (`GREEN_INVOICE_KEY_ID`, `GREEN_INVOICE_KEY_SECRET`) — sandbox + production.
- 🔲 הגדרת סוג העסק (`עוסק מורשה` / `חברה בע"מ`) במורנינג — קובע התנהגות מע"מ.

#### שלבי הביצוע

- [x] **1. תיעוד Green Invoice בריפו** — ארגון `docs/integrations/green-invoice/` (SKILL.md + references/ + scripts/green-invoice-client.py). הכלל ב-CLAUDE.md כבר קיים.

- [x] **2. `integrations/green_invoice.py`** — עטיפה מבודדת. ה-API החיצוני שמודולים אחרים רואים: `create_tax_invoice_receipt(user_id, amount_ils, payment_date, billing_period_label)`, `get_document_pdf_url(document_id)`, `search_documents(user_id)`. JWT token caching פנימי (התוקף ~24 שעות, לחדש בכשל 401). env vars: `GREEN_INVOICE_KEY_ID`, `GREEN_INVOICE_KEY_SECRET`, `GREEN_INVOICE_ENV` (`sandbox`/`production`).

- [x] **3. טבלה חדשה `billing_invoices`** (migration ייעודי):
  - `id` (uuid, PK)
  - `user_id` (FK, RLS)
  - `subscription_id` (FK ל-`subscriptions`)
  - `provider` (text default `'green_invoice'`)
  - `provider_document_id` (text) — המזהה של המסמך אצל Green Invoice
  - `document_number` (int) — המספר העוקב של החשבונית
  - `amount_ils` (numeric)
  - `billing_period_start` (timestamptz)
  - `billing_period_end` (timestamptz)
  - `pdf_url_he` (text, nullable) — מהורדה דרך webhook או polling
  - `created_at`, `updated_at`

  RLS: SELECT למשתמש על שלו (`auth.uid() = user_id`). INSERT/UPDATE רק דרך service (admin client, spec §7.3(ב)).

- [x] **4. שירות הפקה: `billing_service.issue_invoice_for_charge`** — נקרא מתוך handlers של חיוב מוצלח (יום 8 + חיוב חודשי, מ-2.6.1). מקבל: `user_id`, `amount_ils`, `period_start`, `period_end`. זרימה:
  - שליפת פרטי הלקוח מ-`billing_profiles` (נוצר ב-2.6.1 שלב 6.5):
    ```
    profile = billing_profiles.get(user_id)
    green_invoice.create_tax_invoice_receipt(
        client_name = profile.business_name or profile.full_name,
        client_tax_id = profile.tax_id,  # nullable, Green Invoice מקבל בלעדיו
        client_email = users.email,
        ...
    )
    ```
    אם `business_name` קיים — הוא קודם ל-`full_name` (החשבונית על שם החברה). אם `tax_id` חסר — החשבונית מונפקת בלעדיו (הלקוח קיבל הסבר מראש בטופס הפרופיל). שינוי בפרופיל אחרי הפקה לא משפיע על חשבוניות ישנות (דרישה חוקית).
  - קריאה ל-`green_invoice.create_tax_invoice_receipt` עם סוג 320, סוג תשלום 3 (כרטיס אשראי).
  - שמירת `provider_document_id`, `document_number`, `amount_ils` ב-`billing_invoices`.
  - ה-PDF יגיע אחר כך דרך webhook (ראה שלב 5).

- [x] **5. webhook לקבלת PDF + עדכון סטטוס** — `POST /webhooks/billing/green-invoice` במודול המבודד `routers/webhooks.py` (spec §7.3(א)). אימות:
  1. בקבלת payload — קריאה חוזרת ל-`GET /v1/documents/{id}` עם הטוקן שלנו (Green Invoice לא חותמת webhooks ב-HMAC לפי הסקיל; אימות חוזר server-to-server הוא היחיד אמין).
  2. עדכון `pdf_url_he` ב-`billing_invoices` לפי `provider_document_id`.
  3. idempotency דרך `webhook_events` עם `key=provider_document_id` + `event_type='green_invoice_created'`.

- [x] **6. endpoint לרשימת חשבוניות:** `GET /me/invoices` — מחזיר רשימת חשבוניות של הלקוח (חודש, סכום, מספר, קישור PDF). RLS אוכף ראייה רק של שלו. משמש את מסך "החבילה שלי" → "חשבוניות" בפרוטוטייפ (5.4).

- [x] **7. endpoint להורדה ישירה:** `GET /me/invoices/{id}/download` — מחזיר redirect ל-`pdf_url_he` של Green Invoice. RLS אוכף בעלות לפני redirect.

- [x] **8. טיפול בכשלים בהפקה** — אם `green_invoice.create_tax_invoice_receipt` נכשל (401/429/5xx), זה **לא יכול** למנוע את החיוב — הכסף כבר חויב. במקרה כשל:
  - log + Sentry alert.
  - **לא retry אוטומטי כאן** — אם נריץ retry בלולאה, אנחנו עלולים ליצור חשבוניות כפולות. במקום זאת: שדה `last_invoice_error` ב-`subscriptions`, ו-cron יומי שמזהה חיובים מוצלחים בלי חשבונית ויוצר retry מבוקר (אחד בכל פעם).
  - הלקוח מקבל מייל ידני מאיתנו עד שהאוטומציה מתקנת. לא חוסם אותו.

#### הערות חשובות

**1. פרטי הלקוח להנפקת החשבונית — נשלפים מ-`billing_profiles`.**
הטבלה נוצרת ב-Session 2.6.1 (שלב 6.5) ומאוכלסת בזמן הזנת אמצעי התשלום. ב-2.6.2 פשוט שולפים. אם `billing_profiles` לא קיים למשתמש — לא אמור לקרות (חיוב מוצלח מחייב profile קיים); log + Sentry + fail.

**2. סוג העסק במורנינג קובע התנהגות מע"מ.** מומלץ "עוסק מורשה" או "חברה בע"מ" — מע"מ 18% מתווסף אוטומטית לסכום. הגדרה חד-פעמית במורנינג, לא בקוד.

**3. תזמון חשבונית = תזמון חיוב.** ב-MVP: חיוב מצליח → חשבונית מונפקת מיד באותה עסקה (אטומית ככל האפשר — אם החשבונית נכשלת ההפקה היא בעיה צדדית, החיוב כבר מוצלח).

**4. סקיל israeli-e-invoice נדחה ל-Post-MVP.** הסכומים שלנו מתחת לסף השע"מ. אם בעתיד תהיה חבילה שנתית מעל ₪5,000 — נחבר אז.

**Done:** חיוב חודשי מוצלח של לקוח (יום 8 / חודשי) → אוטומטית מונפקת חשבונית מס/קבלה ב-Green Invoice → נשמרת ב-`billing_invoices` → webhook מעדכן את `pdf_url_he` → הלקוח רואה אותה ב-"החבילה שלי" → "חשבוניות" → לוחץ הורדה ומקבל PDF חתום.

**לא לעשות:**
- שע"מ allocation numbers (נדחה).
- חשבונית ידנית מהדשבורד ("צור חשבונית עכשיו"). הפקה אוטומטית בלבד.
- חשבונית זיכוי (סוג 330) על ביטולים. ביטול ב-`canceled` שומר גישה עד סוף period — אין מה לזכות.
- ערוץ מייל ייעודי לחשבוניות (Green Invoice שולח ללקוח בעצמו עם `attachment: true`).

**זהירות (CLAUDE.md):**
- **external-sdk patterns** — JWT token של Green Invoice פג מדי פעם, צריך retry על 401.
- **webhook patterns + idempotency** — `webhook_events` עם `provider_document_id`, אימות חוזר server-to-server חובה.
- **secrets-handling** — `GREEN_INVOICE_KEY_SECRET` env בלבד, אף פעם ב-frontend.
- **financial state** — חיוב מוצלח **בלי** חשבונית = מצב לא תקין שדורש התראה (Sentry + cron retry).

#### תת-החלטות שננעלו

1. **טבלה נפרדת `billing_invoices`, לא עמודה ב-`subscriptions`.** סיבה: לקוח אחד מקבל הרבה חשבוניות לאורך זמן (12 בשנה). עמודה אחת לא תספיק.

2. **PDF לא נשמר אצלנו, רק URL.** Green Invoice מארח את הקובץ. אם בעתיד נרצה backup — נוסיף R2 sync.

3. **שם החבילה בחשבונית = "מנוי Campaign AI — [Basic/Premium/WhatsApp] — [תאריך תקופה]".** ניסוח קבוע, ניתן לשינוי בהמשך.

4. **בלי חתימה דיגיטלית של אימייל ל-לקוח דרכנו.** Green Invoice שולח עם `attachment: true`, אנחנו לא נוגעים.

5. **שגיאת הפקת חשבונית לא מבטלת חיוב.** הכסף נשאר בידי הלקוח כשירות שולם, החשבונית מתעדכנת ידנית/cron retry.

6. **`billing_profiles` נפרד מ-`subscriptions`.** סיבה: אם לקוח יבטל ויחזור, ה-`subscription` יהיה חדש אבל הפרופיל יישאר. גם — אם בעתיד נוסיף עוד שדות פרופיל (לוגו לחשבונית, כתובת...), הם לא בעלי קשר ישיר למנוי.

7. **`tax_id` אופציונלי, עם הסבר ברור ב-UI.** הלקוח שעיקש לא להזין → לא חוסמים. הוא יודע מה הוא מפסיד.

---

## Phase 3 · שאלון + יצירת מודעות (flow שלבים 8,13) — הליבה

### Session 3.0 — תשתית jobs (queue + worker) ✅

**עדכון — הרחבת התכנון (3.0):**

תשתית גנרית לעיבוד רקע. הבסיס לכל ה-jobs בפרויקט — Phase 3 (יצירת קמפיין), Phase 5 (בוט WhatsApp), Phase 6 (תחזוקה: refresh tokens, חיובים יומיים), Phase 8 (אופטימיזציה אוטונומית). **עדיין בלי Redis/Celery** — Postgres queue + worker polling מספיקים לנפח שלנו, ועקביים עם spec §2 ("העומס משלב נקודתי ומתמשך... התשתית הקיימת מספיקה").

**ה-jobs הצפויים בכל הפרויקט (לעיון, לא נבנים ב-3.0):**

| Phase | Job type | מתי | מי מחכה? |
|---|---|---|---|
| 3 | `generate_ad_copy` | אחרי שאלון | המשתמש (polling על UI) |
| 3 | `generate_ad_image` | אחרי קופי, 3 פעמים מקבילים | המשתמש |
| 3 | `push_campaign_to_meta` | אחרי אישור 3 הוריאציות | המשתמש |
| 3.5/3.6 | `edit_existing_image`, `regenerate_image` | פעולות UI | המשתמש |
| 5 | `process_bot_message` | webhook WhatsApp נכנס | webhook (5 שנ' timeout) |
| 6 | `refresh_meta_token` | cron יומי | אף אחד |
| 6 | `daily_billing_charge` | cron יומי | אף אחד |
| 8 | `monitor_campaign` | cron פר-קמפיין פעיל | אף אחד |
| 8 | `optimization_propose`, `optimization_execute` | אחרי זיהוי בעיה | אף אחד (אוטונומי) |

ב-3.0 בונים את התשתית הריקה — handlers אמיתיים יתווספו ב-Sessions הבאים.

#### שלבי הביצוע

- [x] **1. טבלת `jobs`** (migration ייעודי):
  - `id` (uuid, PK)
  - `type` (text + CHECK constraint, enum מתחיל ריק וגדל פר-session)
  - `payload` (jsonb) — פרמטרים ספציפיים ל-job
  - `status` (text + CHECK, ערכים: `pending`/`running`/`done`/`failed`)
  - `attempts` (int, default 0)
  - `max_attempts` (int, default 3)
  - `next_retry_at` (timestamptz, nullable)
  - `last_error` (text, nullable)
  - `user_id` (uuid, nullable, FK ל-`auth.users`) — NULL ל-jobs מערכת (cron-driven)
  - `priority` (int, default 0) — עמודה קיימת בסכמה, **לא בשימוש ב-MVP** (לעתיד)
  - `created_at`, `started_at` (nullable), `completed_at` (nullable)

  RLS: SELECT לבעלים (`auth.uid() = user_id`). jobs מערכת (`user_id IS NULL`) אינם נגישים מהמשתמש. INSERT/UPDATE רק דרך admin client (services יוצרים jobs, לא המשתמש ישירות).

  **חשוב:** `status` ו-`type` הם `TEXT` עם `CHECK constraint`, לא `VARCHAR(N)` — אותו דפוס כמו `subscriptions.status` ב-2.6 (תפיסת ערכים שגויים ברמת ה-DB, גמישות להרחבה).

- [x] **2. `worker/runner.py`** — לולאה אינסופית:
  1. תפיסת job אטומית: `UPDATE jobs SET status='running', started_at=now() WHERE id IN (SELECT id FROM jobs WHERE status='pending' AND (next_retry_at IS NULL OR next_retry_at <= now()) ORDER BY created_at ASC LIMIT 1 FOR UPDATE SKIP LOCKED) RETURNING *`
  2. הרצת handler לפי `type` (מ-`worker/handlers.py`)
  3. הצלחה: `status='done'`, `completed_at=now()`
  4. כשל זמני (`raise Exception`): `attempts++`, `last_error=...`, `next_retry_at=now()+backoff(attempts)`, `status='pending'`
  5. כשל סופי (`attempts >= max_attempts`): `status='failed'`, `last_error=...`
  6. שינה 2 שניות → חזרה לתחילת הלולאה
  7. עיבוד job אחד בכל פעם (sequential, לא concurrent). אם בעתיד צריך throughput — מוסיפים worker process נוסף ב-Render, לא concurrency פנימי.

- [x] **3. retry policy:** backoff בסיסי, 3 ניסיונות מקסימום:
  - ניסיון 1 → כשל → `next_retry_at = now() + 1 minute`
  - ניסיון 2 → כשל → `next_retry_at = now() + 5 minutes`
  - ניסיון 3 → כשל → `status='failed'`, סוף.

  זמני ה-backoff מוגדרים כקבועים ב-`worker/runner.py`, ניתנים להחלפה פר-job-type בעתיד.

- [x] **4. `worker/handlers.py`** — dict שממפה `type → callable`. ב-3.0 הוא **ריק חוץ מ-handler אחד לבדיקה**: `'test_echo'` שמקבל payload, רושם ל-log, מסיים. אין handlers ייצוריים.

- [x] **5. `render.yaml`** — תוספת של שירות worker שני מאותו repo (background worker).

- [x] **6. endpoint לקריאת status:** `GET /me/jobs/{id}` — מחזיר `status`, `attempts`, `last_error`, `created_at`. RLS אוכף שהמשתמש רואה רק jobs שלו. משמש את ה-frontend ל-polling במהלך "הסוכן בונה..." (Phase 3+).

- [x] **7. Sentry integration (מינימלי):** ב-`worker/runner.py` עוטפים את ההרצה של ה-handler ב-`try/except`. בכשל סופי (ניסיון 3) — `sentry_sdk.capture_exception(e)` עם context: `job_id`, `type`, `user_id`, `attempts`. כשלים זמניים (retry) — לא נשלחים ל-Sentry (יציפו).

- [x] **8. תיעוד `CLAUDE.md` — חוק idempotency:** כל handler חייב להיות idempotent. לפני כל קריאה ל-API חיצוני / שינוי state ב-DB, ה-handler בודק "האם הפעולה כבר בוצעה?" אם כן — דלג. הסיבה: ה-worker עלול לקרוס באמצע handler, אחרי שהפעולה החיצונית הצליחה אבל לפני שהסטטוס נכתב ב-DB. retry יריץ את ה-handler שוב — בלי idempotency זה ייצור כפילות (קמפיין כפול, חיוב כפול, חשבונית כפולה). **אסור להוסיף handler חדש בלי שכבת idempotency.** Code review חוסם.

#### תת-החלטות שננעלו

1. **polling interval = 2 שניות.** איזון בין latency לעומס DB (~43k queries ביום, זניח). אם בעתיד נמדוד עומס — אפשר לעלות ל-5 שניות.

2. **`FOR UPDATE SKIP LOCKED` לתפיסת jobs.** מבטיח שאם נריץ 2 workers במקביל (בעתיד), כל אחד יתפוס job אחר. ב-MVP יש worker אחד, אבל הכלל נכתב מהיום הראשון.

3. **`status='running'` ללא timeout cleanup ב-3.0.** אם worker קורס באמצע, job נשאר ב-`running` לנצח. **לא בעיה דחופה ב-MVP** (worker יחיד, נדיר). cron cleanup ייתווסף בפאזה מאוחרת יותר. העמודה `started_at` קיימת כדי לאפשר את זה כשנצטרך.

4. **worker יחיד, sequential.** פשטות. אם דרוש throughput — מוסיפים worker process נוסף ב-Render, לא concurrency פנימי.

5. **קריאת status מ-frontend = HTTP polling (לא WebSocket / SSE).** פשטות. עיכוב של 2 שניות — בלתי מורגש.

6. **Sentry על כשל סופי בלבד, לא על retries.** Retries זמניים = רעש. רק כשל סופי (`status='failed'`) דורש תשומת לב.

7. **idempotency = חוק חוצה-handlers, לא feature של 3.0.** ה-handler עצמו אחראי. אסור להעמיס idempotency לתוך ה-runner — זו הפרת שכבות.

#### תלויות לפני 3.0

- ✅ Session 0.2 חייב להיות done (חיוני כדי לפרוס worker ב-Render).

**Done:** `INSERT INTO jobs (type, payload, status, user_id) VALUES ('test_echo', '{"message":"hello"}', 'pending', '<uuid>');` — ה-worker תופס תוך 2 שניות, רושם ל-log, מסיים `done`. כשל מכוון → 3 ניסיונות עם backoff → `failed` + Sentry capture. `GET /me/jobs/{id}` עם הטוקן של בעל ה-job → 200. עם טוקן של אחר → 0 שורות (RLS). שני שירותים רצים ב-Render: web + worker. `CLAUDE.md` מכיל את חוק ה-idempotency.

**לא ב-3.0 (יתווסף בפאזות הבאות):**
- handlers ייצוריים — `generate_ad_copy` (3.2), `generate_ad_image` (3.3), `push_campaign_to_meta` (3.4), `process_bot_message` (5.x), `refresh_meta_token` (6), `daily_billing_charge` (6, מסתמך על 2.6.1), `monitor_campaign` (8).
- cron מתוזמן ל-jobs מערכת (Phase 6 ואילך).
- timeout cleanup ל-jobs תקועים ב-`running`.
- priority queues בפועל (העמודה קיימת, לא בשימוש).
- מנגנון notifications מ-server ל-frontend (אם בעתיד נחליף את ה-polling).

### Session 3.1 — שמירת שאלון ✅
- [x] endpoint ששומר את תשובות השאלון כ-JSONB ב-`quiz_responses` (כולל טון מותג)

**Done:** תשובות נשמרות, קשורות לקמפיין, RLS עובד.

**מומש (3.1):** מיגרציה `0016_quiz_responses.sql` — טבלה עם `user_id` + composite FK
`(campaign_id,user_id)→campaigns(id,user_id)` (spec §7, + `unique(id,user_id)` על campaigns)
ו-RLS `auth.uid()=user_id`. `brand_tone` = עמודת TEXT+CHECK עם **5 ערכים** אנגליים
(`professional/friendly/luxury/direct/authoritative`, תואם קבצי הטון של 3.1.5); שאר התשובות
ב-`answers jsonb` גמיש. `POST /campaigns/{id}/quiz` (upsert, draft gate) + `GET` — דרך
`quiz_service` (user-client/RLS). ולידציה ב-Pydantic (`QuizResponseInput`) עם הודעות עברית.

### Session 3.1ב — הגדרת שדות טופס ליד (flow שלב 10, מותנה) ✅

**מומש (3.1ב):** מיגרציה `0017_lead_form_fields.sql` — הטבלה **נוצרה** (לא רק אומתה; לא
הייתה ב-DB, רק ב-spec §6) עם `configuration jsonb`, `user_id` + composite FK
`(campaign_id,user_id)→campaigns(id,user_id)` (spec §7), RLS `auth.uid()=user_id`,
`unique(campaign_id)`. `POST/GET /campaigns/{id}/lead-form` דרך `lead_form_service`
(user-client/RLS). ולידציה ב-service עם הודעות עברית (שם/טלפון נעולים, email אופציונלי,
≤4 שאלות, שאלה 5–200 תווים, 2–3 תשובות 1–50). GET מחזיר default אם אין; type≠lead→400,
status≠draft→409, לא-בעלים→404.

**עדכון — הרחבת התכנון (3.1ב):**

הגדרת המבנה של **Meta Lead Form** — איזה שדות הליד יראה כשילחץ על "להשאיר פרטים" במודעה בפייסבוק. מתאים פרוטוטייפ סעיף 7.2 שלב 5.

**חשוב להבחין משלושה דברים אחרים שדומים בשם:**
- ❌ זה **לא** השאלון של הקמפיין (3.1 — קהל/הצעה/בידול/טון/תקציב/מיקום/גיל+מגדר). זה דאטה שמזין את ה-LLM.
- ❌ זה **לא** שאלון "עזור לי להחליט" באונבורדינג (סעיף 3 בפרוטוטייפ).
- ✅ זה **המבנה של הטופס שהליד הסופי ימלא בפייסבוק**. נדחף ל-Meta ב-Session 3.4.

**מותנה: רק כש-`campaigns.type='lead'`.** קמפיין `whatsapp` מדלג על השלב — הליד פותח שיחת WhatsApp ישירה, אין טופס.

### שלבי הביצוע

- [x] **1. אימות מבנה הטבלה `lead_form_fields`** (כבר קיימת בסכמה לפי spec §6). הקונפיגורציה כולה ב-JSONB אחד:
  ```
  configuration jsonb NOT NULL DEFAULT '{}'::jsonb
  ```
  מבנה ה-JSON:
  ```json
  {
    "fields": {
      "name":  {"enabled": true,  "required": true},
      "phone": {"enabled": true,  "required": true},
      "email": {"enabled": false, "required": false}
    },
    "screening_questions": [
      {
        "question": "האם אתה בעל עסק עצמאי?",
        "answers": ["כן", "לא"]
      },
      {
        "question": "מה התקציב?",
        "answers": ["עד ₪5,000", "₪5,000-₪15,000", "מעל ₪15,000"]
      }
    ]
  }
  ```
  RLS: SELECT/INSERT/UPDATE/DELETE לבעלים דרך JOIN על `campaigns.user_id`. אין `user_id` ישיר בטבלה — מקור הבעלות הוא הקמפיין.

- [x] **2. endpoint יחיד:** `POST /campaigns/{id}/lead-form` — שומר את כל הקונפיגורציה במכה אחת. אם כבר קיים — דורס (upsert סמנטית). זרימה:
  1. שליפת הקמפיין מ-DB.
  2. בדיקה: `type='lead'`? אם לא → `400` עם הודעה "טופס ליד זמין רק בקמפיין מסוג לידים."
  3. בדיקה: `status='draft'`? אם לא → `409 Conflict` עם הודעה "הקמפיין כבר פעיל. שינוי טופס לא אפשרי."
  4. ולידציה של ה-payload (ראו שלב 3).
  5. UPSERT ל-`lead_form_fields` עם `configuration` מעודכן.
  6. החזרה: `200` + הקונפיגורציה השמורה.

- [x] **3. ולידציה ב-service (לא ב-Pydantic):**
  - **שם פרטי + טלפון:** תמיד `enabled: true, required: true`. ניסיון לשלוח `false` → `400` עם הודעה ברורה.
  - **email:** אופציונלי. `enabled` יכול להיות `true` או `false`. כש-`enabled: true` → `required` יכול להיות `true` או `false` (לבחירת המשתמש).
  - **שאלות סינון:** עד **4 שאלות** במערך. יותר → `400`.
  - **כל שאלה:**
    - `question`: טקסט, מינ' 5 תווים, מקס' 200.
    - `answers`: מערך של 2-3 מחרוזות. 2 חובה, 3 אופציונלי. כל תשובה: מינ' 1 תו, מקס' 50.
  - הודעות שגיאה **בעברית** (לכן service ולא Pydantic — Pydantic מחזיר באנגלית כברירת מחדל).

- [x] **4. endpoint לקריאה:** `GET /campaigns/{id}/lead-form` — מחזיר את הקונפיגורציה השמורה, או `200` עם default אם עוד לא נשמר:
  ```json
  {
    "fields": {
      "name":  {"enabled": true,  "required": true},
      "phone": {"enabled": true,  "required": true},
      "email": {"enabled": false, "required": false}
    },
    "screening_questions": []
  }
  ```
  RLS אוכף שהמשתמש קורא רק ללקוחות שלו. ניסיון לקרוא ל-`/campaigns/{id}/lead-form` של אחר → 404.

### תת-החלטות שננעלו

1. **multiple choice בלבד, לא טקסט חופשי.** Meta Lead Forms תומך גם בטקסט, אבל הפרוטוטייפ מציג רק multiple choice. אם בעתיד גיא ירצה — הוספה לא-שוברת ל-JSONB.

2. **שם+טלפון נעולים כחובה.** אי אפשר לכבות. סיבה: בלי שם+טלפון אין מה לעשות עם הליד.

3. **email = toggle של `enabled`, לא של `required`.** ה-UI מציג רק "כבוי/פעיל" — כש-`enabled: true`, `required` יורש `true` (זה ברירת המחדל הסבירה). אם בעתיד נצטרך הבחנה דקה — אפשר.

4. **קונפיגורציה כולה ב-JSONB אחד.** לא 3 טבלאות (`fields`, `questions`, `answers`). סיבה: אין צורך בשאילתות פר-תשובה. הקונפיגורציה נקראת ונכתבת כיחידה.

5. **endpoint יחיד, לא פר-שאלה.** עקבי עם 3.1 (שמירת השאלון). פשוט יותר.

6. **אסור עריכה אחרי `status > draft`.** נעילה כשהקמפיין כבר נדחף ל-Meta — שינוי הטופס באמצע יוצר הבדל בין מה שב-DB למה שב-Meta. אם הלקוח רוצה לשנות — נדרשת יצירת קמפיין חדש (תואם להחלטה הכללית של 3.1, שעדיין מחכה לאישור גיא).

7. **frontend אחראי לאזהרת CPL.** הפרוטוטייפ מציג אזהרה כשמוסיפים 2+ שאלות סינון. ה-backend לא יודע על זה — הוא רק שומר את מה ששלחו לו.

### תלויות לפני 3.1ב

- ✅ Session 2.2 חייב להיות done (קמפיין draft נוצר עם `type='lead'`).
- ✅ Session 0.5.2 (subscriptions) — נדרש ל-`has_active_billing` gate אם נחליט לאכוף ב-Phase 3 (כפי שתוכנן). **לעת עתה — לא נאכף ב-3.1ב.** הוא ייאכף ב-Session 3.4 לפני דחיפה ל-Meta.
- 🔲 אין תלות בגיא — סטטוס הקמפיין `draft` כפי שכבר סוכם, שאר הוולידציה בסיסית.

### Done של 3.1ב

- `POST /campaigns/{id}/lead-form` עם קונפיגורציה תקינה → `200`, הקונפיגורציה נשמרת ב-`lead_form_fields.configuration`.
- ניסיון על קמפיין `type='whatsapp'` → `400` בעברית.
- ניסיון על קמפיין `status='live'` → `409` בעברית.
- ניסיון לכבות שם/טלפון → `400`.
- ניסיון לשמור 5 שאלות → `400`.
- שאלה עם תשובה אחת בלבד → `400`.
- שאלה עם 4 תשובות → `400`.
- `GET` מחזיר default אם עוד לא נשמר.
- RLS עובד — משתמש לא רואה טפסים של אחרים.

### לא ב-3.1ב

- דחיפת הטופס ל-Meta — קורה ב-3.4 (`push_campaign_to_meta` handler יקרא ל-`/campaigns/{id}/lead-form` כחלק מבניית הקמפיין).
- שאלות טקסט חופשי. רק multiple choice.
- שאלות מותנות ("אם ענית X, שאלה הבאה היא Y"). Meta תומך, אנחנו לא משתמשים ב-MVP.
- שאלות מובנות של Meta (city, state — Meta יציע אוטומטית). רק שאלות מותאמות.
- אזהרת CPL — frontend בלבד.

### זהירות

- **Meta יכולה לדחות טופס** אם השאלות חורגות ממדיניות הפרטיות שלה (למשל בקשת מידע רגיש). זה מתגלה רק ב-3.4 בעת הדחיפה. ב-3.1ב לא בודקים את זה.
- **תרגום שמות שדות:** ב-MVP — עברית בלבד (`שם פרטי`, `טלפון`, `דוא"ל`). Meta תציג ללידים עברית.
- **`fields.name` הוא שדה אחד.** לא שם פרטי + שם משפחה. אם בעתיד יידרשו — תוספת ל-JSONB.

### Session 3.1.5 — תשתית פרומפטים + ממשק בדיקה ✅

> [!IMPORTANT]
>  **חובה לעיין במסמך `docs/session-3.1.5-instructions.md` קודם המימוש**

**מומש (3.1.5) — לפי ה-doc, שגובר על התכנון המקורי שלהלן:**
- קובץ `copy_generation.txt` **יחיד** (3 וריאציות ב-JSON: emotional/pain_solution/result_success)
  במקום copy_angle_1/2/3 + campaign_strategy (גיא קבע בקריאה אחת).
- **5 קבצי טון** (professional/friendly/luxury/direct/authoritative), **תוכן אמיתי** — לא placeholders.
- `prompts_service.build()` (כלל 8) + `ad_generation_service` (generate_copy_variants /
  generate_image_for_variant; gpt-4o-mini + gpt-image-2, עמיד url/b64) + `routers/admin/prompt_tester`
  (GET form + POST מוגן `X-Admin-Token`) + template `app/templates/admin/prompt_tester.html`.
- credentials (`OPENAI_API_KEY`/`ADMIN_TOKEN`) **optional + degradation** (כלל 10), לא fail-on-startup.
- ה-GET של דף ה-admin אינו מוגן ב-token (דפדפן לא שולח header בניווט); ה-token מגן על POST /generate.

**עדכון — הרחבת התכנון (3.1.5):**

תשתית גנרית לניהול פרומפטים בכל הפרויקט (Phase 3/5/7/8), + ממשק admin פנימי לבדיקה ידנית של pipeline שלם של יצירת מודעה.

**Session זה בונה תשתית, לא תוכן.** הזוויות עצמן (`copy_angle_1/2/3`) יישארו placeholders עד שגיא יחזור עם הזוויות השיווקיות. אבל ה-loader, ה-API, ה-admin UI — הכל בנוי ומוכן.

**בנוסף — `services/ad_generation_service.py` נכתב ב-Session זה**, לא ב-3.2. הסיבה: ה-admin UI חייב לקרוא לאותו pipeline שירוץ בייצור. אם נשמור את הקוד ב-handlers של 3.2 — ה-admin UI יעתיק אותו, ויהיו שני מקורות אמת. **קוד הליבה ב-`ad_generation_service`, handlers ב-3.2/3.3 ידחפו אותו ל-jobs queue.**

### מבנה התיקיות

```
app/
├── prompts/
│   ├── README.md                         ← הסבר למפתחים עתידיים
│   ├── phase3/
│   │   ├── copy_angle_1.txt              ← placeholder
│   │   ├── copy_angle_2.txt              ← placeholder
│   │   ├── copy_angle_3.txt              ← placeholder
│   │   ├── image_generation.txt          ← placeholder
│   │   ├── campaign_strategy.txt         ← placeholder (פרומפט-על)
│   │   └── tones/
│   │       ├── luxury.txt                ← placeholder
│   │       ├── friendly_young.txt        ← placeholder
│   │       ├── professional_direct.txt   ← placeholder
│   │       └── bold_dynamic.txt          ← placeholder
│   └── (phase5/, phase7/, phase8/ — יתווספו בעתיד)
├── services/
│   ├── prompts_service.py                ← loader + הרכבה
│   └── ad_generation_service.py          ← pipeline ליצירת קופי+תמונות
└── routers/
    └── admin/
        ├── __init__.py
        └── prompt_tester.py              ← דף ה-admin
```

### שלבי הביצוע

- [ ] **1. תיקיית `app/prompts/`** — יצירת המבנה לעיל. **כל קובץ `.txt` הוא placeholder עם תוכן פשוט:**
  ```
  TODO: גיא יחזור עם תוכן הזווית.

  Placeholders שיש להחליף:
  - {service_name}: שם השירות
  - {audience}: תיאור הקהל
  - {offer}: ההצעה השיווקית
  - {differentiation}: הבידול
  - {tone_instructions}: הנחיות הטון (יוזרק אוטומטית)
  ```
  קבצי הטון placeholders דומים עם TODO בלבד.

- [ ] **2. `app/prompts/README.md`** — מסמך הסבר למפתחים שיגעו בקבצים:
  - איך נטענים פרומפטים (`prompts_service`)
  - איך עובדת הזרקת הטון (`{tone_instructions}` placeholder)
  - איך מוסיפים פרומפט חדש (להעתיק template, להוסיף ל-`prompts_service` ולתעד)
  - אסור: לקרוא לפרומפט מבחוץ ל-`prompts_service` (לא לקרוא קובץ ישירות מ-handler).

- [ ] **3. `services/prompts_service.py`** — API ציבורי יחיד:
  ```python
  class PromptsService:
      def build(
          self,
          prompt_name: str,           # "copy_angle_1", "image_generation"
          tone: Optional[str] = None, # "luxury", "friendly_young", ...
          **params,                   # service_name, audience, offer, ...
      ) -> str:
          """
          Loads a prompt template, injects tone (if applicable),
          formats with params, and returns the final prompt string.

          Raises:
              PromptNotFoundError: אם הקובץ ריק או TODO בלבד
              PromptFormatError: אם חסר placeholder
          """
  ```

  **הזרימה הפנימית:**
  1. קריאת קובץ הפרומפט מ-`app/prompts/phase3/{prompt_name}.txt`.
  2. אם הקובץ ריק או מכיל רק "TODO" → `PromptNotFoundError("Prompt {prompt_name} not yet defined")`.
  3. אם `tone` סופק → קריאת `app/prompts/phase3/tones/{tone}.txt`, הזרקה דרך `{tone_instructions}` placeholder בקובץ הזווית. אם הקובץ של הטון ריק/TODO → `PromptNotFoundError`.
  4. `str.format(**params)` על המחרוזת המאוחדת.
  5. החזרה.

  **לא ב-Pydantic:** התבניות הן `str.format()`, לא Jinja2. החלטה מפורשת ב-Session.

- [ ] **4. `services/ad_generation_service.py`** — pipeline ליצירת מודעה:
  ```python
  async def generate_copy_variants(
      service_name: str,
      audience: str,
      offer: str,
      differentiation: str,
      tone: str,
  ) -> List[CopyVariant]:
      """
      יוצר 3 וריאציות קופי, סדרתי (כל אחת יודעת על הקודמות).

      Returns:
          [CopyVariant(angle="angle_1", text=...),
           CopyVariant(angle="angle_2", text=...),
           CopyVariant(angle="angle_3", text=...)]
      """

  async def generate_image_for_variant(
      copy_variant: CopyVariant,
      service_name: str,
      audience: str,
      tone: str,
  ) -> str:
      """
      יוצר תמונה אחת לwariant נתון.

      Returns:
          URL של התמונה ב-gpt-image-2.
      """
  ```

  **הזרימה ב-`generate_copy_variants`:**
  1. בניית הקשר משותף (`campaign_strategy` פרומפט) עם הפרמטרים.
  2. קריאה ל-`copy_angle_1` עם `tone` — מקבל text_1.
  3. קריאה ל-`copy_angle_2` עם `tone` + הזרקה "אל תחזור על: {text_1}" — מקבל text_2.
  4. קריאה ל-`copy_angle_3` עם `tone` + הזרקה "אל תחזור על: {text_1}, {text_2}" — מקבל text_3.
  5. החזרת המערך.

  **טיפול בכשל חלקי:** אם angle_2 נכשל — angle_1 ו-angle_3 נשארים בתור. החזרת רשימה עם variant חסר → ה-caller (admin UI או handler עתידי) יחליט אם retry או fail.

  **הזרימה ב-`generate_image_for_variant`:**
  1. קריאת `image_generation` עם הקלט (copy_text + service_name + tone).
  2. קריאה ל-gpt-image-2 API.
  3. החזרת ה-URL.

  **חשוב — לא writes ל-DB ב-Session 3.1.5.** הפונקציות מקבלות פרמטרים, מחזירות פלט. שמירה ל-DB מתבצעת ב-handler של 3.2/3.3 (אחרי 3.1.5).

- [ ] **5. `routers/admin/prompt_tester.py`** — דף ה-admin:
  ```python
  @router.get("/admin/prompt-tester")
  async def prompt_tester_page(token: str = Header(...)):
      if token != settings.ADMIN_TOKEN:
          raise HTTPException(401)
      return HTMLResponse(form_html)

  @router.post("/admin/prompt-tester/generate")
  async def prompt_tester_generate(request: GenerateRequest, token: str = Header(...)):
      if token != settings.ADMIN_TOKEN:
          raise HTTPException(401)

      copies = await ad_generation_service.generate_copy_variants(
          service_name=request.service_name,
          audience=request.audience,
          offer=request.offer,
          differentiation=request.differentiation,
          tone=request.tone,
      )

      images = []
      for copy in copies:
          image_url = await ad_generation_service.generate_image_for_variant(
              copy_variant=copy,
              service_name=request.service_name,
              audience=request.audience,
              tone=request.tone,
          )
          images.append(image_url)

      return {
          "copies": [{"angle": c.angle, "text": c.text} for c in copies],
          "images": images,
      }
  ```

  **דף HTML פשוט** (Jinja2 template):
  - שדה: שם השירות (text)
  - שדה: קהל (textarea)
  - שדה: הצעה (textarea)
  - שדה: בידול (textarea)
  - בורר רדיו: טון (4 אופציות)
  - כפתור Generate → AJAX call ל-`/generate`, מציג תוצאות באותו עמוד
  - שלוש כרטיסים: copy + image
  - כפתור "Show prompts used" — מציג את הפרומפטים המלאים שנשלחו (debugging)
  - כפתור "Save as fixture" — שומר ל-localStorage את הקלט הנוכחי
  - כפתור "Load fixture" — טוען מ-localStorage

  **טכנולוגיה:** HTML + CSS + vanilla JS. **לא React, לא build process.** Jinja2 כתבנית server-side.

- [ ] **6. אבטחה:**
  - `ADMIN_TOKEN` env var — חובה, אחרת השרת לא עולה.
  - הטוקן עובר ב-Header (`X-Admin-Token`), לא ב-query string (לא רוצים שירשם בלוגים).
  - הדף הראשון `GET /admin/prompt-tester` ידחה ללא טוקן — קלסי-ית `Bearer`.
  - **לא חשוף בדשבורד של הלקוח** — URL לא מקושר משום מקום. רק אנחנו יודעים שהוא קיים.

- [ ] **7. בדיקה ידנית (לפני שגיא חוזר):**
  - מחליפים את אחד הקבצים (`copy_angle_1.txt`) בפרומפט dummy שעובד:
    ```
    אתה כותב מודעה.
    שם השירות: {service_name}
    קהל: {audience}
    הצעה: {offer}

    {tone_instructions}

    כתוב פסקה קצרה של עד 100 מילים.
    ```
  - מחליפים את `tones/luxury.txt`:
    ```
    הטון: יוקרתי. השתמש במילים גבוהות-רמה. הימנע מ-emojis.
    ```
  - גולשים ל-`/admin/prompt-tester` (עם הטוקן), מזינים פרטים, לוחצים Generate.
  - **תוצאה צפויה:** angle_1 מקבל קופי. angle_2 ו-angle_3 זורקים `PromptNotFoundError` — זה תקין (placeholders).
  - מאשרים שה-pipeline עובד ב-end-to-end, ולא מחכים לתוכן הסופי.

### תת-החלטות שננעלו

1. **`str.format()` לתבניות, לא Jinja2.** קבצים טקסט נקי, placeholder סטנדרטי. אם בעתיד נצטרך תנאים — שדרוג.

2. **טון מוזרק דרך `{tone_instructions}` בתוך קובץ הזווית.** הזווית קובעת מבנה, הטון משובץ במשבצת מוגדרת.

3. **שמות גנריים** (`copy_angle_1/2/3`) **בינתיים**. אחרי תשובת גיא — שינוי שם הקבצים לפי הזוויות האמיתיות.

4. **`ad_generation_service` נכתב ב-3.1.5, לא ב-3.2.** ב-3.2/3.3 הוא יקבל handler wrapper בלבד.

5. **דף האדמין: server-side rendering עם Jinja2.** לא React, לא build process, לא state management.

6. **`ADMIN_TOKEN` ב-env בלבד, ב-Header.** לא ב-query string, לא ב-cookie.

7. **כשל חלקי ב-`generate_copy_variants` מחזיר רשימה עם variant חסר.** ה-caller מחליט מה לעשות. אין retry פנימי.

8. **`generate_image_for_variant` רץ מקבילית מה-caller.** ה-service עצמו לא יוזם מקביליות. ב-3.3 ה-handler יכול לקרוא לו 3 פעמים מקבילים.

### תלויות לפני 3.1.5

- ✅ Session 0.2 (FastAPI + Render) — חייב להיות done.
- ✅ Session 3.0 (jobs queue) — קיים, אבל **לא בשימוש ב-3.1.5.** ה-admin UI סינכרוני.
- 🔲 **OpenAI credentials** ב-env vars: `OPENAI_API_KEY`. **חוסם 3.1.5** — באחריות אמיר.
- 🔲 **`ADMIN_TOKEN`** ב-env vars: ערך אקראי וארוך. **חוסם 3.1.5** — באחריות אמיר.

### Done של 3.1.5

- `app/prompts/` קיימת עם כל ה-placeholders.
- `prompts_service.build('copy_angle_1', tone='luxury', service_name='X', ...)` מחזיר string (אחרי החלפת ה-placeholders ידנית בקובץ dummy).
- `ad_generation_service.generate_copy_variants(...)` רץ ומחזיר רשימה (גם אם חלק זורקים `PromptNotFoundError`).
- `GET /admin/prompt-tester` (עם טוקן) → דף HTML עם form.
- `POST /admin/prompt-tester/generate` → קופי + תמונה (לפחות angle_1 אחרי dummy fix).
- ניסיון ללא token → 401.
- מחיקת הקובץ dummy → הכל חוזר ל-placeholders → `/generate` מחזיר שגיאה ברורה.

### לא ב-3.1.5

- **תוכן הזוויות עצמן** — ימולא אחרי תשובת גיא.
- **handlers ב-jobs queue** — יתווספו ב-3.2/3.3.
- **שמירה ל-DB** — לא מתבצעת ב-3.1.5 (הפונקציות pure).
- **versioning של פרומפטים** — לא ב-MVP. אם בעתיד נצטרך — נוסיף `prompt_version` column.
- **A/B test בייצור** — Post-MVP.
- **DB-backed prompts** — לא. הפרומפטים בקוד, ב-git, סקירה דרך PR.
- **תרגום ל-EN** — לא רלוונטי, השוק שלנו עברית.

### זהירות

- **OpenAI API key חשוף ב-`/admin/prompt-tester`.** אם הטוקן דולף — מי שיודע אותו יכול להשתמש ב-API key שלנו לכל מטרה. הגנה: לוגינג מפורט של כל קריאה ל-OpenAI מהדף, כולל IP. אם רואים פעילות חשודה — סובבים את ה-token.
- **כל קובץ `.txt` הוא קוד.** שינוי בפרומפט = שינוי בהתנהגות המוצר. PR + סקירה כמו לכל קוד אחר.
- **`prompts_service` עוטף את גישות הקבצים.** אסור לקרוא ישירות מ-handler — הכלל הזה ב-CLAUDE.md.
- **OpenAI rate limits.** הדף הזה יכול לרוץ כמה פעמים ברצף. הוסף retry עם backoff ב-`ad_generation_service` (לא בdf, ב-service).

### עדכון ל-CLAUDE.md

> **חוק חדש: גישה לפרומפטים.**
> אסור לקרוא ישירות קבצי `app/prompts/*.txt` מאף מקום בקוד. הגישה היחידה היא דרך `prompts_service.build(...)`. הסיבה: ה-service מטפל ב-loading, format, injection, ושגיאות — והוא המקום היחיד שצריך לעדכן אם המבנה ישתנה.

# Session 3.1.6 — חילוץ Business Context מתשובות השאלון ✅

> **⚠️ נדרס (refactor מאוחר):** ההחלטה המקורית (חילוץ ב-LLM במקום שדות מובחנים) **התהפכה**.
> `service_name`/`business_description`/`problem_solved` נאספים כעת **ישירות** ב-wizard (מסכי business-context)
> ונשמרים per-user (`business_profile`, migration 0091) + per-campaign (`onboarding_draft` holding, migration 0092).
> `extract_business_context` צומצם ל-`extract_industry` (industry בלבד, מ-`business_description`+`service_name`).
> ה-section למטה מתאר את המימוש המקורי; ה-API `get_or_extract_business_context` נשמר (כעת מרכיב מ-2 מקורות).
>
> **עדכון להוספה ל-ROADMAP.md תחת Session 3.1.6** (חדש, מיד אחרי 3.1.5).
> נוצר בעקבות החלטה של גיא שלא להוסיף שדות מובחנים לשאלון —
> במקום זה נחלץ אותם מהתשובות החופשיות באמצעות LLM.

**מומש (3.1.6):** מיגרציה `0019_quiz_extracted_context.sql` (cache jsonb);
`extract_business_context` + `get_or_extract_business_context` (lazy + cache) ב-ad_generation;
`generate_copy_variants` מקבל `BusinessContext` (services_list הוסר); `OPENAI_TEXT_MODEL=gpt-5.2`;
admin tester מציג את ה-context שחולץ. **השלמה ל-3.1:** `offer`/`differentiation`/`business_name`
נוספו ל-QuizResponseInput (היו אמורים להיות בשאלון; קלט לחילוץ) + invalidation ב-save_quiz.

---

## תיאור Session 3.1.6

**שלב חילוץ בין השאלון לבין יצירת הקופי.** ה-pipeline של 3.1.5 דורש
שדות מובחנים (`service_name`, `business_description`, `problem_solved`)
שלא נשאלים ישירות בשאלון. ב-3.1.6 בונים שלב מקדים שמחלץ אותם מתשובות
ה-quiz באמצעות LLM, שומר ב-DB, וזורם הלאה.

**ההקשר:**
- גיא קבע שהשאלון מכסה את הצרכים — לא מוסיפים שדות נוספים.
- הניסיון יראה אם זה עובד בסביבת הבדיקות, ואם לא — נדע מאיפה.
- 3.1.6 מאפשר את החילוץ באופן מבוקר ושקוף (debugging דרך admin tester).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | באיזה מודל להשתמש לחילוץ? | **GPT-5.2** (gpt-4o-mini הוסר מ-API ב-16/2/2026) |
| 2 | מה קורה אם החילוץ נכשל? | **נכשלים בקול** — שגיאה ברורה, לא fallback. נחוץ ל-debug. |
| 3 | מתי החילוץ רץ? | **Lazy** — רק בקריאה ראשונה ליצירת קופי, לא בזמן submission |
| 4 | האם לחלץ `services_list`? | **לא** — מוסר מהפרומפט. נחלץ 3 שדות בלבד. |
| 5 | האם לעדכן את 3.1.5? | **כן** — אותו PR, חלק מ-3.1.6 |

---

## חלק 1 — מה משתנה ב-3.1.5

### 1.1 — `OPENAI_TEXT_MODEL` עובר ל-`gpt-5.2`

`gpt-4o-mini` הוסר מ-API של OpenAI ב-16 בפברואר 2026. ה-config של
3.1.5 שכבר נכתב צריך להתעדכן לפני שמשהו ירוץ בייצור.

**שינוי ב-`app/config.py`:**
```python
OPENAI_TEXT_MODEL: str = "gpt-5.2"  # היה "gpt-4o-mini"
```

**שינוי ב-Render env vars:** `OPENAI_TEXT_MODEL=gpt-5.2` (או להסיר את ה-
env var ולהשאיר רק את ה-default ב-config).

**הערה ל-CC:** אם בעתיד OpenAI ישחררו `gpt-5.2-mini` שעולה פחות
ועובד טוב לחילוץ — אפשר להוסיף `OPENAI_EXTRACT_MODEL` נפרד. ב-MVP
משתמשים באותו מודל לחילוץ וליצירה.

### 1.2 — `copy_generation.txt` — הסרת `services_list`

הפרומפט של 3.1.5 מכיל 13 placeholders. אחד מהם — `{services_list}` —
מוסר. הסיבה: אין דרך אמינה לחלץ רשימת **כל** שירותי העסק מתשובות
לשאלון שמתמקד בקמפיין ספציפי.

**שינוי ב-`app/prompts/phase3/copy_generation.txt`:**
- הסר את השורה `כל שירותי העסק: {services_list}` מתוך "נתוני העסק".
- שאר הפרומפט נשאר כפי שהוא.

### 1.3 — `ad_generation_service.generate_copy_variants` — signature change

הפונקציה הציבורית מקבלת היום 13 פרמטרים. אחד מהם (`services_list`)
מוסר.

**שינוי ב-`app/services/ad_generation_service.py`:**
- הסר את `services_list: str` מ-signature.
- הסר אותו מה-`prompts_service.build(...)` call.
- ה-`dataclass` `BusinessContext` (יוגדר בהמשך) לא יכיל את השדה.

---

## חלק 2 — שלב החילוץ החדש

### 2.1 — `app/prompts/phase3/extract_business_context.txt`

קובץ פרומפט חדש בתיקיית 3.1.5. מקבל את תשובות השאלון הגולמיות,
מחזיר JSON עם 3 שדות.

```
אתה אנליסט עסקי. תפקידך לחלץ מידע מובחן מתוך תשובות לשאלון עומק שמילא בעל עסק.

## נתוני העסק

שם העסק: {business_name}

## תשובות השאלון

קהל היעד: {audience}
ההצעה השיווקית: {offer}
הבידול והיתרונות: {differentiation}

---

## משימתך

חלץ מהתשובות לעיל את שלושת השדות הבאים:

1. **service_name** — שם השירות הספציפי שעליו הקמפיין הזה. עסק
   יכול להציע כמה שירותים — זה השירות המסוים שמוצג בקמפיין.
   דוגמאות: "שיעורי נהיגה לנשים", "טיפול קוסמטי לעור פנים", "ייעוץ
   עסקי לעצמאים".

2. **business_description** — תיאור קצר של מה העסק עושה ברמה
   הכללית (1-2 משפטים). לא ההצעה הספציפית, אלא הזהות המקצועית.
   דוגמה: "סטודיו לקוסמטיקה רפואית פרימיום בתל אביב, מתמחה
   בטיפולי פנים אנטי-אייג'ינג".

3. **problem_solved** — איזו בעיה או צורך העסק פותר ללקוח. זה
   הבסיס לקופי מסוג "כאב + פתרון". דוגמה: "נשים שמרגישות שהעור
   שלהן מאבד את הזוהר עם השנים ורוצות פתרון מקצועי בלי ניתוחים".

## חוקים נוקשים

1. **אל תמציא** — אם המידע לא קיים בתשובות, החזר string ריק לאותו
   שדה. אל תנחש.

2. **חלץ במדויק** — אם בתשובה כתוב "שיעורי נהיגה לנשים", החזר את
   זה. אל תכליל ל-"שיעורי נהיגה".

3. **כתוב בעברית** — כל השלושה בעברית.

## פורמט הפלט

JSON תקין בלבד, ללא טקסט נוסף:

```json
{{
  "service_name": "...",
  "business_description": "...",
  "problem_solved": "..."
}}
```
```

**הערה ל-CC:** ה-`{{` ו-`}}` בסוף (double braces) הכרחיים — `str.format()`
רואה `{` יחיד כ-placeholder. אותו דפוס כמו ב-3.1.5.

### 2.2 — Pydantic model + dataclass

**הוסף ל-`app/services/ad_generation_service.py`:**

```python
@dataclass
class BusinessContext:
    service_name: str
    business_description: str
    problem_solved: str
```

זה ה-dataclass שמייצג את התוצאה של החילוץ. אותו pattern כמו
`CopyVariant` הקיים.

### 2.3 — פונקציה חדשה: `extract_business_context`

```python
async def extract_business_context(
    business_name: str,
    audience: str,
    offer: str,
    differentiation: str,
) -> BusinessContext:
    """
    Extracts business_description, service_name, and problem_solved
    from quiz answers using LLM.

    Raises:
        BusinessContextExtractionError: אם החילוץ נכשל או החזיר שדה ריק.
    """
```

**הזרימה:**
1. בניית הפרומפט דרך `prompts_service.build('extract_business_context', ...)`.
2. קריאה ל-OpenAI Chat Completions API (`OPENAI_TEXT_MODEL`, כעת `gpt-5.2`):
   - `temperature=0.2` (חילוץ מדויק, לא יצירתי)
   - `response_format={"type": "json_object"}`
   - `max_tokens=500` (החילוץ קצר)
3. פענוח ה-JSON. אם נכשל → `BusinessContextExtractionError`.
4. ולידציה: כל 3 השדות חייבים להיות מחרוזות לא-ריקות. אם משהו ריק →
   `BusinessContextExtractionError("חסר שדה X — נסה לתת תשובות יותר ספציפיות")`.
5. החזרת `BusinessContext`.

**Retry logic:** אם OpenAI מחזיר rate limit → exponential backoff עד
3 ניסיונות (אותו pattern כמו ב-`generate_copy_variants` הקיים).

### 2.4 — `generate_copy_variants` — שינוי בקריאה

הפונקציה הציבורית מקבלת כעת `business_context: BusinessContext`
במקום שלושת השדות הבודדים:

```python
async def generate_copy_variants(
    business_name: str,
    business_context: BusinessContext,  # חדש
    offer: str,
    differentiation: str,
    campaign_goal: Literal["lead", "whatsapp"],
    audience: str,
    gender: Literal["male", "female", "all"],
    age_range: str,
    location: str,
    budget_info: str,
    tone: Literal["professional", "friendly", "luxury", "direct", "authoritative"],
) -> list[CopyVariant]:
    """..."""
```

ה-3 שדות מ-`BusinessContext` מועברים לפרומפט כ-`service_name`,
`business_description`, `problem_solved`.

**הסרה:** `services_list` יורד מהפרמטרים. שאר הפרמטרים נשארים.

---

## חלק 3 — Caching ב-DB

### 3.1 — עמודה חדשה ב-`quiz_responses`

**migration ייעודי** (המספור ייקבע לפי המיגרציות שכבר קיימות, כנראה 0020):

```sql
ALTER TABLE public.quiz_responses
  ADD COLUMN extracted_context jsonb;

COMMENT ON COLUMN public.quiz_responses.extracted_context IS
  'BusinessContext שחולץ מ-LLM. JSON: {service_name, business_description, problem_solved}. ' ||
  'NULL = עוד לא חולץ. lazy-populated בקריאה ראשונה ליצירת קופי.';
```

**למה JSONB ולא 3 עמודות נפרדות:**
- 3 השדות נקראים יחד, לא בנפרד.
- גמישות עתידית — אם נחליט להוסיף שדה רביעי (`industry`, `unique_angle`), זה לא דורש migration.
- עקבי עם `answers` שכבר ב-`quiz_responses`.

### 3.2 — לוגיקת ה-caching

**ב-`ad_generation_service`** — שכבת service חדשה:

```python
async def get_or_extract_business_context(
    quiz_response_id: UUID,
) -> BusinessContext:
    """
    Returns cached BusinessContext if exists, otherwise extracts and caches.
    """
    quiz = fetch_quiz_response(quiz_response_id)

    if quiz.extracted_context:
        # cached → parse and return
        return BusinessContext(**quiz.extracted_context)

    # not cached → extract
    context = await extract_business_context(
        business_name=quiz.business_name,
        audience=quiz.audience,
        offer=quiz.offer,
        differentiation=quiz.differentiation,
    )

    # save to DB
    save_extracted_context(quiz_response_id, context)

    return context
```

**הערות:**
- `fetch_quiz_response` — קיים ב-`quiz_service` (3.1).
- `save_extracted_context` — חדש, UPDATE ב-DB דרך admin client.
- ה-cache עובד **עד שהלקוח משנה את ה-quiz**. אם הוא חוזר ל-3.1 ועדכן תשובות, צריך לאפס את `extracted_context` ל-NULL — נוסיף את זה כאשר 3.1 ייכתב.

---

## חלק 4 — שינוי ב-Admin Tester (3.1.5)

### 4.1 — תצוגה של תוצאת החילוץ

**שינוי ב-`routers/admin/prompt_tester.py`:**

ה-endpoint `/admin/prompt-tester/generate` עכשיו רץ 3 שלבים:
1. `extract_business_context(...)` → `BusinessContext`
2. `generate_copy_variants(business_context=..., ...)` → 3 variants
3. `generate_image_for_variant(...)` × 3 → 3 images

**ה-response הופך ל:**
```python
class GenerateResponse(BaseModel):
    business_context: BusinessContextResponse  # חדש
    copies: list[CopyVariantResponse]
    images: list[str]
    prompts_used: dict[str, str]
```

**ב-HTML:**
- כרטיס חדש בראש התוצאות: "Business Context שחולץ" עם 3 השדות.
- כפתור "Show extraction prompt" שמראה את הפרומפט המלא שנשלח לחילוץ.
- אחרי זה — שלושת הכרטיסים של copy + image (כמו ב-3.1.5).

### 4.2 — שינוי ב-`GenerateRequest`

הפרמטרים נשארים אותם 13 כמו ב-3.1.5 (האדמין מזין את הכל ידנית
לבדיקה). השינוי הוא רק שב-pipeline, ה-`audience`/`offer`/`differentiation`
ניתנים ל-`extract_business_context`, ותוצאת החילוץ מועברת ל-
`generate_copy_variants`.

**בפועל:** האדמין יכול לראות איך אותם 3 שדות שהוא הזין מתפרשים
ע"י ה-LLM כ-`service_name`/`business_description`/`problem_solved`.
זה ה-debug שאמיר רצה.

---

## חלק 5 — Done של 3.1.6

- migration להוספת `extracted_context` ב-`quiz_responses` רצה בהצלחה.
- `OPENAI_TEXT_MODEL` עודכן ל-`gpt-5.2` ב-config וב-Render.
- `services_list` הוסר מ-`copy_generation.txt`, מ-signature של
  `generate_copy_variants`, ומה-`prompts_service.build()` call.
- `app/prompts/phase3/extract_business_context.txt` קיים עם תוכן אמיתי.
- `extract_business_context` ו-`get_or_extract_business_context` בנויים
  ב-`ad_generation_service`.
- `BusinessContext` dataclass קיים.
- `BusinessContextExtractionError` exception class קיים.
- Admin tester מציג את החילוץ לפני הקופי.
- חילוץ נכשל (שדה ריק) → exception ברור, לא fallback.
- caching עובד — קריאה שנייה לאותו `quiz_response_id` לא מבצעת קריאה
  ל-OpenAI.
- כל הטסטים החדשים עוברים.

## חלק 6 — לא ב-3.1.6

- שאלון מעודכן עם שדות מובחנים (גיא דחה).
- חילוץ `services_list` — נדחה לחלוטין.
- invalidation של `extracted_context` כשה-quiz משתנה — נטופל ב-3.1.
- UI ללקוח שמציג את ה-BusinessContext שחולץ — Post-MVP.

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **3.1.6 ו-3.1.5 ב-PR אחד.** השינויים ב-3.1.5 (הסרת `services_list`,
   עדכון מודל) הם חלק מ-3.1.6 מבחינת PR. לא PR נפרד.

2. **`gpt-5.2` כברירת מחדל, ולא `gpt-5.2-mini`.** אם בעתיד יהיה
   `gpt-5.2-mini` שעובד טוב לחילוץ ועלותו זולה משמעותית, נוסיף
   `OPENAI_EXTRACT_MODEL` נפרד. ב-MVP אותו מודל לשתי המשימות.

3. **`temperature=0.2` לחילוץ** — נמוך כדי לקבל תוצאות עקביות.
   ל-`generate_copy_variants` נשאר `temperature=0.8` (גיוון).

4. **שגיאות חילוץ חייבות להיות ברורות בעברית.** הודעות כמו "לא הצלחנו
   לחלץ את שם השירות מהתשובות. נסה לתת תשובות יותר ספציפיות."
   זה לא רק UX — זה גם debug אדם-קריא ב-Sentry.

5. **caching = lazy, לא eager.** אל תוסיף trigger או cron שמחלץ
   אוטומטית. החילוץ קורה רק כשמשתמש מבקש קופי, ונשמר אז.

6. **invalidate cache כש-quiz משתנה — TODO ל-3.1.** כשה-3.1
   ייכתב, ה-PATCH של quiz צריך לאפס `extracted_context` ל-NULL.

7. **ה-admin tester לא משתמש ב-cache.** הוא מריץ extract בכל קריאה.
   הסיבה: זה כלי debug, רוצים לראות את החילוץ הטרי בכל פעם.

8. **שמירה בDB דרך admin client** — `save_extracted_context` הוא
   server-authoritative (spec §7.3-ב). אותו pattern כמו 2.5.



### Session 3.2 — יצירת קופי (ChatGPT) ✅
- [x] endpoint שמקבל שאלון ומייצר 3 גרסאות קופי (כאב/תועלת/קצרה+CTA)
- [x] שמירה ב-`ads` (variant 1/2/3, רק copy_text בשלב זה)

**Done:** 3 רשומות ב-`ads` עם קופי שנוצר אמיתית מה-AI.
**זהירות:** external-sdk patterns — rate limits, timeout, תשובה לא תקינה.


# Session 3.2 — Copy Generation Handler ✅

> **עדכון להוספה ל-ROADMAP.md תחת Session 3.2.** משלב את התשתית של 3.1.5
> (יצירת קופי) ושל 3.1.6 (חילוץ business context) לתוך flow אמיתי עם DB
> ועם רענון פר-וריאציה. כל הפעולות סינכרוניות — לא דרך jobs queue.

**מומש (3.2):** מיגרציה `0020_ads.sql` (RLS select-own + GRANT, כלל 11; composite FK;
UNIQUE(campaign_id,angle)); `AdResponse`; `generate_single_copy_variant` + `PreviousVariant`
+ `copy_generation_single.txt`; `copy_service` (generate_initial_copy / regenerate_variant /
list_ads + formatters); 3 endpoints ב-`routers/ads.py`. 31 טסטים. **2 סטיות מה-spec של
ה-ROADMAP (תיקונים, לא סותרים):** (א) בלי trigger ל-updated_at — אין `update_updated_at_column`
בפרויקט, מתעדכן ב-app code; (ב) נוסף `GRANT select` (ה-spec החסיר אותו). copy_service ניגש
לטבלאות ישירות (דפוס quiz_service) — לא נדרשו helpers ב-campaign_service/quiz_service.

---

## תיאור Session 3.2

**יצירת קופי + רענון פר-וריאציה.** המשתמש לוחץ "צור קופי" → 3 וריאציות
נוצרות ונשמרות ב-`ads`. אם וריאציה מסוימת לא מוצאת חן בעיניו — הוא לוחץ
"רענן" על הכרטיס שלה, וה-variant הזה מתחלף בחדש (התמונה נשארת).

**מה לא ב-3.2:**
- יצירת תמונות — 3.3.
- עריכה ידנית של variant — נדחה ל-UI session או 3.4.
- push לקמפיין — 3.4.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מתי נוצר ה-job? | **לחיצה ייעודית** "צור קופי". לא אוטומטי. |
| 2 | רענון כל 3, או פר-וריאציה? | **פר-וריאציה** (כמו בפרוטוטייפ של גיא) |
| 3 | רענון = מחיקת ישנים? | **כן** — UPDATE על אותה שורה, לא הוספה |
| 4 | תמונה אחרי רענון קופי? | **נשארת.** המשתמש יכול לרענן תמונה בנפרד. |
| 5 | סינכרוני או async? | **סינכרוני לשניהם** (קריאה אחת, ~2-3s) |
| 6 | retry אוטומטי בכשל? | **אין** — שגיאה למשתמש, הוא לוחץ שוב |
| 7 | טרמינולוגיה | **"קופי"** (לא "טקסט מודעה") |
| 8 | variant חדש שונה מהקיימים? | **כן** — הפרומפט מקבל את 3 הקיימים כ"לא לחזור" |
| 9 | עדכון `campaign.status`? | **לא** — נשאר `draft` עד 3.4 |

---

## חלק 1 — Endpoints

### 1.1 — יצירת קופי ראשונית

```
POST /campaigns/:id/generate-copy
→ Validation gates
→ קריאה ל-OpenAI (3 variants בקריאה אחת)
→ INSERT 3 ads
→ Return 200 + 3 ads
```

**Validation gates:**
1. `campaign` קיים ושייך למשתמש (RLS).
2. `campaign.status = 'draft'`.
3. `quiz_responses` קיים עם תשובות מלאות.
4. `subscription.has_paid_access = true`.
5. **`ads` count עבור הקמפיין = 0** — אם כבר יש 3 ads, לא ניתן ליצור מחדש (צריך לרענן פר-variant).

**אם כל ה-gates עברו:**
1. קריאה ל-`get_or_extract_business_context(quiz_id)` (3.1.6).
2. קריאה ל-`generate_copy_variants(...)` (3.1.5) → 3 `CopyVariant` objects.
3. בתוך transaction:
   - INSERT 3 שורות ל-`ads`, אחת לכל angle (`emotional`, `pain_solution`,
     `result_success`).
   - כל שורה: `headline`, `body`, `cta`, `image_concept`, `status='pending'`,
     `image_url=NULL`.
4. Return 200 עם 3 ה-ads ב-Pydantic response.

**זמן צפוי:** ~3 שניות (קריאה אחת ל-OpenAI שמייצרת את כל 3 הוריאציות).

### 1.2 — רענון של variant יחיד

```
POST /ads/:ad_id/regenerate-copy
→ Validation gates
→ קריאה ל-OpenAI (single variant)
→ UPDATE ה-ad
→ Return 200 + the updated ad
```

**Validation gates:**
1. `ad` קיים ושייך למשתמש (RLS).
2. `campaign.status = 'draft'`.
3. `subscription.has_paid_access = true`.
4. אין UI lock — `frontend` אחראי על מניעת double-click. backend לא נועל
   ב-DB (פשטות MVP).

**אם כל ה-gates עברו:**
1. שלוף את כל 3 ה-ads של הקמפיין (3 שורות).
2. הכן `previous_variants` — list של dicts עם headline/body/cta של כל 3
   (כולל ה-ad שמתחדש).
3. קריאה ל-`generate_single_copy_variant(angle, previous_variants, ...)`.
4. UPDATE על ה-ad:
   - `headline`, `body`, `cta`, `image_concept` — חדשים.
   - **`image_url` — לא משתנה.**
   - `updated_at = now()`.
5. Return 200 עם ה-ad המעודכן.

**זמן צפוי:** ~2 שניות.

### 1.3 — תצוגת תוצאות

```
GET /campaigns/:id/ads
→ 200 [{id, angle, headline, body, cta, image_concept, image_url, status}, ...]
```

נשתמש בזה ב-UI לתצוגת הכרטיסים אחרי כל פעולה.

---

## חלק 2 — מבנה טבלת `ads`

### 2.1 — Migration

המספור ייקבע לפי המיגרציות הקיימות:

```sql
CREATE TABLE public.ads (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  campaign_id uuid NOT NULL REFERENCES public.campaigns(id) ON DELETE CASCADE,

  -- composite FK ל-RLS coherence (spec §7.2, כמו ב-leads)
  CONSTRAINT ads_campaign_user_fk
    FOREIGN KEY (campaign_id, user_id)
    REFERENCES public.campaigns(id, user_id)
    ON DELETE CASCADE,

  -- Variant info
  angle text NOT NULL CHECK (angle IN ('emotional', 'pain_solution', 'result_success')),

  -- Copy fields (3.2)
  headline text,
  body text,
  cta text,
  image_concept text,

  -- Image fields (3.3 — נשארים NULL כעת)
  image_url text,
  image_prompt text,

  -- Meta fields (3.4 — נשארים NULL כעת)
  meta_ad_id text,

  -- Status state machine
  status text NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'generating', 'rejected', 'live', 'failed_push')),

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- אינדקסים
CREATE INDEX idx_ads_campaign ON public.ads (campaign_id);
CREATE INDEX idx_ads_meta_id ON public.ads (meta_ad_id) WHERE meta_ad_id IS NOT NULL;
CREATE UNIQUE INDEX uq_ads_campaign_angle ON public.ads (campaign_id, angle);
-- ^ unique constraint: כל קמפיין יש לו variant אחד מכל angle

-- RLS
ALTER TABLE public.ads ENABLE ROW LEVEL SECURITY;

CREATE POLICY ads_select_own ON public.ads FOR SELECT USING (auth.uid() = user_id);

-- INSERT/UPDATE/DELETE — רק דרך admin client (server-authoritative)

-- Trigger לעדכון updated_at
CREATE TRIGGER update_ads_updated_at
  BEFORE UPDATE ON public.ads
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 2.2 — Status state machine

- **`pending`** — נוצר ב-3.2, מחכה ליצירת תמונה (3.3).
- **`generating`** — תמונה בתהליך יצירה (3.3).
- **`rejected`** — Meta דחתה (3.4 או Phase 8).
- **`live`** — באוויר אחרי push (3.4).
- **`failed_push`** — כשל טכני ב-push (3.4).

ב-3.2 ה-status הוא תמיד `pending` (יצירה ראשונית או רענון קופי).

### 2.3 — Unique constraint על angle

`uq_ads_campaign_angle` מבטיח שיש בדיוק 3 ads פר קמפיין — אחד לכל angle.
זה גם מבטיח שרענון פר-variant לעולם לא יוסיף שורה חדשה, רק יעדכן את
הקיימת לפי `angle`.

**שימושי גם ל-3.2 endpoint #1:** אם בטעות ננסה INSERT שני variants עם
אותו angle → ה-DB ידחה. הגנה נוספת על correctness.

---

## חלק 3 — שינויים ב-3.1.5

3.1.5 כבר מומש ב-CC, אבל 3.2 דורש הרחבות:

### 3.1 — פונקציה חדשה: `generate_single_copy_variant`

ב-`app/services/ad_generation_service.py`:

```python
@dataclass
class PreviousVariant:
    angle: str
    headline: str
    body: str
    cta: str

async def generate_single_copy_variant(
    angle: Literal['emotional', 'pain_solution', 'result_success'],
    previous_variants: list[PreviousVariant],
    business_name: str,
    business_context: BusinessContext,
    offer: str,
    differentiation: str,
    campaign_goal: Literal['lead', 'whatsapp'],
    audience: str,
    gender: Literal['male', 'female', 'all'],
    age_range: str,
    location: str,
    budget_info: str,
    tone: Literal['professional', 'friendly', 'luxury', 'direct', 'authoritative'],
) -> CopyVariant:
    """
    Generates a single new copy variant, different from the provided ones.

    The `previous_variants` parameter contains all 3 current variants (including
    the one being regenerated). The LLM is instructed not to produce something
    similar to any of them.
    """
```

הזרימה:
1. בניית פרומפט דרך `prompts_service.build('copy_generation_single', ...)`.
2. קריאה ל-OpenAI (`gpt-5.2`, `temperature=0.8`).
3. פענוח JSON → `CopyVariant`.
4. ולידציה — headline/body/cta לא ריקים.
5. אם נכשל → `CopyGenerationError`.

### 3.2 — פרומפט חדש: `copy_generation_single.txt`

ב-`app/prompts/phase3/copy_generation_single.txt`:

```
[הקדמה זהה ל-copy_generation.txt]

## נתוני העסק
[אותם 12 placeholders, בלי services_list]

## angle ספציפי
תייצר וריאציה עבור angle: {angle}

[הסבר על מה זה ה-angle הזה]

## וריאציות קיימות שאסור לחקות

הנה 3 הוריאציות הנוכחיות של הקמפיין — אסור שהוריאציה החדשה תהיה דומה
לאף אחת מהן בסגנון, בטון, או בניסוח. תייצר משהו שונה מהותית:

{previous_variants_formatted}

## פורמט הפלט

JSON תקין יחיד (לא array), ללא טקסט נוסף:

```json
{{
  "angle": "{angle}",
  "headline": "...",
  "body": "...",
  "cta": "...",
  "image_concept": "..."
}}
```
```

**הערה:** `{previous_variants_formatted}` ייבנה ב-service לפני שליחה ל-
`prompts_service.build`. זה יראה כמו:

```
### וריאציה 1 (emotional)
כותרת: ...
גוף: ...
CTA: ...

### וריאציה 2 (pain_solution)
...
```

---

## חלק 4 — Service layer ב-3.2

### 4.1 — `app/services/copy_service.py` (חדש)

```python
async def generate_initial_copy(
    campaign_id: UUID,
    user_id: UUID,
) -> list[AdResponse]:
    """
    Creates 3 ads for a campaign.

    Raises:
        ValidationError: gate נכשל.
        CopyGenerationError: כשל ב-LLM.
        AlreadyHasAdsError: כבר יש 3 ads (חייב לרענן פר-variant).
    """
    # 1. Validation gates
    # 2. Load quiz, extract context
    # 3. Generate 3 variants
    # 4. Transaction: INSERT 3 ads

async def regenerate_variant(
    ad_id: UUID,
    user_id: UUID,
) -> AdResponse:
    """
    Regenerates a single ad's copy. Image stays.

    Raises:
        ValidationError: gate נכשל.
        CopyGenerationError: כשל ב-LLM.
    """
    # 1. Validation gates
    # 2. Load ad, its campaign, sibling ads
    # 3. Generate single variant
    # 4. UPDATE the ad (NOT image_url)
```

### 4.2 — Pydantic response models

ב-`app/models/ad.py`:

```python
class AdResponse(BaseModel):
    id: UUID
    angle: Literal['emotional', 'pain_solution', 'result_success']
    headline: str | None
    body: str | None
    cta: str | None
    image_concept: str | None
    image_url: str | None
    status: Literal['pending', 'generating', 'rejected', 'live', 'failed_push']
    created_at: datetime
    updated_at: datetime
```

### 4.3 — Error handling

כל הכשלים → HTTP 500 עם הודעת שגיאה ב-עברית.

| Exception | HTTP | הודעה |
|---|---|---|
| `ValidationError` (gate) | 400 | תלוי gate, ב-עברית |
| `AlreadyHasAdsError` | 409 | "כבר יש קופי לקמפיין הזה. השתמש בכפתור 'רענן' לכל וריאציה." |
| `CopyGenerationError` | 500 | "יצירת הקופי נכשלה. נסה שוב בעוד רגע." |
| `BusinessContextExtractionError` | 400 | מועבר מ-3.1.6 כפי שהוא |

**אין retry אוטומטי בשום מקרה.** המשתמש לוחץ שוב.

---

## חלק 5 — Done של 3.2

- Migration לטבלת `ads` רץ (אם לא קיימת). RLS פעיל. Unique constraint
  על `(campaign_id, angle)` פעיל.
- `app/prompts/phase3/copy_generation_single.txt` קיים עם תוכן אמיתי.
- `generate_single_copy_variant` בנוי ב-`ad_generation_service`.
- `generate_initial_copy` ו-`regenerate_variant` בנויים ב-`copy_service`.
- `POST /campaigns/:id/generate-copy` עובד סינכרונית, יוצר 3 ads, ~3s.
- `POST /ads/:ad_id/regenerate-copy` עובד סינכרונית, מעדכן ad יחיד
  (קופי + image_concept), שומר על image_url, ~2s.
- `GET /campaigns/:id/ads` מחזיר 3 ads עם RLS.
- כשלים מחזירים 4xx/5xx עם הודעות עברית. אין retry אוטומטי.
- כל הטסטים עוברים.

## חלק 6 — לא ב-3.2

- יצירת תמונות (3.3).
- עריכה ידנית של variant (`PATCH /ads/:ad_id`) — נדחה.
- jobs queue אינו נדרש ב-3.2.
- Worker handlers — אין ב-3.2.
- Status transitions ל-`generating`/`live`/`rejected`/`failed_push` — Phases מאוחרות.

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **3.2 הוא PR נפרד מ-3.1.5 + 3.1.6.** הפונקציה החדשה
   `generate_single_copy_variant` והפרומפט `copy_generation_single.txt`
   מתווספים ל-`ad_generation_service` הקיים. שינוי קוד, לא refactor.

2. **Unique constraint על `(campaign_id, angle)` מקל על המימוש של
   `regenerate_variant`** — אפשר לעדכן ad לפי `WHERE campaign_id = ? AND
   angle = ?` בלי לחפש את ה-id בנפרד.

3. **`generate_initial_copy` חייב להיות בתוך transaction אטומית.** אם
   INSERT של ad #2 נכשל אחרי שאד #1 נוצר — צריך rollback. אחרת נישאר
   עם state חלקי שלא מאפשר ניסיון חוזר (Validation gate #5 יכשל).

4. **timeout על הקריאה ל-OpenAI: 25 שניות.** Render request timeout הוא
   30s. צריך מרווח לפעולות DB. אם OpenAI איטית מ-25s — מחזירים 504 ל-
   משתמש ("הקריאה ל-AI נכשלה, נסה שוב").

5. **UI אחראי על double-click prevention.** Backend לא מבצע lock. אם
   המשתמש איכשהו שלח שתי בקשות במקביל — שני UPDATE יבוצעו, האחרון
   ינצח. ב-MVP זה מקובל. אם בעתיד יהיו תלונות — נוסיף advisory lock.

6. **ה-image_url נשאר בריענון קופי.** הכרטיס ב-UI יציג את התמונה הישנה
   עם הקופי החדש. אם המשתמש רוצה תמונה חדשה — הוא לוחץ "רענן תמונה"
   בנפרד (3.3).

7. **`updated_at` מתעדכן ע"י trigger ב-DB.** ה-service לא צריך לעדכן
   ידנית. trigger קיים מ-mass-table migration (אם לא — להוסיף).

8. **`temperature=0.8` בשני המקרים** (גיוון בין variants). הפרומפט
   ה-single מקבל את ה-3 הקיימים בקלט ככיוון נוסף לגיוון.


### Session 3.3 — יצירת תמונות (gpt-image-2) — כ-job ✅
- [x] handler שמייצר 3 תמונות (Medium, 1:1) → Storage → `image_url`
- [x] שמירת `image_prompt` לשחזור
- [x] **רץ דרך ה-queue** (job type: `generate_ad_images`), לא סינכרוני
- [x] עדכון status שהמשתמש יראה שהתמונות בהכנה (`generating`)

**Done:** הכנסת job → ה-worker מייצר 3 תמונות אמיתיות ושומר. המשתמש רואה
התקדמות.

**הושלם — הערות יישום (סטיות מהתכנון המקורי שהתגלו בחקירת הקוד):**
- **`generate_image_for_variant` כבר היה קיים** (3.1.5) ובשימוש ב-prompt_tester, מחזיר
  str (URL/data-URL). לא נשבר — נוספה `generate_image_bytes_for_variant` (→ bytes ל-Storage)
  עם `_build_image_prompt` משותף.
- **טבלת `ads` כבר כללה** `image_url`/`image_prompt`/`generating` (0020). 0021 הוסיף רק
  `failed_image` + `image_error`; 0022 הוסיף `generate_ad_images` ל-`jobs.type`; 0023 = bucket.
- **handler = קובץ יחיד** `worker/handlers.py` (לא ספרייה) — `handle_generate_ad_images`
  ב-`HANDLERS`. ה-payload כולל `quiz_id` (ה-handler מקבל `payload` בלבד, בלי user_id בחתימה).
- **orchestration ב-`image_service.py` חדש** (לא ב-handler) + refactor: helpers משותפים
  הוצאו לפי domain — `ads_service` (data layer), `subscription_service.require_paid_access`,
  `campaign_service.fetch_for_edit`, `quiz_service.fetch_for_generation`.
- **402** (Payment Required) לכל endpoints של תוכן — כולל שינוי generate-copy/regenerate-copy
  מ-3.2 (היו 403), לעקביות.
- **idempotency:** `image_url IS NULL` (worker sequential) + CAS `reserve_ads_for_images`
  (status→generating) חוסם double-enqueue. double-charge רק ב-crash אחרי OpenAI לפני DB (MVP).
- **לאמת ב-deploy:** migration 0023 (bucket + policy על `storage.objects`) דורש הרשאות —
  אם נכשל, ליצור ידנית ב-Dashboard. ה-storage3 API + מודל gpt-image מאומתים רק מול Supabase/OpenAI.


# Session 3.3 — Image Generation Handler

## תיאור Session 3.3

**יצירת תמונות פוסט.** אחרי ש-3.2 יצר 3 וריאציות קופי, 3.3 מייצר תמונה לכל וריאציה. התמונות נוצרות ב-gpt-image-2, נשמרות ב-Supabase Storage, וה-URL הקבוע נשמר ב-`ads.image_url`. זה השלב הכבד ביותר ב-Phase 3 (10-15 שניות פר תמונה) ולכן רץ אסינכרונית דרך jobs queue.

**מה לא ב-Session הזה:**
- העלאת תמונות מהמשתמש (Vision/editing) — Phase 3.5.
- העלאת התמונות ל-Meta — Phase 3.4.
- עריכה ידנית של `image_concept` — לא ב-MVP בכלל (לפי הפרוטוטייפ).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | יצירה ראשונה — sync או async? | **async** (jobs queue) — 30-45s זה מעבר ל-timeout |
| 2 | רענון תמונה יחידה — sync או async? | **sync** (~10-15s) — עקבי עם 3.2 |
| 3 | איפה התמונות נשמרות? | **Supabase Storage**, bucket `campaign-images` |
| 4 | מבנה הקבצים | `{user_id}/{ad_id}/{timestamp}.png` (RLS לפי prefix) |
| 5 | קופי ותמונות — flow משולב או נפרד? | **משולב** — קופי sync, תמונות async אוטומטית אחרי |
| 6 | Handler — מקבילי או סדרתי? | **מקבילי** — `asyncio.gather(return_exceptions=True)` |
| 7 | Idempotency של handler | `ads.image_url IS NULL` — דלג על קיימות |
| 8 | איכות + יחס | **Medium 1:1** בלבד ב-MVP |
| 9 | טיפול בכשל פר-תמונה | `ads.status='failed'` פר-ad, השאר מצליחות |
| 10 | עריכת `image_concept` ע"י משתמש? | **לא ב-MVP** (לפי הפרוטוטייפ) |
| 11 | תקרת retry פר-תמונה | 3 ניסיונות עם backoff (תשתית 3.0) |
| 12 | מה ב-`ads.status` במהלך היצירה | `generating` → `pending` (הצלחה) / `failed_image` (כשל) |

---

## חלק 1 — Storage Setup (Supabase Storage)

### 1.1 — Bucket חדש

מיגרציה נפרדת (`0021_campaign_images_bucket.sql`):

```sql
-- Bucket creation
INSERT INTO storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
VALUES (
  'campaign-images',
  'campaign-images',
  false,  -- לא public — גישה דרך signed URLs בלבד
  5242880,  -- 5MB limit per file
  ARRAY['image/png', 'image/jpeg', 'image/webp']
);

-- RLS policies (Supabase Storage uses standard RLS)
-- SELECT: בעלים בלבד
CREATE POLICY "Users can view own campaign images"
  ON storage.objects FOR SELECT
  USING (
    bucket_id = 'campaign-images'
    AND auth.uid()::text = (storage.foldername(name))[1]
  );

-- INSERT/UPDATE/DELETE: רק דרך service_role (admin client)
-- אין policy → ברירת המחדל היא דחייה
```

**מבנה הקבצים:**
```
{user_id}/{ad_id}/{timestamp}.png
```

דוגמה:
```
campaign-images/
  ├── a1b2c3.../              ← user_id
  │   ├── ad-uuid-1/
  │   │   ├── 1718000000.png  ← תמונה ראשונה
  │   │   └── 1718001234.png  ← אחרי רענון
  │   └── ad-uuid-2/
  │       └── 1718000000.png
```

**למה timestamp בשם הקובץ?**
רענון תמונה לא מוחק את הקודמת — שומר היסטוריה (ל-debug, לתביעות עתידיות, לאיחוד עתידי). הקובץ הקודם נשאר ב-Storage, ה-`ads.image_url` מצביע לחדש.

**עלות storage:** 5 תמונות פר ad × 3 ads × 100 לקוחות × 1MB ממוצע = ~1.5GB. Free tier של Supabase = 1GB. נצטרך לעבור ל-Pro tier (8GB ב-$25/חודש) או cleanup cron של תמונות ישנות. **ב-MVP לא בעיה דחופה.**

### 1.2 — RLS לפי prefix

ה-policy למעלה משתמשת ב-`storage.foldername(name)[1]` — Supabase helper שמחזיר את החלק הראשון של ה-path. כך RLS אוכף שמשתמש רואה רק קבצים תחת ה-`user_id` שלו.

**שימוש בקליינט:**
```python
# Service: admin client (server-authoritative, §7.3-ב)
supabase_admin.storage \
  .from_('campaign-images') \
  .upload(f'{user_id}/{ad_id}/{timestamp}.png', file_bytes)

# קריאת signed URL לתצוגה ב-frontend (7 days)
signed_url = supabase_admin.storage \
  .from_('campaign-images') \
  .create_signed_url(path, 60 * 60 * 24 * 7)  # 7 days
```

**הערה חשובה:** ה-`ads.image_url` יכיל את ה-signed URL **לא** את ה-storage path. אם רוצים URL טרי — שולפים מ-Storage שוב. זה עובד כל עוד ה-frontend טוען את ה-URL בתוך 7 ימים. **בעיה פוטנציאלית:** אם המשתמש פותח את הקמפיין אחרי 7 ימים, ה-URL פג. **פתרון:** ב-3.4 לפני push ל-Meta, מחדשים את ה-URL. ב-Phase מאוחר נוסיף refresh אוטומטי.

---

## חלק 2 — Endpoints

### 2.1 — `POST /campaigns/:id/generate-images`

יצירת 3 תמונות לראשונה. אסינכרוני.

**זרימה ב-router:**
1. אימות JWT + שליפת campaign דרך RLS.
2. Validation gates:
   - campaign.status = 'draft' → אחרת 409
   - subscription.has_paid_access = true → אחרת 402
   - 3 ads קיימים עם `copy_text` מאוכלס (נוצרו ב-3.2) → אחרת 400 "צור קופי קודם"
3. CAS: לכל ad → `UPDATE ads SET status='generating' WHERE id=? AND status='pending' AND image_url IS NULL`.
   - אם rowcount=0 → תמונה כבר קיימת או generating → דלג שקט.
4. INSERT לתוך jobs queue: `type='generate_ad_images'`, `payload={'campaign_id': ...}`, `user_id=auth.uid()`.
5. Return `202 Accepted` + `{job_id, message: "התמונות בתהליך יצירה"}`.

**שגיאות אפשריות:**
| Status | משמעות | הודעה (עברית) |
|---|---|---|
| 402 | אין billing פעיל | "אמצעי תשלום אינו פעיל. עדכן ב'החבילה שלי'." |
| 400 | קופי חסר | "צור את הקופי קודם — לחץ 'צור קמפיין'." |
| 404 | קמפיין לא קיים / לא של המשתמש | "קמפיין לא נמצא" |
| 409 | קמפיין לא ב-draft | "הקמפיין כבר נדחף, לא ניתן ליצור תמונות חדשות." |
| 500 | שגיאה בתשתית jobs | "אירעה שגיאה. נסה שוב או פנה לתמיכה." |

### 2.2 — `POST /ads/:ad_id/regenerate-image`

רענון תמונה יחידה. סינכרוני (~10-15s).

**זרימה ב-router:**
1. אימות JWT + שליפת ad דרך RLS.
2. Validation:
   - ad קיים ומשתייך למשתמש → אחרת 404
   - ad.copy_text קיים → אחרת 400 "אין קופי לרענן עליו תמונה"
   - campaign.status = 'draft' → אחרת 409
   - subscription.has_paid_access → אחרת 402
3. **לא CAS** — רענון מאפשר דריסה.
4. קריאה ל-`ad_generation_service.generate_image_for_variant(...)` עם הקופי הקיים.
5. שמירה ל-Storage, יצירת signed URL.
6. UPDATE על `ads.image_url`.
7. Return 200 עם ה-ad המעודכן (כולל ה-URL החדש).

**שגיאות:**
| Status | משמעות | הודעה |
|---|---|---|
| 402 | אין billing | "אמצעי תשלום אינו פעיל." |
| 400 | קופי חסר | "אין קופי לרענן עליו תמונה." |
| 404 | ad לא קיים | "פוסט לא נמצא" |
| 409 | קמפיין לא draft | "הקמפיין כבר נדחף." |
| 503 | OpenAI transient | "שירות יצירת התמונות לא זמין. נסה שוב." |
| 500 | OpenAI permanent | "שגיאה ביצירת התמונה. פנה לתמיכה." |

### 2.3 — `GET /jobs/:job_id` (קיים מ-3.0)

ה-frontend עושה polling כל 2 שניות. כשה-job מסיים → ה-frontend מבקש מחדש את ה-ads דרך `GET /campaigns/:id/ads` ורואה את ה-URLs המעודכנים.

---

## חלק 3 — Worker Handler

### 3.1 — `worker/handlers/generate_ad_images.py`

```python
async def handle_generate_ad_images(payload: dict, user_id: UUID) -> None:
    """
    Handler: יוצר 3 תמונות פר וריאציה, שומר ב-Storage,
    מעדכן ads.image_url.

    Idempotency: דלג על ads שכבר יש להן image_url.
    """
    campaign_id = UUID(payload['campaign_id'])

    # 1. שליפת ה-ads דרך admin client (worker עוקף RLS)
    ads = await ads_service.list_by_campaign(campaign_id, user_id)

    # 2. סינון — רק ads בלי תמונה
    ads_to_generate = [ad for ad in ads if ad.image_url is None]

    if not ads_to_generate:
        # idempotency: הכל כבר נעשה
        return

    # 3. שליפת business context (lazy, מ-3.1.6 cache)
    quiz = await quiz_service.get_by_campaign(campaign_id, user_id)
    business_context = await ad_generation_service.get_or_extract_business_context(
        quiz_response_id=quiz.id
    )

    # 4. 3 קריאות מקבילות
    results = await asyncio.gather(
        *[generate_and_save_image(ad, business_context, quiz, user_id) for ad in ads_to_generate],
        return_exceptions=True
    )

    # 5. עיבוד תוצאות — כל ad בנפרד
    for ad, result in zip(ads_to_generate, results):
        if isinstance(result, Exception):
            # כשל פר-ad → status='failed_image' + error
            await ads_service.mark_image_failed(
                ad_id=ad.id,
                error=str(result),
                user_id=user_id
            )
            # log + Sentry על הניסיון האחרון בלבד (תשתית 3.0)
        else:
            # הצלחה — image_url נשמר ב-generate_and_save_image
            # status חוזר ל-'pending'
            pass
```

### 3.2 — `generate_and_save_image` (פונקציית עזר)

```python
async def generate_and_save_image(
    ad: Ad,
    business_context: BusinessContext,
    quiz: QuizResponse,
    user_id: UUID,
) -> None:
    """
    יוצר תמונה אחת ושומר ל-Storage.

    Raises: OpenAIError, StorageError בכשל.
    """
    # 1. קריאה ל-OpenAI (gpt-image-2, Medium, 1024x1024)
    image_bytes = await ad_generation_service.generate_image_for_variant(
        copy_variant=CopyVariant(angle=ad.angle, text=ad.copy_text),
        business_context=business_context,
        tone=quiz.brand_tone,
        # ... שאר פרמטרים
    )

    # 2. שמירה ל-Supabase Storage
    timestamp = int(time.time())
    path = f'{user_id}/{ad.id}/{timestamp}.png'

    await storage_service.upload(
        bucket='campaign-images',
        path=path,
        file_bytes=image_bytes,
        content_type='image/png'
    )

    # 3. signed URL (7 days)
    signed_url = await storage_service.create_signed_url(
        bucket='campaign-images',
        path=path,
        expires_in=60 * 60 * 24 * 7
    )

    # 4. UPDATE ads — image_url + status חוזר ל-pending
    await ads_service.update_image(
        ad_id=ad.id,
        image_url=signed_url,
        user_id=user_id
    )
```

### 3.3 — Idempotency

**שכבה 1: בדיקה לפני ה-handler.** `ads_to_generate = [ad for ad in ads if ad.image_url is None]`. רק תמונות חסרות מיוצרות.

**שכבה 2: CAS על status.** ב-router לפני יצירת ה-job — `UPDATE ads SET status='generating' WHERE status='pending' AND image_url IS NULL`. מבטיח שלא מריצים שני jobs בו-זמנית על אותו ad.

**מקרה הקצה — worker crash אחרי קריאה ל-OpenAI אבל לפני Storage upload:**
- התמונה ב-OpenAI אבודה (URL זמני פג בתוך 60 דקות).
- retry יקרא ל-OpenAI שוב = חיוב כפול ($0.04 פר תמונה).
- **מקובל ב-MVP.** הסתברות נמוכה, עלות זניחה.

**מקרה הקצה — worker crash אחרי Storage upload אבל לפני UPDATE DB:**
- התמונה ב-Storage, אבל `ads.image_url` עדיין NULL.
- retry יקרא שוב ל-OpenAI ויצור תמונה חדשה. הישנה תישאר ב-Storage (orphan).
- **גם מקובל ב-MVP.** cleanup cron של orphans יכול להיכנס Post-MVP.

---

## חלק 4 — שינויים ב-DB

### 4.1 — אין צורך במיגרציה ל-`ads`

הסכמה של 3.2 כבר תומכת:
- `image_url text` (nullable)
- `status` עם CHECK כולל `'pending'`, `'generating'`, `'live'`, `'rejected'`, `'failed_push'`

**אבל צריך להוסיף ערך:** `'failed_image'` — כשל ספציפי לכשל בtumonih.

### 4.2 — מיגרציה 0022 (הרחבת status enum)

```sql
-- 0022_ads_failed_image_status.sql

BEGIN;

-- הרחבת ה-CHECK constraint של ads.status
ALTER TABLE public.ads DROP CONSTRAINT ads_status_check;

ALTER TABLE public.ads ADD CONSTRAINT ads_status_check
  CHECK (status IN (
    'pending',
    'generating',
    'live',
    'rejected',
    'failed_push',
    'failed_image'  -- ← חדש: כשל ביצירת תמונה
  ));

-- עמודה ל-error message (אופציונלית, אם עוד לא קיימת)
ALTER TABLE public.ads
  ADD COLUMN IF NOT EXISTS image_error text;

COMMENT ON COLUMN public.ads.image_error IS
  'הודעת שגיאה כשיצירת התמונה נכשלה. NULL במצב תקין.';

COMMIT;
```

**הערה:** `failed_image` הוא **לא טרמינלי** — המשתמש יכול לעשות regenerate-image, שיחזיר את ה-status ל-`pending` עם `image_url` חדש.

### 4.3 — ביטוי `failed_image` ב-3.4

ב-Validation gates של 3.4 (חלק ג2): gate 6 ("כל 3 ads עם `image_url`") מתפרש כעת כ-"כל 3 ads עם `image_url IS NOT NULL`". `status='failed_image'` יכשיל את ה-gate הזה אם המשתמש לחץ "שגר" בלי לרענן.

**אסטרטגיית UX:** ה-frontend מציג בכרטיס של ad עם `status='failed_image'` הודעה: "❌ יצירת התמונה נכשלה. לחץ 🔄 רענן תמונה." וכפתור "שגר" disabled עד שכל 3 הצליחו.

---

## חלק 5 — שינויים ב-`ad_generation_service`

### 5.1 — `generate_image_for_variant` כבר קיים מ-3.1.5

הפונקציה הציבורית כבר נכתבה ב-3.1.5. מה שנשאר ב-3.3:
- וידוא שהיא מקבלת **bytes**, לא URL (להעלות ל-Storage).
- אם היא היום מחזירה URL — צריך לעטוף ב-fetch של ה-bytes.

**שינוי קטן ב-`integrations/openai.py`:**
```python
async def generate_image(prompt: str, size: str = '1024x1024', quality: str = 'medium') -> bytes:
    """
    Returns: bytes of the generated PNG image.

    Note: gpt-image-2 מחזיר URL זמני. אנחנו מורידים מיד לbytes
    כדי לא להישען על URL זמני שיפוג.
    """
    response = await openai_client.images.generate(
        model='gpt-image-2',
        prompt=prompt,
        size=size,
        quality=quality,
        n=1,
    )

    # gpt-image-2 returns URL — fetch immediately
    image_url = response.data[0].url
    async with httpx.AsyncClient(timeout=30.0) as client:
        image_response = await client.get(image_url)
        image_response.raise_for_status()
        return image_response.content
```

**הערה:** ה-API של gpt-image-2 משתנה לפעמים. ב-OpenAI יש גם אפשרות לקבל `b64_json` ישירות (`response_format='b64_json'`) — חוסך קריאה נוספת. **לאמת בזמן המימוש** מה הדפוס הנכון. הלוגיקה זהה — מה שחשוב הוא ש-`generate_image` מחזירה bytes.

### 5.2 — `storage_service` חדש

קובץ חדש: `app/services/storage_service.py`

```python
class StorageService:
    """
    עטיפה ל-Supabase Storage. כל הקריאות דרך admin client.
    מבודד מ-services אחרים — אם נחליף ל-R2/S3, שינוי פה בלבד.
    """

    async def upload(
        self,
        bucket: str,
        path: str,
        file_bytes: bytes,
        content_type: str,
    ) -> None:
        """העלאה. raise StorageError בכשל."""

    async def create_signed_url(
        self,
        bucket: str,
        path: str,
        expires_in: int,
    ) -> str:
        """signed URL לתצוגה ב-frontend."""

    async def delete(self, bucket: str, path: str) -> None:
        """מחיקה (לעתיד — cleanup cron)."""
```

מבודד למודול אחד לפי חוק שכבות (spec §2א חוק 3).

---

## חלק 6 — Frontend Integration (תיעוד בלבד, לא קוד)

ה-frontend לא נבנה ב-3.3, אבל תיעוד הזרימה הצפויה:

### 6.1 — זרימת יצירה ראשונה

```
1. משתמש מסיים את האשף (3.1 + 3.1ב) → לוחץ "צור קמפיין"
2. Frontend: POST /campaigns/:id/generate-copy (sync, ~3s)
   → מקבל 3 ads עם copy
3. Frontend: מציג 3 כרטיסים עם copy + spinner על התמונה
4. Frontend: POST /campaigns/:id/generate-images
   → מקבל job_id
5. Frontend: polling כל 2s על GET /jobs/:job_id
6. כשה-job done:
   - GET /campaigns/:id/ads → מקבל 3 ads עם image_url
   - מציג את התמונות בכרטיסים
7. משתמש סוקר → לוחץ "אשר ושגר" פר-ad (3 לחיצות)
8. אחרי 3 האישורים: POST /campaigns/:id/push (3.4)
```

### 6.2 — זרימת רענון תמונה יחידה

```
1. משתמש לוחץ "🔄 רענן תמונה" על ad ספציפי
2. Frontend: POST /ads/:ad_id/regenerate-image
3. Spinner על אותה כרטיס בלבד (~10-15s)
4. Response: ad מעודכן → החלפת התמונה ב-UI
```

### 6.3 — טיפול בכשל פר-ad

```
GET /campaigns/:id/ads מחזיר:
[
  {id: 'a1', status: 'pending',       image_url: 'https://...', copy_text: '...'},
  {id: 'a2', status: 'failed_image',  image_url: null,          copy_text: '...', image_error: '...'},
  {id: 'a3', status: 'pending',       image_url: 'https://...', copy_text: '...'}
]

Frontend:
- a1, a3: כרטיס תקין עם תמונה
- a2: כרטיס עם "❌ יצירת התמונה נכשלה" + כפתור "🔄 רענן"
- כפתור "שגר את הקמפיין": disabled עד שכל 3 עם image_url
```

---

## חלק 7 — Env vars

| שם | משמעות | optional? |
|---|---|---|
| `OPENAI_IMAGE_MODEL` | `gpt-image-2` (default) | optional, יש default |
| `OPENAI_IMAGE_SIZE` | `1024x1024` (default) | optional |
| `OPENAI_IMAGE_QUALITY` | `medium` (default) | optional |
| `STORAGE_BUCKET_CAMPAIGN_IMAGES` | `campaign-images` (default) | optional |
| `STORAGE_SIGNED_URL_EXPIRES_DAYS` | `7` (default) | optional |

`OPENAI_API_KEY` כבר קיים מ-3.1.5.
`SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` כבר קיימים מ-0.1.

---

## חלק 8 — שמות הקבצים החדשים

| קובץ | תוכן |
|---|---|
| `routers/ads.py` | (קיים מ-3.2) הרחבה: `POST /ads/:ad_id/regenerate-image` |
| `routers/campaigns.py` | (קיים) הרחבה: `POST /campaigns/:id/generate-images` |
| `services/ad_generation_service.py` | (קיים) הרחבה: `generate_image_for_variant` מקבלת bytes |
| `services/storage_service.py` | **חדש** — עטיפה ל-Supabase Storage |
| `services/ads_service.py` | (קיים מ-3.2) הרחבה: `mark_image_failed`, `update_image` |
| `worker/handlers/generate_ad_images.py` | **חדש** — handler |
| `integrations/openai.py` | (קיים) הרחבה: `generate_image` מחזירה bytes |
| `supabase/migrations/0021_campaign_images_bucket.sql` | bucket + RLS |
| `supabase/migrations/0022_ads_failed_image_status.sql` | הוספת `failed_image` ל-CHECK |

---

## חלק 9 — Done של 3.3

- bucket `campaign-images` קיים ב-Supabase Storage עם RLS.
- מיגרציה 0022 הוסיפה `failed_image` ל-`ads.status`, ו-`image_error` עמודה.
- `POST /campaigns/:id/generate-images` יוצר job אסינכרוני, מחזיר 202 + job_id.
- worker מבצע 3 קריאות מקבילות ל-gpt-image-2, שומר ב-Storage, מעדכן `ads.image_url`.
- כשל פר-תמונה → `ads.status='failed_image'` + `image_error`, השאר ממשיכות.
- `POST /ads/:ad_id/regenerate-image` סינכרוני (~10-15s), דורס את התמונה הקודמת.
- Idempotency: handler מדלג על ads שכבר עם `image_url`.
- frontend עושה polling על job_status ומתעדכן.
- כל הטסטים החדשים עוברים.

---

## חלק 10 — לא ב-3.3

- העלאת תמונות מהמשתמש (Vision/editing) — **Phase 3.5**.
- עריכת `image_concept` ידנית ע"י משתמש — לא ב-MVP.
- איכות High (only Medium) — אופציה עתידית.
- יחסים אחרים מ-1:1 — Post-MVP.
- העלאה ל-Meta — **Phase 3.4**.
- Cleanup cron של תמונות ישנות/orphans — Post-MVP.
- Refresh אוטומטי של signed URLs שפגו — Post-MVP.
- Backup ל-R2/S3 — Post-MVP.

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **gpt-image-2 API משתנה — לאמת בזמן המימוש.** ייתכן שצריך `response_format='b64_json'` במקום URL fetch. השאלה הזו פתוחה עד שננסה.

2. **`asyncio.gather` עם `return_exceptions=True` חובה.** בלי זה — אם תמונה אחת נכשלת, כל ה-`gather` נופל ושתי תמונות מבוטלות.

3. **`StorageService` עוטף את ה-SDK של Supabase.** אסור שמודולים אחרים יקראו ישירות ל-`supabase.storage.from_(...)`. אותו עיקרון כמו שאר ה-integrations.

4. **`signed_url` עם expires_in של 7 ימים.** אם המשתמש פותח קמפיין אחרי 7 ימים — URL פג. ב-3.4 לפני push ל-Meta נצטרך לחדש (זה אחד הצעדים ב-3.4).

5. **`upload` ל-Storage עם `content_type='image/png'`.** gpt-image-2 מחזיר PNG. אם נשנה למודל אחר, לאמת.

6. **CC חייב להוסיף ל-`integrations/openai.py`** את הפונקציה `generate_image(prompt, size, quality) -> bytes`. השם תואם, החתימה ברורה.

7. **rate limit של OpenAI על gpt-image-2:** ~50 תמונות/דקה ב-tier 1. ב-MVP זה לא בעיה (3 תמונות פר משתמש). אם נראה rate limits → לעבור לbackoff פנימי (תשתית 3.0).

8. **טיפול ב-content policy violation:** OpenAI מחזירה שגיאה ייחודית כשתמונה נדחית מטעמי תוכן. הסיווג ב-`integrations/openai.classify_openai_error` (יורחב ב-3.3 אם עוד לא קיים) מחזיר `permanent` עם error code → ה-ad מקבל `failed_image` עם הודעה ברורה: "התוכן שלך נדחה ע"י המודל. נסה לרענן או לערוך את הקופי."

9. **timeout על generate_image:** 30 שניות. אם OpenAI איטית מזה — כשל ועובר ל-retry של תשתית 3.0.

10. **רענון תמונה לא יוצר job חדש.** סינכרוני, אותו דפוס כמו רענון קופי ב-3.2. עקבי.


### Session 3.5 — עריכת/שיפור תמונות
- [ ] רענון תמונה (יצירה חוזרת מ-gpt-image-2)
- [ ] עריכה/שיפור של תמונה שהועלתה (Vision/editing) → Storage
- [ ] רץ כ-job (כמו 3.3)

**Done:** משתמש יכול לרענן תמונה שנוצרה, או להעלות תמונה ולשפר אותה.

### Session 3.4 — העלאת מודעות ל-Meta — כ-job, אטומי ✅
- [x] handler שדוחף Campaign + Ad Set (תקציב + טרגוט) + 3 מודעות
- [x] **אטומיות:** state machine שמתעד באיזה שלב נמצאים. כשל באמצע
      (למשל מודעה 3 נדחתה) → cleanup/rollback, לא להשאיר קמפיין חצי-בנוי
- [x] שמירת `meta_campaign_id`, `meta_ad_id`, עדכון status + `attempt_error`
- [x] דחיפת הקמפיין כוללת יצירת Lead Form מהשדות שנשמרו ב-3.1ב (בקמפיין לידים)
- [x] טרגוט ה-Ad Set כולל טווח גיל + מגדר מבחירת הלקוח (לא רק גיאוגרפי). הקהלים המפורטים נשארים רחבים/אוטומטיים
- [x] ה-status enum כולל `live`/`failed`/`failed_rollback_pending`; דחייה מאוחרת (webhook) **נדחתה** ל-Phase 8 — `live` אינו טרמינלי
- [x] **כאן ה-MVP המינימלי הוכח** — מודעות עלו לאוויר

**Done:** קמפיין אמיתי עם 3 מודעות חי ב-Meta Ads Manager. כשל חלקי לא
משאיר זבל ב-Meta.
**זהירות (CLAUDE.md):** state-machine patterns. זה החלק הכי מסוכן — טעות
באטומיות עולה ביוקר (קמפיין חצי-בנוי = כסף שמתבזבז אצל הלקוח). עיגון:
"מודעה שעלתה יכולה עוד להידחות" — `live` אינו טרמינלי.

**הושלם ב-3 PRs — הערות יישום (סטיות מהתכנון):**
- **3.4a (תשתית):** מעבר מלא של `integrations/meta.py` ל-**facebook-business SDK v25** (לא
  רק כתיבה — גם reads; refactor של ה-httpx הקיים). **חריג מתועד:** OAuth token exchange נשאר
  httpx (ה-SDK לא מכסה auth) — `docs/decisions/0001-meta-sdk.md`. wrapper per-call
  (thread-safe). migrations **0024+0025** (לא 0013 — היינו כבר ב-0023).
- **3.4b (orchestration):** `campaign_push_service` (8 gates, CAS, 6 שלבים, build),
  `meta_rollback_service`, endpoints (POST push + GET campaign). ה-handler **לא זורק** על
  כשל permanent (rollback + campaign='failed') כדי שה-runner לא יעשה retry; ה-frontend בודק
  `campaigns.status` (live/failed) דרך GET — לא רק job status.
- **3.4c (חוסן):** cleanup cron כ-**tick ב-runner** (כל 5 דק', לא Render cron — אין כזה).
  זיהוי תקועים לפי **DB state** (meta_*_id נשמר מיד → proxy ל-Meta; לא קריאת Meta-truth —
  פישוט מ-ROADMAP). token פג (code 190) → **fail_user** "חבר מחדש" + `mark_token_expired`
  (refresh מלא נשאר ל-Phase 6, לא הוקדם).
- **geo:** location (שם עיר) → Meta key דרך `adgeolocation` search; ריק → country IL.
- **לאמת מול Meta אמיתי (staging):** ערכי objective/optimization_goal/CTA/lead-form
  question-types ב-payload (best-effort לפי v25), ו-migration 0023 (storage) מ-3.3.

# Session 3.4 — חלקים א + ב + ג1: תשתית, state machine, flow

## תיאור Session 3.4

**Push קמפיין ל-Meta.** השלב הכי מסוכן בפרויקט — פעולה ארוכה, אטומית,
אוטונומית, עם כסף לקוח אמיתי. כשל באמצע = קמפיין שבור ב-Meta או חיובים
מיותרים. כל ההחלטות בSession הזה הן הגנות נגד התרחיש הזה.

**מה השלב עושה ברמת המשתמש:** הלקוח לחץ "שגר את הקמפיין" אחרי שאישר 3
הוריאציות. אנחנו יוצרים ב-Meta: Campaign (1) + Ad Set (1) + Lead Form
(1, אם `campaigns.type='lead'`) + 3 Ads. הכל מקושר זה לזה, הכל אטומי
(או הצלחה מלאה או rollback מלא), והמשתמש רואה התקדמות בזמן אמת.

---

## חלקים א + ב + ג1 — סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | Meta SDK או Graph API ידני? | `facebook-business-sdk-python` הרשמי |
| 2 | Idempotency על אילו entities? | כל ה-`meta_*_id` נשמרים ב-DB **תוך כדי** ה-flow |
| 3 | כמה validation gates לפני push? | **7 ראשוניים** (ב-ג2 יורחב ל-8 עם Reach Estimate) |
| 4 | האם להוסיף state ביניים `pushing`? | **כן** — CAS protection + UX |
| 5 | להפריד `rejected` (תוכן) מ-`failed_push` (טכני)? | **כן** — טיפול שונה לכל אחד |
| 6 | DELETE Campaign מ-Meta בכשל? | **כן** — rollback אטומי |
| 7 | מה אם ה-DELETE עצמו נכשל? | Sentry + `failed_cleanup` (קיים מ-2.6.1) → טיפול ידני, **לא retry אוטומטי** |
| 8 | מודל idempotency — טבלה נפרדת או עמודות? | **עמודות ב-`campaigns`** (פשוט יותר מ-2.6.1) |
| 9 | סדר קריאות Meta | Lead Form → Campaign → Ad Set → 3 Ads (סדרתי) |
| 10 | 3 Ads — סדרתי או מקבילי | **סדרתי, ב-job אסינכרוני עם frontend polling** |
| 11 | Rate limiting | Exponential backoff בתוך פעולה + jobs queue כפיזור בין קמפיינים |
| 12 | Validation gates — איפה? | **dual layer** — pre-check ב-endpoint + מלא ב-handler |

---

## חלק א — תשתית טכנית

### א.1 — Meta SDK

**שימוש ב-`facebook-business-sdk-python`** (הרשמי של Meta). לא Graph API
ידני.

**סיבות:**
- Type-safe — model classes ל-Campaign, AdSet, Ad וכו'.
- Pagination אוטומטי בקריאות שמחזירות רשימות.
- טיפול אוטומטי ב-rate limiting headers (`X-Business-Use-Case-Usage`).
- Meta מתעדת אותו ושומרת על תאימות עם גרסאות API חדשות.

**חיסרון:** SDK יכול להיות "thicker" מהדרוש לנו. אם בעתיד נגלה שהוא מקור
לבעיות (למשל גרסה ישנה שלא תומכת ב-feature חדש) — אפשר להחליף בקריאות
HTTP ישירות. בידוד מלא ב-`integrations/meta.py` (חוק שכבות מ-§2א).

**Env var חדש:** `META_API_VERSION` (default: גרסה יציבה אחרונה — לאמת
בזמן המימוש; כיום `v21.0` נראית סבירה אבל עלולה להתעדכן).

### א.2 — Idempotency: שמירת `meta_*_id` תוך כדי flow

**כלל קשיח:** בכל שלב שיוצר entity ב-Meta, ה-ID נשמר ב-DB **לפני**
המעבר לשלב הבא.

**למה זה חשוב:** אם ה-worker קורס באמצע (deploy, OOM, panic), אנחנו
חייבים לדעת מה כבר נוצר ב-Meta כדי לעשות rollback. בלי שמירה ב-DB,
יהיה לנו "צאצא יתום" ב-Meta בלי שום דרך להגיע אליו.

**זרימה לדוגמה:**
```
1. POST /campaigns → meta_campaign_id = "123" ← UPDATE DB מיד
2. POST /adsets    → meta_ad_set_id   = "456" ← UPDATE DB מיד
3. POST /ads (#1)  → meta_ad_ids[0]   = "789" ← UPDATE DB מיד
   [worker crash כאן]
```

ה-cleanup cron (פירוט ב-ג2) ימצא את הקמפיין `pushing`, יראה שיש לו
`meta_campaign_id` עם Ad Set + 1 Ad, ויבצע rollback של מה שכבר נוצר.

### א.3 — 7 Validation gates (ב-ג2 יורחב ל-8)

לפני שמתחילים את ה-push (לפני CAS `draft → pushing`), בודקים:

1. `subscription.has_paid_access = true` — מ-2.6.1 redesign. גישה
   לתשלום קיימת.
2. `fb_connections` קיים + טוקן לא פג (`token_expires_at > now()`).
3. `campaigns.status = 'draft'` — לא ב-`pushing` או `live` כבר.
4. `quiz_responses` קיים ומלא (כל השדות החובה).
5. כל 3 `ads` קיימים עם `copy_text` לא ריק.
6. כל 3 `ads` עם `image_url` לא ריק (נוצר ב-3.3).
7. אם `campaigns.type = 'lead'` → `lead_form_fields` קיים (מ-3.1ב).

**Gate 8 (Reach Estimate)** נוסף בחלק ג2 — ראה את הבלוק של ג2.

**כל gate שנכשל → מחזיר שגיאה ספציפית למשתמש בעברית, לפני ה-push.**

### א.4 — Idempotency כעמודות ב-`campaigns`, לא טבלה נפרדת

ב-2.6.1 השתמשנו בטבלה נפרדת (`billing_charge_attempts`) ל-idempotency
של חיובים. **ב-3.4 אנחנו עושים אחרת:** העמודות `meta_*_id` יושבות
ישירות על `campaigns`.

**הסיבה:** קמפיין יחיד נדחף **פעם אחת בחיים שלו** (אם נכשל = rollback
מלא ו-draft חדש). אין צורך בהיסטוריה של ניסיונות — רק במצב הנוכחי.
טבלת attempts נפרדת מיותרת.

**אם בעתיד נרצה היסטוריה** (למשל אנליטיקס: "כמה פעמים קמפיינים נכשלים
בשלב 4?") — אפשר להוסיף טבלה `campaign_push_attempts` ב-Phase מאוחר.

---

## חלק ב — State machine

### ב.1 — State `pushing` כביניים

**מטרה:** בזמן שה-handler רץ, הקמפיין במצב מובחן. זה מאפשר:

- **CAS protection** — `UPDATE campaigns SET status='pushing' WHERE id=? AND status='draft'`. אם המשתמש לוחץ "שגר" פעמיים במהירות, הניסיון השני יחזור עם 0 rows updated → 409 Conflict.
- **UX מובחן** — frontend יודע להציג "מעלה לפייסבוק..." spinner כשהמצב `pushing`.
- **Cleanup detection** — cron יודע למצוא קמפיינים תקועים (`status='pushing' AND attempt_started_at < now() - interval '5 minutes'`).

### ב.2 — `rejected` vs `failed_push` ב-`ads.status`

שני סוגי כשל שצריך להבדיל ביניהם **ברמת ה-ad הספציפי** (לא הקמפיין):

| status | משמעות | טיפול |
|---|---|---|
| `rejected` | Meta דחתה את התוכן (פוליסי, טקסט בתמונה) | Phase 8 — הסוכן מנסח מחדש ושולח ללקוח לאישור |
| `failed_push` | כשל טכני (timeout, 5xx) | אין פעולה — הקמפיין כולו ב-`failed`, המשתמש ינסה שוב |

**ב-3.4 שני המצבים האלה רק נכתבים** — `rejected` בעיקר אם Meta מחזירה
content rejection בזמן ה-push, `failed_push` אם הכשל היה טכני.
**הטיפול ב-`rejected` (ניסוח מחדש) מתבצע ב-Phase 8**, לא ב-3.4.

### ב.3 — DELETE Campaign כ-rollback אטומי

כשמשהו נכשל באמצע ה-push, הכלל הוא: **למחוק את כל מה שכבר נוצר ב-Meta**,
אז לחזור ל-`status='draft'`. המשתמש יכול לנסות שוב מהתחלה.

**רכיב מרכזי:** ב-Meta, DELETE על Campaign מוחק **אוטומטית** את כל
Children שלו (Ad Set + Ads). לא צריך לעבור ידנית. רק Lead Form נפרד
מהיררכיה — דורש DELETE נפרד.

**סדר חובה ב-rollback:**
1. DELETE Campaign (אם קיים) → Ad Set + Ads נמחקים אוטו.
2. DELETE Lead Form (אם קיים, ורק אם `type='lead'`).

הסדר חשוב כי Lead Form עם Ad פעיל שמצביע אליו לא נמחק (FK בצד Meta).
DELETE Campaign קודם → ה-Ads מתים → ה-Lead Form כבר לא קשור → ניתן
למחוק.

### ב.4 — `failed_rollback_pending` כמצב חריג

אם ה-rollback עצמו נכשל (DELETE Campaign החזיר שגיאה לא חולפת), הקמפיין
לא יכול לחזור ל-`draft` — יש לו entity יתום ב-Meta. במצב הזה:

- `campaigns.status = 'failed_rollback_pending'`
- שורה ב-`failed_cleanup` (טבלה קיימת מ-2.6.1).
- Sentry alert.
- **לא retry אוטומטי** — אמיר/הצוות יטפלו ידנית.

זה מצב נדיר. הסיבה: גם DELETE על entity לא קיים ב-Meta מחזיר 404 שאנחנו
מתייחסים אליו כהצלחה. אז `failed_rollback_pending` קורה רק אם יש בעיה
מתמשכת מול Meta (חשבון הושעה באמצע, רשת מנותקת, וכו').

### ב.5 — State diagram

```
campaigns.status:
    draft ──CAS──→ pushing ──┬──→ live                  (כל 6 השלבים)
                             ├──→ failed                (rollback הצליח)
                             └──→ failed_rollback_pending (rollback נכשל)

ads.status:
    pending ──→ live                  (אחרי push מוצלח)
            ──→ rejected              (Meta דחתה תוכן, → Phase 8)
            ──→ failed_push           (כשל טכני)
```

---

## חלק ג1 — Flow ביצוע

### ג1.1 — סדר קריאות ל-Meta

```
1. Lead Form     (אם type='lead')
2. Campaign
3. Ad Set
4. Ad #1
5. Ad #2
6. Ad #3
```

**6 שלבים סדרתיים.** Lead Form מדולג בקמפיין `whatsapp`.

**סדר 3 ה-Ads דטרמיניסטי** (לפי angle): `emotional` → `pain_solution`
→ `result_success`. בלי סיבה מיוחדת מעבר ל-consistency.

### ג1.2 — 3 ה-Ads סדרתיים (לא מקביליים)

זמן צפוי במצב רגיל:
- כל שלב ~2 שניות מול Meta.
- 6 שלבים × 2 שניות = **~12 שניות סה"כ**.

**למה סדרתי ולא מקבילי על 3 ה-Ads:**

| גורם | סדרתי | מקבילי |
|---|---|---|
| זמן רגיל | 12s | 8s (חיסכון 4s) |
| מורכבות קוד | פשוט | `asyncio.gather` + `return_exceptions=True` |
| Debug במקרה כשל | קל (יודעים בדיוק איפה) | קשה (3 כשלים מקבילים) |
| fail-safe | אם Ad #2 נכשל, Ad #3 לא יתחיל | Ad #3 ימשיך ויצור entity יתום |
| Rate limit pressure | קל | גבוה יותר (3 קריאות במכה) |

**חיסכון של 4 שניות לא קריטי** — המשתמש כבר נטש את ה-UI ועושה
polling. ההחלטה לעדיף פשטות וודאות.

**אם בעתיד נרצה לאופטם** — אפשר להוסיף `asyncio.gather` רק בשלבים 4-6
בלי לשנות את שאר המבנה. עמודת `meta_ad_ids` כ-array תומכת בזה.

### ג1.3 — Job אסינכרוני + frontend polling

```
1. POST /campaigns/:id/push
   → 7 validation gates (8 בחלק ג2)
   → CAS: status 'draft' → 'pushing', שמור attempt_started_at = now()
   → INSERT לתוך jobs queue: type='push_campaign_to_meta', payload={campaign_id}
   → מחזיר 202 + job_id

2. Frontend מקבל job_id, מציג "מעלה לפייסבוק..." spinner

3. Worker מעבד את ה-job:
   - Create Lead Form (~2s, אם lead campaign) → UPDATE meta_lead_form_id
   - Create Campaign (~2s)                   → UPDATE meta_campaign_id
   - Create Ad Set (~2s)                     → UPDATE meta_ad_set_id
   - Create Ad #1 (~2s)                      → APPEND meta_ad_ids
   - Create Ad #2 (~2s)                      → APPEND meta_ad_ids
   - Create Ad #3 (~2s)                      → APPEND meta_ad_ids
   - סה"כ ~12s

4. במקרה הצלחה: UPDATE campaigns.status = 'live'

5. במקרה כשל (פרטים מלאים ב-ג2):
   - rollback: DELETE Campaign + DELETE Lead Form (אם lead)
   - הצליח rollback → status = 'failed', attempt_error = {...}
   - נכשל rollback → status = 'failed_rollback_pending' + Sentry + failed_cleanup

6. Frontend עושה polling כל 2s ל-GET /jobs/:job_id:
   - job_status = 'done' → טוסט הצלחה, ניווט למסך הבית במצב "קמפיין באוויר"
   - job_status = 'failed' → הודעת שגיאה עם attempt_error.message + כפתור "נסה שוב"
```

**למה polling ולא WebSocket/SSE:**
- 12 שניות זה לא הרבה — `setInterval(2s)` = 6 קריאות בסה"כ.
- WebSocket דורש תשתית נוספת ב-Render שלא קיימת היום.
- ב-MVP אנחנו לא רוצים להוסיף שכבת תקשורת חדשה.

### ג1.4 — Rate limiting: שתי שכבות הגנה

**שכבה 1: Exponential backoff בתוך פעולה.** פירוט בחלק ג2 — 3 retries
על transient errors.

**שכבה 2: Jobs queue כפיזור בין קמפיינים.** ה-worker מעבד job אחד בכל
פעם (sequential). אם 10 לקוחות לחצו "שגר" בו-זמנית, הם יעובדו בתור,
לא במקביל. זה מבטל לחץ rate limiting מקביל על Meta API.

**ב-MVP זה מספיק.** אם בעתיד נראה שלקוחות מחכים יותר מדי (תור ארוך) —
נוסיף worker שני ב-Render (אותו repo, sequential per-worker). אבל לא
ב-MVP.

### ג1.5 — Validation gates: dual layer (endpoint + handler)

ה-7 (ועכשיו 8) gates נבדקים **בשני מקומות:**

**ראשון: ב-`POST /campaigns/:id/push` (לפני CAS).**
- אם gate נכשל → 400 Bad Request עם הודעה ספציפית.
- היתרון: המשתמש מקבל feedback מיידי בלי לחכות ל-job.
- אבל: בין הבדיקה ל-CAS, משהו יכול להשתנות (race condition נדיר).

**שני: ב-handler של ה-job (לפני שמתחילים את 6 השלבים).**
- אם gate נכשל → ה-job נכשל מיידית בלי קריאות ל-Meta.
- `status` חוזר ל-`draft`, `attempt_error` נשמר.
- היתרון: דאבל-צ'ק נגד race conditions.

**זה לא duplicate code** — הלוגיקה ב-service אחד (`campaign_push_service.validate_gates()`), והקריאה ממקומיים — endpoint ו-handler.

---

## חלק ג1.6 — Done של ג1 (תשתית מוכנה)

- ה-state machine של `campaigns.status` מורחב עם `pushing` (ב-CHECK
  constraint שלFinal יתבצע ב-migration 0013 של ג2).
- `POST /campaigns/:id/push` בנוי: validation gates → CAS → job creation
  → 202.
- `GET /jobs/:job_id` בנוי (מ-3.0): מחזיר status + result.
- `worker/handlers/push_campaign.py` בנוי עם 6 שלבים סדרתיים.
- כל שמירת `meta_*_id` ב-DB מתבצעת **מיד** אחרי ה-API call המוצלח.
- `integrations/meta.py` הורחב עם פונקציות חדשות: `create_lead_form`,
  `create_campaign`, `create_ad_set`, `upload_image`, `create_ad`,
  `delete_campaign`, `delete_lead_form`.

**מה שלא בג1:** error handling פר-שלב, retry policies, cleanup cron,
migration מלאה. הכל ב-ג2.

---

## חלק ג1.7 — לא ב-ג1 (יטופל ב-ג2)

- מטריצת error handling פר-שלב (איזה כשל → איזו קטגוריה).
- מנגנון retry עם backoff על transient.
- Refresh token בכשל code 190.
- Cleanup cron לקמפיינים תקועים.
- DB migration 0013 עם העמודות החדשות.

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`facebook-business-sdk-python` דורש Python 3.8+** — אנחנו על 3.11
   ב-Render, אין בעיה.

2. **`META_API_VERSION` כ-env var, לא hardcoded.** Meta מעדכנת גרסאות
   כל 3 חודשים, וגרסאות ישנות נסגרות אחרי ~2 שנים. רוצים להחליף בלי
   deploy.

3. **`facebook-business-sdk-python` משתמש ב-`requests` הסינכרוני
   internally.** ב-`worker/handlers/push_campaign.py` שרץ כ-async, יש
   לעטוף קריאות SDK ב-`asyncio.to_thread()` או דומה. **שווה לבדוק
   בזמן המימוש** אם זה pattern שכבר בשימוש ב-`integrations/meta.py`
   הקיים מ-1.2/2.1.

4. **CAS צריך להיות עם RETURNING.** `UPDATE campaigns SET
   status='pushing', attempt_started_at=now() WHERE id=? AND
   status='draft' RETURNING id`. אם החזיר 0 שורות → 409 Conflict (כבר
   ב-pushing/live/וכו'). **לא לסמוך על `isinstance(response, list)`**
   — להשתמש ב-`postgrest_helpers` שנוצרו ב-2.6.1.

5. **Lead Form נוצר על Page, לא על Ad Account.** הקריאה היא
   `POST /{page_id}/leadgen_forms`, לא `POST /act_{account_id}/...`.
   ה-`page_id` נשמר ב-`campaigns.fb_page_id` מ-2.2.


# Session 3.4 — חלק ג2: Error handling, Retry, Cleanup, Migration

## חלק ג2 — סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 13 | resume מנקודת הכשל או restart מלא? | **restart מלא** — כל כשל = rollback + חזרה ל-`draft` |
| 14 | retry אוטומטי או ידני? | **היברידי** — transient = אוטומטי, permanent = ידני |
| 15 | Privacy Policy URL בLead Form | **שילוב** — default לשלנו, אופציונלי ללקוח להזין משלו |
| 16 | קונבנציית שם הקמפיין ב-Meta | `{service_name} — {DD-MM-YYYY}` (סדר ישראלי) |
| 17 | יצירת entities — ACTIVE או PAUSED? | **ACTIVE מההתחלה** — אין שלב Activate נפרד |
| 18 | סדר 3 ה-Ads | דטרמיניסטי לפי angle: `emotional` → `pain_solution` → `result_success` |
| 19 | regenerate אוטומטי לתוכן AI שנדחה? | **לא ב-MVP** — rollback ומסירה למשתמש |
| 20 | Reach Estimate ב-validation gates? | **כן** — gate שמיני, חוסם push לפני שהוא מתחיל |
| 21 | מבנה retry על transient | 3 ניסיונות, backoff 2s/5s |
| 22 | token expired — מה לעשות? | refresh ואז ניסיון יחיד נוסף, אחרת fail |
| 23 | 429 — טיפול מיוחד? | **לא ב-MVP** — transient רגיל. circuit breaker עתידי |
| 24 | cleanup cron — איזה תנאי? | `pushing` יותר מ-5 דקות |
| 25 | מה לעשות עם קמפיין תקוע? | **בדיקת אמת מול Meta** — לא להניח, לבדוק לפי `meta_*_id` |
| 26 | תקרת ניסיונות cleanup? | 5 ניסיונות, אחר כך `failed_rollback_pending` |
| 27 | `attempt_error` — JSONB או טבלה נפרדת? | **JSONB ב-MVP** — היסטוריה ארוכה Post-MVP |

---

## תשתית: state machine סופי

```
campaigns.status:
    draft → pushing → live                          (כל 6 השלבים הצליחו)
                    → failed                         (rollback מלא הצליח)
                    → failed_rollback_pending        (rollback עצמו נכשל)

ads.status:  (לא משתנה מ-2.2)
    pending → live           (אחרי push מוצלח)
            → rejected       (Meta דחתה תוכן, Phase 8)
            → failed_push    (כשל טכני)
```

**זרימת השלבים (6 שלבים סדרתיים):**

```
[Phase: validation gates × 8 — לפני CAS]
    ↓
[CAS: draft → pushing, שמור attempt_started_at]
    ↓
1. Lead Form           (אם type='lead')
2. Campaign            (תמיד, ACTIVE)
3. Ad Set              (תמיד, ACTIVE)
4. Ad #1 (emotional)
5. Ad #2 (pain_solution)
6. Ad #3 (result_success)
    ↓
[UPDATE: pushing → live]
```

כל שלב כולל: העלאת תמונה ל-Meta (אם רלוונטי) → יצירת ה-entity → שמירת ה-`meta_*_id` ב-DB **מיד** (לפני המעבר לשלב הבא).

---

## חלק ג2.1 — Validation gates (מורחב מ-7 ל-8)

ה-7 gates המקוריות מחלק ג1 + gate חדש:

1. `subscription.has_paid_access = true`
2. `fb_connections` valid + non-expired token
3. `campaigns.status = 'draft'`
4. `quiz_responses` exists + complete
5. כל 3 `ads` עם `copy_text`
6. כל 3 `ads` עם `image_url`
7. אם `type='lead'` → `lead_form_fields` exists
8. **חדש:** Reach Estimate מ-Meta ≥ 1500 לטרגוט המוצע

**Gate 8 — איך מתבצע:**

קריאה ל-`POST /act_{ad_account_id}/reachestimate` עם הטרגוט המתוכנן (geo + age + gender). זה endpoint זול ומהיר (< 1s). אם החזיר estimate < 1500 → חוסמים את ה-push עם הודעה ידידותית למשתמש: "הקהל שבחרת קטן מדי. נסה להרחיב את הטווח."

ה-gate הזה חוסך כשל מלא במקרה הנפוץ ביותר (קהל מצומצם בעיר קטנה + גיל צר + מגדר ספציפי).

---

## חלק ג2.2 — Error handling פר-שלב

**עיקרון אחיד:** לכל שלב, כשל מסווג ל-4 קטגוריות פעולה:

| קטגוריה | משמעות | פעולה |
|---|---|---|
| `retry` | transient רגיל (timeout, 5xx, rate limit) | 3 retries, backoff 2s/5s |
| `refresh_and_retry` | token פג (`code 190`) | רענון טוקן → ניסיון יחיד. אם שוב נכשל → `fail_user` |
| `fail_user` | permanent שדורש תיקון מהמשתמש | rollback + הודעה ספציפית |
| `fail_system` | permanent שהוא bug שלנו | rollback + Sentry alert + הודעה גנרית |

**מימוש המיפוי:** ב-`worker/handlers/push_campaign.py`, **שכבת מיפוי שנייה** מעל `integrations/meta.classify_meta_error` הקיים. המסווג הקיים מ-1.2 מחזיר `transient/permanent/unknown`; שכבת המיפוי השנייה מתרגמת ל-4 הקטגוריות לעיל.

**אסור לשנות את `classify_meta_error` עצמו** — הוא בשימוש גם ב-1.2 וב-2.1, וכל שינוי שם הוא breaking change. השכבה החדשה היא תוספת ייעודית ל-3.4.

### טבלת מיפוי כשלים פר-שלב

| שלב | sub-step | סוגי כשלים אופייניים | קטגוריה | הודעה למשתמש (עברית) |
|---|---|---|---|---|
| **1. Lead Form** | יצירה | Page לא תקף / Privacy URL לא תקף | `fail_user` | "Meta לא הצליחה לקשר את הטופס לדף. בדוק שהדף פעיל ושיש לאפליקציה הרשאות." |
| | | Token פג | `refresh_and_retry` | (לאחר כשל) "החיבור ל-Meta פג. אנא חבר מחדש." |
| | | Validation אצלנו (שדה חסר) | `fail_system` | "שגיאה פנימית. הצוות קיבל התראה." |
| **2. Campaign** | יצירה | Ad Account לא תקף / הושעה | `fail_user` | "חשבון המודעות הושעה ב-Meta. בדוק ב-Business Manager לפני שתנסה שוב." |
| | | Token פג | `refresh_and_retry` | (כנ"ל) |
| | | Objective לא תקף | `fail_system` | "שגיאה פנימית..." |
| **3. Ad Set** | יצירה | קהל מצומצם מדי | `fail_user` | "הקהל קטן מדי. הרחב טווח גיאוגרפי/גיל/מגדר." |
| | | תקציב נמוך מהמינימום | `fail_user` | "התקציב נמוך מהמינימום של Meta. העלה ל-₪X." |
| | | מיקום לא תקף | `fail_user` | "Meta לא זיהתה את המיקום. נסה מיקום אחר." |
| **4-6. Ads** | העלאת תמונה | פורמט/גודל לא תקפים | `fail_user` | "תמונה {N} לא עברה בדיקות Meta. ייצר מחדש." |
| | | רזולוציה נמוכה | `fail_user` | (כנ"ל) |
| | יצירת Ad | תוכן מפר מדיניות | `fail_user` | "Meta דחתה את התוכן. סיבה: {meta_message}. ייצר מחדש או ערוך ידנית." |
| | | טקסט בתמונה > 20% | `fail_user` | "תמונה {N} מכילה יותר מדי טקסט. ייצר מחדש." |
| | | Lead Form ID לא תקף | `fail_system` | "שגיאת מערכת בקישור לטופס..." |
| | | WhatsApp phone לא תקף | `fail_system` | (כנ"ל) |
| **כל שלב** | — | Timeout / 5xx / 429 | `retry` | (אם 3 retries נכשלו → fail_user/fail_system לפי המסווג) |

---

## חלק ג2.3 — Rollback ספציפי פר-שלב

**עקרון: rollback אחיד = `DELETE Campaign + DELETE Lead Form (אם lead)`.**

Meta מוחקת אוטומטית את כל ה-children של Campaign (Ad Set + Ads). Lead Form הוא יישות נפרדת ולכן מחיקה נפרדת. סדר חובה: **Campaign קודם, Lead Form אחרי** (אי-אפשר למחוק Lead Form אם יש Ad פעיל שמצביע אליו).

### מטריצת rollback פר-נקודת כשל

| נכשל ב- | קיים ב-Meta לפני הכשל | פעולות rollback |
|---|---|---|
| Lead Form (שלב 1) | כלום | אין rollback ב-Meta. עדכון DB בלבד. |
| Campaign (שלב 2) | Lead Form (אם lead) | DELETE Lead Form |
| Ad Set (שלב 3) | Campaign + Lead Form (אם lead) | DELETE Campaign → DELETE Lead Form |
| Ad #1 (שלב 4) | Campaign + Ad Set + Lead Form | DELETE Campaign (Ad Set נמחק אוטו) → DELETE Lead Form |
| Ad #2 (שלב 5) | + Ad #1 | DELETE Campaign (Ad Set + Ad #1 נמחקים אוטו) → DELETE Lead Form |
| Ad #3 (שלב 6) | + Ad #1, Ad #2 | DELETE Campaign (Ad Set + Ad #1 + Ad #2 נמחקים אוטו) → DELETE Lead Form |

### מה אם ה-rollback עצמו נכשל?

לכל DELETE ב-Meta יש את אותה מטריצת סיווג (transient/permanent). **אם DELETE נכשל:**

- transient → 3 retries (כמו כל קריאה).
- אם 3 retries נכשלו, **או** permanent (לרוב 404 — ה-entity לא קיים, מה שטוב):
  - אם זה 404 (entity לא קיים) — נחשב הצלחה. ייתכן שהיה נמחק ידנית.
  - אם זה כשל אמיתי → `campaigns.status = 'failed_rollback_pending'` + שורה ב-`failed_cleanup` (קיים מ-2.6.1) + Sentry alert.

---

## חלק ג2.4 — Cleanup cron לקמפיינים תקועים

**מטרה:** לתפוס קמפיינים שתקועים ב-`pushing` בגלל crash של ה-worker / DB disconnect / לולאה אינסופית.

### תזמון

- **תדירות:** כל 5 דקות.
- **מקום:** ב-worker עצמו (לא ב-jobs queue). תוספת ל-`worker/runner.py` בדפוס cron פנימי.
- **לוגיקה:** `cleanup_stuck_campaigns()` רץ אם עברו 5 דקות מהריצה האחרונה.

### תנאי "תקוע"

```sql
SELECT * FROM campaigns
WHERE status = 'pushing'
  AND attempt_started_at < now() - interval '5 minutes'
  AND cleanup_attempts < 5
ORDER BY attempt_started_at ASC;
```

המספר 5 דקות נגזר ממקסימום הזמן התיאורטי של הפעולה: 6 שלבים × 19 שניות (כולל 3 retries) ≈ 2 דקות. 5 דקות נותן מרווח של פי 2-3.

### בדיקת אמת מול Meta

לכל קמפיין שזוהה כתקוע, **לא להניח — לבדוק**.

```
לכל קמפיין תקוע:
  1. קרא ל-Meta לבדוק את ה-meta_campaign_id (אם קיים ב-DB):
     - אם קיים ויש 3 Ads → מצב A (הצלחה שלא נשמרה)
     - אם קיים אבל פחות מ-3 Ads → מצב B (חלקי)
     - אם לא קיים / NULL ב-DB → מצב C (לא נוצר כלום)

  2. החלטה לפי מצב:
     - מצב A: UPDATE status='live'. אין rollback.
     - מצב B: rollback מלא (DELETE Campaign + Lead Form) → status='failed'
       + attempt_error = {step: 'cleanup_detected_stuck', ...}
     - מצב C: UPDATE status='draft'. אין rollback (אין מה למחוק).

  3. בכל מצב — UPDATE cleanup_attempts = cleanup_attempts + 1.

  4. אם בדיקת ה-Meta עצמה נכשלה (transient/permanent):
     - לדלג לקמפיין הבא (ינסה שוב ב-5 דקות).
     - אחרי 5 ניסיונות (cleanup_attempts >= 5) → status='failed_rollback_pending'
       + Sentry alert + שורה ב-failed_cleanup.
```

### תקרת ניסיונות

`cleanup_attempts` עולה ב-1 בכל ריצת cron שגעה בקמפיין הזה. אחרי 5 ניסיונות שלא הצליחו לפתור → `failed_rollback_pending`. המשמעות: עברו 25 דקות (5 × 5 דקות) של ניסיונות בלי הצלחה — זה כבר דורש התערבות ידנית.

---

## חלק ג2.5 — Migration 0013

### קובץ: `supabase/migrations/0013_campaigns_push_state.sql`

```sql
-- Migration 0013: Campaigns push state machine
-- Phase 3.4 — adds state machine for push flow + cleanup support

BEGIN;

-- ============================================================
-- 1. עמודות חדשות ב-campaigns
-- ============================================================

ALTER TABLE public.campaigns
  -- מזהי Meta לצורך rollback (כולם nullable, רק קמפיין pushed מקבל ערכים)
  ADD COLUMN meta_lead_form_id text,
  ADD COLUMN meta_campaign_id text,
  ADD COLUMN meta_ad_set_id text,
  ADD COLUMN meta_ad_ids text[] NOT NULL DEFAULT '{}'::text[],

  -- מטא-דאטה של ניסיון push
  ADD COLUMN attempt_started_at timestamptz,
  ADD COLUMN attempt_error jsonb,

  -- תמיכה ב-cleanup cron
  ADD COLUMN cleanup_attempts int NOT NULL DEFAULT 0;

-- ============================================================
-- 2. הרחבת CHECK constraint של status
-- (הוספת failed + failed_rollback_pending לערכים מ-2.2)
-- ============================================================

ALTER TABLE public.campaigns DROP CONSTRAINT campaigns_status_check;

ALTER TABLE public.campaigns ADD CONSTRAINT campaigns_status_check
  CHECK (status IN (
    'draft',
    'pending_review',
    'pushing',
    'live',
    'paused',
    'archived',
    'failed',
    'failed_rollback_pending'
  ));

-- ============================================================
-- 3. אינדקסים חדשים
-- ============================================================

-- ל-cleanup cron — partial index, רק על pushing
CREATE INDEX idx_campaigns_pushing_stuck
  ON public.campaigns (attempt_started_at)
  WHERE status = 'pushing';

-- לתצוגת admin של failed_rollback_pending
CREATE INDEX idx_campaigns_failed_rollback
  ON public.campaigns (updated_at)
  WHERE status = 'failed_rollback_pending';

-- ============================================================
-- 4. הערות תיעוד
-- ============================================================

COMMENT ON COLUMN public.campaigns.meta_lead_form_id IS
  'Meta Lead Form ID. NULL לקמפיין whatsapp ולקמפיין שלא נדחף.';

COMMENT ON COLUMN public.campaigns.meta_ad_ids IS
  'מערך של 0-3 Ad IDs. נצבר במהלך push, מתאפס ב-rollback.';

COMMENT ON COLUMN public.campaigns.attempt_started_at IS
  'נכתב ב-CAS לפני push. נבדק ע"י cleanup cron (יותר מ-5 דקות = תקוע).';

COMMENT ON COLUMN public.campaigns.attempt_error IS
  'JSONB עם {step, error_type, error_code, message, meta_response}. ' ||
  'step ∈ lead_form/campaign/ad_set/ad_1/ad_2/ad_3/rollback/cleanup_detected_stuck.';

COMMENT ON COLUMN public.campaigns.cleanup_attempts IS
  'מספר ניסיונות cleanup שטרם הצליחו. תקרה: 5 → failed_rollback_pending.';

COMMIT;
```

### ולידציה

- **אין צורך ב-backfill** — קמפיינים קיימים הם `draft`, ו-default values יכסו.
- **RLS לא משתנה** — policies מ-2.2 מבוססות על `user_id`, עובדות עם עמודות חדשות.
- **GRANT לא נדרש** — אין עמודות רגישות חדשות. הקיים מ-2.2 מספיק.
- **idempotent** — אם הקובץ ירוץ פעמיים: ALTER TABLE ADD COLUMN ייכשל (idempotency חלקית). מומלץ:

  ```sql
  ALTER TABLE public.campaigns
    ADD COLUMN IF NOT EXISTS meta_lead_form_id text,
    ...
  ```

  (`migrate.py` של 0.1 מטפל ב-tracking, אבל IF NOT EXISTS עוזר ב-development.)

---

## חלק ג2.6 — שמות הקבצים החדשים

| קובץ | תוכן |
|---|---|
| `worker/handlers/push_campaign.py` | ה-handler הראשי. orchestration של 6 שלבים, classification של שגיאות, rollback |
| `services/campaign_push_service.py` | API ציבורי שה-router קורא אליו (`POST /campaigns/:id/push`). יוצר job, מבצע validation gates |
| `services/meta_rollback_service.py` | פעולות rollback (DELETE Campaign + DELETE Lead Form). מבודד לקריאות, idempotent |
| `worker/cleanup_runner.py` | cron של 5 דקות. מזהה תקועים, בודק מול Meta, מתקן |
| `integrations/meta.py` | (קיים) — להרחיב עם פונקציות חדשות: `create_lead_form`, `create_campaign`, `create_ad_set`, `upload_image`, `create_ad`, `delete_*`, `reach_estimate` |

**שכבת המיפוי השנייה (`transient/permanent → retry/refresh/fail_user/fail_system`)** יכולה לשבת ב-`worker/handlers/push_campaign.py` כפונקציה פנימית, או ב-`integrations/meta.py` כפונקציה חדשה `classify_for_push(error) -> Action`. אני נוטה לאופציה השנייה כי זה שומר את כל לוגיקת הסיווג במקום אחד.

---

## חלק ג2.7 — Done של 3.4

- migration 0013 רצה בהצלחה. כל הקמפיינים הקיימים שמרו על המצב שלהם.
- `POST /campaigns/:id/push` מבצע 8 validation gates → CAS `draft → pushing` → יצירת job.
- worker מבצע 6 שלבים סדרתיים, שומר `meta_*_id` בכל שלב.
- כשל transient → 3 retries עם backoff. כשל permanent → rollback מלא.
- frontend עושה polling על job status, רואה התקדמות.
- הצלחה → `status='live'`. כשל → `status='failed'` עם הודעה ספציפית.
- cleanup cron רץ כל 5 דקות, מזהה תקועים, בודק מול Meta, מתקן.
- `failed_rollback_pending` מוצג ב-admin tool (ליישום עתידי) עם Sentry alert.

## חלק ג2.8 — לא ב-3.4

- regenerate אוטומטי לתוכן AI שנדחה (Phase 8 או mid-3.4 v2)
- circuit breaker ל-429 ברמת ה-app
- היסטוריית push attempts כטבלה נפרדת
- retry policy לפעולת cleanup עצמה מעבר ל-5 ניסיונות (דורש התערבות ידנית)
- shutdown gracefully של worker באמצע push (במקרה של deploy)

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`meta_ad_ids` הוא array, לא 3 עמודות נפרדות.** הסיבה: יותר גמיש (אם נרצה להרחיב ל-5 וריאציות בעתיד), ו-rollback פשוט יותר (לולאה אחת על המערך).

2. **`attempt_error` JSONB עם schema לא-נאכף.** הערכים החוקיים של `step` מתועדים ב-COMMENT, אבל לא ב-CHECK. אם נראה bugs ב-frontend בגלל values לא צפויים — נשקול JSON Schema validation ב-application layer.

3. **`cleanup_attempts` לא מתאפס ב-rollback.** אם קמפיין נכשל ב-cleanup ואז המשתמש ניסה שוב והצליח, ה-counter נשאר מ-attempt קודם. זה בסדר כי הוא רלוונטי רק כשהקמפיין ב-`pushing`. ב-`live`/`failed`/`draft` אף אחד לא מסתכל עליו. אם נרצה לאפס — `UPDATE campaigns SET cleanup_attempts = 0 WHERE status IN ('live', 'failed', 'draft')` במעבר ה-status.

4. **Reach Estimate ב-validation gates דורש Token לקריאה ל-Meta.** המשמעות: gate 2 (`fb_connections valid`) חייב לרוץ לפני gate 8. סדר ה-gates משנה.

5. **שכבת המיפוי השנייה (`classify_for_push`) — להחליט אם ב-`integrations/meta.py` או ב-`worker/handlers/push_campaign.py`.** המלצה: ב-`integrations/meta.py` כי זה sister-function של `classify_meta_error` הקיים, ושומר את כל לוגיקת הסיווג במקום אחד.

---

## Phase 4 · קליטת לידים (קמפיין lead)

### Session 4.1 — webhook לקליטת לידים + idempotency ✅
- [x] endpoint webhook ל-Meta Lead Forms (GET challenge + POST)
- [x] **אימות חתימה** (X-Hub-Signature-256) לפני עיבוד (security.verify_meta_signature)
- [x] **idempotency** דרך `webhook_events` (SELECT לפני fetch + UNIQUE ב-RPC)
- [x] שמירת ליד ב-`leads` עם service מהקמפיין

**Done:** ליד אמיתי מטופס פייסבוק נכנס ל-DB, פעם אחת בלבד גם אם Meta שולח
כפול.
**זהירות (CLAUDE.md):** webhooks patterns — זה בדיוק המקום של idempotency
ו-signature. **קריטי.** כתיבה דרך `supabase_admin` בלבד.

**הושלם — הערות יישום:**
- migration **0026** (leads + GIN index על campaigns.meta_ad_ids + RPC `insert_lead_and_event`
  אטומי — leads+webhook_events בטרנזקציה אחת, unique_violation→duplicate).
- **subscribe ל-Page נוסף ל-3.4** (`execute_push` step 1.5, lead בלבד, idempotent) — אחרת
  Meta לא שולחת webhooks. + `meta.subscribe_page_to_leadgen` + `meta.fetch_lead_details` (SDK).
- HMAC על **raw body** (לא JSON parsed); `webhook_events` נכתב רק אחרי הצלחה (reserve-after-success).
- מיפוי החזרות: 403 (חתימה), 500 (transient — Meta retry), 200 (success/duplicate/orphan/
  no-token — אבוד+alert). page_id verify (defense). config: `META_WEBHOOK_VERIFY_TOKEN`.
- **לאמת מול Meta אמיתי:** שמות שדות leadgen (full_name/phone_number/email/field_data),
  ה-subscribed_apps endpoint, ופורמט ה-webhook payload (entry/changes/value).

# Session 4.1 — Webhook לקליטת לידים מ-Meta

> **עדכון להוספה ל-ROADMAP.md תחת Session 4.1.** Phase 4 — קליטת לידים
> מקמפיינים שעלו לאוויר ב-3.4. רלוונטי לקמפיינים מסוג `lead` בלבד
> (קמפיין `whatsapp` יוצר לידים ב-Phase 5).

---

## תיאור Session 4.1

**Webhook + שמירת ליד.** אחרי ש-3.4 העלה קמפיין לאוויר, לידים מתחילים
להיכנס. Meta שולחת webhook על כל ליד חדש. Session זה מטפל בכל הצינור
מהקליטה ועד השמירה ב-DB.

**מה לא בSession הזה:**
- אכיפת מכסה — Session 4.2.
- בוט WhatsApp שמגיב לליד — Phase 5.
- שליפת לידים לדשבורד — Session UI נפרד.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | `leads` משותפת לשני סוגי קמפיין? | **כן** — אופציה B. ב-4.1 רק lead campaigns. WhatsApp בPhase 5. |
| 2 | idempotency — טבלה נפרדת או webhook_events? | **webhook_events** עם `event_type='meta_lead'` (עקבי עם פלאקארד 2.6.1) |
| 3 | מיפוי webhook → user_id | **דרך `ad_id` ל-`campaigns`** + GIN index על `meta_ad_ids` |
| 4 | משיכת lead details מ-Meta — sync או async? | **סינכרוני** בתוך ה-webhook (1-2s, מתחת ל-timeout של Meta) |
| 5 | שגיאה במשיכה — מה מחזירים? | **500 ל-Meta** (Meta תעשה retry); transaction על leads + webhook_events |
| 6 | סכמת `leads` | 11 עמודות (פירוט בחלק 4) |
| 7 | HMAC verification | `hmac.compare_digest` על raw body, לא על JSON parsed |
| 8 | GET endpoint לאימות (challenge) | חובה — לפני subscribe ל-Meta |
| 9 | מתי לעשות subscribe ל-Page? | **ב-3.4** אחרי יצירת Lead Form, לא ב-OAuth |

---

## חלק 1 — זרימת ה-webhook

### 1.1 — ה-flow המלא

```
POST /webhooks/meta-leads
    ↓
[1] חישוב HMAC על raw body
    ↓ HMAC לא תואם → 403
    ↓
[2] SELECT מ-webhook_events (key=leadgen_id, type='meta_lead')
    ↓ קיים → return 200 (idempotency, ליד כבר עובד)
    ↓
[3] GET /{leadgen_id} מ-Meta (קריאה סינכרונית, ~1-2s)
    ↓ שגיאה → return 500 (Meta תעשה retry)
    ↓
[4] SELECT campaigns WHERE meta_ad_ids @> ARRAY[ad_id]
    ↓ לא נמצא → return 200 + log warning (ליד יתום, לא לחזור)
    ↓
[5] BEGIN TRANSACTION
    INSERT INTO leads (...)
    INSERT INTO webhook_events (key=leadgen_id, type='meta_lead')
    COMMIT
    ↓
[6] return 200
```

**שני נקודות תכנון חשובות:**

**א) `webhook_events` נכתב רק אחרי שהכל הצליח.** הסדר הוא:
SELECT-then-mark, לא mark-then-process. אם נכתוב את `webhook_events`
ראשון ואז ה-Meta call נכשל, נסיון retry של Meta יראה idempotent ולא
יחזור — נאבד את הליד.

**ב) Transaction על leads + webhook_events.** או ששניהם נכתבים, או
ששניהם rollback. מבטיח שאין מצב ביניים.

### 1.2 — GET endpoint (verification challenge)

```
GET /webhooks/meta-leads?hub.mode=subscribe&hub.challenge=XYZ&hub.verify_token=OUR_TOKEN
    ↓
[1] hub.verify_token תואם ל-META_WEBHOOK_VERIFY_TOKEN?
    ↓ לא → 403
    ↓ כן → return 200 text/plain עם {hub.challenge}
```

נדרש ע"י Meta לפני שמאפשרים subscribe. **חד-פעמי בזמן setup**, אבל
חייב להיות קיים.

**Env var חדש:** `META_WEBHOOK_VERIFY_TOKEN` — string אקראי שאנחנו
ממציאים, רושמים גם ב-Meta App Dashboard וגם ב-Render config.
optional + degradation: אם חסר → endpoint מחזיר 503.

---

## חלק 2 — אבטחה: HMAC verification

### 2.1 — חישוב

```python
raw_body: bytes = await request.body()  # חייב raw, לא JSON parsed
signature_header: str = request.headers["X-Hub-Signature-256"]
# Format: "sha256=abc123..."

expected = hmac.new(
    META_APP_SECRET.encode("utf-8"),
    raw_body,
    hashlib.sha256
).hexdigest()
provided = signature_header.removeprefix("sha256=")

if not hmac.compare_digest(expected, provided):
    raise HTTPException(403, "Invalid signature")
```

### 2.2 — נקודות קריטיות

**1. `compare_digest`, לא `==`.** השוואה רגילה פגיעה ל-timing attack.

**2. raw body, לא parsed.** אם FastAPI עשה `await request.json()` —
ה-body השתנה (whitespace, סדר שדות) וה-HMAC לא יתאים. חייב
`await request.body()` קודם.

**3. `META_APP_SECRET` כבר קיים** ב-config מ-1.1. משתמשים באותו.

**4. dependency נפרדת.** בנה `verify_meta_signature` כ-FastAPI
dependency ב-`core/security.py` — שימוש חוזר ב-webhooks עתידיים (Phase
8 כשמטא תודיע על rejected ads, וכו').

---

## חלק 3 — מיפוי webhook → user/campaign

### 3.1 — איך מוצאים בעלים

ה-webhook payload כולל `ad_id`. נחפש ב-`campaigns`:

```sql
SELECT id, user_id, service_name, fb_page_id
FROM campaigns
WHERE meta_ad_ids @> ARRAY[?]  -- ad_id מה-webhook
LIMIT 1
```

**שני verifications:**

**א) page_id מ-Meta תואם ל-fb_page_id ב-campaigns?** Defense in depth
— אם מישהו איכשהו זייף ad_id שתואם לקמפיין שלנו אבל מ-Page אחר,
ה-page_id verification יתפוס. log warning + 200 (לא 403, כדי לא לחשוף
מידע למתקיף).

**ב) הקמפיין במצב פעיל?** מסננים `status IN ('live', 'paused')`. ליד
שמגיע על קמפיין ב-`draft`/`failed` הוא bug — log + return 200 בלי
שמירה.

### 3.2 — אינדקס

```sql
CREATE INDEX idx_campaigns_meta_ad_ids
  ON campaigns USING GIN (meta_ad_ids);
```

GIN index על array מאפשר שאילתת `@>` (contains) מהירה. בלי זה — sequential scan
על כל הקמפיינים. בנפח MVP זה בסדר, אבל הדפוס נכון מההתחלה.

### 3.3 — ליד יתום (orphan lead)

תרחיש: webhook מגיע על `ad_id` שלא קיים ב-`campaigns`. סיבות:
- הקמפיין נמחק ב-Meta על-ידי המשתמש ידנית, ועוד היה ליד אחד "בדרך".
- bug בייצור (לא אמור לקרות).

**טיפול:** return 200 + log warning + Sentry alert. **לא מחזירים 500**
כי Meta תנסה שוב לנצח. הליד אבוד — אין מה לעשות.

---

## חלק 4 — סכמת `leads`

### 4.1 — Migration

```sql
-- migration XXXX_leads_table.sql
-- (המספר ייקבע לפי המיגרציות שכבר קיימות)

BEGIN;

CREATE TABLE public.leads (
  -- מזהים
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  campaign_id uuid NOT NULL REFERENCES public.campaigns(id) ON DELETE RESTRICT,

  -- composite FK ל-RLS coherence (spec §7.2)
  CONSTRAINT leads_campaign_user_fk
    FOREIGN KEY (campaign_id, user_id)
    REFERENCES public.campaigns(id, user_id)
    ON DELETE CASCADE,

  -- שירות (denormalized מ-campaigns, שורד גם אם הקמפיין יימחק)
  service_name text NOT NULL,

  -- תוכן הליד
  contact_name text NOT NULL,
  contact_phone text NOT NULL,
  contact_email text,
  contact_key text NOT NULL,  -- טלפון מנורמל לאיחוד עתידי

  -- שאלות סינון (jsonb גמיש)
  screening_answers jsonb DEFAULT '{}'::jsonb,

  -- מטא-דאטה
  meta_leadgen_id text NOT NULL UNIQUE,  -- idempotency layer 2
  meta_ad_id text NOT NULL,
  raw_payload jsonb NOT NULL,

  created_at timestamptz NOT NULL DEFAULT now()
);

-- אינדקסים
CREATE INDEX idx_leads_user_created
  ON public.leads (user_id, created_at DESC);

CREATE INDEX idx_leads_campaign_created
  ON public.leads (campaign_id, created_at DESC);

CREATE INDEX idx_leads_contact_key
  ON public.leads (contact_key);

-- RLS
ALTER TABLE public.leads ENABLE ROW LEVEL SECURITY;

CREATE POLICY leads_select_own
  ON public.leads FOR SELECT
  USING (auth.uid() = user_id);

-- INSERT/UPDATE/DELETE — רק דרך admin client בwebhook
-- אין policy → ברירת המחדל היא דחייה

-- GRANT
GRANT SELECT ON public.leads TO authenticated;

COMMIT;
```

### 4.2 — הערות על הסכמה

**`campaign_id ON DELETE RESTRICT`** — לא לאפשר מחיקת קמפיין שיש לו
לידים. אם הלקוח מנסה למחוק קמפיין — צריך גם להסיר את הלידים שלו,
החלטה מודעת. (לא בMVP — אין UI למחיקת קמפיין.)

**`composite FK` ל-`(campaign_id, user_id)`** — כפי שננעל בשאלה הקודמת
מ-CC (3.1ב). מגן מ-service_role leak.

**`service_name` denormalized מ-`campaigns`** — כדי שאם הקמפיין יימחק
בעתיד, נדע על איזה שירות הליד פנה. זה מתאים לעיקרון של spec §6
("רשומת-אירוע פנייה").

**`contact_key`** — טלפון מנורמל (להסיר מקפים, אפסים מובילים, וכו').
**ב-MVP לא unique** — לפי spec §6, "כל פנייה = ליד נפרד. לא מאחדים לפי
טלפון." אבל ה-index קיים לאיחוד עתידי.

**`raw_payload` כ-jsonb** — שמירת ה-response המלא מ-Meta. עוזר ב-debug
ובתאימות עתידית אם Meta יוסיפו שדות.

**אין `status` ב-MVP.** ניהול pipeline (`new`/`contacted`/`converted`)
הוא Phase מאוחר. הלקוח עובד עם הליד ב-WhatsApp/טלפון, לא דרכנו.

**אין `bot_conversation_id`** — Phase 5. כשהבוט יקושר לליד, נוסיף.

---

## חלק 5 — Subscribe ל-Page (שינוי ל-3.4)

### 5.1 — מתי

**אחרי יצירת Lead Form ב-3.4, לפני יצירת Campaign.** קריאה אחת נוספת
ל-Meta.

### 5.2 — מה

```
POST /{page_id}/subscribed_apps?subscribed_fields=leadgen
   Header: Authorization: Bearer {page_access_token}
```

**חשוב:** ה-token כאן הוא `page_access_token`, לא ה-user token. שולפים
אותו דרך:

```
GET /{page_id}?fields=access_token
   Header: Authorization: Bearer {user_token}
```

ה-page_access_token שמיש כל עוד היוזר לא ביטל הרשאות.

### 5.3 — Idempotency של subscribe

קריאה חוזרת ל-`subscribed_apps` על Page שכבר subscribed — Meta מחזירה
200 בלי לעשות כלום. אין race ואין duplicate. ניתן לקרוא בכל push חדש
בלי לבדוק status קודם.

### 5.4 — איפה זה משתלב ב-3.4

זרימת 3.4 המעודכנת:

```
1. Lead Form         (אם type='lead')
   ↓
1.5. Subscribe Page  (אם type='lead', אחרי Lead Form)  ← חדש
   ↓
2. Campaign
3. Ad Set
4. Ad #1
5. Ad #2
6. Ad #3
```

**רק קמפיין `lead`** עושה subscribe. WhatsApp campaigns לא צריכים
webhook של leadgen.

**מסווג כשל:** כשל ב-subscribe → `fail_user` (probably token issue).
rollback = DELETE Lead Form (לא נוצר Campaign עדיין).

**מסמך 3.4 יעודכן** עם השינוי הזה — אעדכן ידנית כשנשלב את שני
המסמכים.

---

## חלק 6 — Env vars חדשים

| שם | משמעות | optional? |
|---|---|---|
| `META_WEBHOOK_VERIFY_TOKEN` | string אקראי לאימות ה-webhook setup | optional + degradation → 503 |

`META_APP_SECRET` כבר קיים מ-1.1 — משתמשים באותו ל-HMAC.

---

## חלק 7 — שמות הקבצים החדשים

| קובץ | תוכן |
|---|---|
| `routers/webhooks/meta_leads.py` | endpoint יחיד עם GET ו-POST |
| `services/lead_intake_service.py` | הלוגיקה: HMAC, idempotency, fetch, save |
| `integrations/meta.py` | (קיים) הרחבה: `fetch_lead_details`, `subscribe_page_to_leadgen` |
| `core/security.py` | (קיים) הרחבה: `verify_meta_signature` כ-dependency |
| `models/lead.py` | Pydantic models: `LeadCreate`, `LeadResponse` |

**ה-admin client** נדרש ב-`lead_intake_service` כי INSERT ל-`leads` הוא
server-authoritative (spec §7.3-א — webhook handler).

---

## חלק 8 — Done של 4.1

- migration להוספת טבלת `leads` רצה בהצלחה. RLS פעיל.
- GIN index על `campaigns.meta_ad_ids` קיים.
- `POST /webhooks/meta-leads` מאומת HMAC, מבצע idempotency, מחזיר 200/403/500
  כפי שתואר.
- `GET /webhooks/meta-leads` (verification challenge) מחזיר את ה-challenge
  אם הטוקן תואם.
- Subscribe ל-Page קורה ב-3.4 אחרי Lead Form creation (שינוי ל-3.4).
- ליד שמגיע נכנס ל-`leads` עם כל השדות, כולל `contact_key` מנורמל.
- ליד יתום (ad_id לא קיים) → log + 200, לא נופל.
- Meta verification challenge עובד.
- כל הטסטים החדשים עוברים.

## חלק 9 — לא ב-4.1

- אכיפת מכסה (Session 4.2).
- שליפת לידים לדשבורד (UI נפרד).
- בוט WhatsApp שמגיב לליד (Phase 5).
- ניהול pipeline (`lead.status`).
- איחוד לידים לפי `contact_key`.
- התראות ללקוח על ליד חדש (Phase 4.6 בROADMAP).

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **HMAC על raw body — pattern קריטי.** תיעוד מפורש ב-CLAUDE.md
   ובהערה מעל הפונקציה `verify_meta_signature`. סטנדרט לכל webhook
   ב-Meta.

2. **`contact_key` normalization** — implementation:
   - הסר כל מה שלא ספרה (`re.sub(r'\D', '', phone)`).
   - הסר prefix `+` ו-`0` מובילים.
   - אם מתחיל ב-`972`, השאר. אחרת, אם 9-10 ספרות ומתחיל ב-`5`/`0`,
     הוסף `972` prefix.
   - תוצאה סטנדרטית: `972501234567`.

3. **`webhook_events` UNIQUE על `(event_type, key)`**, לא רק `key`.
   מאפשר שני webhook providers שונים עם אותו key הילותי. כבר קיים מ-2.6.1?
   לאמת לפני המימוש.

4. **GIN index על `meta_ad_ids`** דורש שהעמודה תהיה `text[]`, לא
   `jsonb`. כבר נסגר ב-migration של 3.4.

5. **קריאה ל-Meta API שולחת `page_access_token`, לא user token.**
   page_access_token נשלף מ-`fb_connections` (כבר מוצפן ב-Vault מ-1.2).
   `meta_service` יודע לטפל בזה.

6. **timeout על ה-`fetch_lead_details`** — 5 שניות. אם Meta יותר איטית
   מזה, return 500 ל-Meta והם ינסו שוב. עדיף מאשר להחזיק את ה-webhook
   פתוח 30 שניות.

7. **CC חייב להוסיף ל-`integrations/meta.py`:**
   - `fetch_lead_details(leadgen_id, page_access_token) -> dict`
   - `subscribe_page_to_leadgen(page_id, page_access_token) -> bool`
   שתי פונקציות חדשות, אבל עקביות עם ה-pattern הקיים.


### Session 4.2 — ספירת מכסה והתראה ✅
- [x] ספירת לידים בתוך החלון `[current_period_start, +1 month)` מול `lead_quota` (anniversary, לא קלנדרי — ראה spec §5)
- [x] התראת שדרוג בקרבת/חריגת מכסה (לא חוסמים קליטת ליד)

**Done:** כשעוברים מכסה — התראה נשלחת, הליד עדיין נקלט.


# Session 4.2 — ספירת מכסה והתראות שדרוג

> **עדכון להוספה ל-ROADMAP.md תחת Session 4.2.** ממשיך את 4.1
> (קליטת לידים). Phase 4 סוגרת את הצינור של לידים — קליטה (4.1),
> ספירה והתראה (4.2). אכיפה ברמת הבוט (Phase 5) משתמשת בכלים שבונים
> כאן.

---

## תיאור Session 4.2

**ספירה והתראה.** משתמש שהגיע ל-80% מהמכסה מקבל התראה ("מתקרב למכסה").
משתמש שעבר 100% מקבל התראה שנייה ("עברת את המכסה, שקול שדרוג"). הליד
עצמו תמיד נספר ונשמר — לא חוסמים קליטה. אכיפה אמיתית (בוט שמפסיק
להגיב) קורה ב-Phase 5, על בסיס ה-helper שנבנה כאן.

**מה לא ב-4.2:**
- אכיפת בוט (Phase 5).
- שליחת ההתראות בפועל (Phase 4.6 — handler ל-`send_quota_alert`).
- UI של dashboard ("השתמשת ב-X מתוך Y") — UI session נפרד.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | איך סופרים לידים? | **COUNT חי** על `leads` בכל קריאה. לא counter ב-DB. |
| 2 | source-of-truth ל-`current_period_start`? | `subscriptions` (קיים מ-2.6.1) |
| 3 | helper מרכזי לספירה? | **כן** — `get_quota_status(user_id) -> QuotaStatus` ב-`subscription_service` |
| 4 | מתי שולחים התראה? | סף **80%** (warning) + סף **100%** (limit). כל אחד שולח **פעם אחת בלבד** בחלון. |
| 5 | איך מונעים התראה כפולה? | 2 flags ב-`subscriptions`: `notified_80_at`, `notified_100_at`. מתאפסים ב-renewal. |
| 6 | מנגנון שליחת ההתראה? | **jobs queue** (`type='send_quota_alert'`). ה-handler עצמו ב-Phase 4.6. |
| 7 | מה עם `tier=whatsapp`? | `lead_quota=null` → דילוג מוחלט על כל הלוגיקה. |
| 8 | backfill לידים ישנים? | **אין** — COUNT חי תופס הכל אוטומטית. |

---

## חלק 1 — Helper מרכזי: `get_quota_status`

### 1.1 — מבנה ה-dataclass

ב-`app/services/subscription_service.py` (קיים מ-2.5), נוסיף:

```python
@dataclass
class QuotaStatus:
    used: int                    # כמה לידים נספרו בחלון הנוכחי
    quota: int | None            # מכסה (None = unlimited)
    percentage: float | None     # אחוז ניצול (None ב-unlimited)
    is_unlimited: bool           # True ב-tier=whatsapp
    is_near_limit: bool          # >= 80% (False ב-unlimited)
    is_over_limit: bool          # >= 100% (False ב-unlimited)
    period_start: datetime       # current_period_start של המנוי
    period_end: datetime         # period_start + 1 month
```

### 1.2 — מימוש

```python
async def get_quota_status(user_id: UUID) -> QuotaStatus:
    """
    Returns the quota usage status for a user in the current billing period.

    Returns is_unlimited=True for whatsapp tier (lead_quota=null).
    Raises SubscriptionNotFoundError if user has no subscription.
    """
```

**הזרימה:**
1. שלוף את ה-subscription של המשתמש (`subscription_service.get_user_subscription`).
2. אם אין subscription → `SubscriptionNotFoundError`.
3. אם `lead_quota IS NULL` → החזר `QuotaStatus(is_unlimited=True, used=0, quota=None, ...)`. אין צורך בספירה.
4. אחרת:
   - חשב `period_end = period_start + interval '1 month'`.
   - COUNT על `leads` בחלון.
   - חשב `percentage`, `is_near_limit`, `is_over_limit`.
5. החזר `QuotaStatus`.

### 1.3 — שאילתת ה-COUNT

```sql
SELECT COUNT(*) FROM leads
WHERE user_id = $1
  AND created_at >= $2  -- period_start
  AND created_at <  $3  -- period_end
```

האינדקס `idx_leads_user_created` שנוצר ב-4.1 (`leads(user_id, created_at DESC)`) תומך בזה ישירות. שאילתה מהירה בנפח MVP, מהירה גם בעתיד.

### 1.4 — מי קורא ל-helper

- **4.2 עצמו** — אחרי כל INSERT ל-`leads` ב-`lead_intake_service` (בדיקת ספים).
- **Phase 5** — לפני תגובה של הבוט (אכיפה).
- **dashboard endpoint** — `GET /me/quota` (פירוט בחלק 4).

**מקור אחד של אמת לכל הלוגיקה הקשורה למכסה.**

---

## חלק 2 — בדיקת ספים אחרי INSERT

### 2.1 — מתי

ב-`lead_intake_service` (4.1), אחרי שה-transaction של `INSERT leads + INSERT webhook_events` הצליחה. **לא בתוך ה-transaction עצמה** — אם ה-INSERT הצליח, ה-webhook מצליח גם אם בדיקת הספים תזרוק.

```python
# בתוך handle_meta_webhook, אחרי הצלחת ה-transaction:
try:
    await check_quota_thresholds(user_id)
except Exception as e:
    sentry.capture_exception(e)
    # לא לזרוק — ההתראה לא קריטית להצלחת הצינור
```

### 2.2 — לוגיקת `check_quota_thresholds`

```python
async def check_quota_thresholds(user_id: UUID) -> None:
    """
    Checks if user crossed 80% or 100% thresholds and triggers alerts
    if not already notified in the current period.

    Idempotent: re-running won't create duplicate alerts.
    """
    status = await get_quota_status(user_id)
    if status.is_unlimited:
        return

    subscription = await get_user_subscription(user_id)

    if status.is_over_limit and subscription.notified_100_at is None:
        await trigger_alert(user_id, threshold=100, status=status)
        await mark_notified(user_id, threshold=100)

    elif status.is_near_limit and subscription.notified_80_at is None:
        await trigger_alert(user_id, threshold=80, status=status)
        await mark_notified(user_id, threshold=80)
```

**שתי הערות חשובות:**

**א) `elif`, לא `if`+`if`.** משתמש שעובר ישירות מ-79% ל-100% (תרחיש אפשרי אם הגיעו כמה לידים במכה) מקבל **רק** את התראת 100%. ה-80% לא נשלחת. זה במכוון — לא רוצים להציף את המשתמש בשתי הודעות באותו רגע.

**ב) bug אפשרי: אם משתמש מקבל את 100% **לפני** ה-80%** (תרחיש קצה — קפיצה מאפס ל-150%), אז `notified_80_at` נשאר NULL. אם בעתיד הוא חוזר ל-79% (לא יכול לקרות בלי downgrade), נשלח לו 80%. ב-MVP זה אפסי.

### 2.3 — `mark_notified`

```python
async def mark_notified(user_id: UUID, threshold: int) -> None:
    """Sets notified_{80,100}_at = now() in subscriptions."""
```

**דרך admin client** (server-authoritative, spec §7.3-ב). זה אותו pattern כמו ב-2.5/2.6.1.

### 2.4 — איפוס ה-flags ב-renewal

ב-2.6.1 (cron יומי לחיוב חודשי), אחרי שה-`current_period_start` מתעדכן, **צריך לאפס את שני ה-flags ל-NULL.** זה חלק מהמעבר לחלון חדש.

**שינוי קטן ל-2.6.1:** ה-handler של החיוב החודשי (כשמצליח) מעדכן את ה-row של `subscriptions`. צריך להוסיף לאותה עסקה:
```sql
UPDATE subscriptions
SET current_period_start = current_period_start + interval '1 month',
    notified_80_at = NULL,
    notified_100_at = NULL
WHERE user_id = $1
```

**הערה ל-CC:** זה שינוי קטן ל-2.6.1 שכבר נסגר. אפשר לעשות אותו ב-PR של 4.2 עצמו (לא דורש Session נפרד).

---

## חלק 3 — שליחת ההתראה דרך jobs queue

### 3.1 — Job חדש

ב-`worker/handlers.py`, סוג חדש: `send_quota_alert`.

```python
async def trigger_alert(user_id: UUID, threshold: int, status: QuotaStatus) -> None:
    """Creates a job to send a quota alert. Doesn't send directly."""
    await create_job(
        type='send_quota_alert',
        user_id=user_id,
        payload={
            'threshold': threshold,        # 80 או 100
            'used': status.used,
            'quota': status.quota,
            'percentage': status.percentage,
        },
    )
```

### 3.2 — Handler placeholder ב-4.2

ב-Phase 4.6 (התראות מערכת) ייכתב handler אמיתי שיודע איך להתרע (מייל,
WhatsApp). ב-4.2 נכניס **placeholder** שרק לוגג:

```python
# worker/handlers/send_quota_alert.py
async def handle_send_quota_alert(job: Job) -> None:
    """
    Placeholder handler for quota alerts.
    Real notification logic implemented in Phase 4.6.
    """
    logger.info(
        "quota_alert_pending",
        extra={
            "user_id": job.user_id,
            "threshold": job.payload['threshold'],
            "used": job.payload['used'],
            "quota": job.payload['quota'],
        }
    )
    # No-op until Phase 4.6
```

**זה נשמר כ-job פעיל ב-DB** — אחרי Phase 4.6 ה-handler יתחבר ויעבד. ה-job עצמו לא יזרוק שגיאה. **שווה לוודא** שב-Phase 4.6 לא נכשל retroactively על jobs ישנים — או שנתחיל handler מ-`processed_at > X` או שניקה ידנית את ה-jobs הישנים.

### 3.3 — Idempotency

ה-`mark_notified` מבטיח שלא נוצר job כפול. אם בכל זאת קורה (אם ה-transaction של `mark_notified` נכשלה אחרי `create_job`) — לוג ב-Sentry. ב-Phase 4.6 ה-handler יבדוק בעצמו אם זה כבר נשלח לפני שליחה.

---

## חלק 4 — Dashboard endpoint

### 4.1 — `GET /me/quota`

ב-`routers/users.py` (אם קיים) או `routers/me.py`:

```python
@router.get("/me/quota")
async def get_my_quota(
    current_user: User = Depends(get_current_user),
) -> QuotaResponse:
    status = await subscription_service.get_quota_status(current_user.id)
    return QuotaResponse(
        used=status.used,
        quota=status.quota,
        percentage=status.percentage,
        is_unlimited=status.is_unlimited,
        is_near_limit=status.is_near_limit,
        is_over_limit=status.is_over_limit,
        period_start=status.period_start,
        period_end=status.period_end,
    )
```

### 4.2 — Pydantic response model

```python
class QuotaResponse(BaseModel):
    used: int
    quota: int | None
    percentage: float | None
    is_unlimited: bool
    is_near_limit: bool
    is_over_limit: bool
    period_start: datetime
    period_end: datetime
```

ה-frontend יציג: "השתמשת ב-{used} מתוך {quota} לידים החודש" + פס התקדמות.
ב-unlimited: "ללא הגבלה" + ספירה גולמית.

---

## חלק 5 — Migration

### 5.1 — עמודות חדשות ב-`subscriptions`

המספור ייקבע לפי המיגרציות הקיימות:

```sql
ALTER TABLE public.subscriptions
  ADD COLUMN notified_80_at timestamptz,
  ADD COLUMN notified_100_at timestamptz;

COMMENT ON COLUMN public.subscriptions.notified_80_at IS
  'Set when 80%% quota alert was sent in current period. Reset to NULL on renewal.';

COMMENT ON COLUMN public.subscriptions.notified_100_at IS
  'Set when 100%% quota alert was sent in current period. Reset to NULL on renewal.';
```

**אין צורך ב-backfill** — ערכים NULL הם המצב התקין למשתמשים שלא חצו ספים.

**RLS לא משתנה** — policies קיימות (SELECT only) עובדות עם עמודות חדשות.

**Type חדש בעמודת `jobs.type`:**
```sql
ALTER TABLE public.jobs DROP CONSTRAINT jobs_type_check;
ALTER TABLE public.jobs ADD CONSTRAINT jobs_type_check
  CHECK (type IN (
    -- types שכבר קיימים מ-3.0...
    'send_quota_alert'
  ));
```

(אם הטבלה `jobs` עוד לא קיימת — היא תיווצר ב-3.0. במקרה כזה, הוספת
`send_quota_alert` ל-CHECK כאן זה תוספת ל-3.0 ולא migration נפרד.)

---

## חלק 6 — Done של 4.2

- Migration רץ — `subscriptions.notified_80_at`, `notified_100_at` קיימים.
- `get_quota_status(user_id)` מוחזר נכון לכל 3 ה-tiers.
- אחרי כל INSERT ליד (4.1), הצינור בודק ספים ויוצר jobs.
- `mark_notified` מאפס מאחורי הקלעים — לא ניתן ליצור התראה כפולה באותו חלון.
- `send_quota_alert` רשום ב-jobs queue, handler placeholder בלוגים.
- `GET /me/quota` מחזיר QuotaResponse מלא, RLS אוכף.
- ב-2.6.1 renewal cron — איפוס שני ה-flags ל-NULL (שינוי קטן).
- כל הטסטים החדשים עוברים.

## חלק 7 — לא ב-4.2

- שליחת ההתראה בפועל (מייל / WhatsApp) — Phase 4.6.
- אכיפת בוט שמפסיק להגיב — Phase 5.
- UI של dashboard עם פס התקדמות — Frontend session.
- 3 ספים (50%, 80%, 100%) או יותר — נדחה. ב-MVP רק 80% ו-100%.
- shape של ההתראה (איך נראית הודעה) — Phase 4.6.
- downgrade / upgrade — מחוץ ל-MVP.

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`elif` ולא `if`+`if` בבדיקת ספים.** משתמש שעובר ישר מ-79% ל-100%
   מקבל רק הודעת 100%. במכוון — לא להציף.

2. **`tier=whatsapp` → דילוג מוחלט.** ב-`get_quota_status` מחזיר
   `is_unlimited=True` ויוצא מיד. ב-`check_quota_thresholds`
   הראשון בלוק "`if status.is_unlimited: return`" — חוסך COUNT
   מיותר ובדיקות מיותרות.

3. **`mark_notified` חייב להיות דרך admin client.** המשתמש לא יכול
   לעדכן את ה-flags של עצמו. spec §7.3-ב.

4. **`check_quota_thresholds` נקרא אחרי ה-transaction של `leads`, לא בתוכה.**
   כשל בבדיקה לא יכול להפיל את הצינור של 4.1. ה-INSERT ל-`leads`
   חייב להצליח גם אם ההתראה נכשלה.

5. **2.6.1 צריך עדכון קטן.** ה-renewal cron מאפס את שני ה-flags.
   זה תוספת קטנה ל-handler קיים, לא Session נפרד.

6. **`jobs.type='send_quota_alert'`** — אם 3.0 עדיין לא נכתב,
   ה-CHECK constraint יתעדכן יחד איתו. אם 3.0 כבר קיים — שינוי קטן.

7. **`COUNT(*)` עם index — מהיר.** בנפח MVP זה < 5ms. אם בעתיד נראה
   spike בעומס — אפשר להוסיף materialized view או counter. ב-MVP לא נדרש.

8. **timezone — UTC בכל מקום.** המרה לעברית רק ב-frontend.

9. **`is_near_limit` ו-`is_over_limit` ב-`QuotaStatus`** — ערכי boolean
   מחושבים. ה-frontend לא צריך לבדוק `>= 80` בעצמו. logic אחיד.

---

## סטטוס מימוש: הושלם ✅ (Session 4.2)

מומש לפי ה-ROADMAP, עם 2 סטיות מאושרות (root-cause):
- **`_claim_notification` כ-CAS לפני enqueue (reserve-first)** במקום הסדר trigger→mark
  שב-ROADMAP — מונע jobs כפולים כששני webhooks חוצים סף במקביל (CLAUDE.md כלל 2/9).
- **`period_start`/`period_end` כ-`datetime | None`** (ולא חובה) — מנוי לא-מאוקטב יכול
  להיות עם `current_period_start=NULL` (לפני חיוב ראשון).

**קבצים:** `subscription_service` (`QuotaStatus`, `get_quota_status`, `check_quota_thresholds`,
`_claim_notification`, `_enqueue_quota_alert`), `QuotaResponse` + `GET /me/quota`, handler
placeholder `handle_send_quota_alert`, חיבור ב-`lead_intake_service` (אחרי INSERT, try/except),
migrations 0027 (flags) / 0028 (jobs.type) / 0029 (איפוס flags ב-`finalize_charge_attempt`).
**טסטים:** `test_quota.py` (18) + 3 ב-`test_lead_intake.py`.

---

## Phase 4.5 · שליפת Meta Insights

> חוסם את הסוכן (Phase 7) ואת האופטימיזציה (Phase 8) — שניהם צריכים מספרים אמיתיים. קמפיינים צריכים לרוץ (3.4) ולידים להיכנס (4.1) כדי שיהיה CPL.


# Session 4.5 — שליפת Meta Insights ✅

> **עדכון להוספה ל-ROADMAP.md תחת Session 4.5.1.** Phase 4.5 הוא
> תשתית לסוכן (Phase 7) ולאופטימיזציה (Phase 8). בלי 4 המטריקות של
> Meta Insights, הסוכן לא יודע "לצטט מספרים" כפי שדורש spec §2א חוק 7.

---

## תיאור Session 4.5

**שליפת 4 מטריקות פר-קמפיין מ-Meta Insights:** Spend, Leads, CPL, CTR.
זמין דרך helper מרכזי `get_campaign_insights(campaign_id)`. caching
in-memory עם TTL גמיש. stale-while-error לעמידות מול תקלות Meta.

**מה לא ב-4.5:**
- שימוש בנתונים בצ'אט הסוכן — Phase 7.
- שימוש לאופטימיזציה אוטונומית — Phase 8.
- מטריקות נוספות (impressions, reach, frequency) — לא נדרשות ל-MVP.
- aggregation ברמת Ad Set — מיותר (Ad Set אחד פר קמפיין ב-MVP).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מקור ה-leads count | **DB ל-`lead` campaign, Meta API ל-`whatsapp`** |
| 2 | רמות aggregation | **Campaign + Ad** (לא Ad Set) |
| 3 | מיפוי leads → Ads | `leads.meta_ad_id` (כבר נשמר ב-4.1) |
| 4 | מקור ה-cache | **in-memory** dict + TTL. לא DB, לא Redis. |
| 5 | TTL | **גמיש בקריאה**, default 300s (5 דקות) |
| 6 | טווח זמן | **גמיש בקריאה**, default `maximum` (lifetime) |
| 7 | טיפול בכשל | **stale-while-error** ל-transient. fail-loud ל-permanent. |
| 8 | proactive refresh? | **לא ב-MVP** — lazy fetch בלבד |
| 9 | טווח חישוב CPL | **same range** ל-spend ול-leads (לא anniversary) |

---

## חלק 1 — Public API

### 1.1 — dataclasses

ב-`app/services/meta_service.py` (קיים מ-2.1):

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class CampaignInsights:
    campaign_id: str       # מזהה הקמפיין שלנו (UUID)
    meta_campaign_id: str  # מזהה ב-Meta
    spend: float           # ב-שקלים
    leads: int
    cpl: float | None      # None אם leads=0
    ctr: float             # אחוז (0-100)
    date_range_start: datetime
    date_range_end: datetime

@dataclass
class AdInsights:
    ad_id: str             # מזהה ה-Ad שלנו
    meta_ad_id: str
    angle: str             # 'emotional' / 'pain_solution' / 'result_success'
    spend: float
    leads: int
    cpl: float | None
    ctr: float

@dataclass
class FullCampaignInsights:
    campaign: CampaignInsights
    ads: list[AdInsights]
    fetched_at: datetime   # מתי הנתונים נשלפו
    is_stale: bool         # True אם החזרנו cached value בגלל שגיאת Meta
```

### 1.2 — פונקציה ראשית

```python
async def get_campaign_insights(
    campaign_id: UUID,
    date_preset: str = "maximum",
    max_age_seconds: int = 300,
) -> FullCampaignInsights:
    """
    Fetches campaign insights (spend, leads, CPL, CTR) at campaign + ad level.

    Args:
        campaign_id: UUID of campaign in our DB.
        date_preset: Meta API date_preset. Common: 'maximum', 'last_7d',
                     'last_30d', 'today', 'yesterday'.
        max_age_seconds: Use cached value if fresher than this. 0 = always fresh.

    Returns:
        FullCampaignInsights with campaign-level + per-ad metrics.

    Raises:
        CampaignNotFoundError: if campaign doesn't exist or status != 'live'.
        MetaTokenExpiredError: if FB connection token expired.
        MetaPermissionError: if ad account suspended / no permissions.
        MetaServiceError: if Meta unreachable AND no cached value available.
    """
```

---

## חלק 2 — Caching strategy

### 2.1 — מבנה ה-cache

```python
# ב-module level
_insights_cache: dict[tuple[UUID, str], tuple[FullCampaignInsights, datetime]] = {}
#                  ^ key = (campaign_id, date_preset)
#                                                  ^ value = (insights, fetched_at)
```

מפתח ה-cache הוא `(campaign_id, date_preset)` כי אותו קמפיין יכול
להישלף ב-טווחי זמן שונים, וכל אחד מהם cache נפרד.

### 2.2 — Decision logic

```python
async def get_campaign_insights(campaign_id, date_preset, max_age_seconds):
    cache_key = (campaign_id, date_preset)
    cached = _insights_cache.get(cache_key)

    # אם cache פנימי טרי — החזר מיד
    if cached is not None:
        insights, fetched_at = cached
        age = (datetime.now(timezone.utc) - fetched_at).total_seconds()
        if age <= max_age_seconds:
            return insights  # is_stale=False

    # אחרת — נסה fetch חדש
    try:
        fresh = await _fetch_from_meta(campaign_id, date_preset)
        _insights_cache[cache_key] = (fresh, datetime.now(timezone.utc))
        return fresh
    except MetaTransientError:
        if cached is not None:
            logger.warning(
                "returning_stale_insights",
                extra={"campaign_id": str(campaign_id), "age_seconds": age},
            )
            stale = replace(cached[0], is_stale=True)
            return stale
        raise MetaServiceError("Meta unreachable and no cached value")
    except MetaPermanentError:
        raise  # token expired / suspended account — fail loud
```

### 2.3 — איפוס cache בעת רענון טוקן או הזרמת קמפיין חדש

אין צורך לאפס באופן יזום. ה-cache מתעדכן באופן טבעי בקריאה הבאה.

**יוצא דופן:** אם בעתיד יש פעולה שמעדכנת קמפיין ידנית (למשל Phase 8
שמכבה מודעה), כדאי לקרוא ל-`_invalidate_cache(campaign_id)` כדי לא
להציג נתונים ישנים. נטפל בזה ב-Phase 8.

---

## חלק 3 — Fetch logic

### 3.1 — קריאה ל-Meta API

ב-`app/integrations/meta.py`:

```python
async def fetch_campaign_insights_raw(
    meta_campaign_id: str,
    page_access_token: str,
    date_preset: str = "maximum",
) -> dict:
    """
    Calls Meta Marketing API Insights endpoint at campaign level.
    Returns raw dict response.
    """
    # GET /{meta_campaign_id}/insights
    # ?fields=spend,ctr,actions
    # &date_preset={date_preset}
    # &level=campaign

async def fetch_ads_insights_raw(
    meta_campaign_id: str,
    page_access_token: str,
    date_preset: str = "maximum",
) -> list[dict]:
    """
    Calls Meta Marketing API Insights endpoint at ad level.
    Returns list of dicts (one per ad).
    """
    # GET /{meta_campaign_id}/insights
    # ?fields=ad_id,spend,ctr,actions
    # &date_preset={date_preset}
    # &level=ad
```

**שתי קריאות נפרדות** — Campaign level + Ad level. Meta API לא מחזיר
את שתיהן בקריאה אחת בנוחות.

**הערה ל-CC:** Meta יכולה להחזיר 4 שדות נפרדים. במקום `level=ad` עם
breakdown יקר, פשוט קריאה נפרדת לכל רמה.

### 3.2 — Orchestration ב-`meta_service`

```python
async def _fetch_from_meta(
    campaign_id: UUID,
    date_preset: str,
) -> FullCampaignInsights:
    # 1. שלוף campaign מ-DB
    campaign = await fetch_campaign(campaign_id)
    if campaign.status != 'live':
        raise CampaignNotInsightsError("only live campaigns have insights")

    # 2. שלוף page_access_token דרך fb_service
    token = await fb_service.get_page_access_token(campaign.user_id)

    # 3. קריאה ל-Meta — שתי שאילתות מקבילות
    raw_campaign, raw_ads = await asyncio.gather(
        fetch_campaign_insights_raw(campaign.meta_campaign_id, token, date_preset),
        fetch_ads_insights_raw(campaign.meta_campaign_id, token, date_preset),
    )

    # 4. ספור leads מ-DB / Meta לפי type
    campaign_leads_count = await count_leads_for_campaign(
        campaign_id=campaign_id,
        type=campaign.type,
        date_range_start=raw_campaign.get('date_start'),
        date_range_end=raw_campaign.get('date_stop'),
    )

    # 5. בנה CampaignInsights
    campaign_insights = CampaignInsights(
        campaign_id=str(campaign_id),
        meta_campaign_id=campaign.meta_campaign_id,
        spend=float(raw_campaign['spend']),
        leads=campaign_leads_count,
        cpl=(raw_campaign['spend'] / campaign_leads_count) if campaign_leads_count > 0 else None,
        ctr=float(raw_campaign['ctr']),
        date_range_start=parse_meta_date(raw_campaign['date_start']),
        date_range_end=parse_meta_date(raw_campaign['date_stop']),
    )

    # 6. בנה AdInsights — לכל Ad בנפרד
    ads_leads_by_meta_id = await count_leads_per_ad(campaign_id=campaign_id)
    # → dict[meta_ad_id, count]

    ads_insights = []
    for raw_ad in raw_ads:
        meta_ad_id = raw_ad['ad_id']
        ad_db = await fetch_ad_by_meta_id(meta_ad_id)
        leads = ads_leads_by_meta_id.get(meta_ad_id, 0)
        ad_spend = float(raw_ad['spend'])

        ads_insights.append(AdInsights(
            ad_id=str(ad_db.id),
            meta_ad_id=meta_ad_id,
            angle=ad_db.angle,
            spend=ad_spend,
            leads=leads,
            cpl=(ad_spend / leads) if leads > 0 else None,
            ctr=float(raw_ad['ctr']),
        ))

    return FullCampaignInsights(
        campaign=campaign_insights,
        ads=ads_insights,
        fetched_at=datetime.now(timezone.utc),
        is_stale=False,
    )
```

---

## חלק 4 — ספירת leads — שתי דרכים

### 4.1 — קמפיין `lead` — מ-DB

```python
async def count_leads_for_campaign(
    campaign_id: UUID,
    type: Literal['lead', 'whatsapp'],
    date_range_start: datetime,
    date_range_end: datetime,
) -> int:
    if type == 'lead':
        return await count_leads_in_db(campaign_id, date_range_start, date_range_end)
    else:  # whatsapp
        return await count_leads_from_meta(campaign_id, date_range_start, date_range_end)


async def count_leads_in_db(
    campaign_id: UUID,
    start: datetime,
    end: datetime,
) -> int:
    """SELECT COUNT(*) FROM leads WHERE campaign_id=? AND created_at BETWEEN ?,?"""
```

ה-`created_at` ב-`leads` משמש כסינון לטווח הזמן. אינדקס `idx_leads_campaign_created`
שיצרנו ב-4.1 תומך בזה ישירות.

### 4.2 — קמפיין `whatsapp` — מ-Meta

```python
async def count_leads_from_meta(
    campaign_id: UUID,
    start: datetime,
    end: datetime,
) -> int:
    # שלוף מ-meta API את ה-actions עם
    # action_type='onsite_conversion.messaging_conversation_started_7d'
    # זה ה-event ש-Meta מדווחת על "שיחת WhatsApp נפתחה"
```

**הערה חשובה ל-CC:** ה-`action_type` הזה הוא placeholder. שווה לאמת
ב-Meta docs בזמן המימוש את ה-event type המדויק לוואטסאפ click-to-message.
ייתכן ש-Meta שינתה את השם.

### 4.3 — ספירת leads פר Ad

```python
async def count_leads_per_ad(campaign_id: UUID) -> dict[str, int]:
    """
    SELECT meta_ad_id, COUNT(*) FROM leads
    WHERE campaign_id=? AND meta_ad_id IS NOT NULL
    GROUP BY meta_ad_id

    Returns dict mapping meta_ad_id -> count.
    """
```

**Edge case:** ליד עם `meta_ad_id IS NULL` (אם איכשהו לא נשמר) — לא
ייספר ב-Ad level, אבל יספר ב-Campaign level. שווה ל-log warning אם זה
קורה.

ב-`whatsapp` campaign — מיפוי פר Ad דורש קריאה ל-Meta. ב-MVP, ניתן
להחזיר `[]` ל-`ads` ב-whatsapp campaigns ולא לדאוג ל-aggregation per-ad.
Phase 8 ידרוש את זה, אז להחליט אז.

---

## חלק 5 — Error handling

### 5.1 — סוגי exceptions

ב-`app/integrations/meta.py`:

```python
class CampaignNotInsightsError(Exception):
    """Campaign exists but isn't live (status != 'live')."""

class MetaTokenExpiredError(MetaError):
    """Page access token expired."""

class MetaPermissionError(MetaError):
    """Ad account suspended or no permissions."""

class MetaTransientError(MetaError):
    """Transient Meta API issue — timeout, 5xx, 429."""

class MetaServiceError(Exception):
    """Meta unreachable AND no cached value available."""
```

המסווג `classify_meta_error` הקיים (מ-1.2) צריך לדעת להבחין בין
transient ל-permanent. ב-4.5 פשוט מתרגמים את הסיווג ל-exception ספציפי.

### 5.2 — stale-while-error

נדון ב-2.2. כאשר Meta נכשלת ב-transient:
- אם יש cached value → החזר אותו עם `is_stale=True` + log warning.
- אם אין cached value → זרוק `MetaServiceError`.

**ל-permanent errors (token, permissions) — תמיד fail loud.** המשתמש
חייב לדעת על בעיות חיבור.

### 5.3 — מה ה-caller (Phase 7 / 8) עושה?

לא חלק מ-4.5, אבל שווה לציין:
- Phase 7 (סוכן): אם זרק `MetaTokenExpiredError`, הצ'אט יראה "החיבור ל-Meta פג, חבר מחדש".
- Phase 8 (cron): אם זרק `MetaServiceError`, ה-job ייכשל ויתועד. retry ב-cron הבא.

---

## חלק 6 — חישוב CPL — same-range

### 6.1 — העיקרון

`CPL = spend / leads` חייב להיות **באותו טווח זמן** ל-spend ול-leads.

- אם `date_preset='maximum'` (lifetime) → leads מ-תחילת הקמפיין.
- אם `date_preset='last_7d'` → leads מ-7 הימים האחרונים.

### 6.2 — איך מקבלים את הטווח?

Meta מחזירה ב-response של Insights שדות `date_start` ו-`date_stop`.
משתמשים בהם כקלט ל-`count_leads_for_campaign`.

```python
date_range_start = parse_meta_date(raw_campaign['date_start'])
date_range_end = parse_meta_date(raw_campaign['date_stop'])

leads = await count_leads_for_campaign(
    campaign_id, campaign_type,
    date_range_start, date_range_end,
)
```

### 6.3 — שונה מ-4.2 (אכיפת מכסה)

חשוב לבדל:
- **4.2 (ספירת מכסה):** משתמש ב-anniversary window (`current_period_start` ל-`+1 month`).
- **4.5 (Insights):** משתמש ב-Meta date_preset window.

שני subsystems שונים, שני חישובים שונים. **בכוונה.** ה-CPL ב-Insights
מודד "ביצועים", ה-quota מודד "צריכה". זה לא אותו דבר.

---

## חלק 7 — Done של 4.5

- `app/integrations/meta.py` הורחב עם `fetch_campaign_insights_raw` ו-
  `fetch_ads_insights_raw`.
- `app/services/meta_service.py` כולל `get_campaign_insights` (public)
  ו-`_fetch_from_meta` (private).
- in-memory cache עם TTL גמיש פר קריאה עובד.
- `lead` campaign → leads מ-DB; `whatsapp` campaign → leads מ-Meta
  (לפי `action_type` שיאומת).
- שתי רמות agg: Campaign + Ad. Ad Set מדולג.
- 4 exception classes מוגדרים. stale-while-error עובד.
- ה-CPL מחושב נכון על same-range לשני המקורות.
- אין endpoint ציבורי ב-4.5 — קריאה רק מ-Phase 7/8 internally.
- כל הטסטים עוברים.

## חלק 8 — לא ב-4.5

- Endpoint ציבורי (`GET /me/campaigns/{id}/insights`) — UI session.
- Proactive refresh ב-cron — Phase 8 או מאוחר יותר.
- מטריקות נוספות: impressions, reach, frequency, conversions — לא נדרשות
  ל-MVP.
- ניהול cache ב-DB / Redis — in-memory מספיק ב-MVP.
- agg ברמת Ad Set — מיותר ב-MVP.
- per-ad leads ב-`whatsapp` — דרוש Phase 8, נטופל אז.

---

## הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`action_type` ל-whatsapp לידים — צריך אימות מול Meta docs.**
   `onsite_conversion.messaging_conversation_started_7d` הוא ניחוש מבוסס
   ידע ישן. שווה לאמת ב-Meta API Explorer לפני המימוש.

2. **page_access_token — שליפה דרך `fb_service`, לא ישירות.** הטוקן
   מוצפן ב-Vault מ-1.2. `fb_service` יודע לפענח. אסור ל-`meta_service`
   לנגוע ב-Vault ישירות.

3. **`asyncio.gather` לשתי הקריאות במקביל** — חוסך ~500ms. שתי הקריאות
   ל-Meta הן independent.

4. **`fetch_ad_by_meta_id` יצטרך index** על `ads.meta_ad_id`. כדאי לוודא
   שזה קיים — אם לא, להוסיף ב-migration קטן ב-PR זה.

5. **`is_stale=True` חייב להיות גלוי ב-UI עתידי.** Phase 7 יציג איזה
   הסבר ("נתונים מ-10 דקות, Meta לא זמינה עכשיו"). 4.5 לא מטפל ב-UI,
   רק חושף את ה-flag.

6. **in-memory cache לא משותף בין web ו-worker process.** אם web process
   יש לו cache טרי, worker process עדיין יביא טרי שלו. זה בסדר ב-MVP —
   ה-cost של double fetch הוא נמוך.

7. **המסווג `classify_meta_error` מ-1.2 — צריך לוודא שהוא יודע ל-categorize
   את ה-error codes ש-Insights API מחזירה.** Marketing API messages
   קצת שונים מ-Graph API. אם יש פערים — להרחיב את המסווג, לא לכתוב חדש.

8. **ה-cache לא מתנקה אוטומטית.** ב-worker שרץ ימים על ימים, ה-cache
   יגדל מתמיד. ב-MVP זה לא בעיה (עשרות קמפיינים פעילים). בעתיד אפשר
   לאכוף `max_size` או TTL hard cleanup ב-cron.


### Session 4.5.1 — endpoint Insights ✅
- [x] `integrations/meta.py`: שליפת Spend, Leads, CPL, CTR פר-קמפיין
- [x] `meta_service.py`: לוגיקת שליפה + caching סביר (לא לקרוא ל-Meta בכל בקשה)

**Done:** המערכת שולפת את ארבעת השדות פר-קמפיין מ-Meta, מוכן להזנת הסוכן.

**הערות מימוש (סטיות מנומקות מהתכנון המקורי):**
- `get_campaign_insights(campaign_id, date_preset, max_age_seconds)` ב-meta_service — helper
  **internal** (אין endpoint HTTP; נדחה ל-session UI). in-memory cache + stale-while-error.
- **reuse של מסווג השגיאות הקיים** (MetaTransientError/Permanent/Unexpected) — לא נוצרו
  MetaTokenExpiredError/MetaPermissionError (כלל 10 — מסווג יחיד). token פג = Permanent code=190.
- **`lead_stats_service.py` חדש** (count leads בטווח + per-ad) — layering נקי, במקום count
  ישירות ב-meta_service.
- **user token** (`fb_service.get_decrypted_token`) ל-Insights, לא page token — ה-scope כולל
  ads_read; `get_page_access_token` שב-ROADMAP אינו קיים.
- **batch-fetch** של ads (מיפוי meta_ad_id→ad) במקום N+1 per-ad.
- **whatsapp** leads מ-`actions` (קבוע `_WHATSAPP_ACTION_TYPE` מתועד — **דורש אימות ידני מול
  Meta** לפני פרודקשן). per-ad ל-whatsapp נדחה ל-Phase 8.
- **אין migration** (idx_ads_meta_id קיים מ-0021; idx_leads_campaign_created מכסה count).

---

## Phase 4.6 · התראות מערכת ✅

> אחרי שהאירועים קיימים: קמפיין עלה (3.4), trial נגמר+חויב (2.6), 80% מכסה (4.2).
> **הושלם:** תשתית (4.6.1) + 3 triggers — campaign_live (4.6.2), billing_succeeded (4.6.3), quota_80/100 (4.6.4).


# Session 4.6 — System Notifications (Email)


## תיאור Session 4.6

**שלוש התראות, ערוץ אחד, handler אחד.** התראות מערכת ללקוח שולחות מייל ב-3 אירועים תפעוליים: (1) קמפיין עלה לאוויר, (2) חיוב חודשי הצליח, (3) חציית סף מכסה (80%/100%). כולן עוברות דרך טבלת `sent_notifications` (idempotency אטומי), נשלחות דרך Resend, ומעובדות ב-handler יחיד (`send_notification`). השלמת ה-jobs של 4.2 וחיבור ה-triggers מ-3.4 ומ-2.6.1.

**מה לא ב-Session הזה:**
- ערוץ וואטסאפ — דורש Phase 5.0 (provisioning קו וואטסאפ ללקוח). מייל בלבד ב-MVP.
- התראות הסוכן לבעלים — ערוץ נפרד שיתוכנן ב-Phase 8 (אופטימיזציה אוטונומית). זה דורש את `agent_alerts_quota` ושליחה לוואטסאפ של הבעלים, לא מייל. אסור לבלבל.
- UI היסטוריית התראות בדשבורד — אם יידרש, סשן UI נפרד.
- Unsubscribe flow אמיתי — קישור placeholder ב-footer, מימוש בעתיד.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | ערוץ ב-MVP | **מייל בלבד.** וואטסאפ דורש Phase 5.0. |
| 2 | ספק מייל | **Resend.** Developer experience טוב, API פשוט, free tier נדיב (3K/חודש, בלי limit יומי). |
| 3 | שפה | **עברית בלבד**, RTL. |
| 4 | תוכן | **HTML פשוט inline** עם header צבעוני ו-footer בסיסי. Jinja2 inline ב-Python. |
| 5 | Idempotency | **טבלה חדשה `sent_notifications`** עם UNIQUE constraint. מקור אמת אחד לכל הסוגים. |
| 6 | Retry policy | **default של 3.0** — 3 ניסיונות, backoff 1m/5m, אז `failed` + Sentry. |
| 7 | סדר עבודה | **handler-first.** PR אחד עם כל ה-sub-sessions. |
| 8 | flags ב-`subscriptions` מ-4.2 | **מבוטלים** — `sent_notifications` מספיק. נדרש עדכון ל-4.2 לפני שירוץ. |
| 9 | From & Reply-To | **`hello@<domain>` יחיד** ב-MVP — פשטות. אפשר להפריד ל-`notifications@` + `support@` בעתיד בלי שינוי קוד (env vars). |
| 10 | אטומיות יצירת התראה + job | **Transaction אחת** — INSERT ל-`sent_notifications` + INSERT ל-`jobs` ביחד. |
| 11 | At-least-once delivery | **מקובל ב-MVP** — אם handler קורס אחרי שליחה לפני UPDATE, retry ישלח שוב. עלות התראה כפולה נמוכה. |

---

## חלק 1 — תשתית (Sub-session 4.6.1) ✅

> **מומש (תשתית; חיבור ה-triggers הושלם ב-4.6.2-4 ✅):** migration 0030 (טבלת sent_notifications +
> RPC אטומי `create_notification_and_job` + הרחבת jobs.type), `integrations/email.py` (Resend דרך
> httpx + classify_resend_error), `email_templates.py` (Jinja2 **autoescape** — escaping כלל 6),
> `notification_service.py` (create/get/mark_sent/mark_failed + resolve_next_tier), handler
> `send_notification` ב-handlers.py. **סטיות מנומקות:** httpx ולא SDK resend (עקבי עם green_invoice);
> handler ב-handlers.py היחיד (לא package — הטיוטה לא תאמה את הריפו בפועל); builders ב-module נפרד;
> provider_message_id עמודה נוספת ל-debugging; resolve_next_tier basic/500→Basic 1000. 45 טסטים,
> ruff נקי. **אימות ידני לפני פרודקשן:** Resend Idempotency-Key header פעיל + דומיין שולח מאומת
> (אחרת sandbox onboarding@resend.dev).

### 1.1 — Migration 0020: טבלת `sent_notifications`

```sql
-- 0020_sent_notifications.sql
-- Phase 4.6 — idempotent notification log

BEGIN;

CREATE TABLE public.sent_notifications (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  notification_type text NOT NULL CHECK (notification_type IN (
    'campaign_live',
    'billing_succeeded',
    'quota_80',
    'quota_100'
  )),

  anchor_id text NOT NULL,

  channel text NOT NULL DEFAULT 'email' CHECK (channel IN ('email')),

  status text NOT NULL DEFAULT 'pending' CHECK (status IN (
    'pending',
    'sent',
    'failed'
  )),

  payload jsonb NOT NULL DEFAULT '{}'::jsonb,
  error_message text,

  created_at timestamptz NOT NULL DEFAULT now(),
  sent_at timestamptz,

  CONSTRAINT sent_notifications_unique_idempotency
    UNIQUE (user_id, notification_type, anchor_id)
);

-- אינדקסים
CREATE INDEX idx_sent_notifications_user_created
  ON public.sent_notifications (user_id, created_at DESC);

CREATE INDEX idx_sent_notifications_status_pending
  ON public.sent_notifications (created_at)
  WHERE status = 'pending';

-- RLS
ALTER TABLE public.sent_notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY sent_notifications_select_own
  ON public.sent_notifications FOR SELECT
  USING (auth.uid() = user_id);

-- INSERT/UPDATE/DELETE רק דרך admin client (server-authoritative, spec §7.3-ב)
-- אין policy → ברירת המחדל היא דחייה ל-authenticated

GRANT SELECT ON public.sent_notifications TO authenticated;

-- הוספת type חדש ל-jobs
ALTER TABLE public.jobs DROP CONSTRAINT IF EXISTS jobs_type_check;
ALTER TABLE public.jobs ADD CONSTRAINT jobs_type_check
  CHECK (type IN (
    'test_echo',
    'send_quota_alert',
    'send_notification'
    -- ... types נוספים שכבר קיימים, יישמרו כשמטמיעים
  ));

-- הערות
COMMENT ON COLUMN public.sent_notifications.anchor_id IS
  'מזהה ייחודי למניעת כפילות. קמפיין → campaign_id; חיוב → subscription_id:period_start; מכסה → subscription_id:period_start.';

COMMENT ON COLUMN public.sent_notifications.payload IS
  'נתונים לבניית התוכן (service_name, amount, threshold, used, quota). נשמר כדי שניתן לבנות מחדש בעת retry.';

COMMIT;
```

**הערות על הסכמה:**

ה-`UNIQUE` constraint על `(user_id, notification_type, anchor_id)` הוא ה-mechanism היחיד של idempotency. INSERT עם `ON CONFLICT DO NOTHING` יחזיר 0 שורות אם כבר נשלחה התראה מאותו סוג עם אותו anchor. שום מנגנון בקוד צריך לבדוק "האם נשלח?" — ה-DB אוכף.

`anchor_id` הוא `text` כדי לתמוך ב-anchor פשוט (UUID של קמפיין) ובמורכב (`{subscription_uuid}:{iso_period_start}` או `{subscription_uuid}:{iso_period_start}:{threshold}` — אם כי `threshold` כבר ב-`notification_type`, אז זה לא נדרש).

**אין composite FK** ל-`campaigns` או ל-`subscriptions`. הסיבה: anchor_id הוא soft-reference (text), והוא יכול להצביע לישות שנמחקה (קמפיין שנמחק לא ימחק את ההיסטוריה של "קמפיין עלה" שנשלחה בעבר — זה תיעוד היסטורי). RLS על `user_id` + cascade על `auth.users` מספיקים.

### 1.2 — `app/integrations/email.py` (Resend wrapper)

עטיפה מבודדת ל-Resend, לפי spec §2א חוק 3 (כל אינטגרציה חיצונית במודול משלה).

**API חיצוני אגנוסטי:**

```
send_email(
    to: str,
    subject: str,
    html_body: str,
    reply_to: str | None = None,
    idempotency_key: str | None = None,
) -> EmailSendResult
```

החזרה: `EmailSendResult` עם `success: bool`, `provider_message_id: str | None`, `error_type: Literal['transient', 'permanent'] | None`, `error_message: str | None`.

**env vars:**
- `RESEND_API_KEY` — חובה לפעולה, optional + degradation (בלי המפתח → log warning + skip, לא fail-on-startup, לפי החוק שננעל ב-3.1.5/3.1.6).
- `RESEND_FROM_EMAIL` — ברירת מחדל אם לא הוגדר אחרת בקריאה.
- `RESEND_REPLY_TO` — אופציונלי. אם הוגדר → נוסף ל-Reply-To header.

**מסווג שגיאות פנימי:**

`classify_resend_error(error) -> Literal['transient', 'permanent', 'unknown']` — אותו דפוס כמו `classify_meta_error` הקיים. מטא-קוד:
- **transient:** HTTP 429, 5xx, timeout, ConnectionError → ה-handler יזרוק ויקבל retry.
- **permanent:** HTTP 400 (bad request), 422 (validation — אימייל לא תקין), 403 (API key לא תקף) → ה-handler יסמן `failed` בלי retry.
- **unknown:** כל השאר → טיפול שמרני כ-permanent + log + Sentry.

**Resend Idempotency-Key:** Resend תומכת ב-header `Idempotency-Key` שמונע שליחה כפולה אם הקריאה נעשתה פעמיים. ה-`integrations/email.send_email` מקבל `idempotency_key` ומעביר ל-Resend. ה-handler ישתמש ב-`notification.id` כמפתח. **שווה לאמת מול תיעוד Resend בזמן המימוש** שה-header פעיל.

### 1.3 — `app/services/notification_service.py`

ה-service המרכזי. מחזיק את ה-admin client (server-authoritative, §7.3-ב). ה-router/handler לא קוראים ישירות לטבלה.

**Public API:**

```
create_notification(
    user_id: UUID,
    notification_type: NotificationType,
    anchor_id: str,
    payload: dict,
) -> NotificationCreateResult
```

מבצע **transaction אטומית**: INSERT ל-`sent_notifications` עם `ON CONFLICT (user_id, notification_type, anchor_id) DO NOTHING RETURNING id`, ואז INSERT ל-`jobs` עם `type='send_notification'` ו-`payload={notification_id}`.

החזרה: `NotificationCreateResult(created: bool, notification_id: UUID | None)`. אם `created=False` → התראה זהה כבר נוצרה בעבר (idempotency הסיק), לא יוצר job, חוזר שקט. הקורא מתעלם.

**שלוש פונקציות builder לתבניות HTML** (פנימיות, נקראות ע"י ה-handler):
- `build_campaign_live_email(payload: dict) -> tuple[subject, html]`
- `build_billing_succeeded_email(payload: dict) -> tuple[subject, html]`
- `build_quota_email(payload: dict, threshold: int) -> tuple[subject, html]`

כל אחת מקבלת payload שנשמר ב-`sent_notifications.payload` בעת ה-INSERT (כדי שניתן לבנות מחדש בעת retry).

**פונקציות עזר ל-handler:**
- `mark_notification_sent(notification_id: UUID, provider_message_id: str | None)` — UPDATE status='sent', sent_at=now().
- `mark_notification_failed(notification_id: UUID, error_message: str)` — UPDATE status='failed', error_message=error_message.

שתי הפעולות דרך admin client.

### 1.4 — `app/worker/handlers/send_notification.py`

ה-handler היחיד שמטפל בכל 4 ה-notification types.

**זרימה:**

1. שליפת ה-notification מ-DB דרך `notification_service.get_notification(notification_id)`.
2. אם `status='sent'` → exit (idempotency שכבה שנייה, מגן מ-bug ב-handler runner). אם `status='failed'` → exit (לא מנסים שוב על permanent).
3. בניית subject + html_body לפי `notification_type` — קריאה ל-`build_*_email(payload)`.
4. שליפת אימייל המשתמש דרך admin client (`auth.users.email`).
5. קריאה ל-`email.send_email(to, subject, html_body, idempotency_key=notification.id)`.
6. בדיקת `EmailSendResult`:
   - `success=True` → `mark_notification_sent(notification_id, provider_message_id)`. סוף.
   - `error_type='transient'` → `raise EmailTransientError(error_message)`. תשתית 3.0 תרים retry.
   - `error_type='permanent'` → `mark_notification_failed(notification_id, error_message)`. לא raise. job יסתיים כ-`done` (לא `failed`), כי ה-handler עצמו פעל תקין; הכשל מתועד ב-notification.

**הבחנה חשובה:** סטטוס ה-job (`done`/`failed`) לא זהה לסטטוס ה-notification (`sent`/`failed`). job=`failed` רק אם ה-handler קרס באופן בלתי צפוי. handler שטיפל ב-permanent error של Resend וסימן את ה-notification כ-`failed` → job=`done` (הוא עשה את העבודה, אין מה לנסות שוב).

**At-least-once delivery — תרחיש הקצה:**

worker crash אחרי `email.send_email` הצליחה, לפני `mark_notification_sent`. retry יקרא ל-`send_email` שוב עם אותו `idempotency_key` (notification.id). אם Resend מכבדת את ה-header → לא תישלח כפולה. אם לא — תישלח כפולה. עלות זניחה ב-MVP.

### 1.5 — תבניות HTML inline (Jinja2 ב-Python)

מבנה אחיד לכל 3 התראות:
- `<html dir="rtl" lang="he">` עם `<meta charset="utf-8">`.
- פונט Heebo מ-Google Fonts CDN (לעברית). fallback: `Arial, sans-serif`.
- מבנה: header צבעוני (רקע צבע מותג, לוגו טקסטואלי "Campaign AI"), body עם כותרת + פסקאות, footer פשוט עם זכויות יוצרים + placeholder ל-unsubscribe.
- inline CSS (לא `<style>`), כי לקוחות מייל מוחקים `<style>` חיצוני.
- רוחב מקסימלי 600px, mobile-friendly.

**נוסחים סופיים** (אושרו):

**`campaign_live`**:
- Subject: "הקמפיין '{service_name}' עלה לאוויר 🚀"
- Body: "הקמפיין '{service_name}' עלה לאוויר. 3 מודעות פעילות. עוקבים אחרי הביצועים ונעדכן בעוד 4 ימים."

**`billing_succeeded`**:
- Subject: "החיוב הראשון בוצע — Campaign AI"
- Body: "ה-trial הסתיים והחיוב הראשון בסך ₪{amount} בוצע. תודה. החשבונית נשלחה בנפרד."

**הערה:** המייל הזה יישלח גם לחיוב חודשי שגרתי (לא רק "ראשון"). הניסוח צריך לכסות את שני המקרים. הצעת ניסוח אחיד: "החיוב החודשי בסך ₪{amount} בוצע. החשבונית נשלחה בנפרד." את ההבחנה "ראשון" מול "חודשי" ניתן להעביר ב-payload (`is_first_charge: bool`) ולשנות subject בלבד.

**`quota_80`**:
- Subject: "התקרבת ל-80% מהמכסה החודשית"
- Body: "השתמשת ב-{used} מתוך {quota} לידים החודש (80%). שקול שדרוג ל-{next_tier}."

**`quota_100`**:
- Subject: "עברת את המכסה החודשית"
- Body: "עברת את המכסה החודשית ({quota} לידים). הלידים ימשיכו להיכנס, אבל הבוט יפסיק להגיב מעבר ל-{quota} (במנוי Premium). לקבלת שירות מלא — שדרוג למסלול גבוה יותר."

**`next_tier` mapping** (ב-`notification_service`):
- `basic` (500) → "Basic 1000" או "Premium 500"
- `basic` (1000) → "Premium 1000"
- `premium` (500) → "Premium 1000"
- `premium` (1000) → "צור קשר ל-Enterprise" (placeholder; אין tier מעל)
- `whatsapp` → לא רלוונטי, אין מכסה (לא יישלח quota notification בכלל)

---

## חלק 2 — חיבור triggers ✅

### 2.1 — Sub-session 4.6.2: trigger ל-"קמפיין עלה" מ-3.4

ב-`worker/handlers/push_campaign.py` (3.4), אחרי שהמעבר `pushing → live` הצליח (אחרי שלב 6 — Ad #3 נוצר ו-DB עודכן):

```
# בסוף ה-handler, אחרי UPDATE campaigns SET status='live':
await notification_service.create_notification(
    user_id=campaign.user_id,
    notification_type='campaign_live',
    anchor_id=str(campaign.id),
    payload={
        'service_name': campaign.service_name,
        'campaign_id': str(campaign.id),
    },
)
```

**Idempotency:** `anchor_id = campaign.id`. אם 3.4 יעלה את אותו קמפיין שוב (תרחיש כמעט בלתי אפשרי כי `status` כבר `live` והגנת CAS חוסמת push חוזר, אבל ליתר בטחון) → ה-notification לא ייצר כפול.

**מיקום הקריאה:** אחרי ה-UPDATE של campaigns ולפני סיום ה-handler. אם ה-`create_notification` נכשל → ה-handler ייכשל ו-3.0 יעשה retry. אבל ה-UPDATE כבר קרה, ה-cleanup cron של 3.4 לא יראה אותו כתקוע. אז retry יקרא ל-`create_notification` שוב, וה-UNIQUE constraint ימנע כפילות. עובד.

**הערה לבדיקה ב-CC:** אם ה-handler של 3.4 כתוב כך שהוא יוצא מיד אחרי UPDATE, יש לוודא שיש שכבה כלשהי שיכולה לטפל בכשל של `create_notification` בלי לבטל את ה-`live` של הקמפיין. עדיף לעטוף ב-`try/except` ולתעד ב-Sentry — קמפיין שעלה אבל לא נשלחה התראה הוא לא bug חוסם, רק חיסרון UX.

> **✅ מומש.** המיקום בפועל: `campaign_push_service.execute_push` (לא `worker/handlers/push_campaign.py` — ה-orchestration התאחד שם ב-3.4; ה-handler רק קורא ל-execute_push). הקריאה דרך helper `_notify_campaign_live(campaign_id, user_id, service_name)`. payload: `{service_name}` בלבד (כל מה ש-`build_campaign_live_email` צריך; `campaign_id` כבר ב-`anchor_id`).
>
> **⚠️ סטייה מודעת מהתכנון (שורה 4778) — best-effort, לא "זרוק→retry". אל תחזירו!** ה-"זרוק→retry" הנאיבי **שובר** כאן: ה-`except Exception` של `execute_push` קורא ל-`_handle_failure` שעושה **rollback** (מחיקת הקמפיין מ-Meta) על כל כשל permanent. `NotificationUnavailableError` אינה ב-`_is_transient` → הייתה מסווגת permanent → **rollback של קמפיין live תקין** בגלל תקלת DB בהתראה. לכן הקריאה היא **מחוץ** ל-try/except של ה-push, עטופה ב-try/except משלה שבולע (log + `capture_exception` עם campaign_id/user_id/notification_type) — בדיוק כפי שחזה ה-caveat למעלה, ועקבי עם `check_quota_thresholds` (4.2). מי שיחזיר ל-"זרוק→retry" בלי לטפל קודם ב-rollback path — יחזיר את ה-bug. ה-email עצמו עדיין מקבל retry דרך ה-job `send_notification` (רק ה-RPC `create_notification` הוא ה-best-effort, חלון צר). idempotency: `anchor_id=campaign_id` → ON CONFLICT, אין כפילות ב-crash-after-live.

### 2.2 — Sub-session 4.6.3: trigger ל-"חיוב מוצלח" מ-2.6.1

ב-cron החיוב היומי (2.6.1, handler שמטפל ב-trial→active ובחיוב חודשי שגרתי), אחרי שחיוב הצליח ו-`subscriptions.status` עבר ל-`active`:

```
# אחרי UPDATE subscriptions בעת חיוב מוצלח:
await notification_service.create_notification(
    user_id=subscription.user_id,
    notification_type='billing_succeeded',
    anchor_id=f"{subscription.id}:{subscription.current_period_start.isoformat()}",
    payload={
        'amount': float(charged_amount_ils),
        'is_first_charge': previous_status == 'trial',
        'period_start': subscription.current_period_start.isoformat(),
    },
)
```

**Idempotency:** `anchor_id = "{subscription_id}:{period_start_iso}"`. כל חיוב חודשי שונה לפי `current_period_start`, אז כל חודש מקבל התראה. retry על אותו חיוב לא ייצור כפול.

**הערה ל-CC לגבי `period_start` בעת איפוס:** `current_period_start` מתעדכן בעת ה-renewal עצמו (ראה 2.6.1 — `current_period_start = current_period_start + interval '1 month'`). חשוב שה-`create_notification` יקרא **אחרי** ה-UPDATE של `current_period_start`, כך ש-`anchor_id` יהיה של החלון החדש. אם נקרא לפני — נקבל anchor של החלון הקודם → התראה שני שתחזור בחודש הבא תזוהה כדופליקציה ולא תישלח. סדר הפעולות: UPDATE → create_notification.

> **✅ מומש.** המיקום בפועל: `subscription_service._charge_subscription` (ה-helper המשותף ל-3 פונקציות החיוב: charge_trial_end/monthly/past_due) — **לא** `billing_cron`, שרק מתזמן; ה-charge והמעבר ל-active קורים ב-_charge_subscription ושם כל ה-context. מקום אחד מכסה את שלושת ה-flows. הקריאה דרך helper `_notify_billing_succeeded`, **ליד** `issue_invoice_for_charge` (בתוך `if result.success:`), אחרי ה-finalize שכתב active + `current_period_start` החדש — עומד בדרישת ה-caveat למעלה (anchor של החלון החדש).
>
> **best-effort (לא "זרוק→retry"), נפרד מ-issue_invoice** (try/except משלו — כשל באחד לא ידלג על השני). הכסף כבר חויב וה-subscription active (בלתי-הפיך), אז כשל בהתראה אסור שישבור את החיוב (CLAUDE.md financial state) — בדיוק כמו issue_invoice_for_charge לידו. context: amount=`debit_total/100` (הסכום שחויב בפועל, עקבי עם החשבונית), is_first=`source_status=='trial'`, period_start=`new_cps`. payload: `{amount, is_first_charge}` (period_start כבר ב-anchor_id). idempotency: `anchor_id={sub_id}:{new_cps}`. edge: past_due→active (גם אחרי trial) → is_first=False (ROADMAP: prev=='trial' בלבד).

### 2.3 — Sub-session 4.6.4: החלפת ה-placeholder של מכסה מ-4.2

ב-4.2 (תוכנן, לא מומש) יש handler placeholder ל-`send_quota_alert` שרק לוגג. ב-4.6.4 אנחנו **מחליפים את כל ה-flow** — לא נשתמש ב-`send_quota_alert` כלל. במקום זה:

ב-`check_quota_thresholds` של 4.2 (נקרא אחרי INSERT ליד):

```
# במקום create_job('send_quota_alert', ...):
if status.is_over_limit:
    await notification_service.create_notification(
        user_id=user_id,
        notification_type='quota_100',
        anchor_id=f"{subscription.id}:{subscription.current_period_start.isoformat()}",
        payload={
            'used': status.used,
            'quota': status.quota,
            'tier': subscription.tier,
        },
    )
elif status.is_near_limit:
    await notification_service.create_notification(
        user_id=user_id,
        notification_type='quota_80',
        anchor_id=f"{subscription.id}:{subscription.current_period_start.isoformat()}",
        payload={
            'used': status.used,
            'quota': status.quota,
            'tier': subscription.tier,
            'next_tier': suggested_next_tier(subscription.tier, status.quota),
        },
    )
```

**Idempotency:** `anchor_id = "{subscription_id}:{period_start_iso}"`. quota_80 ו-quota_100 הם `notification_type` שונים, אז שניהם יכולים להישלח באותו חלון (אחד אחרי השני אם המשתמש חצה 80% ואז 100%). אבל אותו threshold לא יישלח כפול באותו חלון.

**הסרת ה-`send_quota_alert` job type:** ה-job type הזה היה אמור להיווצר ב-3.0. אם 3.0 כבר נכתב והוא ב-CHECK constraint, עדיף להשאיר אותו ב-enum (תאימות לאחור) ו-handler ריק / no-op בעבורו. אם 3.0 עוד לא נכתב — להסיר את ה-type מההצעה. **חשוב לבדוק את מצב 3.0 לפני יישום 4.6.**

> **✅ מומש.** המיקום: `check_quota_thresholds` (subscription_service) → `_notify_quota` → `create_notification(quota_80/quota_100)` ישירות. הוסרו `_claim_notification`, `_NOTIFIED_COLUMN`, `_enqueue_quota_alert`. `handle_send_quota_alert` → **no-op deprecated** (warning log עם user/threshold; נשאר לתאימות עם jobs ישנים בתור — 3.0 כבר ב-CHECK). anchor=`{sub}:{cps}`; payload=`{used, quota, next_tier}` (next_tier לשני הספים — `build_quota_email` צריך אותו גם ל-100, בניגוד לטיוטה למעלה). `get_quota_status`/`QuotaStatus` הורחבו ב-`subscription_id`+`tier` (לא נחשפים ב-API — `QuotaResponse` מפורש).
>
> **⚠️ עמודות מתות — `notified_80_at`/`notified_100_at` (0027) + ה-reset שלהן ב-finalize RPC (0029):** הקוד **הפסיק לכתוב/לקרוא** אותן ב-**Session 4.6.4 (2026-06-08)** — ה-idempotency עבר ל-`sent_notifications` UNIQUE (מקור יחיד; השארתן הייתה מחזירה את ה-bug "flag נתפס + create_notification נכשל → התראה אבודה"). נשארו ב-DB (CLAUDE.md Postgres 9: DROP נפרד אחרי שהקוד הפסיק). **ה-DROP העתידי (migration נפרד) חייב לכלול גם עדכון ל-`finalize_charge_attempt` RPC — להסיר את ה-`notified_80_at/100_at = NULL` reset (0029), אחרת ה-RPC ייכשל על עמודה שלא קיימת.** עד אז ה-reset הוא no-op בטוח (מאפס עמודות מתות).

---

## חלק 3 — תיקון מקדים ל-4.2 (התייתר — ראו הערה)

> **הערה (4.6.4):** חלק זה תוכנן בהנחה ש-4.2 לא מומש (אז ה-flags לא ייווצרו). בפועל 4.2 מומש **עם** ה-flags (migrations 0027/0029). לכן הגישה שיושמה: הקוד הפסיק להשתמש ב-flags (ראו ✅ למעלה), וה-DROP של העמודות נדחה ל-migration נפרד עתידי. הסעיפים למטה נשמרים כרקע היסטורי.

ה-Phase 4.2 תוכנן (HANDOFF: "רק תוכנן") אבל לא מומש. צריך לעדכן את מסמך 4.2 **לפני** שהוא ירוץ ב-CC, כך שהמיגרציה לא תיצור את העמודות `notified_80_at` ו-`notified_100_at` ב-`subscriptions`. סיבה: `sent_notifications` ב-4.6 תופס את אותה פונקציה, ושני מנגנוני idempotency באותה מערכת = drift.

**שינויים נדרשים במסמך של 4.2:**

1. **חלק 5 (Migration):** להסיר את ה-`ALTER TABLE subscriptions ADD COLUMN notified_80_at`, `notified_100_at`. אם המיגרציה לא רצה — פשוט להסיר את השורות. אם כן רצה — מיגרציה נפרדת ב-4.6 שמוחקת אותן.

2. **חלק 2.2 (`check_quota_thresholds`):** להחליף את הלוגיקה — במקום `if subscription.notified_80_at is None: ... await mark_notified(...)`, להשתמש ב-`notification_service.create_notification(...)` כפי שתואר ב-2.3 לעיל. ה-idempotency נאכף דרך `sent_notifications`, לא דרך flags.

3. **חלק 2.3 (`mark_notified`):** הפונקציה הזו **לא נדרשת**. למחוק לחלוטין.

4. **חלק 2.4 (איפוס ה-flags ב-renewal של 2.6.1):** **לא רלוונטי יותר.** אין flags לאפס. ה-anchor_id מבוסס על `current_period_start` שמתעדכן ממילא בעת renewal → חלון חדש = anchor חדש = שליחה חדשה אפשרית. אוטומטי.

5. **חלק 3 (jobs queue):** ה-job type `send_quota_alert` לא נדרש. הקריאה מ-`check_quota_thresholds` היא ישירות ל-`notification_service.create_notification`, וה-service יוצר את ה-job עם `type='send_notification'`. אם 4.2 מתאר את `send_quota_alert` כפלייסהולדר — להסיר את כל החלק הזה.

**ב-CC operationally:** אמיר ייקח את ה-patch הזה ויעדכן את מסמך 4.2 ב-gist לפני שיורה ל-CC לבצע אותו. או — אם 4.2 ו-4.6 ירוצו ב-PRs נפרדים — אפשר לעדכן את 4.2 רק ברגע שמתחילים אותו.

---

## חלק 4 — Env vars חדשים

| שם | משמעות | הערה |
|---|---|---|
| `RESEND_API_KEY` | מפתח API של Resend | optional + degradation → לוג אם חסר |
| `RESEND_FROM_EMAIL` | כתובת שולח (`hello@<domain>` ב-MVP) | optional + degradation |
| `RESEND_REPLY_TO` | כתובת Reply-To אופציונלית | optional, NULL = ללא Reply-To header |

`SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` כבר קיימים.

---

## חלק 5 — שמות הקבצים החדשים והשינויים

| קובץ | תוכן |
|---|---|
| `supabase/migrations/0020_sent_notifications.sql` | **חדש** — טבלת `sent_notifications` + RLS + extension ל-jobs type |
| `app/integrations/email.py` | **חדש** — Resend wrapper + classify_resend_error |
| `app/services/notification_service.py` | **חדש** — create_notification + builders + mark functions |
| `app/worker/handlers/send_notification.py` | **חדש** — handler יחיד ל-4 הסוגים |
| `app/models/notification.py` | **חדש** — Pydantic types: NotificationType enum, NotificationCreateResult, EmailSendResult |
| `app/templates/emails/*.html` | **חדש** (אופציונלי) — תבניות Jinja2 אם רוצים להפריד מהקוד. אפשר גם inline. |
| `app/worker/handlers/push_campaign.py` | **תיקון** — קריאה ל-create_notification בסוף ההצלחה |
| `app/worker/handlers/billing_cron.py` | **תיקון** — קריאה ל-create_notification אחרי חיוב מוצלח |
| `app/services/subscription_service.py` או היכן שמוגדר `check_quota_thresholds` | **תיקון** — החלפת לוגיקת ה-flags בקריאה ל-create_notification |
| מסמך `session-4.2-roadmap.md` | **תיקון מקדים** (לפני יישום 4.2) — להסיר flags ו-mark_notified, להחליף בקריאה ל-create_notification |

---

## חלק 6 — Done של 4.6

- מיגרציה 0020 רצה — `sent_notifications` קיימת עם UNIQUE constraint, RLS פעיל, `jobs.type` כולל `send_notification`.
- `integrations/email.py` עובד מול Resend (sandbox `onboarding@resend.dev` ב-development, דומיין אמיתי בייצור כשגיא קונה).
- `services/notification_service.create_notification` יוצר התראה + job בעסקה אחת. UNIQUE constraint מונע כפילויות.
- `worker/handlers/send_notification` שולח 4 סוגי התראות, מטפל ב-transient (retry) ו-permanent (mark failed).
- 3 ה-triggers מחוברים: 3.4 קורא ל-`create_notification('campaign_live')`, 2.6.1 קורא ל-`create_notification('billing_succeeded')`, 4.2 (אחרי תיקון) קורא ל-`create_notification('quota_80'/'quota_100')`.
- ניסיון לשליחת אותה התראה פעמיים (אותו `anchor_id`) → לא יוצר job שני, חוזר שקט.
- כשל transient ב-Resend (5xx) → 3 retries עם backoff → אם כל הכישלונות הם transient, סוף `failed`. כשל permanent (400) → mark failed מיד.
- מייל מגיע ללקוח (בדיקה ידנית במייל של אמיר/גיא): בעברית, RTL, HTML תקין ב-Gmail וב-Outlook.
- מסמך 4.2 עודכן ב-gist להסיר flags.
- כל הטסטים החדשים עוברים.

---

## חלק 7 — לא ב-4.6

- ערוץ וואטסאפ — דורש Phase 5.0 (provisioning קו) ו-`whatsapp_lines`.
- התראות הסוכן לוואטסאפ של הבעלים (תובנות אופטימיזציה) — Phase 8.
- UI היסטוריית התראות בדשבורד — UI session.
- Unsubscribe flow אמיתי — קישור placeholder ב-footer, לא פונקציונלי.
- A/B testing על subject lines — Post-MVP.
- Localization (אנגלית) — לא רלוונטי, ישראל בלבד.
- Tracking opens/clicks — Resend תומכת, אבל לא נדרש ב-MVP.
- התראת trial פג בלי חיוב (failed_billing) — שווה לחשוב, אבל לא ברשימת ה-3 שגיא ביקש. אם תידרש, להוסיף `notification_type='billing_failed'` ל-CHECK בעתיד.

---

## חלק 8 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **Resend API טרי — לאמת בזמן המימוש.** ה-API משתנה. ה-SDK הרשמי הוא `resend-python` (pip install resend). תיעוד: https://resend.com/docs. אם ה-`Idempotency-Key` header לא נתמך בגרסה הנוכחית, ה-MVP יחיה עם at-least-once delivery.

2. **`RESEND_FROM_EMAIL` בלי דומיין מאומת = sandbox.** אם הדומיין לא verified ב-Resend (SPF/DKIM/DMARC), הם מאלצים שליחה דרך `onboarding@resend.dev`. זה מספיק לפיתוח. ייצור דורש דומיין מאומת.

3. **`auth.users.email` הוא ה-source-of-truth של המייל.** אסור לשמור עותק ב-`subscriptions` או במקום אחר — drift. ה-handler שולף בכל קריאה. אם המשתמש החליף מייל ב-Supabase Auth → ההתראה הבאה תלך לחדש. עקבי.

4. **Idempotency שכבות:** שכבה ראשונה היא UNIQUE constraint ב-`sent_notifications` (מונע יצירת job כפול). שכבה שנייה היא בדיקת `status='sent'` בתחילת ה-handler (מונע שליחה כפולה אם איכשהו נוצר job כפול — לא אמור לקרות). שכבה שלישית היא `Idempotency-Key` של Resend (מונע שליחה מ-Resend עצמה ב-retry של handler). שלוש שכבות = ביטחון.

5. **`payload` נשמר ב-DB.** בעת retry, ה-handler בונה מחדש את ה-HTML מ-`payload`. אסור לבנות את ה-HTML בזמן `create_notification` ולשמור אותו — זה ייצור בעיות אם נשנה template (התראות retry יישלחו ב-template ישן). לכן payload גמיש, ה-HTML נבנה בעת ההגשה.

6. **CC חייב להוסיף ל-`worker/handlers/__init__.py`** את הרישום: `'send_notification': handle_send_notification`. ללא זה ה-runner לא ימצא את ה-handler.

7. **תזמון בדיקה ידנית:** אחרי המימוש, להריץ סדרה של בדיקות ידניות:
   - signup → push קמפיין (sandbox) → לוודא שמייל "קמפיין עלה" מגיע
   - גרירת `current_period_start` אחורה ידנית לקמפיין סנדבוקס + הרצת cron החיוב → לוודא מייל "חיוב מוצלח"
   - INSERT 400 לידים ידני לטבלת `leads` (משתמש Basic 500) → לוודא מייל 80%
   - INSERT עוד 100 → לוודא מייל 100%
   - INSERT עוד 50 → לוודא שאין מייל נוסף (idempotency)
   - חיכייה ל-renewal cycle (או הזזה ידנית של current_period_start) → INSERT 400 → לוודא מייל 80% חדש (חלון חדש)

8. **Sentry alerts על failed notifications.** ב-`mark_notification_failed` להוסיף `sentry_sdk.capture_message(...)` עם context. גם אם הסיווג הוא "permanent" וה-handler לא יזרוק, כדאי שמישהו ידע על כשל permanent (יכול להעיד על בעיית קונפיגורציה, אימייל לא תקין במשתמש, וכו').

9. **טבלת `sent_notifications` תגדל לאט.** ~3-4 התראות לחודש × 100 לקוחות = ~400 רשומות לחודש = ~5K רשומות לשנה. שום בעיית scaling ב-MVP. אם בעתיד יהיה צורך — cleanup cron של רשומות ישנות מ-12 חודשים (אבל זה גם תיעוד תפעולי, אז עדיף לשמור).

10. **בדיקת deliverability לפני בטא.** לפני שמשגרים ללקוח אמיתי הראשון: לשלוח את 3 ההתראות לחשבונות בדיקה ב-Gmail, Outlook, ו-Walla (הספק הישראלי הנפוץ ביותר אחרי Gmail). לוודא שאף אחת לא נופלת ל-spam. אם נופלת — בודקים SPF/DKIM/DMARC ב-Resend dashboard.


### Session 4.6.1 — התראות מערכת
- [x] תשתית: migration + email.py (Resend) + notification_service + handler send_notification
      + תבניות HTML (מייל בלבד; וואטסאפ נדחה ל-Phase 5.0). חיבור ה-triggers (קמפיין עלה /
      חיוב / מכסה) ב-Sub-sessions 4.6.2-4.

**Done:** אירועים תפעוליים שולחים התראה ללקוח (מייל/וואטסאפ).

---

## Phase 5 · בוט WhatsApp (Premium) — הכי מורכב, אחרון

### Session 5.0 — הקצאת קו וואטסאפ
- [ ] provisioning קו דרך 360dialog (בבטא: ידני / קו משותף)
- [ ] שמירת `phone_number_id` + credentials ב-`whatsapp_lines`
- [ ] רישום webhook לאותו מספר

**Done:** ללקוח Premium מוקצה קו, ה-credentials נשמרים, ה-webhook רשום.
**לא לעשות:** self-serve provisioning אוטומטי — זה דורש WABA ואישור מטא, מאוחר יותר.

# Session 5.0 — WhatsApp Line Provisioning (Manual)

> **עדכון להוספה ל-ROADMAP.md תחת Session 5.0.** Phase 5.0 — הקצאת קו WhatsApp ייעודי ללקוח Premium, במודל manual-provisioning לבטא (5-10 לקוחות). הקו מוקצה ידנית ע"י אדמין (אמיר/גיא) דרך Meta Business Manager UI, וה-Admin endpoint רק רושם את הקו ב-DB ומוודא שהוא תחת ה-WABA שלנו. תלוי ב-3.0 (jobs queue), 0.5 (auth + RLS), 2.6 (subscriptions). חוסם את כל Phase 5 המשך (5.1-5.4).

---

## תיאור Session 5.0

**מטרה:** לאפשר ללקוח Premium לקבל קו WhatsApp פעיל אחרי שאמיר/גיא ביצע provisioning ידני ב-Meta Business Manager. ה-Session **לא** עושה provisioning אוטומטי — זה Post-MVP (5.0.1). ה-Session כן מספק את התשתית ש-5.1-5.4 ייבנו עליה: טבלה `whatsapp_lines`, עטיפה ל-Meta Cloud API, Admin endpoint לרישום קו, ו-webhook handshake כדי ש-Meta יוכל להתחבר ל-webhook שלנו.

**מודל ה-WABA:** WABA יחיד שלנו (Campaign AI), N קווים תחתיו (אחד פר לקוח Premium). System User Token גלובלי. כל קו עם Display Name של הלקוח (העסק שלו) — לא של Campaign AI. הקוד תומך מ-day 1 ב-multi-line; ה-line context נשלף מ-`whatsapp_lines` לפי `user_id`.

**מה לא ב-Session הזה:**
- POST webhook handler (קליטת הודעות נכנסות מלידים) — Session 5.2.
- שליחת הודעות יוצאות (replies, follow-ups) — Session 5.2.
- Bot logic (RAG, conversation state, AI) — Session 5.3.
- WhatsApp Templates ו-pre-approved follow-up messages — Session 5.4.
- Provisioning אוטומטי של phone number ו-Display Name דרך Meta APIs — Post-MVP 5.0.1.
- Periodic sync של `name_status` ו-`quality_rating` מ-Meta (הערכים נלקחים פעם אחת ב-provision, לא מתעדכנים אוטומטית) — Post-MVP.
- UI ללקוח לראות את הקו שלו (admin בלבד ב-MVP, לקוח רואה דרך הדשבורד הקיים שמציג את ה-subscription tier).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מודל WABA | **WABA יחיד שלנו**, N phone numbers תחתיו. Display Name פר לקוח. |
| 2 | Provisioning | **Manual ב-MVP** — אדמין עושה ב-Meta Business Manager, ואז קורא ל-Admin endpoint שלנו לרישום. |
| 3 | Auth ל-Admin endpoint | **ENV var `ADMIN_USER_IDS`** עם רשימת UUIDs. ה-dependency שולפת user מ-JWT ובודקת חברות ברשימה. |
| 4 | ולידציה שהמספר תחת ה-WABA שלנו | **קריאה ל-`GET /{waba-id}/phone_numbers`** ואימות שה-`phone_number_id` שהאדמין הזין מופיע ברשימה. |
| 5 | קריאה ראשונית ל-Meta API לקבל פרטי הקו | **בעת provision_line** — שולפים `display_phone_number`, `verified_name`, `name_status`, `quality_rating` ושומרים ב-DB. אחרי זה לא נסנכרן (Post-MVP). |
| 6 | מה אם `name_status='pending'` בעת provision | **מאפשרים** — הקו נרשם כ-active, השדה pending נשמר. ב-WhatsApp הלקוח יראה "(Display Name pending)" עד שאושר. לא חוסם MVP. |
| 7 | מה אם `quality_rating='RED'` | **מאפשרים אבל מסמנים** — לא חוסם רישום, ה-UI של admin מציג alert. החלטה תפעולית של אדמין, לא חסימה אוטומטית. |
| 8 | קו אחד פר user או יותר | **קו אחד בלבד** — `UNIQUE(user_id)`. multi-line פר user לא נדרש ב-MVP. |
| 9 | webhook subscription ל-WABA | **חד-פעמי, ידני** דרך Meta Business Manager — לא בקוד. ה-Session רק חושף את ה-handshake endpoint כדי ש-Meta תוכל להתחבר. |
| 10 | סביבת dev | **Test phone number** של Meta (חינמי, ניתן אוטומטית עם WhatsApp Business Platform signup). שולח עד 5 נמענים מוגדרים מראש. מספיק לבדיקות 5.0-5.3. |

---

## חלק 1 — תשתית

### 1.1 — Migration: טבלת `whatsapp_lines`

```sql
-- 0021_whatsapp_lines.sql
-- Phase 5.0 — WhatsApp line provisioning (manual)

BEGIN;

CREATE TABLE public.whatsapp_lines (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,

  -- Meta identifiers
  phone_number_id text NOT NULL UNIQUE,
  display_phone_number text NOT NULL,
  display_name text NOT NULL,

  -- Meta-managed status (snapshot from time of provisioning)
  name_status text NOT NULL CHECK (name_status IN ('pending', 'approved', 'rejected')),
  quality_rating text NOT NULL DEFAULT 'UNKNOWN' CHECK (quality_rating IN (
    'GREEN', 'YELLOW', 'RED', 'UNKNOWN'
  )),

  -- Our status (independent of Meta)
  status text NOT NULL DEFAULT 'active' CHECK (status IN (
    'active', 'paused', 'disabled'
  )),

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- Auto-update updated_at
CREATE TRIGGER whatsapp_lines_updated_at
  BEFORE UPDATE ON public.whatsapp_lines
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();
-- ההנחה: `set_updated_at()` כבר קיים מ-Phase 0.5. אם לא — ליצור.

-- אינדקסים
CREATE INDEX idx_whatsapp_lines_user ON public.whatsapp_lines (user_id);
CREATE INDEX idx_whatsapp_lines_status ON public.whatsapp_lines (status) WHERE status = 'active';

-- RLS
ALTER TABLE public.whatsapp_lines ENABLE ROW LEVEL SECURITY;

CREATE POLICY whatsapp_lines_select_own
  ON public.whatsapp_lines FOR SELECT
  USING (auth.uid() = user_id);

-- אין policies של INSERT/UPDATE/DELETE → רק service_role דרך admin endpoints
-- (spec §7.3-ב — server-authoritative).

GRANT SELECT ON public.whatsapp_lines TO authenticated;

-- הערות
COMMENT ON COLUMN public.whatsapp_lines.phone_number_id IS
  'Meta phone_number_id (גלובלי) — המזהה לשליחה דרך Cloud API. UNIQUE כי קו אחד שייך ל-WABA אחד פר user.';

COMMENT ON COLUMN public.whatsapp_lines.display_phone_number IS
  'מספר טלפון בפורמט E.164 (+972...) — להצגה ב-UI ולוגים. **לא** מזהה השליחה (זה phone_number_id).';

COMMENT ON COLUMN public.whatsapp_lines.name_status IS
  'סטטוס אישור Display Name ב-Meta בעת provisioning. ב-MVP לא מסונכרן אחרי זה — sync תקופתי הוא Post-MVP.';

COMMENT ON COLUMN public.whatsapp_lines.status IS
  'הסטטוס שלנו, נפרד מ-Meta. active=מותר לשלוח. paused=הלקוח השהה (לא ב-MVP). disabled=השבתה ידנית של אדמין.';

COMMIT;
```

**הערות על הסכמה:**

ה-`UNIQUE(user_id)` ב-MVP מבטיח קו אחד פר לקוח. אם נצטרך בעתיד להעביר קו (החלפת מספר), נמחק ונרשום מחדש. multi-line פר user לא בתכנון.

ה-`UNIQUE(phone_number_id)` חיוני — Meta phone_number_id הוא מזהה גלובלי. אם אותו ID יצוץ פעמיים = bug ב-provisioning (אדמין הזין שגוי, או למישהו אחר כבר רשום).

**אין composite FK** ל-`subscriptions` (כלומר, אין אכיפה ב-DB שהמשתמש Premium). הוולידציה הזו ב-service layer בלבד. סיבה: tier יכול להשתנות (downgrade), והקו צריך להמשיך להתקיים (לסטטוס `paused`/`disabled`) ולא להימחק אוטומטית.

**`set_updated_at` trigger:** אם הפונקציה לא קיימת ב-0.5, להוסיף ב-0021:
```sql
CREATE OR REPLACE FUNCTION public.set_updated_at()
RETURNS trigger AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;
```

### 1.2 — `app/integrations/meta_whatsapp.py` (Meta Cloud API wrapper)

עטיפה מבודדת ל-Meta Cloud API, לפי spec §2א חוק 3 (כל אינטגרציה חיצונית במודול משלה). מודול זה ישמש גם 5.2/5.3 (שליחת הודעות) ו-5.4 (templates), אז ה-API שלו צריך להיות נקי.

**ENV vars:**
- `META_WABA_ID` — מזהה ה-WABA שלנו (תחתיו כל הקווים).
- `META_ACCESS_TOKEN` — System User Token, גלובלי. **לא** ENV optional + degradation במקרה הזה — אם חסר, ה-Service זורק error ב-startup. בלי טוקן אי אפשר לעבוד מול Meta.
- `META_APP_SECRET` — לחתימת webhook (יהיה בשימוש ב-5.2).
- `META_VERIFY_TOKEN` — לhandshake (בשימוש בסעיף 3).
- `META_GRAPH_API_VERSION` — אופציונלי, ברירת מחדל `v21.0`. שמירה כ-ENV מאפשרת bump גרסה בלי redeploy אם Meta דורשת.
- `META_GRAPH_BASE_URL` — אופציונלי, ברירת מחדל `https://graph.facebook.com`. שינוי רק לטסטים.

**Public API (מה ש-5.0 משתמש; 5.2-5.4 יוסיפו פונקציות):**

```
list_phone_numbers_in_waba() -> list[MetaPhoneNumber]
```
קוראת ל-`GET /{waba_id}/phone_numbers?fields=id,display_phone_number,verified_name,name_status,quality_rating`. מחזירה רשימה של dataclass `MetaPhoneNumber(id, display_phone_number, verified_name, name_status, quality_rating)`. שימוש: ולידציה ש-`phone_number_id` שהאדמין הזין באמת תחת ה-WABA.

```
get_phone_number_details(phone_number_id: str) -> MetaPhoneNumber | None
```
קוראת ל-`GET /{phone_number_id}?fields=display_phone_number,verified_name,name_status,quality_rating`. אם 404 → מחזיר None (המספר לא קיים או לא תחת WABA שאליו יש לנו גישה).

**`classify_meta_error(error) -> Literal['transient', 'permanent', 'auth', 'unknown']`** — אותו דפוס כמו `classify_resend_error` של 4.6. מטא-קוד:
- **auth:** HTTP 401, error code `190` (Invalid OAuth access token) → התרעת אדמין דחופה (הטוקן פג).
- **transient:** HTTP 429, 5xx, timeout, ConnectionError → retry-able.
- **permanent:** HTTP 400 (bad request), 404 → לא לנסות שוב, לסמן failed.
- **unknown:** כל השאר → טיפול שמרני כ-permanent + log + Sentry.

ב-5.0 משתמשים רק ב-permanent handling (provision_line כושל → return error לאדמין). retry לא נדרש ב-5.0 (פעולה סינכרונית מ-admin endpoint, אין job).

**הערה לעתיד:** ב-5.2 תיווסף `send_text_message(phone_number_id, to, body) -> message_id`, `send_template_message(...)`, `download_media(media_id)`. ב-5.4 — `submit_template(name, category, components) -> meta_template_id`, `list_templates() -> list`. כל הפונקציות יחלקו את אותה תשתית של auth/retry/error classification.

### 1.3 — `app/services/whatsapp_line_service.py`

ה-service המרכזי. מחזיק admin client (server-authoritative, §7.3-ב). ה-Admin endpoint לא קורא ישירות לטבלה.

**Public API:**

```
provision_line(
    user_id: UUID,
    phone_number_id: str,
    display_name: str,
) -> WhatsAppLineProvisionResult
```

זרימה:
1. שליפת ה-user מ-`auth.users` — אם לא קיים → `UserNotFoundError`.
2. שליפת ה-subscription של ה-user — אם לא `tier='premium'` או `status NOT IN ('trial', 'active')` → `NotEligibleError("רק לקוחות Premium פעילים יכולים לקבל קו WhatsApp")`. **הערה:** ב-`trial` כן מאפשרים — כדי שאפשר יהיה לרשום את הקו במהלך onboarding, ולא לחכות לחיוב הראשון.
3. בדיקה אם כבר יש לו קו → אם כן → `LineAlreadyExistsError(existing_line_id)`. אפשר להסיר ידנית דרך admin endpoint נפרד (Post-MVP) אם צריך להעביר מספר.
4. בדיקה ש-`phone_number_id` לא בשימוש בקו אחר → SELECT לבדיקה. אם כן → `PhoneNumberAlreadyUsedError`.
5. **ולידציה מול Meta** — קריאה ל-`meta_whatsapp.list_phone_numbers_in_waba()` ובדיקה שה-`phone_number_id` ברשימה. אם לא → `PhoneNumberNotInWabaError("המספר לא תחת ה-WABA שלנו — ודא ש-provisioning ידני בוצע במלואו")`.
6. שליפת פרטי הקו מ-Meta (מאותה תוצאה של list, או קריאה נפרדת אם נדרש field שלא ב-list) — `display_phone_number`, `verified_name`, `name_status`, `quality_rating`.
7. ולידציה ש-`verified_name` תואם ל-`display_name` שהאדמין הזין — אם לא → log warning אבל **לא חוסם**. ההנחה: Meta הוא source-of-truth ל-Display Name, האדמין אולי טעה בהזנה. שומרים את הערך מ-Meta (`verified_name`), לא מהאדמין.
8. INSERT ל-`whatsapp_lines` עם הערכים מ-Meta. status='active'.
9. החזרת `WhatsAppLineProvisionResult(line_id, display_phone_number, display_name, name_status, quality_rating, warnings)`. `warnings` הוא רשימה — לדוגמה: "Display Name pending approval — Meta tracks `verified_name` separately, may take 1-3 business days" אם `name_status='pending'`, או "Quality Rating is RED — review provisioning before sending traffic" אם `quality_rating='RED'`.

**פונקציות נוספות:**

- `get_line_for_user(user_id: UUID) -> WhatsAppLine | None` — שליפה לפי user. בשימוש ב-5.2/5.3 כשמטפלים בליד שצריך לדעת איזה קו לשלוח ממנו.
- `get_line_by_phone_number_id(phone_number_id: str) -> WhatsAppLine | None` — שליפה הפוכה. בשימוש ב-5.2 כשמקבלים webhook מ-Meta — ה-payload מגיע עם `phone_number_id` (לא user_id), וצריך לקשר חזרה.
- `update_status(line_id: UUID, new_status: Literal['active','paused','disabled']) -> WhatsAppLine` — אדמין בלבד. לא בשימוש ב-5.0 אבל מומלץ לבנות כבר עכשיו (10 שורות).

---

## חלק 2 — Admin endpoint

### 2.1 — `require_admin` dependency

ב-`app/dependencies/auth.py` (אם המודול קיים מ-0.5; אחרת ליצור).

הרעיון: dependency שמרחיב את ה-`require_authenticated_user` הקיים (שדואג ל-JWT validation) ומוסיף בדיקה שהמשתמש הוא admin.

```
ADMIN_USER_IDS env var format: comma-separated UUIDs
דוגמה: "uuid-1,uuid-2,uuid-3"
parsing: split על פסיק, strip whitespace, dedupe, validate UUID format
חישוב פעם אחת ב-startup (לא בכל request) — נשמר ב-set גלובלי

require_admin(user = Depends(require_authenticated_user)):
    if user.id not in ADMIN_USER_IDS_SET:
        raise HTTPException(403, "Admin access required")
    return user
```

**הערות:**
- אם `ADMIN_USER_IDS` ENV var לא מוגדר → ה-app רץ אבל **כל** admin endpoint יחזיר 403. זה safer default מ"לאפשר הכל".
- ולידציית UUID חשובה — אחרת string לא תקני ב-ENV יתפרש כ-admin (כי הוא לא יתאים לאף user.id).
- ל-bata: אמיר + גיא ב-ENV. אם מצטרף עוד אדמין → redeploy עם ENV מעודכן.

### 2.2 — `POST /api/v1/admin/whatsapp/lines`

ב-`app/routers/admin/whatsapp.py` (חדש).

**Auth:** `Depends(require_admin)`.

**Request body** (Pydantic schema):
```
class ProvisionLineRequest(BaseModel):
    user_id: UUID
    phone_number_id: str = Field(min_length=1, max_length=255)
    display_name: str = Field(min_length=1, max_length=200)
```

**Response (201 Created):**
```
class ProvisionLineResponse(BaseModel):
    line_id: UUID
    user_id: UUID
    display_phone_number: str
    display_name: str  # מהערך של Meta verified_name, לא מהקלט
    name_status: Literal['pending', 'approved', 'rejected']
    quality_rating: Literal['GREEN', 'YELLOW', 'RED', 'UNKNOWN']
    status: Literal['active', 'paused', 'disabled']
    warnings: list[str]  # ראה service.provision_line
```

**Error responses:**
- `403` — האדמין לא ב-`ADMIN_USER_IDS`.
- `404` — `user_id` לא קיים (`UserNotFoundError`).
- `409 Conflict` — קו כבר קיים ל-user, או `phone_number_id` בשימוש (`LineAlreadyExistsError`, `PhoneNumberAlreadyUsedError`).
- `422` — `phone_number_id` לא תחת ה-WABA שלנו (`PhoneNumberNotInWabaError`).
- `402` — ה-user לא Premium פעיל (`NotEligibleError`). שימוש ב-402 עקבי עם 3.2/3.3 (אין מנוי = 402).

**זרימה:**
1. Validation אוטומטי של Pydantic.
2. `whatsapp_line_service.provision_line(...)`.
3. החזרת ProvisionLineResponse.

**הערה לפיתוח:** ה-router הזה תחת `/admin/` — חשוב שכל הראוטרים תחת `/admin/` ידרשו `Depends(require_admin)` (אפשר ברמת router prefix). זה מקנה defense-in-depth: גם אם שכחנו להוסיף לendpoint ספציפי, ה-prefix דואג.

### 2.3 — `GET /api/v1/admin/whatsapp/lines` (אופציונלי, מומלץ)

list endpoint לאדמין לראות את כל הקווים הקיימים. שימושי לוודא ש-provisioning עבד, ולהבחין בקווים בעייתיים (quality_rating=RED, name_status=rejected).

לא קריטי ל-MVP, אבל 5 שורות קוד וחוסך SQL ידני.

---

## חלק 3 — Webhook handshake

### 3.1 — `GET /api/v1/webhooks/whatsapp`

ב-`app/routers/webhooks/whatsapp.py` (חדש).

**ללא auth** — זה endpoint ציבורי ש-Meta מתחבר אליו.

**Query params:**
- `hub.mode` — Meta שולחת "subscribe"
- `hub.verify_token` — Meta שולחת את הטוקן שהגדרנו ב-Meta Business Manager
- `hub.challenge` — string רנדומלי ש-Meta מצפה לקבל בחזרה

**זרימה:**
1. אם `hub.mode != "subscribe"` → 403.
2. אם `hub.verify_token != META_VERIFY_TOKEN` → 403.
3. אם תאים → להחזיר את `hub.challenge` כ-plain text, status 200.

**Response:** `PlainTextResponse(hub.challenge, status_code=200)`. **חשוב — לא JSON, plain text** — Meta מצפה לטקסט גולמי.

**הערה ל-CC:** ה-`POST` של אותו endpoint (קליטת הודעות מ-Meta) **לא ב-5.0**. ה-router יכיל רק `GET` ב-5.0. ב-5.2 נוסיף `POST` עם signature verification ו-message processing.

### 3.2 — הגדרה ידנית ב-Meta Business Manager (מחוץ לקוד)

אחרי deploy של 5.0, אמיר/גיא מבצעים פעם אחת ב-Meta Business Manager:
1. נכנסים ל-Meta Developer Console → App settings → WhatsApp → Configuration.
2. **Webhook URL:** `https://{render_url}/api/v1/webhooks/whatsapp`.
3. **Verify Token:** הערך של `META_VERIFY_TOKEN` (string רנדומלי ארוך).
4. לוחצים "Verify and Save" — Meta תקרא ל-GET שלנו, ה-handshake יעבוד, החיבור נשמר.
5. **Webhook fields:** סימון "messages" (יהיה בשימוש ב-5.2 לקבלת הודעות). יתר ה-fields לא נדרשים ב-MVP.

זה ההגדרה היחידה הנדרשת ברמת ה-WABA — לא חוזרים עליה פר לקוח. כל phone number תחת ה-WABA אוטומטית משתמש באותו webhook.

---

## חלק 4 — Env vars חדשים

| שם | משמעות | חובה? |
|---|---|---|
| `META_WABA_ID` | מזהה ה-WhatsApp Business Account שלנו | **חובה** — אם חסר, `provision_line` יזרוק error. |
| `META_ACCESS_TOKEN` | System User Token של ה-WABA, גלובלי | **חובה** — אם חסר, אף קריאה ל-Meta לא תעבוד. |
| `META_APP_SECRET` | חתימת webhook (`X-Hub-Signature-256`) | יידרש ב-5.2. ב-5.0 לא בשימוש פעיל, אבל מומלץ להגדיר כבר. |
| `META_VERIFY_TOKEN` | handshake token ל-webhook subscription | **חובה** — בלעדיו ה-handshake נכשל. |
| `META_GRAPH_API_VERSION` | גרסת Graph API (ברירת מחדל `v21.0`) | אופציונלי. |
| `META_GRAPH_BASE_URL` | Base URL (ברירת מחדל `https://graph.facebook.com`) | אופציונלי, לטסטים. |
| `ADMIN_USER_IDS` | UUIDs של אדמינים מופרדים בפסיק | **חובה** — אם חסר, כל admin endpoint יחזיר 403. |

הוספה ל-Render Environment Group של ה-API service.

---

## חלק 5 — שמות הקבצים החדשים והשינויים

| קובץ | תוכן |
|---|---|
| `supabase/migrations/0021_whatsapp_lines.sql` | **חדש** — טבלת `whatsapp_lines` + RLS + trigger updated_at |
| `app/integrations/meta_whatsapp.py` | **חדש** — Meta Cloud API wrapper (list_phone_numbers_in_waba, get_phone_number_details, classify_meta_error) |
| `app/services/whatsapp_line_service.py` | **חדש** — provision_line + get_line_for_user + get_line_by_phone_number_id + update_status |
| `app/models/whatsapp.py` | **חדש** — Pydantic: WhatsAppLine, MetaPhoneNumber, WhatsAppLineProvisionResult, ProvisionLineRequest, ProvisionLineResponse |
| `app/exceptions/whatsapp.py` | **חדש** — UserNotFoundError, NotEligibleError, LineAlreadyExistsError, PhoneNumberAlreadyUsedError, PhoneNumberNotInWabaError |
| `app/dependencies/auth.py` | **תיקון** — הוספת `require_admin` (אם המודול קיים מ-0.5; אחרת ליצור) |
| `app/routers/admin/whatsapp.py` | **חדש** — POST + GET /api/v1/admin/whatsapp/lines |
| `app/routers/webhooks/whatsapp.py` | **חדש** — GET /api/v1/webhooks/whatsapp (handshake) |
| `app/main.py` | **תיקון** — רישום של ה-routers החדשים |

---

## חלק 6 — Done של 5.0

- מיגרציה 0021 רצה — `whatsapp_lines` קיימת עם UNIQUE constraints, RLS פעיל, trigger updated_at עובד.
- `integrations/meta_whatsapp.py` עובד מול ה-WABA הבטא שלנו. בדיקה ידנית: `list_phone_numbers_in_waba()` מחזיר רשימה לא ריקה (לפחות test phone number של Meta).
- `services/whatsapp_line_service.provision_line(...)` עובד end-to-end עם user Premium fake (created via test fixture).
- POST `/api/v1/admin/whatsapp/lines` עובד עם curl מקומי + admin JWT. error cases מוחזרים נכון (403, 404, 409, 422, 402).
- GET `/api/v1/webhooks/whatsapp` עובר handshake — meta דרך Meta Business Manager מצליחה ב-"Verify and Save".
- ה-admin endpoint של list (`GET`) עובד ומחזיר את הקווים הקיימים.
- כל הטסטים החדשים עוברים.

---

## חלק 7 — לא ב-5.0

- POST webhook handler (קליטת הודעות מלידים) — 5.2.
- שליחת הודעות יוצאות — 5.2.
- Bot logic / RAG / state machine — 5.3.
- Templates + follow-up — 5.4.
- Provisioning אוטומטי (יצירת phone number ו-Display Name דרך Meta APIs) — Post-MVP 5.0.1.
- Periodic sync של name_status ו-quality_rating — Post-MVP.
- UI ללקוח לראות פרטי הקו — Post-MVP (אם נדרש; כרגע הלקוח רואה רק שיש לו Premium tier).
- חיוב נפרד פר line / metered billing — לא רלוונטי, ה-line כלול ב-Premium tier.
- העברת קו בין users — Post-MVP. ב-MVP: למחוק ולהוסיף.

---

## חלק 8 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **Meta API טריות — לאמת בזמן המימוש.** Meta Graph API משתנה. הגרסה הסטנדרטית נכון להיום היא v21.0 אבל שווה לבדוק https://developers.facebook.com/docs/graph-api/changelog בעת המימוש. אם יש breaking changes — לעדכן `META_GRAPH_API_VERSION` ENV.

2. **`META_ACCESS_TOKEN` הוא System User Token, לא User Access Token.** System User נוצר ב-Meta Business Manager תחת ה-WABA, ויש לו טוקן עם life ארוך (60 יום עם renewal אוטומטי, או "never expires" עם הגדרה ספציפית). User Access Token פג אחרי 60 יום בלי תזכורת — לא להשתמש. גיא צריך ליצור System User לפני שאמיר מתחיל בפיתוח.

3. **Test phone number של Meta.** סביבת dev — כל פעולת `list_phone_numbers_in_waba()` תחזיר אותו test phone number (חינמי, שולח ל-5 נמענים מוגדרים מראש שאמיר/גיא הוסיפו). זה מספיק ל-5.0-5.3. ל-5.4 ו-לבטא אמיתי — צריך production phone number אחרי App Review.

4. **Idempotency של provision_line.** אם האדמין קורא פעמיים עם אותם נתונים — שתי הקריאות יוצרות `LineAlreadyExistsError` ב-409, לא יוצרות כפול. זה צפוי וטוב.

5. **CC צריך להוסיף ל-`app/main.py`** את רישום ה-routers:
```
app.include_router(admin_whatsapp_router, prefix="/api/v1/admin/whatsapp", tags=["admin", "whatsapp"])
app.include_router(webhooks_whatsapp_router, prefix="/api/v1/webhooks/whatsapp", tags=["webhooks"])
```

6. **Verification flow ידני (אחרי deploy):**
   - אמיר/גיא יוצרים phone number ידנית ב-Meta Business Manager תחת ה-WABA.
   - מגישים Display Name (לוקח 1-3 ימי עסקים).
   - מקבלים את ה-`phone_number_id` מ-Meta Business Manager UI.
   - קוראים ל-POST `/api/v1/admin/whatsapp/lines` עם user_id של הלקוח, phone_number_id, ו-display_name.
   - ה-API מאמת מול Meta, רושם ב-DB, מחזיר את הפרטים.
   - הקו פעיל מבחינתנו. ב-5.2 ההודעות יתחילו לזרום.

7. **התרעת אדמין על `quality_rating='RED'`.** ב-MVP — warning ב-response של provision_line. אם רוצים אקטיבי יותר — Sentry message ב-`services.provision_line` כשהערך RED. לא חוסם, רק מתעד.

8. **טסטים מומלצים (לא חובה ל-MVP):**
   - test_provision_line_success — user Premium, phone_number_id תחת WABA, אין קונפליקטים → מצליח.
   - test_provision_line_user_not_premium → 402.
   - test_provision_line_phone_not_in_waba → 422.
   - test_provision_line_duplicate_user → 409.
   - test_provision_line_duplicate_phone_number_id → 409.
   - test_webhook_handshake_valid_token → 200 + challenge.
   - test_webhook_handshake_invalid_token → 403.
   - test_admin_endpoint_without_admin_user_id → 403.

9. **לא לכלול ב-`META_ACCESS_TOKEN` בלוגים.** ה-token יחשף בלוגי אינטגרציה אם לא זהירים. ב-`meta_whatsapp.py` להוסיף `classify_meta_error` הגנה — sanitize שגיאות מ-API לפני log. הדפוס מ-4.6 (sanitize של `sk-` keys ב-Resend) עובד כאן עם regex על Bearer tokens.

10. **לא לסנכרן name_status/quality_rating אחרי provisioning.** בכוונה. Meta מעדכנת אותם בקצב משלה, וב-MVP אנחנו לא צריכים לדעת בזמן אמת. אם מישהו (אדמין או לקוח) מבחין שהסטטוס לא תואם ל-Meta — לבצע sync ידני (UPDATE direct ב-DB, או admin endpoint שזה Post-MVP). Periodic sync אוטומטי = Post-MVP, לא במסגרת MVP.

---

## נספח: מה משתמע לסשנים הבאים

5.0 מספק את התשתית. סשנים הבאים יסתמכו עליה:

- **5.1 (Bot configuration UI):** הלקוח יראה בדשבורד את ה-`whatsapp_lines.display_phone_number` שלו ויידע שזה הקו שאליו הלידים יכתבו. ה-Display Name נקבע ב-provisioning, לא ב-UI של הלקוח (Meta-managed).
- **5.2 (Inbound webhook + outbound):** ה-POST handler ישתמש ב-`get_line_by_phone_number_id` כדי לקשר webhook נכנס ל-user. שליחה יוצאת תקרא ל-`get_line_for_user` כדי לדעת מאיזה phone_number_id לשלוח.
- **5.3 (Bot logic):** האמור משתמש ב-`whatsapp_lines` רק כדי לדעת איזה line קיים — שאר הלוגיקה (RAG, conversation state) נטועה בטבלאות חדשות שייווצרו ב-5.2/5.3.
- **5.4 (Templates):** templates יוגשו ברמת ה-WABA (לא פר line), אז יש טבלת `whatsapp_templates` גלובלית — לא תלויה ב-`whatsapp_lines`.

ה-decoupling הזה מבטיח שמודל ה-provisioning יכול להשתנות (5.0 manual → 5.0.1 automated) בלי לגעת בשאר Phase 5.

---

### Session 5.1 — הגדרת בוט (flow שלב 11)
- [ ] שמירת `bot_configs` (הודעת פתיחה, פעולת סיום) + `bot_questions`
      (עד 5)
- [ ] זמין רק ל-tier=premium

**Done:** מנוי Premium יכול להגדיר בוט; הנתונים נשמרים נכון.
> **מיקום ב-flow:** הגדרת הבוט מופיעה **אחרי טופס הליד, בתוך הקמת הקמפיין** (שלב 11) — לא שלב מוקדם. מותנה `tier=premium`. **סדר הבנייה לא משתנה** (הבוט עדיין נבנה ב-Phase 5, אחרון) — רק מיקומו ב-flow.

# Session 5.1 — Bot Configuration (Backend)

> **עדכון להוספה ל-ROADMAP.md תחת Session 5.1.** Phase 5.1 — תשתית backend להגדרת בוט הסינון של הלקוח Premium: הודעת פתיחה, עד 5 שאלות סינון, ופעולת סיום. תלוי ב-5.0 (`whatsapp_lines` קיימת) ובpatch של 5.0 שהופך את `phone_number_id` ל-nullable ומוסיף state `pending_provisioning`. חוסם את 5.2 (webhook handler צריך לדעת מה לשאול ואיך לסיים) ו-5.3 (AI logic צריך את ה-`expected_answer_hint`).

---

## תיאור Session 5.1

**מטרה:** לאפשר ללקוח Premium להגדיר את הבוט שלו דרך API — הודעת פתיחה, עד 5 שאלות סינון עם רמזים סמנטיים ל-AI, ופעולת סיום (Calendly / העברה אנושית / תיאום תור — האחרון חסום עד 5.5). בשמירה הראשונה — נוצרת אוטומטית שורת `whatsapp_lines` במצב `pending_provisioning`, שאדמין ישלים ב-Meta Business Manager.

**מודל קונספטואלי:** בוט אחד לכל user (account-level, לא campaign-level). Premium tier הוא account-wide ולכן גם הבוט. אם בעתיד יידרש multi-bot — שינוי schema (`UNIQUE(user_id)` יהפוך ל-`UNIQUE(user_id, campaign_id)`), בלי שבירת לוגיקה.

**מה ב-Session הזה:**
- Migration לטבלאות `bot_configs` ו-`bot_questions`.
- `GET /api/v1/bot/config` — שליפה.
- `PUT /api/v1/bot/config` — upsert אטומי של config + שאלות, יחד עם auto-create של `whatsapp_lines` row אם אין.
- ולידציה מלאה: תקרת שאלות, אורכים, פורמט E.164, פורמט URL, fallback_action תלוי-ערך.
- Premium gating (402).

**מה לא ב-Session הזה:**
- UI / frontend — session נפרד.
- Webhook handler לקליטת הודעות נכנסות מ-WhatsApp — 5.2.
- AI לפירוש תשובות חופשיות של ליד — 5.3.
- תיאום תורים בפועל (Google Calendar integration) — 5.5.
- Follow-up אחרי 23.5 שעות — 5.4.
- Interpolation של משתנים בהודעת פתיחה (`[שם]`, `[שירות]`) — מתבצע ב-5.3 בעת שליחה.
- מסך admin לרשימת `pending_provisioning` lines — Post-MVP (אפשר SQL ידני).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | בוט פר-user או פר-קמפיין? | **פר-user** — `UNIQUE(user_id)` ב-`bot_configs`. account-level. |
| 2 | endpoint אחד או מספר? | **endpoint אחד אטומי** (`PUT /bot/config`) — upsert של config + שאלות בעסקה אחת. DELETE+INSERT לשאלות (ה-UI שולח את הסט המלא). |
| 3 | יצירת `whatsapp_lines` | **אוטומטית בשמירה ראשונה** של bot_config. INSERT באותה transaction עם `status='pending_provisioning'`, `phone_number_id=NULL`. תלוי ב-patch של 5.0. |
| 4 | `expected_answer_hint` | **שדה אופציונלי** פר שאלה — רמז סמנטי ל-AI ב-5.3, לא ולידציה קשיחה. |
| 5 | `fallback_action` values | שלושה: `calendly_link`, `human_handoff`, `bot_schedule_appointment`. השלישי קיים ב-CHECK אבל ה-service דוחה אותו ב-422 עד 5.5. |
| 6 | משתנים בהודעת פתיחה | נשמרים as-is. interpolation ב-5.3. ולידציה ב-5.1 רק על שמות מוכרים (warning, לא error). |
| 7 | Premium gating | **402** אם לא `tier='premium'` ו-`status IN ('trial','active')`. אותו דפוס כ-3.2/3.3. |
| 8 | `default_appointment_duration_minutes` | נכנס כבר עכשיו, default 60, range 15-240. לא בשימוש ב-5.1 (הערך `bot_schedule_appointment` חסום), אבל forward-compatible. |

---

## חלק 1 — Migration

### 1.1 — קובץ `0022_bot_config.sql`

המספר ייקבע לפי המיגרציות הקיימות. אם 5.0 כבר תפס 0021, אז 5.1 הוא 0022. אם CC השאיר רווח — לבדוק `ls supabase/migrations/`.

```sql
-- 0022_bot_config.sql
-- Phase 5.1 — Bot configuration backend for Premium users

BEGIN;

-- ============================================================
-- 1. טבלת bot_configs
-- ============================================================

CREATE TABLE public.bot_configs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,

  -- הודעת פתיחה (תיווצר תוך 30 שניות מקליטת הליד)
  opening_message text NOT NULL CHECK (
    length(trim(opening_message)) BETWEEN 10 AND 1500
  ),

  -- פעולת סיום
  fallback_action text NOT NULL CHECK (fallback_action IN (
    'calendly_link',
    'human_handoff',
    'bot_schedule_appointment'  -- חסום ב-service עד 5.5
  )),

  -- ערך לפעולת הסיום:
  -- calendly_link → URL
  -- human_handoff → טלפון E.164 של בעל העסק
  -- bot_schedule_appointment → NULL (לא בשימוש ב-5.1)
  fallback_value text,

  -- ל-5.5 — לא בשימוש ב-5.1
  default_appointment_duration_minutes int NOT NULL DEFAULT 60 CHECK (
    default_appointment_duration_minutes BETWEEN 15 AND 240
  ),

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TRIGGER bot_configs_updated_at
  BEFORE UPDATE ON public.bot_configs
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

CREATE INDEX idx_bot_configs_user ON public.bot_configs (user_id);

-- RLS — SELECT-only למשתמש (כתיבה דרך admin client בלבד, spec §7.3-ב)
ALTER TABLE public.bot_configs ENABLE ROW LEVEL SECURITY;

CREATE POLICY bot_configs_select_own
  ON public.bot_configs FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.bot_configs TO authenticated;

-- ============================================================
-- 2. טבלת bot_questions
-- ============================================================

CREATE TABLE public.bot_questions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  bot_config_id uuid NOT NULL REFERENCES public.bot_configs(id) ON DELETE CASCADE,

  -- composite FK לקוהרנטיות RLS (spec §7.2 — defense against service_role leak)
  CONSTRAINT bot_questions_config_user_fk
    FOREIGN KEY (bot_config_id, user_id)
    REFERENCES public.bot_configs(id, user_id)
    ON DELETE CASCADE,

  question_text text NOT NULL CHECK (
    length(trim(question_text)) BETWEEN 5 AND 500
  ),

  -- רמז סמנטי ל-AI ב-5.3 (לא ולידציה קשיחה על תשובת הליד)
  expected_answer_hint text CHECK (
    expected_answer_hint IS NULL OR
    length(trim(expected_answer_hint)) BETWEEN 1 AND 300
  ),

  order_index int NOT NULL CHECK (order_index BETWEEN 1 AND 5),

  created_at timestamptz NOT NULL DEFAULT now(),

  -- מבטיח ייחודיות סדר פר config + max 5 שאלות (CHECK + UNIQUE יחד)
  CONSTRAINT bot_questions_unique_order UNIQUE (bot_config_id, order_index)
);

-- composite UNIQUE על (id, user_id) — נדרש ל-composite FK
ALTER TABLE public.bot_configs
  ADD CONSTRAINT bot_configs_id_user_unique UNIQUE (id, user_id);

CREATE INDEX idx_bot_questions_config
  ON public.bot_questions (bot_config_id, order_index);

CREATE INDEX idx_bot_questions_user
  ON public.bot_questions (user_id);

-- RLS — SELECT-only למשתמש
ALTER TABLE public.bot_questions ENABLE ROW LEVEL SECURITY;

CREATE POLICY bot_questions_select_own
  ON public.bot_questions FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.bot_questions TO authenticated;

-- ============================================================
-- 3. הערות תיעוד
-- ============================================================

COMMENT ON TABLE public.bot_configs IS
  'הגדרת בוט סינון WhatsApp פר Premium user. אחד לכל user (UNIQUE).';

COMMENT ON COLUMN public.bot_configs.opening_message IS
  'הודעה שתישלח לליד תוך 30 שניות מקליטה. תומכת במשתנים [שם], [שירות], [שם_עסק] — interpolation ב-5.3.';

COMMENT ON COLUMN public.bot_configs.fallback_action IS
  'פעולת סיום אחרי כל השאלות. calendly_link/human_handoff פעילים ב-MVP; bot_schedule_appointment חסום עד 5.5.';

COMMENT ON COLUMN public.bot_configs.fallback_value IS
  'תלוי ב-fallback_action: URL ל-calendly_link, טלפון E.164 ל-human_handoff, NULL ל-bot_schedule_appointment.';

COMMENT ON COLUMN public.bot_questions.expected_answer_hint IS
  'רמז סמנטי ל-LLM ב-5.3 — מה התשובה הצפויה. למשל "מספר בין 1000 ל-100000" או "כן/לא". אופציונלי.';

COMMENT ON COLUMN public.bot_questions.order_index IS
  'סדר השאלה (1-5). UNIQUE פר bot_config_id מבטיח שלא יהיו שתי שאלות באותו סדר.';

COMMIT;
```

### 1.2 — הערות על הסכמה

**`UNIQUE(user_id)` ב-`bot_configs`:** מאכף "בוט אחד לכל user" ברמת ה-DB. אם בעתיד נרצה multi-bot (פר קמפיין) — הסרת ה-UNIQUE והוספת `campaign_id` עם UNIQUE composite. שינוי מקומי, לא שובר.

**Composite FK על `(bot_config_id, user_id)`:** דפוס מ-spec §7.2 (כמו ב-`quiz_responses`, `lead_form_fields`, `ads`). מגן מ-`service_role` leak — אם איכשהו admin client ינסה לכתוב `bot_question` עם user_id שלא תואם ל-config, ה-FK ייכשל. דורש `UNIQUE(id, user_id)` על הטבלה האב.

**`UNIQUE(bot_config_id, order_index)`:** מבטיח ש-2 שאלות לא יוכלו לתפוס את אותו סדר. בשילוב עם CHECK `order_index BETWEEN 1 AND 5` → מקסימום 5 שאלות פר config, נאכף ב-DB.

**אין `priority` או `is_required`:** הפרוטוטייפ מציג שאלות כסדרתיות חובה. אם בעתיד יידרש "שאלה אופציונלית" — תוספת עמודה, לא שינוי שובר.

**SELECT-only RLS:** כל הכתיבות (PUT) עוברות דרך service ב-admin client (spec §7.3-ב). המשתמש לא יכול לעשות INSERT/UPDATE/DELETE ישירות דרך PostgREST. ההיגיון בקוד, לא ב-RLS.

---

## חלק 2 — Service layer

### 2.1 — `app/services/bot_config_service.py` (חדש)

מחזיק admin client. ה-router קורא אליו, לא נוגע בטבלאות.

#### Public API

```
get_config(user_id: UUID) -> BotConfigResponse | None
```

זרימה: שליפת `bot_configs` + `bot_questions` ב-2 שאילתות (או JOIN אחד). מחזיר `None` אם אין config. RLS אוכף בשליפה (user client) — או admin client + WHERE user_id = ? בלבד.

```
upsert_config(
    user_id: UUID,
    opening_message: str,
    fallback_action: Literal['calendly_link', 'human_handoff', 'bot_schedule_appointment'],
    fallback_value: str | None,
    default_appointment_duration_minutes: int = 60,
    questions: list[BotQuestionInput],
) -> BotConfigResponse
```

זרימה אטומית בעסקה אחת (`BEGIN ... COMMIT`):

1. **Premium gating** — `subscription_service.require_premium_active(user_id)`. אם נכשל → `NotPremiumError` → 402.
2. **ולידציה של `fallback_action` תלוי-ערך** (פירוט בחלק 3).
3. **ולידציה של `bot_schedule_appointment`** — אם זה הערך → `FeatureNotAvailableError("תיאום תורים יהיה זמין בעדכון הבא")` → 422.
4. **upsert של bot_configs:**
   - אם אין שורה ל-user → INSERT.
   - אם יש → UPDATE.
   - בכל מקרה — תופסים את ה-`bot_config.id` המוחזר.
5. **DELETE + INSERT לשאלות:**
   - `DELETE FROM bot_questions WHERE bot_config_id = ?`
   - `INSERT` שאלות חדשות לפי הסדר שהתקבל מהקלט (`order_index = 1..N`).
   - לפני INSERT — בדיקת max 5 (נאכף גם ב-CHECK של DB, אבל error ידידותי מהservice).
6. **auto-create של `whatsapp_lines`** (אם זו שמירה ראשונה):
   - בדיקה: האם יש שורה ב-`whatsapp_lines` ל-user.
   - אם לא → INSERT עם `status='pending_provisioning'`, `phone_number_id=NULL`, `display_phone_number=NULL`, `display_name=NULL`, `name_status='pending'`, `quality_rating='UNKNOWN'`.
   - **תלוי ב-patch של 5.0** שהופך את `phone_number_id` ו-`display_phone_number` ל-nullable.
7. **COMMIT** — או rollback אם משהו נכשל.
8. שליפה חוזרת + החזרת `BotConfigResponse`.

**הערה חשובה על patch של 5.0:** ה-patch צריך להוסיף `display_phone_number`, `display_name`, ו-`name_status` ל-nullable. בשמירה הראשונה אין לנו את הערכים האלה — Meta מחזירה אותם רק ברגע ה-provisioning. אם 5.0 כבר מומש כ-NOT NULL — `bot_config_service` לא יוכל ליצור את ה-line. **חובה לסנכרן** עם אמיר אם 5.0 כבר ב-CC או לא.

#### Helper פנימי

```
_validate_fallback(action, value) -> ValidationResult
```

מקבץ את הוולידציה של `fallback_action` + `fallback_value`. מחזיר תוצאה מובנית או זורק `ValidationError` עם הודעה ידידותית בעברית. פירוט בחלק 3.

```
_validate_opening_message(text) -> list[str]
```

מחזיר רשימת warnings (לא errors) על משתנים לא מוכרים בהודעת פתיחה. למשל אם המשתמש כתב `[שם_מלא]` במקום `[שם]` — נשמר as-is אבל מוחזר warning שיוצג ב-UI. ה-API מחזיר את הקונפיג + רשימת warnings.

### 2.2 — Pydantic models

ב-`app/models/bot.py` (חדש):

```python
from typing import Literal
from uuid import UUID
from datetime import datetime
from pydantic import BaseModel, Field, field_validator, model_validator

FallbackAction = Literal['calendly_link', 'human_handoff', 'bot_schedule_appointment']

class BotQuestionInput(BaseModel):
    question_text: str = Field(min_length=5, max_length=500)
    expected_answer_hint: str | None = Field(default=None, max_length=300)

class BotConfigInput(BaseModel):
    opening_message: str = Field(min_length=10, max_length=1500)
    fallback_action: FallbackAction
    fallback_value: str | None = Field(default=None, max_length=500)
    default_appointment_duration_minutes: int = Field(default=60, ge=15, le=240)
    questions: list[BotQuestionInput] = Field(min_length=0, max_length=5)

    @model_validator(mode='after')
    def validate_questions_unique(self):
        # לא לאכוף ייחודיות טקסט — שתי שאלות עם אותו טקסט יכולות להיות לגיטימיות
        # (למשל מקרה edge בעריכה). הסדר מוגדר ע"י המיקום במערך.
        return self

class BotQuestionResponse(BaseModel):
    id: UUID
    question_text: str
    expected_answer_hint: str | None
    order_index: int

class BotConfigResponse(BaseModel):
    id: UUID
    opening_message: str
    fallback_action: FallbackAction
    fallback_value: str | None
    default_appointment_duration_minutes: int
    questions: list[BotQuestionResponse]
    warnings: list[str] = Field(default_factory=list)
    created_at: datetime
    updated_at: datetime
```

**הערה על `warnings`:** מוחזר ב-200, לא 422. ה-UI מציג אותם ב-toast צהוב, לא חוסם שמירה. רק errors קשיחים (פורמט E.164 לא תקני, action=bot_schedule_appointment) מחזירים 422.

---

## חלק 3 — ולידציה מפורטת

### 3.1 — Opening message

**Errors (422):**
- ריק / רק רווחים.
- פחות מ-10 תווים אחרי trim.
- יותר מ-1500 תווים.

**Warnings (200 עם רשימת warnings):**
- משתנה לא מוכר. רשימת המשתנים המוכרים: `[שם]`, `[שירות]`, `[שם_עסק]`. כל `[XXX]` אחר → warning `unknown_variable: [XXX]`.
- אין משתנים בכלל. warning: `no_variables` — "ההודעה ללא משתנים — היא תהיה זהה לכל ליד." (לא error כי לפעמים זה לגיטימי.)

### 3.2 — Fallback action + value

לוגיקה תלוית-ערך:

**`calendly_link`:**
- `fallback_value` חובה — אם NULL/empty → 422 "נדרש קישור Calendly".
- ולידציית URL — `https://` קידומת, host תקין. ב-MVP regex פשוט; אם Calendly מספק SDK לוולידציה — שווה לבדוק.
- אורך מקסימלי 500.

**`human_handoff`:**
- `fallback_value` חובה — אם NULL/empty → 422 "נדרש מספר טלפון של בעל העסק".
- ולידציית E.164: `+972XXXXXXXXX` (9-15 ספרות אחרי `+`).
- שימוש: ב-5.2 כשהבוט מסיים את הסינון, פרטי הליד נשלחים למספר הזה ב-WhatsApp. **שונה** מקו ה-WhatsApp של Campaign AI (זה המספר של בעל העסק שמקבל את ה-handoff).

**`bot_schedule_appointment`:**
- אם `fallback_action = 'bot_schedule_appointment'` → 422 `FeatureNotAvailableError("תיאום תורים יהיה זמין בעדכון הבא")`.
- `fallback_value` צריך להיות NULL (תיאום תורים לא דורש value סטטי — הוא דינמי).
- ב-5.5 הערך הזה ייפתח לשימוש.

### 3.3 — Questions

**Errors (422):**
- יותר מ-5 שאלות → "מקסימום 5 שאלות סינון".
- שאלה אחת קצרה מ-5 תווים אחרי trim → "שאלה {N} קצרה מדי".
- שאלה אחת ארוכה מ-500 תווים → "שאלה {N} ארוכה מדי".
- `expected_answer_hint` ארוך מ-300 → "רמז לשאלה {N} ארוך מדי".

**אין error** אם:
- 0 שאלות (לקוח יכול לבחור לדלג על סינון — הבוט פותח עם הודעת פתיחה ועובר ישר ל-fallback_action). שווה לאמת מול גיא, אבל הגיוני לאפשר. אם גיא יחליט שחובה לפחות שאלה אחת — שינוי ל-`min_length=1` ב-Pydantic.

### 3.4 — Premium gating

**מבנה הבדיקה ב-`subscription_service.require_premium_active(user_id)`:**

1. שליפת subscription של ה-user.
2. אם אין → `SubscriptionNotFoundError` → 402.
3. אם `tier != 'premium'` → `NotPremiumError("פיצ'ר זה זמין רק למסלול Premium")` → 402.
4. אם `status NOT IN ('trial', 'active')` → `SubscriptionNotActiveError("המנוי אינו פעיל")` → 402.

**הערה ל-CC:** הפונקציה כבר אמורה להיות קיימת אחרי 3.3 (שם הוסיפו את `require_paid_access`). אם היא `require_paid_access` בלבד (לא בודקת tier=premium ספציפית), צריך להוסיף `require_premium_active` כפונקציה נפרדת או פרמטר.

---

## חלק 4 — API Endpoints

### 4.1 — `GET /api/v1/bot/config`

**Auth:** `Depends(require_authenticated_user)`.

**זרימה:**
1. שליפת user_id מה-JWT.
2. `bot_config_service.get_config(user_id)`.
3. אם `None` → 404 `{"detail": "bot config not found"}`.
4. אחרת → 200 `BotConfigResponse`.

**הערה:** אין Premium gating ב-GET. אם user לא Premium ולא יצר config מעולם → 404 (תקין). אם המשתמש שדרג ל-Basic אחרי שהיה Premium ויצר config — ה-GET יחזיר אותו (נשמר ב-DB). זה לא דליפת מידע (זה שלו), אבל ה-UI יסתיר בהתאם ל-tier.

### 4.2 — `PUT /api/v1/bot/config`

**Auth:** `Depends(require_authenticated_user)`.

**Request body:** `BotConfigInput`.

**זרימה:**
1. שליפת user_id מה-JWT.
2. Pydantic validation אוטומטי.
3. `bot_config_service.upsert_config(user_id, **body)`.
4. החזרה: 200 `BotConfigResponse` (כולל `warnings`).

**Error responses:**

| Status | Exception | מתי |
|---|---|---|
| 402 | `NotPremiumError` / `SubscriptionNotActiveError` | user לא Premium פעיל |
| 422 | `ValidationError` (Pydantic) | פורמט קלט שגוי |
| 422 | `FeatureNotAvailableError` | `fallback_action='bot_schedule_appointment'` |
| 422 | `InvalidFallbackValueError` | URL/E.164 לא תקין |
| 503 | `MetaTransientError` | (לא רלוונטי ב-5.1, אבל אם 5.0 service יוקרא במעלה הקריאה — לכסות) |
| 500 | unhandled | log + Sentry |

**הערה ל-CC:** אין endpoint נפרד ל-DELETE. אם user רוצה "לאפס" — שולח PUT עם `questions: []` ועם opening_message חדש. אם רוצה למחוק את הbot config לגמרי (use case לא ברור ב-MVP) — Post-MVP.

### 4.3 — אין endpoint נפרד לשאלות

החלטה מודעת. שאלות הן חלק מ-`bot_config` (לא יישות עצמאית), ולכן לא נחשפות ב-CRUD נפרד. ה-UI שולח את הסט המלא של השאלות בכל PUT, ה-service עושה DELETE+INSERT. פשוט יותר ל-frontend, פחות API surface.

יתרון נוסף: idempotent. ניסיון 2 של PUT עם אותו payload → אותה תוצאה.

---

## חלק 5 — אינטגרציה עם 5.0

### 5.1 — תלות ב-patch של 5.0

**ה-patch של 5.0 שדרוש לפני 5.1:**

1. הוספת state `pending_provisioning` ל-CHECK constraint של `whatsapp_lines.status` (כברירת מחדל החדשה).
2. הפיכת `phone_number_id` ל-nullable (`text NULL UNIQUE`).
3. הפיכת `display_phone_number` ל-nullable.
4. הפיכת `display_name` ל-nullable.
5. ערכי ברירת מחדל ל-`name_status` ו-`quality_rating` שמתאימים למצב pending (למשל `name_status='pending'`, `quality_rating='UNKNOWN'`).

**אם 5.0 כבר רץ ב-CC:** צריך migration נפרד (`0022_fix_whatsapp_lines_for_pending.sql`) שמשנה את העמודות. אז 5.1 הופך ל-`0023_bot_config.sql`.

**אם 5.0 עדיין לא רץ:** אמיר מעדכן את 5.0 לפני שמורידים ל-CC, ו-5.1 הוא `0022`.

**אקשן לאמיר:** לבדוק מצב 5.0 ב-CC לפני שמורידים את 5.1.

### 5.2 — Auto-create של `whatsapp_lines`

הקוד ב-`bot_config_service.upsert_config`:

```python
# בתוך ה-transaction של upsert_config:
existing_line = await whatsapp_line_service.get_line_for_user(user_id)
if existing_line is None:
    await whatsapp_line_service.create_pending_line(user_id)
```

**`create_pending_line` הוא פונקציה חדשה ב-`whatsapp_line_service`** (תוספת ל-5.0):

```python
async def create_pending_line(user_id: UUID) -> WhatsAppLine:
    """
    יוצר שורת whatsapp_lines במצב pending_provisioning.
    נקרא מ-bot_config_service בשמירה ראשונה של bot_config.
    אדמין משלים את ה-provisioning ב-Meta Business Manager + admin endpoint.
    """
    return await admin_client.from_('whatsapp_lines').insert({
        'user_id': str(user_id),
        'status': 'pending_provisioning',
        'phone_number_id': None,
        'display_phone_number': None,
        'display_name': None,
        'name_status': 'pending',
        'quality_rating': 'UNKNOWN',
    }).execute()
```

**הערה:** ה-admin endpoint של 5.0 (`POST /admin/whatsapp/lines`) צריך לעבור התאמה — במקום ליצור שורה חדשה, הוא **מעדכן** שורה קיימת מ-`pending_provisioning` ל-`active`. שינוי ל-5.0 ה-admin flow:

```
לפני: POST /admin/whatsapp/lines → INSERT
אחרי: PATCH /admin/whatsapp/lines/{user_id} → UPDATE (phone_number_id, display_phone_number, status='active', ...)
```

זה patch נוסף ל-5.0 שצריך להחיל. **שווה לעדכן את ה-HANDOFF.**

### 5.3 — מה קורה אם ה-INSERT של `whatsapp_lines` נכשל

ה-INSERT הוא חלק מה-transaction של `upsert_config`. אם נכשל — כל ה-transaction מתבטלת, ה-bot_config לא נשמר, המשתמש מקבל 500.

זה רצוי: ה-invariant הוא "Premium user עם bot_config → יש לו שורה ב-whatsapp_lines". בלי השורה, 5.2 לא יידע איך לשלוח לליד.

**Edge case אחד שצריך לחשוב עליו:** מה אם user היה Premium, יצר bot_config, ירד ל-Basic, ועכשיו עולה חזרה ל-Premium ועושה PUT שני? `whatsapp_lines` כבר קיים — ה-`get_line_for_user` יחזיר אותו, `create_pending_line` לא יקרא. עובד נכון.

---

## חלק 6 — ENV vars

אין ENV vars חדשים ב-5.1. כל מה שצריך כבר קיים מ-0.5/5.0.

---

## חלק 7 — שמות הקבצים החדשים והשינויים

| קובץ | תוכן |
|---|---|
| `supabase/migrations/0022_bot_config.sql` | **חדש** — bot_configs + bot_questions + RLS + composite unique |
| `app/services/bot_config_service.py` | **חדש** — get_config + upsert_config + ולידציה |
| `app/services/whatsapp_line_service.py` | **תיקון** — הוספת `create_pending_line(user_id)` |
| `app/services/subscription_service.py` | **תיקון** (אם נדרש) — הוספת `require_premium_active(user_id)` אם לא קיים |
| `app/models/bot.py` | **חדש** — Pydantic: BotConfigInput, BotQuestionInput, BotConfigResponse, BotQuestionResponse, FallbackAction |
| `app/exceptions/bot.py` | **חדש** — NotPremiumError (אם לא ב-subscription), FeatureNotAvailableError, InvalidFallbackValueError |
| `app/routers/bot.py` | **חדש** — GET + PUT /api/v1/bot/config |
| `app/main.py` | **תיקון** — רישום של ה-router |
| `app/routers/admin/whatsapp.py` | **תיקון** — שינוי מ-POST ליצירה ל-PATCH לעדכון (אם 5.0 כבר רץ — patch נפרד) |

---

## חלק 8 — Done של 5.1

- מיגרציה 0022 רצה — `bot_configs`, `bot_questions` קיימים עם UNIQUE constraints, RLS פעיל, composite FK עובד.
- patch של 5.0 יושם (phone_number_id nullable, pending_provisioning state).
- `whatsapp_line_service.create_pending_line(user_id)` עובד.
- `bot_config_service.upsert_config(...)` עובד אטומית — bot_config + שאלות + whatsapp_line (אם ראשון).
- `subscription_service.require_premium_active(user_id)` עובד (קיים או נוסף).
- `GET /api/v1/bot/config` מחזיר 200 עם config או 404 אם אין. RLS אוכף.
- `PUT /api/v1/bot/config` עובד:
  - Premium user → 200 + warnings.
  - Basic user → 402.
  - `fallback_action='bot_schedule_appointment'` → 422.
  - URL לא תקין → 422.
  - טלפון לא ב-E.164 → 422.
  - יותר מ-5 שאלות → 422 (גם דרך CHECK ב-DB, גם דרך Pydantic).
- DELETE+INSERT של שאלות עובד אטומית (התראת PUT שמסיר שאלה → השאלה הישנה נמחקת באמת).
- בשמירה ראשונה — `whatsapp_lines` נוצר אוטומטית עם `pending_provisioning`.
- בשמירה חוזרת — לא נוצרת שורה כפולה ב-`whatsapp_lines`.
- כל הטסטים עוברים.

---

## חלק 9 — לא ב-5.1

- UI של ה-frontend (אשף קמפיין / מסך עריכה נפרד).
- Webhook handler לקליטת הודעות נכנסות — 5.2.
- AI לפירוש תשובות חופשיות של ליד — 5.3.
- Interpolation אמיתי של משתנים בהודעת פתיחה — 5.3 (בעת שליחה).
- Templates של WhatsApp — 5.4.
- Follow-up 23.5h — 5.4.
- תיאום תורים בפועל — 5.5.
- UI ל-admin לראות רשימת `pending_provisioning` lines — אפשר SQL ידני ב-MVP, UI ב-Post-MVP.
- Versioning של bot_config (היסטוריית שינויים) — לא נדרש ב-MVP.
- A/B testing של הודעות פתיחה שונות — Post-MVP.

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **תלות קריטית ב-patch של 5.0.** אם 5.0 כבר ב-CC עם `phone_number_id NOT NULL` — חובה migration נפרד לפני 5.1, אחרת `create_pending_line` ייכשל. אמיר חייב לבדוק מצב 5.0 לפני להעביר את 5.1 ל-CC.

2. **`require_premium_active` — לבדוק שהפונקציה קיימת ב-`subscription_service`.** אחרת CC צריך להוסיף אותה. אותו דפוס כמו `require_paid_access` הקיים מ-3.3, רק מוסיף בדיקת `tier='premium'`.

3. **Composite UNIQUE על `bot_configs (id, user_id)`:** דרוש ל-composite FK של `bot_questions`. אם נשכח — ה-FK ייכשל ב-migration. אותו pattern כמו ב-`campaigns (id, user_id)` מ-2.2.

4. **DELETE+INSERT לשאלות — לא UPDATE.** הסיבה: ה-UI שולח את הסט המלא, וזה pattern פשוט ו-idempotent. אם בעתיד נרצה לשמור על UUIDs של שאלות לצורך אנליטיקס (כמה שאלות פר-שאלה ענו) — שינוי. ב-MVP לא נדרש.

5. **`expected_answer_hint` הוא רמז ל-AI, לא ולידציה.** אסור שה-service ינסה לאמת תשובה של ליד מולו ב-5.1 או 5.3. הוא נקרא רק כקלט ל-prompt של LLM ב-5.3.

6. **משתנים בהודעת פתיחה — נשמרים as-is.** הוולידציה ב-5.1 רק מוציאה warnings על שמות לא מוכרים. ה-interpolation עצמו (`[שם] → "דני"`) קורה ב-5.3 כשהבוט שולח את ההודעה לליד הספציפי. אם נעשה את ה-interpolation ב-5.1 (בעת שמירה) — לא יהיה לנו שם של ליד עדיין.

7. **`fallback_value` עם 500 תווים זה אורך גדול ל-URL.** כיסוי לקצוות. Calendly URLs עד ~100 תווים בדרך כלל.

8. **טסטים מומלצים:**
   - test_get_config_not_found → 404.
   - test_put_config_basic_user → 402.
   - test_put_config_premium_first_time → 200 + whatsapp_lines נוצר.
   - test_put_config_premium_second_time → 200 + לא נוצרת שורה כפולה ב-whatsapp_lines.
   - test_put_config_with_appointment_action → 422.
   - test_put_config_calendly_invalid_url → 422.
   - test_put_config_handoff_invalid_phone → 422.
   - test_put_config_6_questions → 422.
   - test_put_config_unknown_variable → 200 + warning.
   - test_put_config_delete_question → שאלה ישנה לא חוזרת.
   - test_atomic_rollback_on_whatsapp_lines_failure → אם create_pending_line נכשל, bot_config לא נשמר.

9. **Race condition לא ריאלי ב-MVP:** משתמש שלוחץ PUT פעמיים במהירות. ב-MVP — אחרון מנצח. אם בעתיד יהיה צורך ב-optimistic locking — `updated_at` כ-version.

10. **CC חייב להוסיף ל-`app/main.py`:**
    ```python
    app.include_router(bot_router, prefix="/api/v1/bot", tags=["bot"])
    ```

---

## חלק 11 — אקסטרא: מה קורה ב-flow אחרי 5.1

לפי גיא (בשיחה ב-WhatsApp):

```
לקוח Premium → אשף קמפיין:
  שאלות עומק → טון → תקציב+מיקום → קהל (גילאים+מין) →
  טופס לידים → ★ הגדרת בוט (5.1) ★ → 3 קופי + 3 תמונות → פרסום
```

ה-bot config = שלב לפני האחרון באשף הקמפיין. ה-UI של 5.1 (frontend session נפרד) צריך:
- להופיע אוטומטית אחרי שלב טופס הלידים.
- להציג spinner "הבוט בהכנה (1-3 ימי עסקים)" אחרי שמירה, עד שאדמין משלים provisioning.
- לאפשר עריכה מאוחרת מ-sidebar "הגדרות הבוט" — אותם endpoints, מסך נפרד.

ב-Backend אין הבדל בין "ראשונה" ל"עריכה מאוחרת" — שניהם PUT אטומי.

---

**סוף מסמך 5.1.**

---

### Session 5.2 — webhook הודעות נכנסות + מכונת מצבים
- [ ] webhook מ-360dialog/Meta Cloud (עם אימות + idempotency)
- [ ] ניהול `bot_state` / `bot_current_question` / `bot_answers` ב-`bot_conversations` (פר-`contact_key`), לא ב-`leads`
- [ ] כלל הצמדה מפורש: הודעה נכנסת ממספר → השיחה הפעילה של אותו מספר. קישור לליד-אירוע בפעולת הסיום.

**Done:** הודעה נכנסת מקדמת את מצב השיחה נכון.
**זהירות:** state-machine patterns — מעברי מצב לא תקינים, הודעה כפולה.

# Session 5.2 — Webhook נכנס, מכונת מצבים, ושליחה יוצאת

> **עדכון להוספה ל-ROADMAP.md תחת Session 5.2.** ה-Session הגדול והקריטי של Phase 5. מאחד את 5.0 (קו פיזי קיים, `whatsapp_lines.status='active'`) ו-5.1 (קונפיגורציה — הודעת פתיחה, שאלות, פעולת סיום) ל-flow חי שעובד מקצה לקצה: ליד נכנס דרך טופס Meta → הבוט פונה תוך 30 שניות → סבב שאלות → סיום (Calendly link / handoff למספר בעל העסק). תלוי בכל מה שקדם ב-Phase 5, ב-4.1 (lead intake), ובתשתית 3.0 (jobs queue). חוסם את 5.3 (אינטליגנציה ב-AI) ו-5.4 (templates ו-follow-up). מאחר ש-5.2 בונה את התשתית, אבל **לא** את ה-templates עצמם — בייצור אמיתי הבוט לא ישלח עד 5.4. בסביבת dev עם ה-test phone number של Meta — עובד מקצה לקצה.

---

## תיאור Session 5.2

**מטרה:** לבנות את שלוש השכבות המתואמות שמרכיבות את ה-runtime של הבוט. אני אסביר כל אחת בנפרד כי שלושתן יחד הן הליבה של Phase 5, וחשוב להבין למה כל אחת קיימת:

**שכבה 1 — Webhook נכנס.** Meta שולחת **webhook** (קריאת HTTP מרחוק אל ה-endpoint שלנו) על כל הודעה שמגיעה לקו ה-WhatsApp שלנו. Webhook הוא "טלפון" שצד שני (במקרה הזה Meta) מצלצל אליו כשמשהו קורה — בניגוד לקריאה רגילה שאנחנו יוזמים. ה-endpoint שלנו צריך לאמת שזה באמת Meta (חתימה קריפטוגרפית), למנוע עיבוד כפול (idempotency), ולענות מהר (Meta דורשת 200 בתוך 30 שניות, אחרת מנסה שוב). אם נעבד את ההודעה מיד inline — נחרוג. הפתרון: לקבל, לשמור ב-DB, להכניס job לתור, להחזיר 200. עיבוד אמיתי בנפרד.

**שכבה 2 — State machine.** **מכונת מצבים** היא דרך לתאר תהליך שעובר בין מצבים מוגדרים. במקרה שלנו, שיחה עם ליד עוברת בין `awaiting_opening_ack` (חיכיתי שיענה להודעת הפתיחה), דרך `asking_question_1` עד `asking_question_5`, ל-terminal states כמו `completed_calendly` או `completed_handoff`. כל הודעה נכנסת מהליד מקדמת את ה-state. שמירת ה-state בעמודה מפורשת ב-DB מבטיחה שאם ה-worker נופל באמצע, retry יודע איפה היינו. אפשר היה להסיק את ה-state מספירת התשובות, אבל זה שביר ולא תומך ב-states מילוליים שאינם "שאלה N".

**שכבה 3 — שליחה יוצאת.** אחרי שעיבדנו הודעה נכנסת והחלטנו מה לענות, צריך לשלוח את התגובה ל-WhatsApp דרך Meta Cloud API. הקריאה היא לנקודת קצה (`/{phone_number_id}/messages`) עם header `Idempotency-Key` (מזהה ייחודי שמונע ממטא לשלוח את אותה הודעה פעמיים אם הקריאה תרוץ פעמיים — דפוס סטנדרטי ב-APIs לכסף ולתקשורת). כשליחה מצליחה, מטא מחזירה `message_id` שלה. אנחנו שומרים אותו ב-`bot_messages` (היסטוריית הודעות) כדי שאם בעתיד מטא תשלח לנו webhook על הודעה זו (status update: delivered/read/failed) נדע איך לקשר.

**מה ב-Session הזה:**
- Migration לטבלאות `bot_conversations` (state) ו-`bot_messages` (היסטוריה).
- `POST /api/v1/webhooks/whatsapp` — webhook handler נכנס, עם אימות חתימה ו-idempotency.
- שני handlers ב-worker: `initiate_bot_conversation` (פותח שיחה אחרי שליד חדש נכנס ב-4.1) ו-`process_inbound_message` (מטפל בהודעה שהגיעה דרך ה-webhook).
- שליחת הודעות יוצאות דרך Meta Cloud API, כולל idempotency דו-שכבתי.
- Patch ל-4.1 שמפעיל את הבוט אחרי שליד חדש נשמר.

**מה לא ב-Session הזה:**
- אינטליגנציה אמיתית (זיהוי סטיות, פירוש סמנטי של תשובות) — 5.3. ב-5.2 כל הודעה לא-ריקה במצב "ממתין לתשובה לשאלה N" מתקבלת כתשובה תקפה וה-state מתקדם ל-N+1.
- Templates של WhatsApp ו-follow-up אחרי 23.5 שעות — 5.4.
- תיאום תור ביומן — 5.5.
- Status updates של הודעות (delivered/read) — לא ב-MVP. נטפל רק ב-message events (הודעות נכנסות), לא ב-status events.
- Quality monitoring (`quality_rating` של הקו) — ב-Meta יש sync שכבת אזהרות; ב-MVP לא קוראים, אדמין בודק ידנית.
- Multi-language detection — הבוט תמיד עברית.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | עיבוד webhook נכנס | **תמיד אסינכרוני דרך jobs queue.** Meta דורשת 200 תוך 30 שניות; עיבוד inline יחרוג. |
| 2 | State machine | **עמודה מפורשת `state`** ב-`bot_conversations` עם enum מילולי. לא נגזרת מספירת תשובות. |
| 3 | הפעלת השיחה | **דרך job חדש `initiate_bot_conversation`** שמופעל מ-4.1 אחרי INSERT ליד. Patch קטן ל-4.1. |
| 4 | מספר שיחות פעילות פר אדם | **שיחה אחת פעילה לכל `(user_id, contact_key)`**. אם יש כבר פעילה — מקשרים את הליד החדש (`lead_ids` כ-array), לא פותחים חדשה. |
| 5 | פירוש תשובות ב-5.2 | **נאיבי** — כל הודעה לא-ריקה במצב `asking_question_N` נחשבת תשובה ומקדמת state. AI ב-5.3. |
| 6 | הודעות מדיה (תמונה/אודיו) | **תגובה אוטומטית** "תומך רק בטקסט כרגע", state לא מתקדם. |
| 7 | שליחה ראשונה (business-initiated) | **בייצור — דורש template (5.4).** ב-5.2 שולחים טקסט חופשי, עובד עם test phone number של Meta. 5.4 יעטוף ב-template wrapper. |
| 8 | Idempotency של שליחה יוצאת | **דו-שכבתי:** `Idempotency-Key` של Meta + בדיקת `bot_messages` ל-`internal_message_key`. |
| 9 | Routing webhook → user | **לפי `phone_number_id`** (`whatsapp_lines.phone_number_id → user_id`). ה-`from` משמש רק לאיתור השיחה. |
| 10 | Retry על שליחה כושלת | **transient — דרך 3.0** (3 ניסיונות, backoff). **permanent** — `state='send_failed'` + Sentry, אין retry. |

---

## חלק 1 — מודל הנתונים

לפני שאני נכנס ל-SQL, אסביר את התפקיד של כל טבלה ולמה היא נפרדת מהאחרות. הסכימה כאן מורכבת יחסית למה שראינו עד עכשיו, אבל כל טבלה מצדיקה את עצמה.

### 1.1 — תפקיד `bot_conversations` מול `bot_messages` מול `webhook_events`

**`bot_conversations`** היא היחידה שמייצגת "שיחה" — דיאלוג מתמשך עם אדם ספציפי. שדה אחד שלה — `state` — מספר היכן אנחנו בתסריט הסינון. שדה אחר — `answers` — מצטבר התשובות שהליד נתן עד עכשיו (JSONB גמיש, כי השאלות עצמן מוגדרות פר user). זוהי השורה שעוברת mutation תוך כדי השיחה. תיקרא לעיתים קרובות, לכל הודעה נכנסת.

**`bot_messages`** היא יומן append-only של **כל הודעה** שעברה — נכנסת או יוצאת. אם `bot_conversations` היא הוויקיפדיה (מצב נוכחי), `bot_messages` היא הלוג של כל השינויים. שני שימושים: (א) **debugging** — לראות בדיוק מה הליד שלח ומה הבוט השיב; (ב) **idempotency של שליחה יוצאת** — לפני שליחה אנחנו בודקים אם כבר שלחנו הודעה עם `internal_message_key` זהה (ראה שכבה דו-שכבתית בהחלטה 8). אין mutations — רק INSERT.

**`webhook_events`** קיימת כבר מ-4.6 / 2.6.1. זוהי שכבת idempotency **גלובלית** לכל webhook נכנס מכל ספק (Pelecard, Resend, ועכשיו Meta WhatsApp). לפני שאנחנו אפילו מסתכלים על תוכן ההודעה, אנחנו עושים INSERT ל-`webhook_events` עם המזהה שמטא נתנה ל-event הזה — ועם `ON CONFLICT DO NOTHING`. אם כבר עיבדנו אותו (Meta שלחה שוב כי לא קיבלה 200 במהירות) — INSERT לא יבצע כלום, ואנחנו מחזירים 200 בלי לעבד. **חשוב:** ב-Meta WhatsApp ה-event ID הייחודי הוא `messages[].id` (מזהה ההודעה הספציפית) בתוך payload — לא `entry[].id`. עוד הסבר על זה בחלק 2.

### 1.2 — Migration

```sql
-- 0023_bot_conversations.sql
-- Phase 5.2 — Conversation state machine and message history

BEGIN;

-- ============================================================
-- 1. ENUM ל-state (כ-CHECK constraint, לא Postgres enum — עקבי עם הפרויקט)
-- ============================================================

-- ה-states:
-- awaiting_opening_ack — נשלחה הודעת פתיחה, מחכים לתגובה כלשהי מהליד.
-- asking_question_1..5 — שאלת השאלה ה-Nth, מחכים לתשובה.
-- completed_calendly — הסינון הסתיים, נשלח Calendly link.
-- completed_handoff — הסינון הסתיים, פרטי הליד נשלחו לבעל העסק.
-- abandoned — הליד לא ענה אחרי X שעות (יסומן ב-5.4 דרך follow-up cron).
-- send_failed — שליחת הודעה ל-Meta נכשלה permanent. אדמין צריך לבדוק.

-- ============================================================
-- 2. טבלת bot_conversations
-- ============================================================

CREATE TABLE public.bot_conversations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  -- מזהה הליד/האדם — מספר טלפון מנורמל (E.164 ללא +, כמו 972501234567)
  -- מאפשר לקשר בין מספר ההודעה הנכנסת לבין שיחה קיימת.
  contact_key text NOT NULL,

  -- snapshot של שם הליד ב-Meta (לטובת interpolation של [שם] בהודעת פתיחה)
  contact_name text,

  -- רשימת leads.id שקושרו לשיחה. בדרך כלל אחד, אבל יכול לגדול אם אותו אדם
  -- מילא טופס שני בזמן שהשיחה הראשונה עוד פעילה.
  lead_ids uuid[] NOT NULL DEFAULT '{}',

  -- ה-bot_config שהיה פעיל ברגע פתיחת השיחה. שמירת snapshot כדי שאם הלקוח
  -- ישנה את ההגדרות באמצע שיחה — השיחה ממשיכה לפי המקור.
  bot_config_id uuid NOT NULL REFERENCES public.bot_configs(id) ON DELETE RESTRICT,

  -- ה-WhatsApp line שדרכו השיחה התנהלה
  whatsapp_line_id uuid NOT NULL REFERENCES public.whatsapp_lines(id) ON DELETE RESTRICT,

  -- ה-state הנוכחי
  state text NOT NULL DEFAULT 'awaiting_opening_ack' CHECK (state IN (
    'awaiting_opening_ack',
    'asking_question_1',
    'asking_question_2',
    'asking_question_3',
    'asking_question_4',
    'asking_question_5',
    'completed_calendly',
    'completed_handoff',
    'abandoned',
    'send_failed'
  )),

  -- מצטבר התשובות. מבנה: {"1": "תשובה לשאלה 1", "2": "...", ...}
  answers jsonb NOT NULL DEFAULT '{}'::jsonb,

  -- timestamp של ההודעה האחרונה (נכנסת או יוצאת) — לטובת זיהוי abandoned
  last_message_at timestamptz NOT NULL DEFAULT now(),

  -- ספירת ניסיונות שליחה שנכשלו ב-permanent (לא transient). אם > 0 → צריך בדיקה.
  send_failure_count int NOT NULL DEFAULT 0,
  last_send_error text,

  started_at timestamptz NOT NULL DEFAULT now(),
  completed_at timestamptz,

  -- אחת פעילה (state NOT IN terminal) לכל (user_id, contact_key).
  -- שתי שיחות completed_calendly לאותו אדם — מותר.
  -- partial unique index — מאפשר רק שיחה אחת לא-terminal פר אדם פר user.
  CONSTRAINT bot_conversations_unique_active EXCLUDE USING btree (
    user_id WITH =,
    contact_key WITH =
  ) WHERE (state NOT IN (
    'completed_calendly', 'completed_handoff', 'abandoned', 'send_failed'
  ))
);

-- composite UNIQUE על (id, user_id) — נדרש ל-FK composite מ-bot_messages
ALTER TABLE public.bot_conversations
  ADD CONSTRAINT bot_conversations_id_user_unique UNIQUE (id, user_id);

CREATE INDEX idx_bot_conversations_user_contact
  ON public.bot_conversations (user_id, contact_key);

CREATE INDEX idx_bot_conversations_state_active
  ON public.bot_conversations (last_message_at)
  WHERE state NOT IN ('completed_calendly', 'completed_handoff', 'abandoned', 'send_failed');

-- RLS — SELECT-only למשתמש (כתיבה דרך admin client בלבד)
ALTER TABLE public.bot_conversations ENABLE ROW LEVEL SECURITY;

CREATE POLICY bot_conversations_select_own
  ON public.bot_conversations FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.bot_conversations TO authenticated;

-- ============================================================
-- 3. טבלת bot_messages
-- ============================================================

CREATE TABLE public.bot_messages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  conversation_id uuid NOT NULL REFERENCES public.bot_conversations(id) ON DELETE CASCADE,

  -- composite FK לקוהרנטיות RLS (spec §7.2 — כמו ב-quiz_responses, bot_questions)
  CONSTRAINT bot_messages_conversation_user_fk
    FOREIGN KEY (conversation_id, user_id)
    REFERENCES public.bot_conversations(id, user_id)
    ON DELETE CASCADE,

  -- כיוון: inbound = מהליד אלינו, outbound = מאיתנו ללליד
  direction text NOT NULL CHECK (direction IN ('inbound', 'outbound')),

  -- סוג ההודעה. ב-MVP רק text. עתידי: image, audio, video, document, button, list
  message_type text NOT NULL DEFAULT 'text' CHECK (message_type IN (
    'text', 'image', 'audio', 'video', 'document', 'unsupported'
  )),

  -- תוכן הודעת הטקסט. במדיה — תיאור או caption.
  body text,

  -- מזהי ההודעה — שניהם:
  -- meta_message_id: ID שמטא נתנה (נשמר ב-inbound מה-webhook payload, וב-outbound מה-response).
  -- internal_message_key: UUID שאנחנו מייצרים ל-outbound לפני שליחה. משמש כ-Idempotency-Key למטא,
  --                       וגם לבדיקה ב-DB אם כבר שלחנו (שכבה שנייה של idempotency).
  meta_message_id text,
  internal_message_key uuid UNIQUE,

  -- ל-outbound: סטטוס השליחה (לא Meta status update, אלא שלנו)
  send_status text CHECK (send_status IN ('pending', 'sent', 'failed_transient', 'failed_permanent')),
  send_error text,

  created_at timestamptz NOT NULL DEFAULT now(),

  -- ולידציה לוגית: inbound חייב meta_message_id, outbound חייב internal_message_key.
  CONSTRAINT bot_messages_direction_check CHECK (
    (direction = 'inbound' AND meta_message_id IS NOT NULL) OR
    (direction = 'outbound' AND internal_message_key IS NOT NULL)
  )
);

CREATE INDEX idx_bot_messages_conversation
  ON public.bot_messages (conversation_id, created_at);

CREATE INDEX idx_bot_messages_meta_id
  ON public.bot_messages (meta_message_id)
  WHERE meta_message_id IS NOT NULL;

CREATE UNIQUE INDEX idx_bot_messages_internal_key
  ON public.bot_messages (internal_message_key)
  WHERE internal_message_key IS NOT NULL;

-- RLS — SELECT-only למשתמש
ALTER TABLE public.bot_messages ENABLE ROW LEVEL SECURITY;

CREATE POLICY bot_messages_select_own
  ON public.bot_messages FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.bot_messages TO authenticated;

-- ============================================================
-- 4. הוספת job types חדשים (אם 3.0 כבר רץ — extension ל-CHECK constraint)
-- ============================================================

ALTER TABLE public.jobs DROP CONSTRAINT IF EXISTS jobs_type_check;
ALTER TABLE public.jobs ADD CONSTRAINT jobs_type_check CHECK (type IN (
  -- types שכבר קיימים מ-Phases קודמים — לבדוק ב-CC ולשלב הכל ביחד:
  'test_echo',
  'send_notification',
  'generate_ad_images',
  'push_campaign_to_meta',
  'initiate_bot_conversation',  -- חדש ב-5.2
  'process_inbound_message'     -- חדש ב-5.2
  -- אם יש types נוספים מ-Phases קודמים שלא ברשימה — להוסיף לפני MERGE.
));

-- ============================================================
-- 5. הערות תיעוד
-- ============================================================

COMMENT ON TABLE public.bot_conversations IS
  'דיאלוג מתמשך בין הבוט לליד. אחת פעילה לכל (user_id, contact_key) באמצעות EXCLUDE constraint.';

COMMENT ON COLUMN public.bot_conversations.contact_key IS
  'מספר טלפון מנורמל E.164 ללא + (e.g. 972501234567). דפוס זהה ל-leads.contact_key מ-4.1.';

COMMENT ON COLUMN public.bot_conversations.bot_config_id IS
  'snapshot של bot_config בזמן פתיחת השיחה. ON DELETE RESTRICT כדי שלא נאבד את ההקשר.';

COMMENT ON COLUMN public.bot_conversations.lead_ids IS
  'מערך של leads.id שקושרו לשיחה זו. בד"כ אחד; גדל אם אותו אדם מילא טופס שני.';

COMMENT ON COLUMN public.bot_conversations.answers IS
  'מצטבר התשובות. מבנה: {"1": "תשובה לשאלה 1", ...}. ב-5.2 נשמר ה-raw text של הליד.';

COMMENT ON TABLE public.bot_messages IS
  'יומן append-only של כל הודעה (נכנסת/יוצאת). משמש ל-debugging ול-idempotency של שליחה יוצאת.';

COMMENT ON COLUMN public.bot_messages.internal_message_key IS
  'UUID שאנחנו מייצרים לפני שליחה ל-Meta. משמש כ-Idempotency-Key במטא, וכ-dedupe ב-DB.';

COMMIT;
```

### 1.3 — הסבר על `EXCLUDE USING btree ... WHERE`

זה דפוס פחות מוכר ב-PostgreSQL ושווה הסבר. `EXCLUDE` constraint דומה ל-UNIQUE אבל מאפשר תנאי **WHERE** — כלומר ייחודיות שמותנית.

במקרה שלנו: "אסור שיהיו שתי שורות עם אותו `(user_id, contact_key)` **אם** ה-state לא טרמינלי". המשמעות: יכולות להיות 10 שיחות `completed_calendly` של אותו ליד עם אותו עסק לאורך זמן — מותר. אבל לעולם לא יכולות להיות 2 שיחות במצב `asking_question_3` באותו רגע — אסור.

זה מאכף את ה-invariant ברמת ה-DB, לא ברמת הקוד. אם service ינסה להפר — INSERT ייכשל עם constraint violation, ונדע מיד שיש באג. אלטרנטיבה הייתה partial UNIQUE INDEX — בסיס נתונים נמוך יותר, אבל EXCLUDE מתאר את הכוונה יותר ברור.

### 1.4 — `JSONB` ל-`answers` במקום טבלה נפרדת

יכולנו ליצור `bot_conversation_answers` (שורה לכל תשובה). יתרון: ניתן ל-query לפי תוכן ספציפי, אבל זה לא משהו שאנחנו צריכים ב-MVP. חסרון: עוד טבלה, עוד JOIN, עוד RLS. JSONB עם מבנה פשוט (`{"1": "...", "2": "..."}`) זול לקריאה ולכתיבה, ומשתלב יפה עם `bot_questions.order_index`. אם בעתיד נרצה analytics על תשובות — נוכל לפתוח את ה-JSONB עם פונקציות SQL סטנדרטיות (`->`, `->>`, `jsonb_each`).

---

## חלק 2 — ה-Webhook handler

### 2.1 — מה Meta שולחת לנו

Meta שולחת **POST** ל-endpoint שלנו עם payload בפורמט קבוע. דוגמה מקוצרת של הודעת טקסט:

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "WABA_ID",
    "changes": [{
      "field": "messages",
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {
          "display_phone_number": "972501234567",
          "phone_number_id": "PHONE_NUMBER_ID"
        },
        "contacts": [{
          "profile": {"name": "דני"},
          "wa_id": "972501234567"
        }],
        "messages": [{
          "from": "972501234567",
          "id": "wamid.HBgL...",
          "timestamp": "1717000000",
          "type": "text",
          "text": {"body": "שלום, ראיתי את המודעה שלכם"}
        }]
      }
    }]
  }]
}
```

מפתחות חשובים:
- `messages[].id` — המזהה הייחודי של ההודעה במטא. **זה ה-`event_id` שלנו ל-idempotency** (לא `entry[].id`, שיכול לחזור על עצמו על-פני webhooks שונים).
- `metadata.phone_number_id` — איזה קו של Meta קיבל את ההודעה. **זה מה שמקשר אותנו ל-user** (`whatsapp_lines.phone_number_id`).
- `messages[].from` — מספר הליד ששלח. **זה ה-`contact_key`** אחרי נורמליזציה.
- `contacts[].profile.name` — שם פרופיל של הליד ב-WhatsApp. שמיר ב-`bot_conversations.contact_name` ל-interpolation.

יש גם payloads אחרים שמטא יכולה לשלוח — status updates של הודעות שלנו (delivered/read), errors, וכו'. ב-5.2 **נתעלם** מהם בשקט אם `value.messages` ריק. נטפל בהם רק אם נצטרך, בעתיד.

### 2.2 — אימות חתימה: דפוס סטנדרטי ולמה הוא חשוב

ה-endpoint שלנו ציבורי (חייב להיות, כדי שמטא תוכל להגיע). אם לא נאמת — כל אחד יוכל לזייף webhook ולגרום לבוט שלנו להזיז state, לשלוח הודעות, וכו'. אימות חתימה הוא הגנה קריפטוגרפית: מטא חותמת את ה-payload שלה עם ה-`App Secret` שלנו (שגיא הגדיר ב-Meta Business Manager), ושולחת את החתימה ב-header `X-Hub-Signature-256`. אנחנו מחשבים את אותה חתימה על ה-payload שקיבלנו, ואם התאמה — זה באמת מטא.

החישוב הוא HMAC-SHA256 על raw body (לפני שקראנו אותו כ-JSON), עם המפתח `META_APP_SECRET`. תוצאה: `sha256=<hex>`. השוואה חייבת להיות **constant-time** (`hmac.compare_digest` ב-Python, לא `==`) — אחרת תוקף יכול ללמוד את החתימה הנכונה תו-תו דרך מדידת זמן (timing attack).

```python
# Pseudocode — לא לקוד ישיר
import hmac
import hashlib

async def verify_meta_signature(request: Request) -> bool:
    raw_body = await request.body()
    signature_header = request.headers.get("X-Hub-Signature-256", "")

    if not signature_header.startswith("sha256="):
        return False

    expected = hmac.new(
        META_APP_SECRET.encode("utf-8"),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    actual = signature_header.removeprefix("sha256=")

    return hmac.compare_digest(expected, actual)
```

**חשוב:** האימות חייב להיות על ה-raw body שקיבלנו, **לפני** שעשינו `.json()`. אם נפענח כ-JSON ונסריאליז שוב — הסדר של ה-keys יכול להשתנות, רווחים, וכו', והחתימה לא תתאים. ב-FastAPI: `await request.body()` לפני `await request.json()`.

### 2.3 — Idempotency: למה ואיך

מטא לא תמיד מקבלת את ה-200 שלנו (רשת איטית, שרת עמוס) ואז שולחת שוב את אותו webhook. בלי מנגנון idempotency — נעבד את אותה הודעה פעמיים, נקדם state פעמיים, נשלח תגובה כפולה.

**דפוס סטנדרטי** (זה מה ש-4.6 ו-2.6.1 משתמשים בו): טבלת `webhook_events` עם UNIQUE על `(event_type, key)`. לפני עיבוד — INSERT עם `ON CONFLICT DO NOTHING`. אם INSERT הצליח (שורה חדשה) — זה event חדש, נמשיך לעבד. אם DO NOTHING (כבר קיים) — זה event שכבר עיבדנו, נחזיר 200 ונצא בלי לעשות כלום.

**`key` במקרה שלנו:** `messages[].id` של מטא (ה-`wamid.HBg...`). זה ייחודי גלובלית במטא. **לא** `entry[].id` (זה ה-WABA, חוזר על עצמו).

**`event_type`:** `'meta_whatsapp_message'` — מבדיל בין webhook providers שונים שעלולים להשתמש באותו key.

### 2.4 — זרימת ה-handler

```
POST /api/v1/webhooks/whatsapp
    ↓
[1] חישוב HMAC על raw body מול X-Hub-Signature-256
    ↓ לא תואם → 403, מחיקת lo
    ↓
[2] Parse JSON. אם אין value.messages → 200 (status update, מדלגים בשקט).
    ↓
[3] לכל message ב-value.messages:
    a. INSERT ל-webhook_events עם key=message.id. אם DO NOTHING → דלג להודעה הבאה.
    b. בדיקת message.type: אם לא text → INSERT ל-bot_messages עם message_type='unsupported'.
       צור job send_unsupported_response (או לבצע inline ב-process_inbound_message — נשקול).
    c. נורמליזציה של from: הסר '+', אפסים מובילים, וודא 9-15 ספרות.
    d. שליפת whatsapp_line לפי phone_number_id. אם לא קיים → log warning, החזר 200, צא.
       (אדם שולח למספר שאינו שלנו? לא יכול לקרות אם ה-webhook הוגדר נכון.)
    e. INSERT job 'process_inbound_message' עם payload:
       { user_id, conversation_lookup_key: (user_id, contact_key), meta_message_id, body, ... }
[4] return 200
```

### 2.5 — חשוב: 200 חייב לחזור מהר

כל הצעדים בזרימה לעיל מקיימים את ה-budget של 30 שניות בקלות (DB writes ב-מילישניות). העיבוד עצמו (קריאה ל-OpenAI ב-5.3, שליחה ל-Meta) — בתוך ה-job, לא בתוך ה-handler. ה-handler חייב להיות זריז, defensive, ולחזור 200 גם בתרחישי קצה — אחרת מטא תציף אותנו ב-retries.

**עיקרון:** הצלחה = שמרנו את ההודעה הנכנסת וקיבלנו אחריות לעבד אותה. הכשל ב-עיבוד יהיה ב-job, ושם יש לנו retry policy, Sentry, וכו'. ב-handler — אנחנו רק שומרים ומאשרים.

---

## חלק 3 — מכונת המצבים

### 3.1 — דיאגרמת ה-states

```
[ליד נכנס דרך 4.1]
    ↓
[initiate_bot_conversation job]
    ↓
    שולח הודעת פתיחה
    ↓
awaiting_opening_ack
    │
    ├─ הליד עונה (כל הודעה) ──→ asking_question_1
    │                            │
    │                            ├─ עונה ──→ asking_question_2 ──→ ... ──→ asking_question_5
    │                            │                                          │
    │                            │                                          └─ עונה ──→ סיום
    │                            │
    │                            └─ עונה בשאלה אחרונה ──→ סיום
    │
    └─ X שעות ללא תגובה ──→ abandoned (יסומן ב-5.4 דרך follow-up cron)

סיום (לפי fallback_action ב-bot_config):
    ├─ calendly_link → שולח URL + עובר ל-completed_calendly
    └─ human_handoff → שולח לבעל העסק את פרטי הליד + עובר ל-completed_handoff

כשל שליחה permanent בכל שלב → send_failed
```

### 3.2 — מעברי state חוקיים

ה-CHECK constraint ב-DB מגדיר אילו ערכים מותרים, אבל לא אילו מעברים חוקיים. את זה אוכפים בקוד (ב-`bot_service`). מעברים מותרים:

| מ- | אל | טריגר |
|---|---|---|
| (initial) | `awaiting_opening_ack` | יצירת השיחה ושליחת הודעת פתיחה |
| `awaiting_opening_ack` | `asking_question_1` | הליד הגיב להודעת פתיחה |
| `awaiting_opening_ack` | `completed_calendly`/`completed_handoff` | אם אין שאלות בכלל ב-bot_config — דילוג ישר לסיום |
| `asking_question_N` | `asking_question_N+1` | הליד ענה לשאלה N |
| `asking_question_<last>` | `completed_calendly`/`completed_handoff` | הליד ענה לשאלה האחרונה |
| **כל מצב לא-טרמינלי** | `send_failed` | כשל שליחה permanent |
| **כל מצב לא-טרמינלי** | `abandoned` | follow-up cron ב-5.4 מסמן |

מעברים אסורים: לאחור (אסור לחזור משאלה 3 לשאלה 2), מ-terminal לאן שהוא (terminal הוא terminal), בין מצבי terminal שונים.

**מה אם הליד שולח הודעה אחרי שהגיעו ל-`completed_calendly`?** ב-5.2 — מתעלמים בשקט (INSERT ל-`bot_messages` עם הערה, לא מקדמים state, לא עונים). ב-5.3 ייתכן שנרצה לזהות "הליד שואל שאלה אחרי הסיום" ולענות. אבל זה כבר 5.3.

### 3.3 — שאלה: מה אם `bot_config` הוא 0 שאלות

ב-5.1 ולידציה מאפשרת 0 שאלות (max=5, min=0). הלוגיקה ב-`bot_service`: אם `len(questions) == 0` בעת מעבר מ-`awaiting_opening_ack` — דילוג ישר ל-fallback. הוא מקבל הודעת פתיחה, מגיב כלשהו, ומיד מקבל את ה-Calendly link / handoff. שווה לוודא עם גיא שזה ה-flow שהוא רוצה, אבל הגיוני.

---

## חלק 4 — Job handlers

### 4.1 — `initiate_bot_conversation`

נקרא מ-4.1 (lead intake) דרך job. payload:

```json
{
  "user_id": "uuid",
  "lead_id": "uuid"
}
```

זרימה:

1. שליפת ה-lead ב-DB.
2. בדיקת tier+status של ה-user. אם לא Premium פעיל → log + סיום (לא error, פשוט אין בוט).
3. בדיקת `bot_configs` של ה-user. אם אין → log + סיום.
4. בדיקת `whatsapp_lines` של ה-user. אם אין או `status != 'active'` → log + סיום. (אם `pending_provisioning` — הליד נכנס לפני שהקו מוכן; ב-MVP נאבד את ההזדמנות הזו. ב-Post-MVP אפשר לתור.)
5. **בדיקת שיחה קיימת** ב-`bot_conversations` עם אותם `(user_id, contact_key)` ו-state לא-terminal. אם קיימת → רק מעדכן את `lead_ids` להוסיף את ה-lead החדש, מסיים. **לא יוצר שיחה חדשה, לא שולח הודעת פתיחה שנייה.**
6. אם לא קיימת — יוצר שורה חדשה ב-`bot_conversations` עם state ההתחלתי `awaiting_opening_ack`.
7. בונה את הודעת הפתיחה: לוקח את `opening_message` מ-`bot_config`, מבצע interpolation על המשתנים (`[שם]` → `lead.contact_name`, `[שירות]` → `campaign.service_name`, `[שם_עסק]` → לא ברור מאיפה ב-MVP — אולי מ-`subscriptions` או מ-quiz_responses, ראה הערה 1 בחלק 9).
8. שולח את ההודעה דרך `meta_whatsapp.send_text_message(...)`. שמירה ב-`bot_messages` עם `direction='outbound'`, `internal_message_key`, ו-`send_status='sent'` אם הצליח.
9. אם השליחה נכשלה — תלוי בסוג הכשל. transient → throw כדי ש-3.0 יעשה retry. permanent → mark `state='send_failed'`, increment `send_failure_count`, Sentry alert.

**Idempotency של ה-handler עצמו:** אם 3.0 retry את ה-job (worker crash אחרי שליחה לפני שמירה ב-DB), הזרימה הזו צריכה להיות בטוחה. שכבת ההגנה: לפני שליחה, `bot_service` בודק אם כבר יש שיחה ב-`bot_conversations` ל-`(user_id, contact_key)`. אם כן (מצעד 5 ב-retry) — לא יוצר חדשה, ובודק ב-`bot_messages` אם כבר נשלחה הודעת פתיחה (outbound, direction=outbound, conversation_id=...). אם כן — מסיים. אם לא — שולח.

### 4.2 — `process_inbound_message`

נקרא מה-webhook handler. payload:

```json
{
  "user_id": "uuid",
  "contact_key": "972501234567",
  "meta_message_id": "wamid.HBg...",
  "message_type": "text",
  "body": "כן, אני בעל עסק",
  "contact_name": "דני",
  "phone_number_id": "..."
}
```

זרימה:

1. שליפת `bot_conversations` לפי `(user_id, contact_key)` עם state לא-terminal. אם אין → log warning ("inbound message without conversation"), שמירה ב-`bot_messages` עם `conversation_id=NULL`?, סיום. **בעיה:** ה-FK של `bot_messages` לא מאפשר NULL. החלטה: במקרה זה, נשמור ב-לוג בלבד, לא ב-DB. שווה לדון — אולי טבלת `orphan_messages` עתידית.
2. INSERT ל-`bot_messages` עם `direction='inbound'`, ה-`meta_message_id`, וה-`body`.
3. בדיקת `message_type`. אם `unsupported` (image/audio/video) — שולח תגובה אוטומטית "תומך רק בטקסט כרגע", state לא מתקדם, סיום.
4. אם text:
   - אם `state == 'awaiting_opening_ack'`: state ← אם יש שאלות → `asking_question_1`, אחרת → fallback ישיר. שאל את שאלה 1 (אם יש), או בצע fallback (אם אין).
   - אם `state == 'asking_question_N'`: שמור את ה-body ב-`answers[N]`. אם N < מספר השאלות → state ← `asking_question_N+1`, שאל את שאלה N+1. אם N == מספר השאלות → בצע fallback.
   - בכל מקרה אחר → log + סיום בלי תגובה.
5. `last_message_at = now()`.

**ביצוע fallback:**

- `calendly_link`: שולח הודעה "נשמח לקבוע איתך פגישה: {fallback_value}". state ← `completed_calendly`. `completed_at = now()`.
- `human_handoff`: שולח הודעה לליד "תודה! אדם מהצוות יחזור אליך בקרוב". בנוסף — שולח הודעה למספר של בעל העסק (`fallback_value`) עם פרטי הליד (`contact_name`, `contact_key`, `answers` כטקסט מעוצב). state ← `completed_handoff`. `completed_at = now()`.
- `bot_schedule_appointment`: ב-5.2 לא יקרה (חסום ב-5.1). 5.5 יממש.

**שתי שליחות נפרדות ב-handoff:** הראשונה (ליד) — דרך `whatsapp_line` של ה-user. השנייה (בעל העסק) — דרך **אותו** `whatsapp_line` (כי בעל העסק הוא מספר חיצוני, לא תחת הקו שלנו). זוהי הודעה business-initiated **שנייה** מאיתנו לבעל העסק — וגם זה דורש template (5.4). ב-test phone number עובד.

### 4.3 — Idempotency של `process_inbound_message`

אם ה-handler נופל אחרי קידום state ב-DB אבל לפני שליחת תגובה — retry יראה ש-state כבר מתקדם וינסה לקדם שוב, מה שייצור bug. הפתרון: **בדיקה לפני קידום**.

לפני שינוי state, בודקים אם כבר יש `bot_message` outbound עם `created_at > inbound_message.created_at` (כלומר, התגובה שלנו כבר נשלחה). אם כן — לא מקדמים שוב, מסיימים.

זה לא perfect — race conditions קצרות אפשריות. אבל ב-MVP מספיק. שכבת idempotency הראשונה (`webhook_events`) ממילא קוטעת את רוב ה-duplicates לפני שמגיעים לכאן.

---

## חלק 5 — שליחת הודעות יוצאות

### 5.1 — `meta_whatsapp.send_text_message(...)`

תוספת ל-integration שכבר נבנה ב-5.0. הקריאה:

```
POST /{phone_number_id}/messages
Authorization: Bearer {META_ACCESS_TOKEN}
Idempotency-Key: {internal_message_key}
Content-Type: application/json

{
  "messaging_product": "whatsapp",
  "to": "972501234567",
  "type": "text",
  "text": {"body": "שלום, ראיתי שהשארת פרטים..."}
}
```

תשובה מוצלחת:

```json
{
  "messaging_product": "whatsapp",
  "contacts": [{"input": "972501234567", "wa_id": "972501234567"}],
  "messages": [{"id": "wamid.HBg..."}]
}
```

ה-`messages[0].id` הוא ה-`meta_message_id` שאנחנו שומרים.

**טיפול בכשלים** (דרך `classify_meta_error` הקיים מ-5.0, עם תוספת ל-מיפוי):

- **400 + recipient blocked / not registered:** permanent. ה-מספר אינו תקין או שהליד חסם. mark `send_status='failed_permanent'`, state → `send_failed`, Sentry.
- **401 + token expired:** permanent מבחינת ה-job (אבל קריטי גלובלית). Sentry urgent alert. ה-state של השיחה → `send_failed`. אדמין צריך לחדש את ה-System User Token.
- **429 + rate limit:** transient, retry דרך 3.0.
- **5xx, timeout:** transient, retry.
- **400 + 24h window violation:** business-initiated message ללא template. ב-5.2 לא יקרה בייצור (test phone number לא אוכף). ב-5.4 — שולח template במקום. mark permanent ל-בינתיים.

### 5.2 — Idempotency דו-שכבתי

**שכבה 1 (Meta side):** ה-header `Idempotency-Key` עם UUID שאנחנו מייצרים. אם נריץ retry על אותו job עם אותו UUID — מטא תזהה ותחזיר את אותו `messages[0].id` בלי לשלוח באמת. **שווה לאמת בתיעוד Meta בזמן המימוש** שה-header נתמך ב-Cloud API messages endpoint. אם לא — נסתמך רק על שכבה 2.

**שכבה 2 (DB side):** לפני שליחה, בודקים אם כבר יש `bot_message` עם אותו `internal_message_key`. UNIQUE constraint ב-`bot_messages` מבטיח שאם ננסה להוסיף שוב — INSERT ייכשל. אם כשל — דלג על השליחה (כבר נשלח).

הרצף:

```
1. צור internal_message_key = uuid4()
2. INSERT ל-bot_messages עם direction='outbound', send_status='pending', internal_message_key
   - אם נכשל ב-UNIQUE → כבר נשלחה, צא
3. שלח ל-Meta עם Idempotency-Key=internal_message_key
4. UPDATE bot_messages SET send_status='sent', meta_message_id={response.id}
```

יש race קטן בין שלב 3 ל-4 (אם worker נופל אחרי שליחה לפני UPDATE), אבל מטא תזהה את ה-key ב-retry ותחזיר את אותו message_id, ו-UPDATE יהיה idempotent.

### 5.3 — `last_message_at` ו-`abandoned`

כל שליחה מוצלחת ל-`bot_conversations.last_message_at = now()`. גם כל קליטת inbound מעדכנת. השדה הזה משמש ב-5.4 ל-follow-up: cron שרץ פעם בשעה, מוצא שיחות עם `state` לא-טרמינלי ו-`last_message_at < now() - 23.5h`, ושולח follow-up. אם follow-up גם לא מקבל תשובה תוך 24h — מסומן `abandoned`. ב-5.2 רק לוודא שהשדה מתעדכן נכון; ה-cron עצמו הוא 5.4.

---

## חלק 6 — Patch ל-4.1

### 6.1 — מה צריך לשנות

ב-4.1 (lead intake — `process_meta_lead_webhook` או דומה), אחרי INSERT מוצלח של ה-lead, נוספת קריאה:

```python
# בסוף ה-handler, אחרי שה-lead נשמר ובדיקת מכסה ב-4.2 הסתיימה:
await jobs_service.enqueue_job(
    type='initiate_bot_conversation',
    user_id=lead.user_id,
    payload={'lead_id': str(lead.id)},
)
```

**אבל רק אם** ה-user באמת זכאי (Premium פעיל). אפשר לעשות את הבדיקה כאן (חוסך job מיותר), או להעביר ל-`initiate_bot_conversation` עצמו (מרכזי יותר). הצעה: **בדוק ב-4.1**, חסוך עומס, ולוג ברור. ה-job עצמו עושה בדיקה כפולה defensively.

זרימה מלאה אחרי patch:

```
[Meta webhook → 4.1]
    ↓
INSERT ל-leads (4.1)
    ↓
Update quota counters (4.2)
    ↓
[בדיקה: tier=premium + bot_config exists + whatsapp_line=active]
    ↓ זכאי
enqueue initiate_bot_conversation job
    ↓ לא זכאי
log, סיום
```

### 6.2 — מה אם ה-enqueue נכשל

ה-INSERT של ה-lead כבר הצליח. אם enqueue נכשל (DB issue) — הליד נשמר, אבל הבוט לא ייפנה. ב-MVP זה acceptable; הליד עדיין מתועד. ב-Sentry יראו warning. אם בעתיד יידרש garantía — אפשר לעשות INSERT ל-lead + enqueue באותה transaction. ב-MVP לא דחוף.

---

## חלק 7 — Env vars

| שם | משמעות | חובה? |
|---|---|---|
| `META_APP_SECRET` | חתימת webhook ב-HMAC-SHA256 | **חובה** — בלעדיו אימות לא יעבוד. הוגדר כבר ב-5.0 אבל ב-5.2 בשימוש פעיל. |
| `META_ACCESS_TOKEN` | System User Token לשליחה | **חובה** (מ-5.0). |
| `META_WABA_ID` | מזהה ה-WABA | **חובה** (מ-5.0). |

אין ENV חדש ב-5.2.

---

## חלק 8 — שמות הקבצים החדשים והשינויים

| קובץ | תוכן |
|---|---|
| `supabase/migrations/0023_bot_conversations.sql` | **חדש** — bot_conversations + bot_messages + EXCLUDE constraint + RLS + extension ל-jobs.type |
| `app/integrations/meta_whatsapp.py` | **תיקון** — הוספת `send_text_message(phone_number_id, to, body, idempotency_key) -> SendResult` + הרחבת `classify_meta_error` למקרים חדשים |
| `app/services/bot_service.py` | **חדש** — הליבה: handle_inbound, advance_state, perform_fallback, send_outgoing |
| `app/services/conversation_service.py` | **חדש** — get_active_conversation, create_conversation, link_lead_to_conversation |
| `app/worker/handlers/initiate_bot_conversation.py` | **חדש** — handler ל-Phase initial |
| `app/worker/handlers/process_inbound_message.py` | **חדש** — handler ל-inbound |
| `app/worker/handlers/__init__.py` | **תיקון** — רישום שני ה-handlers החדשים |
| `app/routers/webhooks/whatsapp.py` | **תיקון** — הוספת POST handler (ה-GET של handshake כבר קיים מ-5.0) |
| `app/services/lead_intake_service.py` (מ-4.1) | **תיקון** — קריאה ל-`enqueue_job('initiate_bot_conversation')` בסיום |
| `app/models/bot_conversation.py` | **חדש** — Pydantic: BotConversation, BotMessage, ConversationState enum |
| `app/exceptions/bot.py` | **תיקון** — הוספת SendFailedError, NoActiveConversationError, OrphanMessageError |

---

## חלק 9 — Done של 5.2

- מיגרציה 0023 רצה — bot_conversations + bot_messages + EXCLUDE constraint + jobs.type extension.
- POST `/api/v1/webhooks/whatsapp` עובד: מאמת חתימה, idempotent דרך webhook_events, מכניס job, מחזיר 200 בתוך 30 שניות.
- `initiate_bot_conversation` handler עובד: בודק זכאות, יוצר conversation, שולח opening_message מ-bot_config (עם interpolation על משתנים מוכרים), שומר ב-bot_messages.
- `process_inbound_message` handler עובד: מקדם state נכון לפי הטבלה בחלק 3.2, שומר ב-bot_messages, שולח תגובה (שאלה הבאה / fallback).
- שני סוגי fallback עובדים: `calendly_link` (URL ללליד) ו-`human_handoff` (הודעה ללליד + הודעה לבעל העסק עם פרטים).
- 4.1 הורחב — patch ל-`process_meta_lead_webhook` שמכניס job אחרי INSERT.
- Idempotency דו-שכבתי עובד: webhook נשלח פעמיים → רק עיבוד אחד. job retry → לא שולח כפול.
- Retry policy עובד: transient (429, 5xx) → 3 ניסיונות עם backoff. permanent (400 invalid recipient, 401 token expired) → mark send_failed + Sentry.
- הודעת מדיה (image/audio) → "תומך רק בטקסט כרגע", state לא מתקדם.
- בדיקה ידנית מקצה לקצה: שולחים ליד דרך טופס Meta של ה-test phone number, רואים שהבוט פונה, עונים על השאלות, מקבלים את ה-fallback. כל הטסטים החדשים עוברים.

---

## חלק 10 — לא ב-5.2

- אינטליגנציה ב-AI (זיהוי סטיות, פירוש סמנטי) — 5.3.
- WhatsApp Templates ל-business-initiated messages — 5.4. בייצור Meta תדחה שליחה ראשונה אם אינה template.
- Follow-up 23.5h cron — 5.4.
- abandoned state — סומן ב-5.4 (cron), לא ב-5.2.
- תיאום תור ב-bot_schedule_appointment — 5.5.
- Status updates של הודעות (delivered/read/failed) — לא ב-MVP. ה-webhook payload יכול להכיל אותם, אנחנו מתעלמים.
- UI ב-frontend להצגת היסטוריית שיחות — Post-MVP. ה-DATA קיים, ה-UI לא.
- Multi-language detection — תמיד עברית.
- Outbound message editing or deletion — לא ב-MVP.
- שיחות יזומות מהלקוח (Inbound-only שיחות חדשות) — לא ב-MVP. גיא ננעל ב-Outbound-only.

---

## חלק 11 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **משתנה `[שם_עסק]` — מאיפה?** ב-`bot_config.opening_message` יכול להופיע. ב-MVP אין שדה ייעודי "שם העסק". אופציות: (א) להוסיף ב-`subscriptions` או ב-`bot_configs`; (ב) למשוך מ-`quiz_responses` של הקמפיין הקשור (דרך `leads.campaign_id`); (ג) להשתמש בשם של ה-user מ-`auth.users.user_metadata.name` או דומה. **שווה לדון עם גיא** — הוא לא חשב על זה במפורש. ב-INTRO של 5.2 לא חוסם, אבל interpolation נכון דורש החלטה.

2. **EXCLUDE constraint דורש btree_gist extension?** לא — EXCLUDE עם `WITH =` משתמש ב-btree רגיל. בדיקה ב-CC: `CREATE EXTENSION IF NOT EXISTS btree_gist;` רק אם הכשל קורה.

3. **Composite UNIQUE על `bot_conversations (id, user_id)`:** דרוש ל-FK composite מ-`bot_messages`. אם נשכח, ה-FK ייכשל ב-migration. דפוס זהה ל-2.2/5.1.

4. **`require_authenticated_user` ב-webhook?** לא! ה-webhook ציבורי — Meta לא שולחת JWT. האימות הוא HMAC על הגוף. החזרת 401 על webhook = בלוק את Meta. חשוב שה-router לא יהיה מתחת לdependency של auth.

5. **תור עצמאי לקליטת webhook?** לא נחוץ ב-MVP. ה-worker הקיים מ-3.0 יטפל. אם בעתיד יהיו רבבות הודעות בדקה — אפשר לפצל לתור ייעודי. עכשיו אין צורך.

6. **bot_messages גודל לאורך זמן.** כל הודעה (נכנסת ויוצאת) שמורה. ב-MVP — לא בעיה. ב-Post-MVP אולי cleanup cron של הודעות ישנות מ-90 ימים, או archive ל-S3. לא ב-5.2.

7. **`message_type='unsupported'` — נשלחת תגובה אוטומטית?** הצעה: כן, כדי שהמשתמש יבין למה אין תגובה. תיכנס דרך אותו `process_inbound_message` עם זרימה מקבילה (שולח את ההודעה הקבועה, לא מקדם state).

8. **טסטים מומלצים:**
   - test_webhook_invalid_signature → 403.
   - test_webhook_idempotent → 200 פעמיים, INSERT רק פעם אחת.
   - test_webhook_status_event_no_messages → 200, אין job.
   - test_initiate_creates_conversation_and_sends_opening.
   - test_initiate_skips_if_existing_active_conversation → רק linking.
   - test_process_advances_state_from_awaiting_to_q1.
   - test_process_completes_at_last_question_calendly.
   - test_process_completes_at_last_question_handoff_sends_to_owner.
   - test_process_skips_when_conversation_terminal.
   - test_process_unsupported_media_sends_apology.
   - test_send_idempotent_via_db_uniqueness.
   - test_send_transient_failure_retries.
   - test_send_permanent_failure_marks_send_failed.
   - test_4_1_patch_enqueues_only_for_premium_with_active_line.

9. **CC חייב להוסיף ל-`app/main.py`:**
   ```python
   app.include_router(webhooks_whatsapp_router, prefix="/api/v1/webhooks/whatsapp", tags=["webhooks"])
   # אם 5.0 כבר הוסיף את ה-GET — להוסיף POST לאותו router, לא ליצור חדש.
   ```

10. **Test phone number של Meta — הגבלות:** אפשר לשלוח רק ל-5 מספרים שהוגדרו מראש ב-Meta Business Manager. אמיר/גיא יוסיפו את עצמם + 2-3 בודקי בטא. ל-production אמיתי דרוש real phone number אחרי App Review של Meta.

11. **Sentry context:** כל log/error מהsessions של 5.2 צריך לכלול context: `conversation_id`, `lead_id`, `user_id`, `state` נוכחי, `meta_message_id`. בלי זה — debug של בעיות ייצור בלתי אפשרי. ב-`bot_service` להגדיר `sentry_sdk.set_context("bot_conversation", {...})` בכניסה לכל פונקציה ציבורית.

---

## חלק 12 — אקסטרא: זרימה מלאה לבדיקה ידנית

אחרי deploy של 5.2, הזרימה לבדיקה end-to-end:

1. אמיר/גיא מילאו טופס Meta על קמפיין מ-test phone number עם המספר של אמיר.
2. 4.1 קולט את ה-webhook (קיים מ-4.1).
3. `lead` נשמר ב-DB.
4. 4.2 מעדכן מכסה.
5. (Patch של 5.2) אם המשתמש (test premium account) זכאי → enqueue job `initiate_bot_conversation`.
6. Worker מקבל את ה-job, יוצר `bot_conversation`, שולח הודעת פתיחה.
7. אמיר מקבל את ההודעה ב-WhatsApp שלו.
8. אמיר עונה "כן".
9. Meta שולחת webhook ל-`/api/v1/webhooks/whatsapp`.
10. ה-handler מאמת חתימה, INSERT ל-webhook_events, enqueue job `process_inbound_message`.
11. Worker מעבד: state עובר מ-`awaiting_opening_ack` ל-`asking_question_1`, שולח שאלה 1.
12. אמיר עונה.
13. ... ממשיך עד שאלה אחרונה.
14. fallback מתבצע — Calendly link נשלח, או הודעה למספר של גיא (אם handoff).
15. state → terminal. `bot_conversations.completed_at = now()`.

אם משהו נופל באמצע — Sentry alert, אדמין בודק.

---

**סוף מסמך 5.2.**

---

### Session 5.3 — לוגיקת הבוט (AI סינון/חימום)
- [ ] הודעת פתיחה תוך 30 שניות (עם `service_name`)
- [ ] שאלות סינון לפי הסדר → ChatGPT לניהול שיחה
- [ ] פעולת סיום (Calendly / העברה אנושית)
- [ ] שמירת היסטוריה ב-`bot_conversations`
- [ ] בחריגת מכסה: הבוט מפסיק להגיב ללידים חדשים + הצעת שדרוג. הליד עצמו עדיין נקלט (4.1) ונספר (4.2).

**Done:** ליד נכנס מקבל הודעה, עובר סינון, ומגיע לפעולת סיום — מקצה לקצה.

# Session 5.3 — אינטליגנציה ב-AI: סיווג תשובות וניהול סטיות

> **עדכון להוספה ל-ROADMAP.md תחת Session 5.3.** Phase 5.3 — מחליף את הלוגיקה הנאיבית של 5.2 ("כל הודעה לא-ריקה = תשובה תקפה") בשכבת אינטליגנציה אמיתית: LLM שמסווג כל הודעה נכנסת ב-state `asking_question_N` — האם זו תשובה לשאלה או סטיה. סטיות מקבלות תגובת ניסוח חכמה ("אעביר לבעל העסק. בינתיים, נחזור לשאלה...") וה-state לא מתקדם עד שהליד עונה. תלוי ב-5.2 (תשתית webhook + state machine + שליחה יוצאת), 3.1.5 (תשתית `prompts_service`), 3.0 (jobs queue). חוסם את 5.4 (templates + follow-up) רק במובן זה ש-5.4 לא תלוי ב-5.3 ישירות — שניהם יכולים לרוץ במקביל אם רוצים, אבל אני ממליץ 5.3 לפני 5.4 כי בלי 5.3 הבוט חוויית-משתמש גרועה בלי קשר ל-templates.

---

## תיאור Session 5.3

**מטרה:** להחליף את ה-`process_inbound_message` הנאיבי של 5.2 בשני שלבים חכמים. שלב 1 — **classification**: LLM מקבל את ההודעה של הליד + השאלה שאנחנו שאלנו + רמז סמנטי (`expected_answer_hint`) + 3 הודעות אחרונות מההקשר. הוא מחזיר JSON: זו תשובה תקפה, או סטיה? אם תקפה — באיזה קטגוריה (כן/לא/לא-ברור)? שלב 2 (רק אם זוהתה סטיה) — **generation**: LLM מנסח תגובת fallback אישית ("אעביר לבעל העסק לבדוק את השאלה שלך, בינתיים נמשיך — מה התקציב שלך?"). 

**שני דברים שחשוב להבחין בהם בהתחלה:**

ראשון, **חוק 7 ב-spec** (הפרדת LLM מלוגיקה דטרמיניסטית) מתבטא כאן בצורה הכי ברורה בכל הפרויקט. ה-state machine מ-5.2 לא משתנה ב-5.3 — אותם states, אותם מעברים, אותו DB. מה שמשתנה זה רק **מה גורם** למעבר. ב-5.2 כל הודעה גרמה למעבר. ב-5.3 רק הודעה שסווגה כתשובה תקפה גורמת. מי שמסווג הוא LLM (לא דטרמיניסטי, פלט גמיש). מי שמקדם את ה-state בפועל הוא קוד דטרמיניסטי שקורא את התוצאה ב-DB. אסור ש-LLM יחזיר "תקדם state ל-asking_question_4". הוא מחזיר רק את הסיווג; הקוד מחליט מה לעשות.

שני, **גישה ל-LLM היא יקרה יחסית בזמן** (~2-3 שניות לקריאה). ב-5.2 הבוט הגיב כמעט מיד אחרי שהליד שלח. ב-5.3 בכל הודעה ייווסף ~2-3 שניות לתגובה (ולפעמים 6 שניות אם הייתה סטיה ויש שתי קריאות). זה בסדר ב-WhatsApp — הליד לא ציפה לתשובה מיידית כמו בצ'אט אנושי. אבל כדאי לזכור ולהציג ב-UI/לוגים אם המתנה ארוכה הופכת בעיה.

**מה ב-Session הזה:**
- שני קבצי פרומפט חדשים תחת `app/prompts/phase5/`.
- פונקציות חדשות ב-`integrations/openai.py` שעוטפות את הקריאה למודל.
- שירות חדש `app/services/bot_intelligence_service.py` שמתאם את שתי הקריאות (classification → fallback).
- שינוי ב-`bot_service` (מ-5.2): במקום לקדם state מיד, להפעיל את ה-intelligence ולקבל החלטה.
- Patch קטן ל-מבנה `bot_conversations.answers` (ראה החלטה 7).
- שני קבצי טון/סגנון להזרקה ל-fallback response (ניצול של ה-`tone` מ-`quiz_responses` של הקמפיין).

**מה לא ב-Session הזה:**
- שינוי ב-state machine עצמה (states, מעברים) — נשאר כמו 5.2.
- Templates / business-initiated messages — 5.4.
- Follow-up cron — 5.4.
- חכמה מתקדמת כמו "הליד אמר לא ספציפי לשאלה X, האם זה אומר שהוא לא רלוונטי לחלוטין?" — דורש שינוי בלוגיקה העסקית עתידי, לא ב-5.3.
- Multi-turn conversations בתוך תשובה אחת (הליד שואל שאלת המשך → בוט עונה → הליד עונה לשאלה המקורית) — ב-MVP, אחרי תגובת fallback אחת אנחנו ממתינים לתשובה לשאלה המקורית, אם הליד שואל שוב — שוב fallback.
- Caching של תוצאות classification — לא רלוונטי, כל הודעה ייחודית.
- Streaming responses מה-LLM — לא נדרש; אנחנו ממילא ממתינים לתשובה מלאה לפני שליחת ההודעה ל-WhatsApp.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מודל LLM ל-classification | `gpt-5.2` — אותו ש-3.1.5/3.1.6 משתמשים. נמדוד עלות, נרד אם צריך. |
| 2 | פורמט פלט | **JSON מובנה** דרך `response_format={"type": "json_object"}`. אסור טקסט חופשי. |
| 3 | גבול classification מ-generation | **שתי קריאות נפרדות.** classification תמיד. generation רק אם סווג כסטיה. |
| 4 | מיקום ה-prompts | `app/prompts/phase5/` עם `prompts_service.build()` (חוק 8 מ-3.1.5). |
| 5 | `expected_answer_hint` | **אופציונלי בשימוש, חיוני בערך.** אם ריק — LLM עובד עם heuristics; פחות מדויק אבל לא חוסם. |
| 6 | confidence threshold | **אין.** `answer_category` עצמו הוא ההכרעה: `yes`/`no`/`unclear` → תקפה; `deviation` → סטיה. |
| 7 | מבנה `answers` ב-DB | **patch ל-5.2:** `{raw, category, value}` במקום string. כל תשובה היא object. |
| 8 | conversation context ל-LLM | **3 הודעות אחרונות** מ-`bot_messages` (inbound+outbound), בסדר כרונולוגי. |

---

## חלק 1 — Patch ל-5.2 על מבנה `answers`

הנה הסיבה לפני שאני מציג את ה-SQL. ב-5.2 `bot_conversations.answers` היא JSONB עם מבנה:

```json
{
  "1": "כן, אני עצמאי",
  "2": "5000 שקל"
}
```

זה raw text של הליד. בעיה: ב-5.3 אנחנו רוצים גם את הסיווג של ה-LLM ("ענה 'כן'", "ענה ערך מספרי 5000"). יכולנו לעשות טבלה נפרדת (`bot_answer_classifications`) אבל זה overhead. עדיף להעשיר את ה-JSONB עצמו.

**המבנה החדש:**

```json
{
  "1": {
    "raw": "כן, אני עצמאי",
    "category": "yes",
    "value": null,
    "classified_at": "2026-06-07T20:00:00Z"
  },
  "2": {
    "raw": "5000 שקל",
    "category": "unclear",
    "value": "5000",
    "classified_at": "2026-06-07T20:01:30Z"
  }
}
```

**שדות:**
- `raw`: הטקסט המלא שהליד שלח. נשמר תמיד, לעולם לא נמחק. שימושים: handoff לבעל העסק (לראות את הליד במילותיו), debugging.
- `category`: אחד מ-`yes`/`no`/`unclear` (ב-`deviation` ה-state לא מתקדם, אז לא מגיעים לשמירה כאן). יוצא מהסיווג של ה-LLM.
- `value`: ערך נומרי/טקסטואלי שה-LLM חילץ אם רלוונטי. למשל לשאלה "מה התקציב?" — `value="5000"`. אופציונלי; null אם אין ערך מובחן.
- `classified_at`: timestamp של סיווג. שימושי ל-debug אם הסיווג היה בעייתי.

### 1.1 — האם זה דורש Migration

תיאורטית — לא, כי JSONB גמיש. הקוד הישן של 5.2 כותב string, הקוד החדש של 5.3 כותב object. אבל אם משתמש קיים יש לו שיחה פעילה עם `answers` במבנה הישן — הקוד החדש יפול ב-parsing.

**שתי אפשרויות:**

א. **Migration שמעדכן רשומות קיימות** — מבטל את ההפרדה בין שתי השכבות, סיכון נמוך אם בכלל יש רשומות.

ב. **קוד שמתמודד עם שני המבנים** — אם `answers["1"]` הוא string, להמיר ל-object on-the-fly.

**הצעה: אופציה א'**, כי 5.2 עדיין לא ב-CC לפי ה-HANDOFF. אם הוא יגיע ל-CC לפני 5.3 — בכל מקרה כתוב את ה-SQL החדש מההתחלה ב-5.2. **שווה לעדכן את 5.2** כדי שלא יהיה patch מיותר.

```sql
-- 0024_bot_answers_structure.sql (אם 5.2 כבר רץ עם המבנה הישן)
-- אחרת — לעדכן את 0023 ישירות.

BEGIN;

-- אין שינוי לסכימת ה-DB (JSONB גמיש), אבל יש לאפס שיחות פעילות במבנה הישן.
-- ב-MVP אין שיחות אמיתיות עדיין, אז זה safe.

UPDATE public.bot_conversations
SET answers = '{}'::jsonb
WHERE state NOT IN ('completed_calendly', 'completed_handoff', 'abandoned', 'send_failed')
  AND answers != '{}'::jsonb;

-- (אופציונלי — תיעוד ב-comment)
COMMENT ON COLUMN public.bot_conversations.answers IS
  'מצטבר התשובות עם סיווג. מבנה (5.3 ואילך): {"1": {raw, category, value, classified_at}, ...}.';

COMMIT;
```

**אם 5.2 עוד לא ב-CC** — אמיר יעדכן את ה-COMMENT במסמך 5.2 כדי שהמבנה החדש יהיה ה-default מההתחלה.

---

## חלק 2 — הפרומפטים

### 2.1 — מבנה התיקייה

מתרחב על 3.1.5:

```
app/prompts/
├── phase3/
│   └── ... (קיים)
└── phase5/
    ├── classify_answer.txt
    ├── generate_deviation_response.txt
    └── tones/   # אופציונלי, אם נחליט שגם ה-deviation response תלוי בטון
        ├── professional.txt
        ├── friendly.txt
        ├── luxury.txt
        ├── direct.txt
        └── authoritative.txt
```

ההחלטה לגבי תיקיית `tones/` ב-phase5: כן, נשתמש בה. תגובת fallback של הבוט צריכה לשמור על הטון של המותג, כפי שהוגדר ב-`quiz_responses.brand_tone` של הקמפיין (אחד מ-5 הערכים מ-3.1.5). זה משתמש באותם קבצי טון בדיוק כמו ב-phase3 — אז במקום לשכפל, נציע **symlink** או **import** מ-`app/prompts/phase3/tones/`. ב-`prompts_service.build()` יכול לקבל פרמטר `phase` שמכוון לתיקייה הנכונה. **הצעה פשוטה יותר:** להעתיק את 5 הקבצים ל-`phase5/tones/` (כפילות מינורית, אבל כל phase עצמאי). ניהול שינויים — אם הטון משתנה ב-phase3, גם phase5 משתנה ידנית. ב-MVP — סבבה.

### 2.2 — `classify_answer.txt`

הפרומפט הראשון. תפקידו: לקבל את ההודעה + הקשר + השאלה ולהחזיר JSON עם הסיווג. אני אכתוב את התוכן המלא כאן כי זה הליבה של 5.3.

```
You are an answer classifier for a sales screening bot.

Your job: classify whether the lead's message is an answer to the question we asked, or a deviation (asking something else, off-topic, irrelevant).

## Context

The lead is interacting with a WhatsApp bot from a small business. The business asked them a screening question. We need to determine if their reply answers the question.

## Inputs

The question we asked the lead:
"{question_text}"

What we expected as a valid answer (semantic hint, may be empty):
"{expected_answer_hint}"

The lead's reply:
"{lead_message}"

Recent conversation history (last 3 messages, chronological order):
{conversation_history}

## Your Task

Classify the lead's reply into ONE of these categories:

1. **yes** — The lead clearly answered the question with an affirmative or positive response. Examples: "yes", "sure", "I am", "definitely", "yeah I'm interested".

2. **no** — The lead clearly answered with a negative response. Examples: "no", "I don't", "not interested", "I'm not".

3. **unclear** — The lead replied in a way that addresses the question but isn't a clean yes/no. This includes numeric values, specific text values, or ambiguous responses that are still ON TOPIC. Examples for "What is your budget?": "5000 shekel", "around 3K", "depends on the value". Examples for "Where do you live?": "Tel Aviv", "north of Israel".

4. **deviation** — The lead is NOT answering the question. They might be asking something else, sharing irrelevant info, expressing frustration, or going off topic. Examples: "How much does this cost?" (when we asked about their budget), "Can I speak to someone?", "Tell me more about your service".

## Critical Rules

- "yes"/"no"/"unclear" all mean: the lead engaged with the question. State should advance.
- "deviation" means: the lead did not engage with the question. State should NOT advance. We need to redirect them.
- If the expected_answer_hint is empty, rely on semantic understanding of whether the reply addresses the question.
- A short reply like "כן" or "לא" is enough to classify as yes/no — don't require elaboration.
- Hebrew or English replies are equally valid; don't penalize either.
- When in doubt between "unclear" and "deviation", prefer "unclear" if the reply is on-topic.

## Value Extraction (for "unclear" category)

If category is "unclear" and the reply contains a specific value (number, name, location, etc.), extract it in the "value" field. Otherwise, set "value" to null.

For "yes"/"no" categories, "value" is always null.
For "deviation" category, "value" is always null and "deviation_topic" describes what they asked about.

## Output

Return ONLY valid JSON, no other text:

```json
{{
  "category": "yes" | "no" | "unclear" | "deviation",
  "value": "extracted value if unclear, else null",
  "deviation_topic": "brief description of what they asked, only if deviation, else null"
}}
```
```

הפרומפט שם דגש על כמה דברים:
- **כללים מפורשים** עם דוגמאות לכל קטגוריה — LLMs לומדים טוב מדוגמאות.
- **שפה גמישה** (עברית/אנגלית) — חשוב כי הלקוח ישראלי, אבל מודלים נטויים לעבוד טוב יותר באנגלית.
- **טוב יותר unclear מ-deviation** במקרי ספק — דחיית unclear שהיה צריך להיות תקפה היא חוויה גרועה יותר מאשר deviation שהיה צריך להיות unclear.

### 2.3 — `generate_deviation_response.txt`

הפרומפט השני. רק אם classifier החזיר `deviation`. תפקידו: לנסח תגובה שמכירה בסטייה, מבטיחה למה שהליד שאל, ומחזירה אותו לשאלה.

```
You are a polite and warm WhatsApp sales bot.

The lead has just gone off-topic in our screening conversation. They asked something instead of answering the question we asked.

Your job: write ONE short message (1-3 sentences in Hebrew) that:
1. Acknowledges what they asked
2. Tells them we'll forward it to the business owner
3. Gently redirects back to the question we were asking
4. Maintains the brand's tone

## Brand Tone

{tone_instructions}

## Context

The question we asked:
"{question_text}"

What the lead said:
"{lead_message}"

Topic they deviated to (classifier's analysis):
"{deviation_topic}"

Recent conversation history (last 3 messages):
{conversation_history}

## Critical Rules

- Reply in Hebrew.
- Keep it short — 1-3 sentences maximum. WhatsApp users hate walls of text.
- Don't actually try to answer their question. Just acknowledge it'll be passed to the owner.
- Don't be apologetic or formal — be human and warm.
- End by re-asking the question (rephrased naturally — don't just paste it).
- Do NOT include the original question word-for-word. Paraphrase it.

## Output

Return ONLY the message text. No JSON, no metadata, no quotes around it.
```

הפרומפט הזה מוציא טקסט חופשי (לא JSON) כי הוא לטובת שליחה ישירה ללליד. אין מה לעבד מהפלט.

### 2.4 — קבצי הטון

כפי שצוין, אופציה פשוטה: העתקה של 5 קבצי הטון מ-`phase3/tones/` ל-`phase5/tones/`. כל קובץ קצר (~10 שורות). ה-`prompts_service.build()` יודע להזריק את `{tone_instructions}` במקום ה-placeholder לפי הפרמטר `tone` שהוא מקבל.

---

## חלק 3 — Integration: `integrations/openai.py`

תוספת שתי פונקציות:

### 3.1 — `classify_bot_answer(...)`

```python
async def classify_bot_answer(
    question_text: str,
    expected_answer_hint: str | None,
    lead_message: str,
    conversation_history: list[BotMessage],
) -> ClassificationResult:
    """
    קורא ל-gpt-5.2 לסיווג תשובה של ליד.

    Returns ClassificationResult with:
        - category: 'yes' | 'no' | 'unclear' | 'deviation'
        - value: str | None (extracted value if unclear)
        - deviation_topic: str | None (if deviation)

    Raises:
        ClassificationError: אם הסיווג נכשל (LLM החזיר JSON לא תקף, או שגיאת API).
    """
```

זרימה:

1. בניית הפרומפט דרך `prompts_service.build('classify_answer', phase='phase5', question_text=..., expected_answer_hint=..., lead_message=..., conversation_history=format_history(conversation_history))`.
2. קריאה ל-OpenAI Chat Completions:
   - `model=OPENAI_TEXT_MODEL` (מ-config).
   - `messages=[{"role": "user", "content": prompt}]`.
   - `temperature=0` (deterministic — חשוב לסיווג!).
   - `response_format={"type": "json_object"}`.
   - `max_tokens=200` (פלט קצר).
   - `timeout=15` שניות.
3. פענוח ה-JSON. אם נכשל → `ClassificationError`.
4. ולידציה ש-`category` הוא אחד מ-4 הערכים. אם לא → `ClassificationError`.
5. החזרת `ClassificationResult`.

**Retry:** אם הקריאה ל-OpenAI נכשלת ב-transient (429, 5xx) — דרך `classify_openai_error` הקיים מ-3.1.5. retry אוטומטי ב-job דרך 3.0. permanent → throw, job ייכשל סופית.

### 3.2 — `generate_deviation_response(...)`

```python
async def generate_deviation_response(
    question_text: str,
    lead_message: str,
    deviation_topic: str,
    tone: str,
    conversation_history: list[BotMessage],
) -> str:
    """
    קורא ל-gpt-5.2 לניסוח תגובת fallback ללליד שסטה.

    Returns the response text (Hebrew, 1-3 sentences).

    Raises:
        GenerationError: אם הניסוח נכשל.
    """
```

זרימה:

1. בניית הפרומפט דרך `prompts_service.build('generate_deviation_response', phase='phase5', tone=tone, question_text=..., lead_message=..., deviation_topic=..., conversation_history=...)`.
2. קריאה ל-OpenAI:
   - `temperature=0.7` (יותר יצירתי — אנחנו רוצים variation בניסוחים).
   - `max_tokens=300`.
   - **בלי** `response_format` (טקסט חופשי).
3. החזרת ה-text. ולידציה: לא ריק, פחות מ-1500 תווים (WhatsApp limit).

### 3.3 — `format_history(messages)`

פונקציית עזר פנימית שלוקחת רשימת `BotMessage` (3 הודעות אחרונות) וממירה לטקסט שיוזרק לפרומפט:

```
Lead: "שלום, ראיתי את המודעה שלכם"
Bot: "שלום! קודם כל, מה התקציב שלך?"
Lead: "כמה זה עולה בעצם?"
```

מיון כרונולוגי. inbound = "Lead", outbound = "Bot". המרה פשוטה.

---

## חלק 4 — `bot_intelligence_service.py` (חדש)

ה-service שמתאם בין classification ל-generation. זוהי השכבה היחידה שמדברת עם `integrations/openai.py` בהקשר של בוט. ה-`bot_service` (מ-5.2) קורא ל-`bot_intelligence_service` ולא ל-OpenAI ישירות.

### 4.1 — Public API

```python
async def process_lead_reply(
    conversation: BotConversation,
    current_question: BotQuestion,
    lead_message: str,
    recent_history: list[BotMessage],
    brand_tone: str,
) -> ProcessedReply:
    """
    הליבה של 5.3.

    מסווג את ההודעה של הליד. אם תקפה → מחזיר ProcessedReply עם action='advance'.
    אם סטיה → מנסח תגובת fallback ומחזיר ProcessedReply עם action='redirect'.

    Returns:
        ProcessedReply(
            action='advance' | 'redirect',
            classification=ClassificationResult,
            redirect_message=str | None,  # רק אם action='redirect'
        )
    """
```

זרימה פנימית:

1. קריאה ל-`integrations.openai.classify_bot_answer(...)`.
2. אם `category in ('yes', 'no', 'unclear')` → `ProcessedReply(action='advance', classification=...)`.
3. אם `category == 'deviation'`:
   a. קריאה ל-`integrations.openai.generate_deviation_response(...)`.
   b. `ProcessedReply(action='redirect', classification=..., redirect_message=...)`.

### 4.2 — מה ה-service **לא** עושה

- **לא משנה את ה-state.** ה-state בידי `bot_service`.
- **לא שולח הודעות.** השליחה ב-`bot_service` ← `meta_whatsapp.send_text_message`.
- **לא קורא ל-DB.** כל ה-context (`recent_history`, `current_question`, `brand_tone`) מועבר כפרמטרים. ה-service טהור מבחינת DB.

זה חשוב כי שמירה על service טהור (pure-ish) מקלה על טסטים — אפשר לבדוק את הלוגיקה בלי DB mocks.

---

## חלק 5 — שינוי ב-`bot_service` (מ-5.2)

ב-5.2 הלוגיקה של `process_inbound_message` הייתה (פסאודו-קוד):

```
if state == 'asking_question_N':
    save answer to answers[N]
    advance state to asking_question_{N+1} OR fallback
    send next question or fallback message
```

ב-5.3 הלוגיקה הופכת:

```
if state == 'asking_question_N':
    current_question = bot_questions[N]
    recent_history = fetch last 3 messages from bot_messages
    brand_tone = fetch from quiz_responses (via campaign_id from lead → leads → campaigns)
    
    result = await bot_intelligence_service.process_lead_reply(
        conversation, current_question, lead_message, recent_history, brand_tone
    )
    
    if result.action == 'advance':
        save {raw, category, value, classified_at} to answers[N]
        advance state to asking_question_{N+1} OR fallback
        send next question or fallback message
    
    elif result.action == 'redirect':
        # state לא מתקדם!
        # תשובה לא נשמרת ב-answers.
        send result.redirect_message
        
        # אופציונלי: רישום הסטייה ב-bot_messages עם metadata.
```

### 5.1 — מאיפה brand_tone מגיע

החיפוש: `lead.campaign_id → campaigns.id → quiz_responses.campaign_id → quiz_responses.brand_tone`. שאילתה אחת JOIN-ית.

**Edge case:** מה אם ל-conversation יש מספר leads (מערך)? אנחנו צריכים לבחור אחד לטון. הצעה: הראשון (FIFO). הגיוני כי הוא ה-context הראשוני של השיחה.

**Edge case 2:** מה אם אין `quiz_responses`? לא אמור לקרות (4.1 לא מפיקה lead בלי קמפיין שעבר את האשף, ואשף דורש quiz). אם בכל זאת — fallback ל-`tone='professional'` עם warning ב-Sentry.

### 5.2 — שמירת hand-off לבעל העסק

ב-fallback של `human_handoff` (כשהשיחה מסתיימת), ההודעה לבעל העסק כוללת את `answers`. ב-5.2 זה היה raw strings. ב-5.3 — מבנה מורכב. ההצגה לבעל העסק צריכה לכבד את ה-`raw` (התשובה במילים של הליד), לא את ה-`category`. דוגמה:

```
ליד חדש מהקמפיין:
שם: דני (053-1234567)

תשובות:
1. האם אתה בעל עסק עצמאי?
   → כן, אני עצמאי כבר 3 שנים [סווג: כן]

2. מה התקציב שלך?
   → אזורי 5000 שקל [ערך: 5000]
```

הקטגוריה והערך בסוגריים — opcional, ב-MVP אפשר להשאיר רק את ה-raw. החלטה ב-implementation.

---

## חלק 6 — Models ו-Exceptions

### 6.1 — Pydantic types חדשים

ב-`app/models/bot_intelligence.py`:

```python
from typing import Literal
from pydantic import BaseModel
from datetime import datetime

ClassificationCategory = Literal['yes', 'no', 'unclear', 'deviation']

class ClassificationResult(BaseModel):
    category: ClassificationCategory
    value: str | None = None
    deviation_topic: str | None = None

class ProcessedReply(BaseModel):
    action: Literal['advance', 'redirect']
    classification: ClassificationResult
    redirect_message: str | None = None

class AnswerEntry(BaseModel):
    raw: str
    category: ClassificationCategory
    value: str | None
    classified_at: datetime
```

`AnswerEntry` הוא המבנה ב-`bot_conversations.answers[N]`. שימוש ב-Pydantic לוולידציה בקריאה.

### 6.2 — Exceptions חדשים

ב-`app/exceptions/bot.py`:

- `ClassificationError` — LLM החזיר JSON לא תקף או category לא חוקי.
- `GenerationError` — LLM החזיר טקסט ריק/ארוך מדי.

שני אלה חולפים דרך retry policy של 3.0 כ-transient (במקרים מסוימים) או permanent. הצעה: שניהם permanent — אם LLM לא הצליח לסווג טוב, retry לא יעזור (אותם inputs). במקום זה — fallback to deviation behavior (לראות הסבר בסעיף 7).

---

## חלק 7 — מה לעשות אם LLM נכשל

### 7.1 — Classification נכשל

תרחישים:
- OpenAI timeout (15s).
- OpenAI החזיר JSON לא תקף.
- OpenAI rate limit.

**הצעה:** אם classification נכשל אחרי retries (transient), הגישה ה-safest היא **לטפל בהודעה כסטייה גנרית**. כלומר: לא לקדם state, לשלוח תגובה fallback סטטית ("רגע אחד, אני בודק..."), לתעד ב-Sentry שה-classifier נפל.

**למה לא להתייחס כתשובה תקפה?** כי קידום state על בסיס תשובה לא-מסווגת יכול לסמן "לא" כ"כן" ולהמשיך — איבוד דאטה לעסק. עדיף לחזור עוד פעם.

**טקסט הfallback הסטטי:** "תודה, רגע אחד אני בודק עם בעל העסק..." — לא מקדם state, אבל גורם לליד להבין שמשהו קורה.

### 7.2 — Generation נכשל

תרחיש: classification הצליח (סטייה זוהתה) אבל ניסוח התגובה נכשל.

**הצעה:** fallback סטטי: "אעביר את השאלה שלך לבעל העסק. בינתיים, בוא נחזור לשאלה הקודמת — {question_text}". הליד לא יודע שמשהו נכשל. ה-context נשמר.

### 7.3 — היכן זה מתבטא בקוד

ב-`bot_service.process_inbound_message`, עוטפים את `bot_intelligence_service.process_lead_reply` ב-try/except. ב-`ClassificationError` — שולחים תגובת fallback סטטית, לא מקדמים state. ב-`GenerationError` — שולחים תגובת deviation גנרית, גם לא מקדמים state.

---

## חלק 8 — Env vars

| שם | משמעות | חובה? |
|---|---|---|
| `OPENAI_API_KEY` | מ-3.1.5 | חובה. |
| `OPENAI_TEXT_MODEL` | מ-3.1.5, default `gpt-5.2` | אופציונלי. |

אין ENV חדש ב-5.3.

---

## חלק 9 — שמות הקבצים החדשים והשינויים

| קובץ | תוכן |
|---|---|
| `app/prompts/phase5/classify_answer.txt` | **חדש** — פרומפט סיווג |
| `app/prompts/phase5/generate_deviation_response.txt` | **חדש** — פרומפט ניסוח תגובת fallback |
| `app/prompts/phase5/tones/{5 קבצים}.txt` | **חדש** — העתקה מ-phase3/tones |
| `app/integrations/openai.py` | **תיקון** — הוספת `classify_bot_answer`, `generate_deviation_response`, `format_history` |
| `app/services/bot_intelligence_service.py` | **חדש** — `process_lead_reply` orchestrator |
| `app/services/bot_service.py` (מ-5.2) | **תיקון** — `process_inbound_message` משנה לקריאה ל-`bot_intelligence_service` |
| `app/services/prompts_service.py` (מ-3.1.5) | **תיקון** — תמיכה בפרמטר `phase` (אם עוד אין) |
| `app/models/bot_intelligence.py` | **חדש** — Pydantic: `ClassificationResult`, `ProcessedReply`, `AnswerEntry` |
| `app/exceptions/bot.py` (מ-5.2) | **תיקון** — הוספת `ClassificationError`, `GenerationError` |
| `supabase/migrations/0024_bot_answers_structure.sql` | **אופציונלי** — רק אם 5.2 כבר רץ עם מבנה ישן (תלוי בלוח זמנים) |

---

## חלק 10 — Done של 5.3

- שני קבצי פרומפט קיימים תחת `app/prompts/phase5/`, נטענים דרך `prompts_service.build()`.
- 5 קבצי טון תחת `app/prompts/phase5/tones/`.
- `integrations.openai.classify_bot_answer(...)` עובד: מחזיר `ClassificationResult` תקין. בדיקה ידנית עם 4 תרחישים (yes/no/unclear/deviation) → סיווג נכון בכולם.
- `integrations.openai.generate_deviation_response(...)` עובד: מחזיר תגובה בעברית 1-3 משפטים.
- `bot_intelligence_service.process_lead_reply(...)` מתאם נכון בין שני הקריאות.
- `bot_service.process_inbound_message` (מ-5.2) משתמש ב-intelligence ל-state advancing. תשובה תקפה → state מתקדם + שאלה הבאה. סטייה → state לא מתקדם, נשלחת תגובת fallback מנוסחת.
- `bot_conversations.answers` נשמר במבנה החדש `{raw, category, value, classified_at}`.
- Brand tone נמשך נכון מ-`quiz_responses` (דרך לקוח → קמפיין).
- Fallback סטטי עובד: classification timeout → לא מקדם state, שולח "רגע אחד...". generation timeout → שולח תגובה גנרית.
- בדיקה ידנית מקצה לקצה: ליד עונה ישר → state מתקדם. ליד שואל שאלה אחרת → לא מתקדם, מקבל ניסוח חכם. שתי תרחישי הקצה (סיווג נכשל, ניסוח נכשל) מתנהגים כצפוי.
- כל הטסטים החדשים עוברים.

---

## חלק 11 — לא ב-5.3

- שינוי במכונת המצבים — נשארת מ-5.2.
- ניסוח של הודעת פתיחה וייעוץ עתידי — האזור הזה מתועד ב-5.4.
- Multi-turn deviations (הליד סוטה, הבוט עונה, הליד סוטה שוב על משהו אחר) — ב-MVP מטופל אותו דבר. שווה לעקוב אם זה הופך ל-pain point.
- Sentiment analysis (הליד כועס? מתוסכל?) — Post-MVP.
- Detection של hate speech / inappropriate content — Post-MVP. אם הליד כותב משהו פוגעני, ה-classifier יראה את זה כ-deviation וה-generator יענה בנימוס.
- LLM cost tracking פר-conversation — UI ל-dashboard עתידי, לא 5.3.
- Hybrid classification (קוד דטרמיניסטי + LLM) למקרים פשוטים — כרגע תמיד LLM. אם רואים שהרבה תשובות הן "כן"/"לא" טהורים — אפשר לחסוך עלות עם regex/keyword matching לפני קריאה ל-LLM.

---

## חלק 12 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`gpt-5.2` תמיכה ב-`response_format`:** **לאמת בזמן המימוש.** OpenAI שינו את ה-API לאחרונה. אם `response_format={"type": "json_object"}` לא נתמך — להחליף ב-`response_format={"type": "json_schema", "json_schema": {...}}`. ה-schema פשוט: `{category: enum, value: nullable string, deviation_topic: nullable string}`.

2. **`temperature=0` לסיווג קריטי.** עם temperature חיובי, אותו input יכול להחזיר category שונה בקריאות שונות. ב-classification רוצים דטרמיניסטיות.

3. **רמז סמנטי ריק (`expected_answer_hint=None`):** הפרומפט מקבל "" במקום. ה-LLM ממילא יודע להתמודד עם הנחיה ריקה ("rely on semantic understanding"). אבל **שווה לוודא** שזה עובד טוב באיכות בבדיקה ידנית.

4. **conversation_history עם 0 הודעות (השיחה רק התחילה):** הפרומפט מקבל "" או "No prior history" במקום. השאלה עצמה מספיק context לסיווג ההודעה הראשונה.

5. **טוקנים — הערכת עלות.** קריאה ממוצעת לסיווג: ~600 input + ~50 output. ב-gpt-5.2 (לאמת מחיר!) — ~$0.001-0.002 לקריאה. בקמפיין עם 100 לידים × 5 שאלות = 500 קריאות = $0.50-1.00. זניח ביחס למחיר החבילה (₪597+).

6. **`gpt-5.2-mini` אם קיים** — לבחון בעתיד. אם המודל המיני יודע לסווג טוב — חיסכון של ~80% בעלות. ב-MVP נישאר על gpt-5.2 ונבדוק.

7. **טסטים מומלצים:**
   - test_classifier_yes_simple → "כן" → category=yes.
   - test_classifier_no_simple → "לא" → category=no.
   - test_classifier_unclear_with_value → "5000 שקל" לשאלת תקציב → category=unclear, value="5000".
   - test_classifier_deviation → "מה המחיר?" לשאלת תקציב → category=deviation, deviation_topic מתאר.
   - test_classifier_empty_hint → עובד גם בלי hint.
   - test_generator_returns_hebrew → תגובה בעברית.
   - test_generator_respects_tone → השוואת luxury vs friendly — אורך/סגנון משתנים.
   - test_intelligence_advance_on_valid → ProcessedReply.action='advance'.
   - test_intelligence_redirect_on_deviation → ProcessedReply.action='redirect', message מנוסח.
   - test_bot_service_advances_state_on_valid → state עובר מ-3 ל-4.
   - test_bot_service_skips_state_on_deviation → state נשאר 3, ניסיון שני יסווג שוב.
   - test_bot_service_fallback_on_classification_error → לא מקדם state, שולח הודעה סטטית.

8. **CC חייב לעדכן את `prompts_service.build()`** כדי לתמוך ב-`phase` parameter — אם זה עוד לא קיים מ-3.1.5. אם יש — לוודא שעובד עם `phase5`.

9. **לוג כל classification ל-Sentry context** (לא רק שגיאות). שימושי ל-tuning של הפרומפט אם נראה שסיווגים שגויים שכיחים. רמת INFO, לא ERROR.

10. **בדיקת מקצועי לפני release:** אמיר וגיא לרוץ סימולציה ידנית של 20 הודעות (טיוטה ב-Excel — "ליד שלח X, מה הסיווג הנכון לכן?") ולהריץ ב-classifier. אם accuracy < 90% — חזור ל-prompt tuning. אם > 95% — מוכן.

11. **ייתכן שיהיו hallucinations:** LLM ינסח תגובת fallback שכוללת מידע שגוי על העסק (למשל "המחיר הוא $500" כשבעצם הוא $300). הסיכון נמוך כי הפרומפט אומר "אל תענה — תעביר", אבל לא אפס. הצעה: prompt-level mitigation (חוק מפורש "אסור לתת מידע על העסק, רק להפנות לבעל העסק"). monitoring: לקוח חוזר עם תלונה — בדיקה ידנית.

---

**סוף מסמך 5.3.**

---

# Session 5.4 — WhatsApp Templates ו-Follow-up אחרי 23.5h

> **עדכון להוספה ל-ROADMAP.md תחת Session 5.4.** Phase 5.4 — המעבר של הבוט מ-dev לייצור אמיתי. מטא דורשת **template מאושר מראש** לכל הודעה business-initiated (הודעה שאנחנו פותחים, לא הליד). הסשן הזה בונה את שלוש ה-templates שהבוט שלנו צריך, את ה-cron שמזהה לידים שלא ענו ושולח להם follow-up, ואת מנגנון ה-soft-disable שמונע מהבוט לשלוח לפני שכל ה-templates אושרו. תלוי ב-5.0 (קו ייצור), 5.1 (`bot_configs` עם `business_name`/`service_label`/`closing_cta`), 5.2 (state machine + outbound sender), 5.3 (intelligence). חוסם את ההפעלה האמיתית בייצור — בלי 5.4 הבוט עובד רק עם ה-test phone number של מטא.

---

## תיאור Session 5.4

**מטרה:** להפוך את הבוט שעובד ב-dev (עם 5.0-5.3) למוכן לייצור אמיתי. שני שינויים מרכזיים:

**ראשון — מעבר ל-templates.** ה-`bot_service.send_text_message` של 5.2 שולח טקסט חופשי. ב-test phone number של מטא זה עובד. בייצור, מטא חוסמת כל הודעה business-initiated שאינה template. ה-Session הזה מוסיף `send_template_message` שמשתמש ב-Meta Cloud API templates endpoint, ומחליף 3 שליחות באפליקציה לקרוא בעיקר אליו. שאר הקוד נשאר זהה — אחרי שהליד ענה, אנחנו בתוך חלון 24h ויכולים להמשיך עם טקסט חופשי כמו עכשיו.

**שני — Follow-up cron.** בוט שאיבד 50% מהלידים שלא הספיקו לענות מיד = בוט גרוע. ה-Session מוסיף scheduler שרץ כל שעה, מוצא שיחות עם `last_message_at` לפני 23.5 שעות, ושולח template `bot_followup` כדי לעורר את הליד לפני שחלון ה-24h נסגר. אם גם ה-follow-up לא עורר תגובה — אחרי 48h מ-`last_message_at` השיחה מסומנת `abandoned`.

**שלוש templates שיוגשו למטא:**

| שם ה-template | שפה | קטגוריה | מתי נשלחת |
|---|---|---|---|
| `bot_opening_v1` | `he` | UTILITY | מ-`initiate_bot_conversation` job מיד אחרי קליטת ליד |
| `bot_followup_v1` | `he` | UTILITY | מ-`bot_followup_cron` כל שעה, ללידים שלא ענו 23.5h |
| `bot_handoff_to_owner_v1` | `he` | UTILITY | בסיום `human_handoff` flow, נשלחת לבעל העסק (לא לליד) |

**מה ב-Session הזה:**
- 3 קבצי טקסט תחת `app/prompts/phase5/templates/` שמכילים את ה-template bodies.
- מסמך הגשה ידני למטא (אדמין מבצע פעם אחת ב-Meta Business Manager UI).
- הוספת `send_template_message` ל-`integrations/meta_whatsapp.py`.
- שינוי ב-`bot_service`: כל שליחה business-initiated עוברת דרך templates במקום טקסט חופשי. שליחות בתוך חלון 24h נשארות טקסט חופשי.
- מנגנון `template_registry` שמכיר ב-3 ה-templates ומחזיר את המבנה הנכון לכל אחת.
- `bot_followup_cron.py` חדש ב-worker, רץ כל שעה.
- Migration קטן ל-`bot_conversations` להוסיף `followup_sent_at`.
- `app_settings` table חדש (אם לא קיים) עם flag גלובלי `WHATSAPP_PRODUCTION_READY` שמושבת עד שמטא אישרה את כל ה-3.
- שינוי ב-5.1 — `opening_message` (טקסט יחיד) מתפצל לשלושה שדות: `business_name`, `service_label`, `closing_cta`. patch על 5.1 (ראה חלק 1).

**מה לא ב-Session הזה:**
- UI ללקוח לעריכת ה-templates — לא ב-MVP. שמות ה-templates קבועים בקוד.
- ניהול multiple template versions ידני בייצור (v2, v3) — דפוס מוסבר אבל לא מומש; אם גיא ירצה לשנות אחרי אישור, ימומש פעם נפרדת.
- מעבר מעבר ל-categorization של MARKETING (אם מטא יורידו את ה-quality rating שלנו ונצטרך לעבור) — לא רלוונטי ב-MVP.
- Localization — עברית בלבד.
- Template buttons (Quick Reply / URL / Phone) — ה-templates שלנו plain text. אם בעתיד נרצה כפתורים — extension.
- Status updates של templates מ-Meta webhooks (אם מטא דוחה template אחרי שאישרה) — לא ב-MVP. אדמין יראה ידנית ב-Business Manager.
- Scheduled jobs ייעודיים פר-שיחה (run_at = started_at + 23.5h) — דחיתי במפורש בהחלטה 6 לטובת cron גלובלי שעתי.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מספר templates | **3 — opening, followup, handoff_to_owner.** |
| 2 | מבנה template | **template אחד גלובלי עם 3-4 משתנים**, לא template פר-לקוח. |
| 3 | שפה | **עברית בלבד.** |
| 4 | קטגוריה | **UTILITY** לכל ה-3. שום שפה שיווקית. |
| 5 | ניהול templates | **proactive** — אמיר/גיא מגישים פעם אחת, לא לקוחות. |
| 6 | follow-up cron | **גלובלי כל שעה**, סורק `last_message_at < now() - 23.5h`. |
| 7 | abandoned | **48h** מ-`last_message_at` (כולל follow-up). |
| 8 | אסטרטגיית מעבר | **soft-disable** עד אישור כל ה-3. הרשמה ל-Premium ממשיכה (לא חוסמים). |
| 9 | הודעת פתיחה | **3 משתנים מהלקוח** (business_name, service_label, closing_cta) — patch ל-5.1. |
| 10 | שינוי נוסחים אחרי אישור מטא | **versioning ידני** — `bot_opening_v2` חדש, אישור חדש, feature flag. לא ב-5.4. |

---

## חלק 1 — Patch ל-5.1: שלושה שדות במקום `opening_message`

הניסוח של ה-template דורש שינוי במבנה הקיים של `bot_configs`. ב-5.1 נכתב `opening_message` כשדה טקסט יחיד שהלקוח כותב בחופשיות. עכשיו אנחנו מבינים שזה לא יעבוד כי המבנה חייב להיות סטנדרטי לטובת אישור מטא. הלקוח שולט רק על 3 ערכים קטנים, לא על הניסוח המלא.

### 1.1 — שינוי בסכימה

```sql
-- 0024_bot_configs_template_fields.sql (אחרי 5.3)
-- אם 5.1 עוד לא ב-CC — לעדכן את 5.1 ישירות.

BEGIN;

ALTER TABLE public.bot_configs
  ADD COLUMN business_name text,
  ADD COLUMN service_label text,
  ADD COLUMN closing_cta text;

-- ולידציה (לא NOT NULL בהתחלה כדי לאפשר migration של רשומות קיימות)
ALTER TABLE public.bot_configs
  ADD CONSTRAINT bot_configs_business_name_length CHECK (
    business_name IS NULL OR length(trim(business_name)) BETWEEN 1 AND 50
  ),
  ADD CONSTRAINT bot_configs_service_label_length CHECK (
    service_label IS NULL OR length(trim(service_label)) BETWEEN 1 AND 80
  ),
  ADD CONSTRAINT bot_configs_closing_cta_length CHECK (
    closing_cta IS NULL OR length(trim(closing_cta)) BETWEEN 1 AND 30
  );

-- backfill (במקרה שיש כבר רשומות; ב-MVP לא צפוי)
UPDATE public.bot_configs
SET business_name = 'העסק',
    service_label = 'השירות',
    closing_cta = 'מעוניין?'
WHERE business_name IS NULL;

-- אחרי backfill — אכיפת NOT NULL
ALTER TABLE public.bot_configs
  ALTER COLUMN business_name SET NOT NULL,
  ALTER COLUMN service_label SET NOT NULL,
  ALTER COLUMN closing_cta SET NOT NULL;

-- הסרת opening_message הישן (אם הוא בעצם לא בשימוש)
-- שווה לדון: אולי להשאיר לתיעוד? בכוונה אני משאיר ולא מוחק
-- כי אם 5.1 כבר ב-CC ואי אפשר לוודא — עדיף לא לשבור.

COMMIT;
```

### 1.2 — שינוי ב-Pydantic

ב-`app/models/bot.py`:

```python
class BotConfigInput(BaseModel):
    business_name: str = Field(min_length=1, max_length=50)
    service_label: str = Field(min_length=1, max_length=80)
    closing_cta: str = Field(min_length=1, max_length=30)
    fallback_action: FallbackAction
    fallback_value: str | None = Field(default=None, max_length=500)
    default_appointment_duration_minutes: int = Field(default=60, ge=15, le=240)
    questions: list[BotQuestionInput] = Field(min_length=0, max_length=5)
```

### 1.3 — מה לעשות עם `opening_message` הקיים

שתי אופציות. **א'** — להסיר את העמודה ב-migration. אם 5.1 ב-CC ויש פיצ'ר בקוד שמשתמש בו — שובר. **ב'** — להשאיר את העמודה אבל לא לכתוב אליה יותר. בעיה: גורר חוסר עקביות עתידית.

**הצעה:** אם 5.1 עוד לא ב-CC — לעדכן את 5.1 ישירות, אין `opening_message` בכלל. אם 5.1 כבר ב-CC — להסיר את העמודה ב-`0024` הזה. שווה לוודא לפני המימוש.

---

## חלק 2 — קבצי ה-templates

ה-templates יישבו כקבצי טקסט תחת `app/prompts/phase5/templates/`. הסיבה: עקבי עם דפוס הפרומפטים של 3.1.5, גרסת control ב-git, ועריכה פשוטה לפני הגשה למטא.

**הערה חשובה:** הקבצים האלה הם **טיוטה לפני הגשה**. אחרי שמטא מאשרת — שינוי דורש versioning (`bot_opening_v2`). השמות שיוגדרו כאן הם **placeholder** בקוד; השמות הסופיים נקבעים בעת ההגשה למטא.

### 2.1 — `app/prompts/phase5/templates/bot_opening.txt`

```
שלום {{1}}, אני הנציג הדיגיטלי של {{2}}. ראינו שהשארת אצלנו פרטים, אשאל אותך כמה שאלות קצרות, כדי שנמצא לך את {{3}} הכי מתאים. {{4}}
```

מיפוי משתנים:
- `{{1}}` — שם הליד מטופס Meta (`lead.contact_name`). אם ריק — fallback ל-"חבר".
- `{{2}}` — `bot_configs.business_name`.
- `{{3}}` — `bot_configs.service_label`.
- `{{4}}` — `bot_configs.closing_cta`.

### 2.2 — `app/prompts/phase5/templates/bot_followup.txt`

```
היי {{1}}, השיחה שלנו נעצרה באמצע. אם זה עדיין רלוונטי, תכתוב לי בחזרה ונמשיך מאיפה שעצרנו.
```

מיפוי משתנים:
- `{{1}}` — שם הליד. אם ריק — fallback ל-"חבר".

### 2.3 — `app/prompts/phase5/templates/bot_handoff_to_owner.txt`

```
ליד חדש מהקמפיין של {{1}}. שם: {{2}}, טלפון: {{3}}. {{4}}
```

מיפוי משתנים:
- `{{1}}` — `bot_configs.service_label` (שם השירות שעליו פנו).
- `{{2}}` — `lead.contact_name`. אם ריק — fallback ל-"לא צוין".
- `{{3}}` — `lead.contact_phone` (פורמט קריא, לא E.164).
- `{{4}}` — סיכום התשובות בפורמט "שאלה 1: תשובה | שאלה 2: תשובה | ...". חתוך ל-1024 תווים אם חורג (מטא limit).

### 2.4 — דוגמת ערכים להגשה למטא

בעת הגשת template, מטא דורשת **ערכים לדוגמה** עבור כל משתנה כדי שיוכלו לראות איך ההודעה תיראה. הצעה לדוגמאות:

**`bot_opening_v1`:**
- `{{1}}`: דני
- `{{2}}`: אוטומציה בקלות
- `{{3}}`: שירות האוטומציה
- `{{4}}`: מעוניין?

**`bot_followup_v1`:**
- `{{1}}`: דני

**`bot_handoff_to_owner_v1`:**
- `{{1}}`: שירות האוטומציה
- `{{2}}`: דני כהן
- `{{3}}`: 050-1234567
- `{{4}}`: שאלה 1: כן | שאלה 2: 5000 שח | שאלה 3: מהיר ככל האפשר

ערכים אלה משמשים את מטא רק לבדיקה. הם לא נשלחים בייצור.

---

## חלק 3 — Template Registry

מתאם לוגי בין שמות ה-templates ל-DB ובין שמות ה-templates במטא. הוא מבודד את הקוד מ-hard-coded names.

### 3.1 — `app/services/whatsapp_template_registry.py` (חדש)

```python
from typing import Literal
from dataclasses import dataclass

TemplateName = Literal['bot_opening', 'bot_followup', 'bot_handoff_to_owner']

@dataclass(frozen=True)
class TemplateDefinition:
    internal_name: TemplateName       # שם פנימי בקוד
    meta_name: str                     # שם בפועל ב-Meta אחרי אישור (כולל version)
    language_code: str                 # 'he'
    variable_count: int                # כמה משתנים יש בtemplate
    category: str                      # 'UTILITY'
```

ה-registry מחזיק dict פנימי ומחזיר את ה-`TemplateDefinition` הנכון לפי שם פנימי.

**שמות ב-Meta** מוגדרים ב-ENV vars (לא קבועים בקוד) כך שאם נעבור ל-v2 בעתיד — שינוי env vars ב-Render בלבד.

```python
# ENV vars:
# META_TEMPLATE_BOT_OPENING_NAME=bot_opening_v1
# META_TEMPLATE_BOT_FOLLOWUP_NAME=bot_followup_v1
# META_TEMPLATE_BOT_HANDOFF_NAME=bot_handoff_to_owner_v1

TEMPLATES = {
    'bot_opening': TemplateDefinition(
        internal_name='bot_opening',
        meta_name=os.getenv('META_TEMPLATE_BOT_OPENING_NAME', 'bot_opening_v1'),
        language_code='he',
        variable_count=4,
        category='UTILITY',
    ),
    'bot_followup': TemplateDefinition(
        internal_name='bot_followup',
        meta_name=os.getenv('META_TEMPLATE_BOT_FOLLOWUP_NAME', 'bot_followup_v1'),
        language_code='he',
        variable_count=1,
        category='UTILITY',
    ),
    'bot_handoff_to_owner': TemplateDefinition(
        internal_name='bot_handoff_to_owner',
        meta_name=os.getenv('META_TEMPLATE_BOT_HANDOFF_NAME', 'bot_handoff_to_owner_v1'),
        language_code='he',
        variable_count=4,
        category='UTILITY',
    ),
}

def get_template(name: TemplateName) -> TemplateDefinition:
    return TEMPLATES[name]
```

---

## חלק 4 — `send_template_message` ב-`meta_whatsapp.py`

תוספת ל-integration שנבנה ב-5.0 ו-5.2.

### 4.1 — פונקציה חדשה

```python
async def send_template_message(
    phone_number_id: str,
    to: str,
    template_name: str,        # שם meta_name, כפי שמטא אישרה
    language_code: str,        # 'he'
    parameters: list[str],     # ערכים למשתנים בסדר {{1}}, {{2}}, ...
    idempotency_key: str,
) -> SendResult:
    """
    שולח template message ל-Meta Cloud API.

    Returns SendResult with meta_message_id.
    Raises MetaError on failure (classified by classify_meta_error).
    """
```

הקריאה ל-Meta:

```
POST /{phone_number_id}/messages
Authorization: Bearer {META_ACCESS_TOKEN}
Idempotency-Key: {idempotency_key}
Content-Type: application/json

{
  "messaging_product": "whatsapp",
  "to": "972501234567",
  "type": "template",
  "template": {
    "name": "bot_opening_v1",
    "language": {"code": "he"},
    "components": [
      {
        "type": "body",
        "parameters": [
          {"type": "text", "text": "דני"},
          {"type": "text", "text": "אוטומציה בקלות"},
          {"type": "text", "text": "שירות האוטומציה"},
          {"type": "text", "text": "מעוניין?"}
        ]
      }
    ]
  }
}
```

תשובה זהה ל-`send_text_message`:

```json
{
  "messages": [{"id": "wamid.HBg..."}]
}
```

### 4.2 — Error handling

נשען על `classify_meta_error` הקיים. תרחישים ספציפיים ל-templates:

- **132xxx — Template error.** ה-template לא קיים, או הוסר, או נדחה. permanent. החזרה: `MetaPermanentError("Template '{name}' not approved or not found")`. Sentry alert חמור.
- **131x — Parameter mismatch.** שלחנו פחות/יותר משתנים ממה שה-template דורש. permanent — bug שלנו ב-registry. Sentry alert.
- **131056 — Recipient cannot receive.** הליד חסם או לא רשום ב-WhatsApp. permanent. mark שיחה כ-`send_failed`, לא retry.
- **כל השאר** — לפי הסיווג הקיים.

---

## חלק 5 — שינוי ב-`bot_service`

ה-service המרכזי של 5.2 קורא ל-`send_text_message` בכל שליחה. עכשיו צריך להבחין בין שני סוגים.

### 5.1 — מתי template ומתי טקסט חופשי

**Template (business-initiated):**
- הודעת פתיחה ב-`initiate_bot_conversation` → `bot_opening`.
- Follow-up ב-`bot_followup_cron` → `bot_followup`.
- Handoff לבעל העסק ב-fallback של `human_handoff` → `bot_handoff_to_owner`.

**טקסט חופשי (user-initiated, בתוך חלון 24h):**
- שאלות הסינון אחרי שהליד ענה להודעת פתיחה.
- תגובות fallback לסטיות (מ-5.3).
- הודעת סיום (Calendly link, או הודעת אישור handoff ללליד).
- תגובה ל-unsupported media (5.2).

### 5.2 — ה-flow המעודכן

ב-`bot_service.send_outgoing_message`:

```python
async def send_outgoing_message(
    conversation: BotConversation,
    message_type: Literal['template', 'free_form'],
    template_name: TemplateName | None,
    parameters: list[str] | None,
    body: str | None,
) -> SendResult:
    """
    מאחד את שתי דרכי השליחה. נקרא מ-bot_service ב-handlers שונים.
    """
    if message_type == 'template':
        template_def = template_registry.get_template(template_name)
        result = await meta_whatsapp.send_template_message(
            phone_number_id=conversation.whatsapp_line.phone_number_id,
            to=conversation.contact_key,
            template_name=template_def.meta_name,
            language_code=template_def.language_code,
            parameters=parameters,
            idempotency_key=uuid4().hex,
        )
    else:
        result = await meta_whatsapp.send_text_message(
            phone_number_id=conversation.whatsapp_line.phone_number_id,
            to=conversation.contact_key,
            body=body,
            idempotency_key=uuid4().hex,
        )
    
    # שמירה ב-bot_messages (לוגיקה זהה לשתי הדרכים)
    await save_outbound_message(conversation, result, message_type, ...)
    return result
```

### 5.3 — הוספת `message_type` לטבלת `bot_messages`

ב-5.2 ה-`message_type` היה רק `text`/`image`/וכו'. עכשיו צריך להוסיף `template` למבחין שליחה דרך template.

```sql
-- בתוך 0024 או patch ל-0023:
ALTER TABLE public.bot_messages DROP CONSTRAINT IF EXISTS bot_messages_message_type_check;
ALTER TABLE public.bot_messages ADD CONSTRAINT bot_messages_message_type_check CHECK (
  message_type IN ('text', 'template', 'image', 'audio', 'video', 'document', 'unsupported')
);

-- עמודה חדשה: שם ה-template (רק ב-outbound template messages)
ALTER TABLE public.bot_messages ADD COLUMN template_name text;
```

המטרה: לוגים ברורים על מה נשלח כ-template. אם נצטרך debug — אפשר לראות בדיוק איזה template נשלח לאיזו שיחה.

---

## חלק 6 — Follow-up cron

### 6.1 — מבנה ה-job

ב-`worker/handlers/bot_followup_cron.py` (חדש):

```python
async def bot_followup_cron():
    """
    רץ כל שעה (configured ב-Render scheduled jobs).
    
    שני בדיקות באותו run:
    1. שיחות שצריכות follow-up — last_message_at < now() - 23.5h, state non-terminal,
       followup_sent_at IS NULL → שלח template bot_followup.
    2. שיחות שצריכות abandon — last_message_at < now() - 48h, state non-terminal,
       followup_sent_at IS NOT NULL → mark abandoned.
    """
    
    # שלב 1 — שלח follow-ups
    eligible_for_followup = await fetch_conversations_eligible_for_followup()
    for conversation in eligible_for_followup:
        await send_followup(conversation)
    
    # שלב 2 — סמן abandoned
    eligible_for_abandon = await fetch_conversations_eligible_for_abandon()
    for conversation in eligible_for_abandon:
        await mark_abandoned(conversation)
```

### 6.2 — בחירת שיחות ל-follow-up

```sql
SELECT * FROM bot_conversations
WHERE state NOT IN ('completed_calendly', 'completed_handoff', 'abandoned', 'send_failed')
  AND last_message_at < now() - interval '23 hours 30 minutes'
  AND followup_sent_at IS NULL
ORDER BY last_message_at ASC
LIMIT 100;  -- batch size
```

ה-`LIMIT 100` חשוב. cron לא אמור לסרוק את כל ה-DB בכל פעם. אם יש יותר מ-100 מועמדים — הם יטופלו בשעה הבאה. במציאות לא צפויה.

### 6.3 — שליחת follow-up

```python
async def send_followup(conversation: BotConversation):
    parameters = [conversation.contact_name or 'חבר']
    
    try:
        result = await bot_service.send_outgoing_message(
            conversation=conversation,
            message_type='template',
            template_name='bot_followup',
            parameters=parameters,
            body=None,
        )
        
        # עדכון הspeak:
        await update_conversation(
            conversation.id,
            followup_sent_at=now(),
            last_message_at=now(),
        )
    except MetaPermanentError as e:
        await mark_send_failed(conversation, str(e))
    except MetaTransientError:
        # 3.0 יעשה retry של ה-job, אבל זה cron — אין retry במשמעות הרגילה.
        # נסה שוב בשעה הבאה (followup_sent_at עדיין NULL).
        sentry_sdk.capture_exception()
```

**חשוב — race condition:** מה אם הליד עונה ב-23.45h, וב-23.51h ה-cron רץ ומחליט לשלוח follow-up? ה-`last_message_at` כבר התעדכן (ל-23.45h), ולכן ה-cron לא יבחר אותה. בטוח.

### 6.4 — סימון abandoned

```sql
SELECT * FROM bot_conversations
WHERE state NOT IN ('completed_calendly', 'completed_handoff', 'abandoned', 'send_failed')
  AND last_message_at < now() - interval '48 hours'
  AND followup_sent_at IS NOT NULL
LIMIT 100;
```

```python
async def mark_abandoned(conversation: BotConversation):
    await update_conversation(
        conversation.id,
        state='abandoned',
        completed_at=now(),
    )
    # לא שולחים הודעה נוספת. השיחה פשוט מסתיימת.
```

### 6.5 — מאיפה ה-cron רץ

ב-Render, ה-`bot_followup_cron` נקבע כ-**Cron Job service**. הגדרה ב-`render.yaml`:

```yaml
services:
  - type: cron
    name: bot-followup-cron
    schedule: "0 * * * *"  # כל שעה ב-:00
    buildCommand: pip install -r requirements.txt
    startCommand: python -m app.worker.handlers.bot_followup_cron
```

תזמון `0 * * * *` = ריצה כל שעה ב-:00. דיוק זה מספיק — שיחה שצריכה follow-up תופי 23.5-24.5h אחרי `last_message_at`.

### 6.6 — Migration ל-`followup_sent_at`

```sql
ALTER TABLE public.bot_conversations
  ADD COLUMN followup_sent_at timestamptz;

CREATE INDEX idx_bot_conversations_followup_pending
  ON public.bot_conversations (last_message_at)
  WHERE state NOT IN ('completed_calendly', 'completed_handoff', 'abandoned', 'send_failed')
    AND followup_sent_at IS NULL;

CREATE INDEX idx_bot_conversations_abandon_pending
  ON public.bot_conversations (last_message_at)
  WHERE state NOT IN ('completed_calendly', 'completed_handoff', 'abandoned', 'send_failed')
    AND followup_sent_at IS NOT NULL;
```

שני partial indexes מאיצים את ה-queries של ה-cron.

---

## חלק 7 — Soft-disable עד אישור מטא

### 7.1 — הבעיה

אחרי deploy של 5.4, אנחנו מגישים 3 templates למטא. הם יאשרו תוך 1-3 ימי עסקים. אבל בינתיים, לקוחות יכולים להיכנס לדשבורד, לבחור Premium, להגדיר bot ב-5.1, ול-4.1 קולט לידים. בלי 5.4 פעיל, ה-`initiate_bot_conversation` יישבר כי `send_template_message` ייכשל (template לא קיים).

### 7.2 — הפתרון: `app_settings` table עם flag גלובלי

```sql
-- 0025_app_settings.sql

BEGIN;

CREATE TABLE IF NOT EXISTS public.app_settings (
  key text PRIMARY KEY,
  value jsonb NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now(),
  updated_by text  -- 'admin' / 'system' / 'cron'
);

INSERT INTO public.app_settings (key, value) VALUES
  ('whatsapp_production_ready', 'false'::jsonb)
ON CONFLICT (key) DO NOTHING;

-- אין RLS — table זה רק לaccess admin/server, לא נחשף ללקוח.
-- לא נוסיף policy → ברירת המחדל היא דחייה.

COMMIT;
```

### 7.3 — בדיקה ב-`bot_service`

ב-תחילת `send_outgoing_message`, אם `message_type='template'` — בדיקה מקדימה:

```python
async def send_outgoing_message(conversation, message_type, ...):
    if message_type == 'template':
        is_ready = await app_settings.get('whatsapp_production_ready', default=False)
        if not is_ready:
            # לא שולחים. מצב ה-conversation נשאר זהה.
            sentry_sdk.capture_message(
                "Template send skipped — WhatsApp not production ready yet",
                level='warning',
                extras={'conversation_id': conversation.id},
            )
            return SendResult(skipped=True, reason='production_not_ready')
    
    # ... המשך זרימה רגילה
```

### 7.4 — מצב ב-5.1 (תצוגה ללקוח)

כשלקוח Premium נכנס לעריכת bot config, אם `whatsapp_production_ready=false` — ה-UI מציג:

> "הבוט שלך מוגדר ומחכה. ההפעלה הסופית מותנית באישור של מטא — אנחנו מצפים שזה ייגמר תוך 24-72 שעות. נעדכן אותך כשמוכן."

**Backend logic:** `GET /api/v1/bot/config` מחזיר את ה-config + שדה חדש `is_active`:

```json
{
  "business_name": "...",
  "service_label": "...",
  ...,
  "is_active": false,
  "activation_pending_reason": "templates_pending_meta_approval"
}
```

ה-UI מציג את ההודעה לפי `is_active`.

### 7.5 — הפעלה ידנית ע"י אדמין

אחרי שמטא אישרה את כל ה-3, אדמין רץ ידנית:

```sql
UPDATE public.app_settings
SET value = 'true'::jsonb, updated_at = now(), updated_by = 'admin'
WHERE key = 'whatsapp_production_ready';
```

או דרך admin endpoint (אופציה ל-Post-MVP):

```python
# POST /api/v1/admin/whatsapp/activate-production
@require_admin
async def activate_production():
    await app_settings.set('whatsapp_production_ready', True, updated_by='admin')
    return {"status": "activated"}
```

### 7.6 — מה קורה ללידים שצברו לפני ההפעלה

תרחיש: לקוח Premium הגדיר bot ב-5.1, השאיר טופס ב-Meta, 4.1 קלט את הליד, `initiate_bot_conversation` רץ אבל skip לפי flag. השיחה לא נוצרה. הליד שמור ב-`leads` table אבל הבוט לא פנה.

**הצעה למימוש:** ה-`initiate_bot_conversation` handler **כן** יוצר את שורת ה-`bot_conversations` (state=`awaiting_opening_ack`), אבל לא שולח את הודעת הפתיחה. במקום זה, שומר עמודה חדשה `pending_initial_send=true`.

cron נוסף ייעודי (`bot_pending_init_cron`) רץ כל 5 דקות, בודק אם `whatsapp_production_ready=true`. אם כן — מטפל בכל השיחות עם `pending_initial_send=true`, שולח להן opening_message. ה-flag הזה נמחק. שיחות "מצטברות" יקבלו הודעת פתיחה ברגע ההפעלה.

**warning:** אם צברו הרבה שיחות (נגיד 50+), כולן ישלחו ב-windows קצרה. מטא יכולה לראות זאת כ-spam burst. **mitigation:** rate limiting ב-cron — לא יותר מ-10 שליחות לדקה. שיחה ממתינה תיפתח בשעה הבאה.

---

## חלק 8 — מסמך הגשה ידני למטא

הסשן הזה לא כולל קוד שמגיש templates ל-Meta — זה תפעולי, אדמין עושה דרך Meta Business Manager UI. אבל המסמך כן יכלול תיעוד של מה להגיש.

### 8.1 — תוכן `docs/deployment/meta-templates-submission.md` (חדש)

```markdown
# הגשת WhatsApp Templates למטא

מסמך זה מתאר את שלושת ה-templates שצריכים אישור מטא לפני שהבוט של Phase 5 יוכל לעבוד בייצור.

## דרישות מקדימות

- WABA פעיל (`META_WABA_ID` מוגדר).
- System User Token (`META_ACCESS_TOKEN`).
- גישה ל-Meta Business Manager → WhatsApp → Message Templates.

## Template 1: bot_opening_v1

**Category:** UTILITY
**Language:** Hebrew (he)
**Variables:** 4
**Body:**

```
שלום {{1}}, אני הנציג הדיגיטלי של {{2}}. ראינו שהשארת אצלנו פרטים, אשאל אותך כמה שאלות קצרות, כדי שנמצא לך את {{3}} הכי מתאים. {{4}}
```

**Sample values:**
- {{1}}: דני
- {{2}}: אוטומציה בקלות
- {{3}}: שירות האוטומציה
- {{4}}: מעוניין?

**Justification (אם מטא מבקשים):**
"Sent to leads who submitted a form on the business's Facebook ad. The message confirms receipt of their inquiry and indicates that a screening conversation will follow. Pure utility — no marketing offers."

## Template 2: bot_followup_v1

**Category:** UTILITY
**Language:** Hebrew (he)
**Variables:** 1
**Body:**

```
היי {{1}}, השיחה שלנו נעצרה באמצע. אם זה עדיין רלוונטי, תכתוב לי בחזרה ונמשיך מאיפה שעצרנו.
```

**Sample values:**
- {{1}}: דני

**Justification:**
"Sent within the 24-hour customer service window when a lead has stopped responding to a screening conversation. Polite reminder, no marketing content."

## Template 3: bot_handoff_to_owner_v1

**Category:** UTILITY
**Language:** Hebrew (he)
**Variables:** 4
**Body:**

```
ליד חדש מהקמפיין של {{1}}. שם: {{2}}, טלפון: {{3}}. {{4}}
```

**Sample values:**
- {{1}}: שירות האוטומציה
- {{2}}: דני כהן
- {{3}}: 050-1234567
- {{4}}: שאלה 1: כן | שאלה 2: 5000 שח | שאלה 3: מהיר ככל האפשר

**Justification:**
"Sent to the business owner (our customer) when a screened lead is ready for human follow-up. Internal notification, not marketing."

## הליך הגשה

1. Meta Business Manager → WhatsApp Manager → Message Templates → Create.
2. הקלד את ה-Name (שווה ל-meta_name ב-ENV var, למשל `bot_opening_v1`).
3. בחר Category = UTILITY.
4. בחר Language = Hebrew.
5. הזן את ה-Body כפי שמופיע למעלה.
6. הוסף sample values.
7. Submit.

חזור על השלבים לכל 3 ה-templates.

## אחרי אישור

- וודא את שמות ה-templates המאושרים ב-Meta Business Manager.
- עדכן ENV vars ב-Render:
  - `META_TEMPLATE_BOT_OPENING_NAME`
  - `META_TEMPLATE_BOT_FOLLOWUP_NAME`
  - `META_TEMPLATE_BOT_HANDOFF_NAME`
- הפעל את ה-production flag:
  ```sql
  UPDATE public.app_settings
  SET value = 'true'::jsonb, updated_by = 'admin'
  WHERE key = 'whatsapp_production_ready';
  ```
- בדוק שהבוט שולח template אמיתי במקום skip.

## דחייה — מה לעשות

אם מטא דוחים template:

1. קרא את ה-rejection reason ב-Business Manager.
2. תקן את הטקסט בקובץ הרלוונטי תחת `app/prompts/phase5/templates/`.
3. Commit ל-git.
4. הגש מחדש למטא עם שם חדש (לדוגמה `bot_opening_v2`) או אותו שם (יבדוק שזה אפשרי).
5. עדכן ENV var.

עד שכל ה-3 מאושרים — ה-production flag נשאר false.
```

---

## חלק 9 — Env vars חדשים

| שם | משמעות | חובה? |
|---|---|---|
| `META_TEMPLATE_BOT_OPENING_NAME` | שם template הפתיחה ב-Meta (אחרי אישור) | חובה לפני production_ready. default `bot_opening_v1`. |
| `META_TEMPLATE_BOT_FOLLOWUP_NAME` | שם template ה-follow-up | חובה. default `bot_followup_v1`. |
| `META_TEMPLATE_BOT_HANDOFF_NAME` | שם template ה-handoff | חובה. default `bot_handoff_to_owner_v1`. |

---

## חלק 10 — שמות הקבצים החדשים והשינויים

| קובץ | תוכן |
|---|---|
| `app/prompts/phase5/templates/bot_opening.txt` | **חדש** — body של ה-template |
| `app/prompts/phase5/templates/bot_followup.txt` | **חדש** |
| `app/prompts/phase5/templates/bot_handoff_to_owner.txt` | **חדש** |
| `app/services/whatsapp_template_registry.py` | **חדש** — TEMPLATES dict + get_template() |
| `app/integrations/meta_whatsapp.py` (מ-5.0) | **תיקון** — הוספת `send_template_message` + extension ל-`classify_meta_error` |
| `app/services/bot_service.py` (מ-5.2) | **תיקון** — `send_outgoing_message` המאוחד, סינוף לטיפול ב-template vs free-form, בדיקת production_ready flag |
| `app/services/app_settings_service.py` | **חדש** — `get('key', default)`, `set('key', value, updated_by)` |
| `app/worker/handlers/bot_followup_cron.py` | **חדש** — cron מלא: follow-up + abandoned |
| `app/worker/handlers/bot_pending_init_cron.py` | **חדש** — cron של 5 דקות ל-pending opening messages |
| `app/services/bot_config_service.py` (מ-5.1) | **תיקון** — שלושת השדות החדשים + הסרת opening_message |
| `app/models/bot.py` (מ-5.1) | **תיקון** — Pydantic עם השדות החדשים |
| `app/services/initiate_bot_conversation.py` (מ-5.2) | **תיקון** — בדיקת production_ready, אם false → שמירה `pending_initial_send` |
| `supabase/migrations/0024_bot_templates_and_init.sql` | **חדש** — שדות חדשים ל-bot_configs + bot_conversations + bot_messages |
| `supabase/migrations/0025_app_settings.sql` | **חדש** — table חדש + insert ראשוני |
| `render.yaml` | **תיקון** — הוספת cron services |
| `docs/deployment/meta-templates-submission.md` | **חדש** — מסמך תפעולי |

---

## חלק 11 — Done של 5.4

- 3 קבצי templates קיימים תחת `app/prompts/phase5/templates/`.
- `whatsapp_template_registry` עובד — `get_template('bot_opening')` מחזיר TemplateDefinition עם meta_name מ-ENV.
- `meta_whatsapp.send_template_message(...)` עובד — קריאה ל-Meta Cloud API עם template payload נכון.
- `bot_service.send_outgoing_message` מאוחד — מקבל `message_type='template'/'free_form'`, מנתב לפונקציה הנכונה.
- שינויי 5.1 ב-DB ויישום — `business_name`, `service_label`, `closing_cta` קיימים, `opening_message` הוסר.
- `bot_followup_cron` רץ כל שעה — שולח follow-up ללידים לא-עונים אחרי 23.5h, מסמן abandoned אחרי 48h.
- `bot_pending_init_cron` רץ כל 5 דקות — מטפל בשיחות שצברו ב-`pending_initial_send`.
- `app_settings.whatsapp_production_ready` flag עובד — בדיקה ב-bot_service, אם false → skip שליחה, log warning.
- בדיקה ידנית: אדמין מפעיל את ה-flag → שיחות שצברו מקבלות opening_message מ-template.
- בדיקה ידנית 2: ליד נכנס דרך 4.1 → `initiate_bot_conversation` שולח template → הליד מקבל הודעה.
- בדיקה ידנית 3: ליד לא עונה 23.5h → cron שולח template follow-up → הליד מקבל.
- מסמך `docs/deployment/meta-templates-submission.md` קיים ומעודכן.
- כל הטסטים החדשים עוברים.

---

## חלק 12 — לא ב-5.4

- Auto-resubmission של templates שמטא דוחה — תפעולי, אדמין מטפל ידנית.
- Webhook מ-Meta על template status changes — לא ב-MVP. אדמין בודק ידנית ב-Business Manager.
- A/B testing בין גרסאות שונות של templates (v1 vs v2 לחצי מהלקוחות) — Post-MVP.
- UI ללקוח לעריכת template הניסוח — לא. הניסוח הוא שלנו, הלקוח רק שולט במשתנים.
- Anti-spam protection (rate limiting פר-לקוח) — מטא בעצמה אוכפת limits.
- Template analytics (כמה פעמים נשלח כל template, response rate) — Post-MVP.
- Multi-language — עברית בלבד.
- אינטגרציה עם 5.5 (Appointment scheduling). 5.5 ייצור template נוסף `bot_appointment_confirmation_v1` כשייכתב.

---

## חלק 13 — הערות לפיתוח

1. **`META_TEMPLATE_*_NAME` env vars — חובה לעדכן אחרי אישור מטא.** אם נשאיר את ה-defaults (`bot_opening_v1`) ומטא יאשרו עם שם אחר — כל השליחות יכשלו עם 132xxx. אדמין חייב לעדכן ENV ב-Render אחרי אישור.

2. **`Idempotency-Key` של Meta בשליחת templates — לאמת תמיכה.** ב-5.2 דנו בזה ל-text messages. בטכנולוגיה זה אמור להיות זהה לכל message type, אבל שווה לבדוק שזה עובד גם ל-templates בעת המימוש.

3. **`bot_pending_init_cron` עם rate limiting:** 10 שליחות לדקה כדי לא להיראות כ-spam burst בעת ההפעלה הראשונה. אם יש 100 שיחות ממתינות — ההפעלה לוקחת 10 דקות. סביר.

4. **בעיה אפשרית עם שמות שלא בעברית במשתנים:** מטא דורשת שערכים יהיו בשפת ה-template (`he`). אם הלקוח מכניס `business_name = "Plumbing Pro"` (אנגלית) ב-template עברי — מטא יכולה לדחות. **mitigation:** בולידציה ב-5.1, להזהיר על תכן שאינו עברי. לא חוסם, רק warning.

5. **`closing_cta` עם 30 תווים מקסימום** — מבטיח שלא יהפכו לכתבה. אם הלקוח רוצה "לחץ כאן עכשיו לקבלת הצעה מיוחדת!" (יותר מ-30) — חוסם.

6. **טסטים מומלצים:**
   - test_send_template_success → mock Meta API, return meta_message_id.
   - test_send_template_132xxx_permanent_error → MetaPermanentError.
   - test_send_template_skip_when_production_not_ready → SendResult(skipped=True).
   - test_followup_cron_picks_only_eligible → query מחזירה את הנכונה.
   - test_followup_cron_no_double_send → אם followup_sent_at!=NULL → לא נבחר.
   - test_followup_cron_marks_abandoned_after_48h.
   - test_pending_init_cron_sends_when_ready → flag=true → שולח.
   - test_pending_init_cron_skips_when_not_ready → flag=false → לא שולח.
   - test_bot_config_business_name_too_long → 422.
   - test_bot_config_closing_cta_too_long → 422.

7. **CC חייב להוסיף ל-`render.yaml`** את שני ה-cron services (`bot-followup-cron`, `bot-pending-init-cron`).

8. **`app_settings` כ-pattern גנרי** — שווה לזכור שזו טבלה לשימוש כללי. אם בעתיד נצטרך feature flags נוספים — אותו table. לא כל פעם table חדש.

9. **דחיית template ע"י מטא — סיוט ב-MVP.** אם תמלה יקח 5 ימים, ובכל פעם מטא דוחה ואנחנו צריכים לתקן ולהגיש שוב — הבטא נתקעת. **mitigation:** הגשה זהירה בפעם הראשונה. הניסוחים שעיצבנו (חלק 2) הם UTILITY נטו, ללא טון שיווקי. הסבירות שייפסלו נמוכה. אבל יש חשיפה.

10. **abandoned לא שולח הודעה ללליד.** הליד נשאר בלי תגובה אחרונה מאיתנו. **שווה לדון:** האם להוסיף הודעה קצרה לפני סימון abandoned? למשל "בסדר, נשמח לעמוד לרשותך בעתיד". יצירת template נוסף = יותר אישורים. **הצעה:** לא ב-MVP. אם בעתיד נראה ש-abandoned רבים — נוסיף.

11. **חלון 24h מטא — מה אם follow-up נשלח ב-23.55h ו-יחזור 23.9h?** ה-follow-up עצמו פותח את החלון מחדש (משעון מטא). הליד שעונה ל-follow-up — בתוך חלון 24h חדש. הבוט יכול להגיב בטקסט חופשי. עובד.

---

**סוף מסמך 5.4.**

---

# Session 5.5 — Appointment Scheduling

> תיאום פגישות אוטומטי דרך הבוט: ליד בוחר slot פנוי מיומן Google של בעל העסק → תור pending → בעל העסק מאשר/דוחה בפאנל → אישור לליד + סנכרון יומן. גם ביטול על ידי הליד דרך WhatsApp.

**סטטוס מימוש (CC):** 5.5 פוצלה ל-5 sub-sessions (היקף ~2400 שורות; ה-ROADMAP כתוב כ-pseudo-code
מול codebase דמיוני). הסדר: 5.5a בסיס → 5.5b Google Calendar OAuth → 5.5c מנוע זמינות → 5.5d booking
flow → 5.5e ביטול+סנכרון+פאנל.
- ✅ **5.5a — Foundation** (migrations `0040`-`0045` + config `google_*` + soft-disable flag +
  un-block מגודר + sync של `ConversationState`). סטיות מה-ROADMAP (תיקוני התאמה לריפו): **Vault**
  ל-tokens (לא Fernet — ROADMAP §2.4 עצמו מורה reuse) → אין `crypto.py`/`SECRETS_ENCRYPTION_KEY`,
  ה-RPCs נדחו ל-5.5b; **אין `set_updated_at` trigger** בריפו → הוסר, `updated_at` ב-RPC/app; מספור
  **0040-0045** (לא 0027-0031); `status`/`state` כ-**text+CHECK** (לא ENUM); ה-states האמיתיים של
  `bot_conversations` (0034) → שוחזר גם `idx_bot_conversations_active` עם 2 ה-terminal החדשים; ה-flag
  כ-**שורת app_settings** (לא ALTER COLUMN). ה-un-block חלקי — ולידציית calendar-connected (412)
  נדחתה ל-5.5b (אין עדיין `is_connected`).
- ✅ **5.5b — Google Calendar OAuth + connection** (integration `google_calendar.py` + RPCs `0046`
  ל-Vault + `google_calendar_service` + router `/auth/google/*` + `GET/DELETE /me/google-connection`
  + השלמת ה-un-block: ולידציית `is_connected` → 412 + runbook `docs/deployment/google-calendar-setup.md`).
  סטיות מה-ROADMAP: **scope יחיד** `calendar` (email+timezone מה-primary calendar — אין
  `userinfo.email`/`openid`, פחות חיכוך ב-consent); redirect **`/auth/google/callback`** (לא `/api/v1`,
  עקבי עם Meta); **name=NULL** ב-Vault secrets (disconnect מוחק שורה → name קבוע היה מתנגש ב-reconnect,
  ואין `vault.delete_secret` בריפו); **alert על ניתוק נדחה** ל-slice ה-notifications (`auth_invalid_at`
  מסומן + `mark_google_auth_invalid` מחזיר was_first — תשתית alert-once מוכנה). הדגל נשאר כבוי
  (ה-runtime ב-5.5c-d), אבל "חבר/נתק יומן" עובד מקצה-לקצה ושמירת `bot_schedule_appointment` נחסמת ב-412.
- ✅ **5.5c — מנוע זמינות** (FreeBusy ב-`integrations/google_calendar.get_freebusy` + `availability_service`
  [engine + CRUD שעות/ימים] + `models/scheduling` + router `/bot/working-hours` · `/bot/special-days` ·
  `/bot/availability[/dates]`). חישוב slots = חיתוך **3 מקורות**: שעות פעילות ∩ NOT(FreeBusy) ∩ NOT(תורים
  pending/confirmed). **אין migration** — הטבלאות קיימות מ-5.5a (0041-0043); כתיבה דרך admin client
  (service_role עוקף RLS). סטיות מה-ROADMAP: **CalendarUnavailable → exception (409), לא `[]`** (לא מבליעים
  שגיאה כ-"אין slots" — External SDK 10); **`get_available_dates` עם FreeBusy אחת ל-range** (לא 60 קריאות —
  פונקציה טהורה `_compute_slots_for_day` משותפת); tz מה-credentials (fallback Asia/Jerusalem); Premium gating
  ב-router (ה-service agnostic — 5.5d יקרא לו ישירות); decision engine נדחה ל-5.5d.
- ✅ **5.5d — Booking flow ב-WhatsApp** (`core/booking_decision` טהור + `appointment_service` [create reserve-first
  + gather_and_decide] + interactive WhatsApp [`send_interactive_list/buttons`] + scheduling handlers ב-`bot_service`
  + template `bot_pending_appointment`). הזרימה: סיום סינון עם `bot_schedule_appointment` → date picker → time picker
  → confirm → `create_appointment(pending)` → התראה לבעל העסק → `completed_appointment_scheduled`. **אין migration**
  (states/טבלאות קיימים; הזרימה inline ב-`process_inbound_message`). הכרעות: **גבול 5.5d↔5.5e** — ב-5.5d התור `pending`
  בלבד (אין Google event עדיין), אז ביטול/אישור/calendar-sync נדחים ל-5.5e; **interactive reply → text** (ה-webhook
  ממפה `list_reply.id`/`button_reply.id` ל-body); **manual mode** תמיד → pending, ו-`gather_and_decide` מדלג על
  `has_active`/FreeBusy (ה-double-booking + idempotency דרך `create_appointment` reserve-first+ownership — אחרת retry
  היה רואה את התור של עצמו ו-rejected כוזב). הדגל נשאר כבוי (`bot_config` חוסם עד 5.5e).
- ✅ **5.5e-1 — Owner Panel + Calendar Sync** (`gcal.create_event/delete_event` + `gcal_service` orchestration +
  `appointment_service` [`update_appointment_status` CAS / list / get] + `bot_service` [approve/reject/cancel +
  סנכרון יומן + הודעה לליד] + `routers/appointments` [list/approve/reject/cancel/ICS] + `core/ics` + templates
  `bot_appointment_confirmation/rejected`). בעל העסק מאשר→confirmed+אירוע ביומן+אישור+ICS לליד / דוחה / מבטל.
  **אין migration** (עמודות קיימות ב-0043; CAS table-op, sends inline). הכרעות: **calendar sync best-effort**
  (כשל→log, ה-DB source-of-truth; delete 404/410=הצלחה); **ICS public** (UUID-protected + status check); owner-cancel
  → reuse `bot_appointment_rejected`; orchestration ב-`bot_service` (ה-send primitives שם). הדגל נשאר כבוי עד 5.5e-2.
- ✅ **5.5e-2 — Lead Cancellation (ביטול ע"י הליד)** (`bot_intelligence_service.classify_intent` [LLM, fail-safe
  ל-'other'] + prompt `classify_intent.txt` + `appointment_service.get_active_appointment_for_conversation` +
  `bot_service` [terminal-entry → כפתורי אישור → `_handle_cancellation_confirm` → `_execute_lead_cancellation`
  CAS+מחיקת אירוע+`_notify_owner_cancellation`] + template `bot_appointment_cancelled_owner`). הליד שולח "לבטל"
  אחרי תור מאושר → זיהוי כוונה → אישור (כן/לא, state `awaiting_cancellation_confirm`) → ביטול אטומי
  (`cancelled`+`cancelled_by=lead`) + מחיקת אירוע מהיומן + התראת בעל העסק + אישור לליד. **אין migration**
  (state+template קיימים ב-0044). הכרעות: **זיהוי כוונה ב-LLM** (כ-`classify_bot_answer`; כשל→'other', לא מבטלים
  על ספק); **ביטול אטומי = CAS** (reuse `update_appointment_status`, ה-WHERE הוא ה-idempotency); **side-effects
  best-effort** (DB source-of-truth); **'other'→שתיקה** (לא nudge). **ה-lifecycle של 5.5 הושלם בקוד.** הפעלת
  הדגל `appointment_scheduling_enabled` נשארת שלב runbook ידני (אדמין, **אחרי אישור Meta ל-4 templates התורים**).

---

## מבוא והקשר

זהו ה-session האחרון של Phase 5. הוא ממשש את הערך השני (אחרי הסינון עצמו) של הבוט: תיאום תורים אוטומטי. עד כה (sessions 5.0–5.4), הבוט יודע ליצור קשר עם ליד, לסנן אותו ב-5 שאלות, לסווג תשובות עם gpt-5.2, ולסיים עם אחת משלוש פעולות: שליחת לינק Calendly, handoff לבעל העסק, או תיאום פגישה. השלישית (`bot_schedule_appointment`) קיימת ב-CHECK של הטבלה אבל ה-service דוחה אותה ב-422 עם `FeatureNotAvailableError`. ב-5.5 אנחנו פותחים אותה.

הליבה הטכנית מבוססת על חילוץ מ-`ai-business-bot` (הצ'אטבוט הקודם של אמיר): OAuth ל-Google Calendar עם הצפנת tokens, חיתוך שלושה מקורות לחישוב slots פנויים, מנוע החלטה כפונקציה טהורה, הגנת UNIQUE חלקי מ-double-booking, ו-polling reminders. רוב הקוד עובר adaptation ל-multi-tenant (החלפת `CHECK(id=1)` ב-`user_id` FK) אבל הדפוסים זהים.

### מה ה-session **כן** מכסה

- חיבור Google Calendar בפאנל (OAuth flow, multi-tenant, אחסון מוצפן, refresh, ניטור בריאות)
- ניהול שעות פעילות + ימים מיוחדים (vacation, override) בפאנל
- חישוב slots פנויים על ידי חיתוך שלושה מקורות: שעות פעילות ∩ NOT(יומן Google busy) ∩ NOT(תורים ב-DB)
- Flow של בחירת תור ב-WhatsApp: תאריך → שעה → אישור (List Picker + Quick Reply)
- מנוע החלטה במצב `manual` בלבד: כל תור נשמר כ-`pending`, בעל העסק מאשר/דוחה בפאנל
- 4 templates חדשים למטא (2 לבעל העסק, 2 לליד)
- Flow ביטול תור על ידי הליד דרך WhatsApp (NLU intent → Quick Reply → ביטול אטומי)
- סנכרון יומן Google אחרי אישור (יצירת אירוע + הוספת bookingId ב-description)
- ICS link לליד אחרי אישור
- UI בפאנל: רשימת תורים + כפתורי אישור/דחייה

### מה ה-session **לא** מכסה (נדחה)

- `auto_with_check` ו-`auto_always` modes (manual בלבד ב-MVP)
- שינוי תור על ידי הליד (reschedule)
- תורים חוזרים (recurring appointments)
- יותר מיומן אחד לכל user (multi-calendar)
- כמה sub-services / סוגי שירות נפרדים (קיים שדה `default_appointment_duration_minutes` בלבד, אז משך אחיד לכל התורים של אותו user)
- חגים ישראליים אוטומטיים (override ידני בלבד דרך `special_days`)
- תזכורות לליד (24 שעות / שעתיים לפני) — חוסך session ל-polling job; ניתן להוסיף בעדכון
- פרסור זמן בשפה טבעית בעברית (List Picker בלבד; טקסט חופשי → deviation handler מ-5.3)
- בעל העסק יכול לעדכן תור דרך WhatsApp (פאנל בלבד)

---

## החלטות ננעלו

| # | החלטה | רציונל |
|---|---|---|
| 1 | `auto_booking_mode = manual` בלבד | בטיחות ב-MVP. אין סיכון double-booking או אישור עיוור. נפתח בעדכון אחרי שיהיו דאטה. |
| 2 | יומן אחד לכל user | תואם ל-`bot_configs` 1:1 עם user. multi-calendar = ROI שלילי ב-MVP. |
| 3 | שימוש חוזר ב-`fallback_value` לאחסון טלפון E.164 של בעל העסק | אותו ה-pattern כמו `human_handoff`. עוקף הצורך בשדה נפרד. ולידציה זהה. |
| 4 | ביטול ע"י הליד ב-WhatsApp עם Quick Reply | הקוד הקיים מראה שזה לא משתנה ארכיטקטונית; ה-NLU כבר במקום (5.3). UX משמעותית טוב יותר. |
| 5 | List Picker בלבד לזמנים | טקסט חופשי = deviation flow מ-5.3 (החזרה ל-Picker). חוסך פיתוח LLM לפרסור עברית. |
| 6 | חיבור יומן חייב לפני שמירת bot_config עם `bot_schedule_appointment` | משדר 412 Precondition Failed עם הוראה ברורה ל-UI. עקבי עם 5.1 (לא מאפשרים action חסר תלות). |
| 7 | בעל העסק שולח הודעה ראשונית לבוט ב-onboarding (פתיחת חלון 24h) | חיוני להתראות חופשיות. template אחד גנרי כ-fallback לחלון סגור. |
| 8 | אישור/דחייה ע"י בעל העסק — בפאנל בלבד | פשוט יותר מ-Quick Reply ב-template. מאפשר context מלא (שאר תורים, היסטוריית הליד). |
| 9 | סנכרון יומן הוא shadow של DB | DB = source of truth. אם הסנכרון נכשל — התור עדיין קיים. busy ranges מה-DB מגנים מ-double-booking. |

---

## סקירת הזרימה המלאה

```
  בעל העסק (onboarding, פעם אחת)               ליד (אחרי 5 שאלות סינון של 5.2)
  ┌─────────────────────────────────┐         ┌────────────────────────────────┐
  │ 1. חיבור Google Calendar (OAuth)│         │ סיום סינון →                    │
  │ 2. הגדרת שעות פעילות           │         │ fallback_action=                │
  │ 3. שליחת הודעה ראשונית לבוט    │         │   bot_schedule_appointment      │
  │    (פותח חלון 24h להתראות)      │         └────────────┬───────────────────┘
  │ 4. שמירת bot_config             │                      ▼
  └─────────────────────────────────┘    בדיקת is_calendar_connected(user_id)
              │                                            ▼
              │ (פעיל)                       ① List Picker תאריכים פנויים
              ▼                                            ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ get_available_slots(user_id, date, duration):                     │
   │   working_hours ∩ NOT(GCal busy) ∩ NOT(DB pending/confirmed)      │  ← הליבה
   │   ─ special_days dominates ─                                       │
   │   = ["09:00","09:30","11:30",...]                                  │
   └──────────────────────────────────────┬───────────────────────────┘
                                          ▼
                          ② List Picker שעות → ③ Quick Reply אישור
                                          ▼
                   ┌──────────────────────────────────────┐
                   │ gather_and_decide(user_id, ...)       │
                   │ manual mode → decision = "pending"    │  ← מנוע החלטה
                   └──────────────────┬───────────────────┘
                                      ▼
        ┌────────────────────────────────────────────────────────────┐
        │ INSERT appointments (status='pending') ← UNIQUE partial    │
        │ index מגן מ-double-booking                                 │
        └─────────────────────┬───────────────────────────────────┘
                              ▼
        ┌──────────────────────────────────────────────────────┐
        │ ניסיון שליחת הודעה חופשית לבעל העסק (חלון פתוח?)        │
        │   ├─ הצלחה → בעל העסק קיבל פירוט + לינק לפאנל          │
        │   └─ כשלון (Error 131047) →                            │
        │       fallback: template bot_pending_appointment_v1     │
        └──────────────────────┬─────────────────────────────────┘
                               ▼
                  הליד: "בקשת התור הועברה, תקבל תשובה תוך X"
                               ▼
                       (התור pending. שיחה terminal.)
                               ▼
        ┌──────────────────────────────────────────────────────┐
        │ בעל העסק נכנס לפאנל → רואה pending → אשר/דחה          │
        └──────────┬─────────────────────────────────┬──────────┘
                   ▼ אישור                            ▼ דחייה
   ┌────────────────────────────┐    ┌─────────────────────────────┐
   │ status=confirmed            │    │ status=rejected             │
   │ create_event ב-GCal         │    │ (אין סנכרון יומן)            │
   │ template confirmation לליד   │    │ template rejected לליד       │
   │ + ICS link                   │    │ הליד יכול להתחיל שוב          │
   └────────────────────────────┘    └─────────────────────────────┘

  (תרחיש מקביל: הליד כותב "אני רוצה לבטל" — NLU מזהה intent → Quick Reply
   "אישור ביטול" → DELETE event ב-GCal + status=cancelled + הודעה לבעל העסק.)
```

---

## Patches נדרשים לפני התחלת 5.5

לפני שמתחילים מימוש 5.5 בפועל, יש 4 patches קטנים ל-sessions קודמים. כולם נדרשים כדי ש-5.5 יתחבר ל-pipeline הקיים בלי לשבור regressions. אם CC לא יישם את 5.0–5.4 עדיין — ה-patches מתועדים גם בקבצי ה-roadmap המקוריים. אם הוא **כן** יישם — צריך לעדכן בנפרד.

### Patch 5.1.B — ולידציה ל-bot_schedule_appointment

ב-`session-5.1-roadmap.md` ה-validator של `fallback_action='bot_schedule_appointment'` מחזיר `422 FeatureNotAvailableError("תיאום תורים יהיה זמין בעדכון הבא")`. ב-5.5 צריך:

1. **להסיר** את ה-422 הזה.
2. **להוסיף** את הולידציות הבאות כש-`fallback_action='bot_schedule_appointment'`:
   - `fallback_value` חובה — אם NULL/empty → `422 "נדרש מספר טלפון של בעל העסק"`
   - `fallback_value` בפורמט E.164: regex `^\+9725\d{8}$` (ישראל בלבד ל-MVP)
   - יומן חייב להיות מחובר עבור user — אם לא → `412 Precondition Failed` עם body:
     ```json
     {
       "error": "calendar_not_connected",
       "message": "יש לחבר את יומן Google לפני בחירת תיאום פגישות",
       "action_required": "connect_calendar",
       "connect_url": "/settings/bot/calendar"
     }
     ```
3. **בדיקת חיבור היומן** — קריאה ל-`google_calendar_service.is_connected(user_id)` שמחזיר True רק אם:
   - יש שורה ב-`google_calendar_credentials` ל-user_id הזה
   - `refresh_token` לא ריק
   - `auth_invalid_at IS NULL` (token עדיין תקף)

הקוד צריך לבדוק את שלושת התנאים יחד. בדיקה מבוססת רק על קיום שורה לא מספיקה.

### Patch 5.2.B — טיפול ב-bot_schedule_appointment בסיום סינון

ב-`session-5.2-roadmap.md` יש לוגיקה שמטפלת ב-completion של conversation אחרי שכל השאלות נענו. הלוגיקה כיום בודקת את `fallback_action` ומבצעת אחת משתי פעולות (`calendly_link` / `human_handoff`). צריך:

1. **להוסיף ענף** ל-`bot_schedule_appointment`:
   - לא לסיים את ה-conversation עם `state='completed'`
   - להעביר ל-state חדש `scheduling_date`
   - להכניס job חדש לתור — `send_date_picker_job` (ראה חלק 7 למטה)
   - לא לשלוח הודעת handoff לבעל העסק (זה לא הזמן — זה אחרי שהליד יבחר slot)

2. **defense in depth** — לפני המעבר ל-state החדש, לבדוק שוב את `is_calendar_connected(user_id)`. אם איכשהו ה-calendar התנתק בין ה-config save ל-runtime (refresh token נשלל בינתיים) — ליפול ל-handoff message כמו `human_handoff`: שולחים לליד "תיצור קשר עם בעל העסק בטלפון X" ולבעל העסק התראה שהיומן מנותק.

### Patch 5.3.B — NLU intent detection לביטול תור

ב-`session-5.3-roadmap.md` יש classifier שמסווג תשובות לשאלות סינון. צריך **להרחיב** את ה-NLU כך שיזהה כוונת ביטול בכל שלב של ה-conversation. שני אופציות:

**אופציה א' — classifier חדש נפרד**: פונקציה `classify_intent(text)` שמחזירה אחת מ-`{"answer", "cancel_appointment", "deviation"}`. נקראת לפני ה-classifier הקיים.

**אופציה ב' — הרחבת ה-classifier הקיים**: ה-classifier הקיים מקבל גם את ה-state של ה-conversation, וה-prompt משתנה לפי הקשר.

ההמלצה: **אופציה א'**. הפרדה ברורה בין "סיווג תשובה לשאלה" (סיווג בינארי בעיקרון) ל"זיהוי intent" (סיווג רב-מחלקתי). כל פונקציה ב-prompt משלה, ב-cost נפרד, ב-test נפרד.

הסיווג `cancel_appointment` נכון לכל אחד מהמצבים הבאים:
- ה-conversation ב-state `completed_appointment_scheduled` (יש pending/confirmed appointment)
- הליד כותב משהו אחרי שהשיחה הסתיימה, וב-DB יש לו appointment במצב `pending` / `confirmed` ובתאריך עתידי

הקלט ל-prompt צריך לכלול אינדיקציה אם לליד יש כרגע תור פעיל. אם אין — אין סיבה לזהות `cancel_appointment` (מי מבטל תור שלא קיים?).

### Patch 5.4.B — 4 templates חדשים

ב-`session-5.4-roadmap.md` מוגשים 3 templates למטא. צריך להוסיף עוד 4 בקובץ `docs/deployment/meta-templates-submission.md`:

1. `bot_pending_appointment_v1` (UTILITY) — לבעל העסק
2. `bot_appointment_cancelled_owner_v1` (UTILITY) — לבעל העסק
3. `bot_appointment_confirmation_v1` (UTILITY) — לליד
4. `bot_appointment_rejected_v1` (UTILITY) — לליד

ניסוחים מפורטים בחלק 10 למטה.

ה-`app_settings.whatsapp_production_ready` (soft-disable flag מ-5.4) ימשיך לחסום שליחה עד שכל ה-7 templates מאושרים. גיא צריך לאמת אישור של ה-7 לפני שמסירים את הדגל.

---

## חלק 1 — Migration

מספור: מניח שה-migration האחרון לפני 5.5 הוא 0026 (אחרי 5.0–5.4). אם CC כבר השתמש במספרים אחרים — לבדוק `ls supabase/migrations/` ולעדכן.

### 1.1 — `0027_google_calendar_credentials.sql`

```sql
-- 0027_google_calendar_credentials.sql
-- Phase 5.5 — אחסון מוצפן של credentials של יומן Google לכל user

BEGIN;

CREATE TABLE public.google_calendar_credentials (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id               uuid NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,

  google_account_email  text NOT NULL,                  -- email של חשבון Google שחובר
  calendar_id           text NOT NULL DEFAULT 'primary',-- בעתיד אולי calendar_id ספציפי
  timezone              text NOT NULL DEFAULT 'Asia/Jerusalem',

  -- Tokens — מוצפנים עם Fernet (prefix v1:<base64>)
  refresh_token         text NOT NULL,                  -- ארוך-טווח. דליפה = גישה ליומן.
  access_token          text,                           -- קצר (1h). חידוש דרך refresh.
  token_expiry          timestamptz,

  -- ניטור בריאות
  auth_invalid_at       timestamptz,                    -- refresh נכשל → מסומן. NULL = תקין.
  owner_alert_sent_at   timestamptz,                    -- אנטי-spam: התראה פעם אחת לכל אירוע ניתוק.

  connected_at          timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now()
);

CREATE TRIGGER google_calendar_credentials_updated_at
  BEFORE UPDATE ON public.google_calendar_credentials
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

CREATE INDEX idx_gcal_creds_user ON public.google_calendar_credentials (user_id);
CREATE INDEX idx_gcal_creds_auth_invalid ON public.google_calendar_credentials (auth_invalid_at)
  WHERE auth_invalid_at IS NOT NULL;  -- partial index ל-health monitoring

-- RLS — אין SELECT למשתמש. גישה רק דרך admin client.
-- אנחנו לא רוצים שטוקן refresh יזלוג גם דרך SELECT לקליינט.
ALTER TABLE public.google_calendar_credentials ENABLE ROW LEVEL SECURITY;
-- ללא policies → אין גישה מ-auth.uid().
-- כל פעולה דרך service_role או admin client בלבד.

COMMIT;
```

**הערות עיצוב:**

- **`user_id UNIQUE`** — יומן אחד לכל user. החלטה ננעלה. אם בעתיד נרצה multi-calendar, נצטרך drop של UNIQUE ולעדכן ה-FK הזה ב-bot_configs.
- **`refresh_token NOT NULL`** — בלי refresh token אין שום ערך לשורה (אי אפשר לרענן access). אם OAuth מחזיר ריק, נסרב לשמור (זה קורה אם המשתמש נתן הרשאה בעבר ועכשיו מתחבר שוב בלי `prompt=consent` — נמנע באמצעות התנאי הזה).
- **partial index ל-`auth_invalid_at`** — בריאות צריכה להישלף מהר ב-CRON שבודק חיבורים מנותקים. partial index כי רוב השורות `NULL`.
- **RLS ללא policies = אין גישה מ-client** — שונה משאר הטבלאות. הסיבה: tokens הם משאב רגיש מספיק שאסור שהם יוחזרו ל-client אפילו ב-SELECT. כל הקריאות דרך הסרבר.

### 1.2 — `0028_working_hours.sql`

```sql
-- 0028_working_hours.sql
-- שעות פעילות שבועיות לכל user

BEGIN;

CREATE TABLE public.working_hours (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  -- 0=ראשון, 1=שני, ..., 6=שבת. החלטה: ספירה ישראלית (לא Python).
  day_of_week int NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),

  is_open     boolean NOT NULL DEFAULT false,
  open_time   time,   -- NULL אם is_open=false
  close_time  time,

  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now(),

  UNIQUE (user_id, day_of_week),
  CHECK (
    (is_open = false AND open_time IS NULL AND close_time IS NULL)
    OR
    (is_open = true AND open_time IS NOT NULL AND close_time IS NOT NULL)
  )
);

CREATE TRIGGER working_hours_updated_at
  BEFORE UPDATE ON public.working_hours
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

CREATE INDEX idx_working_hours_user ON public.working_hours (user_id);

ALTER TABLE public.working_hours ENABLE ROW LEVEL SECURITY;

CREATE POLICY working_hours_select_own
  ON public.working_hours FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.working_hours TO authenticated;
-- כתיבה דרך admin client (אותו דפוס כמו 5.1)

COMMIT;
```

**הערות:**

- **לא משמרים `is_overnight`**. למעט במקרים נדירים מאוד, עסקים ישראליים לא חוצים חצות. אם close_time < open_time — נחזיר שגיאת ולידציה ב-API.
- **`day_of_week`** הוא ספירה ישראלית. הקוד שמחשב slots צריך להמיר מספירת Python: `israeli_dow = (python_dow + 1) % 7` (Python: Monday=0; Israeli: Sunday=0).
- **CHECK constraint** מבטיח עקביות בין `is_open` לבין `open_time/close_time`. בלי זה, אפשר להגיע למצב `is_open=true` עם NULL זמנים → קריסה ב-runtime.
- **ברירת מחדל לכל user** — כשמשתמש מתחבר לראשונה (או נכנס למסך הגדרת שעות בפעם הראשונה), API יוצר seed של 7 שורות עם `is_open=false`. ה-UI מציג ומאפשר עריכה.

### 1.3 — `0029_special_days.sql`

```sql
-- 0029_special_days.sql
-- ימים מיוחדים: חופשה (סגור) או שעות מותאמות לתאריך ספציפי

BEGIN;

CREATE TABLE public.special_days (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  special_date date NOT NULL,

  -- is_closed=true → יום סגור (חופשה, חג). open_time/close_time NULL.
  -- is_closed=false → שעות שונות מהרגיל. open_time/close_time חובה.
  is_closed   boolean NOT NULL,
  open_time   time,
  close_time  time,

  reason      text,                              -- תיוג חופשי לבעל העסק ("חופשה", "ערב חג")

  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now(),

  UNIQUE (user_id, special_date),
  CHECK (
    (is_closed = true AND open_time IS NULL AND close_time IS NULL)
    OR
    (is_closed = false AND open_time IS NOT NULL AND close_time IS NOT NULL)
  )
);

CREATE INDEX idx_special_days_user_date ON public.special_days (user_id, special_date);

ALTER TABLE public.special_days ENABLE ROW LEVEL SECURITY;

CREATE POLICY special_days_select_own
  ON public.special_days FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.special_days TO authenticated;

COMMIT;
```

**הערות:**

- **`UNIQUE (user_id, special_date)`** — חופף ל-`special_date`, לכן אם בעל העסק מוסיף יום שכבר קיים — צריך UPSERT, לא INSERT. ה-API צריך לטפל בזה.
- **חגים אוטומטיים — לא ב-MVP**. הספרייה `holidays` של Python (שמשמשת בקוד הישן) ספציפית לישראל ועובדת מצוין. אבל זה מוסיף תלות, צריך לדאוג למפת רוב/מיעוט (מאות חגים), ויש סיכון של false positives. בעדכון אפשר להוסיף job שמייצר אוטומטית רשומות `special_days` עם `is_closed=true` לכל החגים — והמשתמש יכול להסיר ידנית.
- **`reason`** הוא טקסט חופשי, ל-UX של בעל העסק (יראה ברשימה). לא משפיע על לוגיקה.

### 1.4 — `0030_appointments.sql`

זו הטבלה המרכזית של 5.5. הסכמה מעט שונה מהקוד הישן — adaptation ל-Postgres + UUIDs + multi-tenant.

```sql
-- 0030_appointments.sql
-- תורים שהבוט תיאם עם לידים

BEGIN;

-- enum לסטטוס תור (מומלץ enum על CHECK ב-Postgres ל-validation מהיר)
CREATE TYPE appointment_status AS ENUM (
  'pending',     -- מחכה לאישור בעל העסק
  'confirmed',   -- בעל העסק אישר. אירוע ביומן Google.
  'cancelled',   -- בוטל (ע"י ליד או בעל העסק). slot שוחרר.
  'rejected',    -- בעל העסק דחה את הבקשה (לא בוצע, לא יוצר).
  'passed'       -- התאריך עבר. הגרר אוטומטי דרך CRON או lazy.
);

CREATE TABLE public.appointments (
  id                          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id                     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  -- אינדיקציה ליצירת קישור חזרה לשיחה
  conversation_id             uuid NOT NULL REFERENCES public.bot_conversations(id) ON DELETE CASCADE,
  lead_phone                  text NOT NULL,     -- E.164 — נשמר כפילות מ-conversation לעקיבות
  lead_name                   text,              -- שם של הליד מהטופס (לתצוגה)

  -- פרטי התור
  preferred_date              date NOT NULL,
  preferred_time              time NOT NULL,
  confirmed_duration_minutes  int NOT NULL,      -- נלקח מ-bot_configs.default_appointment_duration_minutes ביצירה

  status                      appointment_status NOT NULL DEFAULT 'pending',

  -- סנכרון יומן
  google_event_id             text,              -- מזהה האירוע ב-GCal (אחרי אישור)

  -- audit
  rejection_reason            text,              -- אם status=rejected, בעל העסק מציין סיבה (אופציונלי)
  cancellation_reason         text,              -- אם status=cancelled, מי ביטל
  cancelled_by                text CHECK (cancelled_by IN ('lead', 'owner', NULL)),

  created_at                  timestamptz NOT NULL DEFAULT now(),
  confirmed_at                timestamptz,
  cancelled_at                timestamptz,

  CHECK (preferred_date >= '2026-01-01'),    -- sanity, לא תאריך עברי
  CHECK (preferred_time >= '00:00' AND preferred_time <= '23:59')
);

-- ⭐ שכבת ההגנה הקריטית מ-double-booking: UNIQUE חלקי על שורות פעילות
-- "פעיל" = pending או confirmed. cancelled/rejected/passed → השורה לא חוסמת slot.
-- זה מאפשר לליד לקבוע מחדש slot שביטל בעבר.
CREATE UNIQUE INDEX idx_appointments_active_slot
  ON public.appointments (user_id, preferred_date, preferred_time)
  WHERE status IN ('pending', 'confirmed');

CREATE INDEX idx_appointments_user_status ON public.appointments (user_id, status);
CREATE INDEX idx_appointments_lead ON public.appointments (lead_phone, status);
CREATE INDEX idx_appointments_date ON public.appointments (preferred_date) WHERE status IN ('pending', 'confirmed');

ALTER TABLE public.appointments ENABLE ROW LEVEL SECURITY;

CREATE POLICY appointments_select_own
  ON public.appointments FOR SELECT
  USING (auth.uid() = user_id);

GRANT SELECT ON public.appointments TO authenticated;

COMMIT;
```

**הערות עיצוב:**

- **`UNIQUE INDEX חלקי` (`WHERE status IN ('pending','confirmed')`)** — זה ה-pattern הקריטי שמונע double-booking ברמת ה-DB. גם אם שני webhook concurrent מנסים ליצור תור באותו slot — רק אחד יצליח (השני יקבל `UniqueViolation`). הוא דורש שה-INSERT יהיה אטומי. הקוד צריך להיות מוכן לטפל ב-`UniqueViolation` ולהחזיר הודעה ידידותית לליד: "השעה שבחרת נתפסה לפני שניות, אנא בחר אחרת" (ולחזור ל-time picker).
- **`status IN ('cancelled','rejected','passed')` → שורה לא חוסמת**. זה מאפשר UPSERT חוזר על אותו slot. אם הליד ביטל ב-10:00 ועכשיו רוצה שוב באותה שעה — יעבוד.
- **`confirmed_duration_minutes`** — נשמר ב-INSERT מתוך `bot_configs.default_appointment_duration_minutes`. ב-MVP זה ערך אחד לכל ה-user. בעתיד אם נוסיף services נשמור duration לפי service.
- **`conversation_id`** — קישור חזרה ל-`bot_conversations` מ-5.2. מאפשר לטרגט את התשובות בחזרה ל-conversation הנכון, ולעדכן את ה-state שלה כשהתור משתנה.
- **`cancelled_by` enum-style** — חשוב לאודיט: ביטל הליד או בעל העסק? משפיע על הניסוח של ההודעה האוטומטית.
- **`passed` נשאר כ-status היסטורי** — תורים שעברו ועדיין `pending` הם בעיה (בעל העסק שכח לאשר ועבר זמן). job יומי שמסמן אותם כ-`passed`. לחילופין lazy: בכל קריאה ל-`get_appointments_busy_ranges`, אם תור pending והתאריך עבר — לעדכן ל-`passed` בו במקום.

### 1.5 — `0031_bot_conversations_states.sql`

הרחבת ה-CHECK של `bot_conversations.state` בהוספת 6 states חדשים.

```sql
-- 0031_bot_conversations_states.sql
-- הוספת states ל-flow תיאום תורים

BEGIN;

-- ה-CHECK הקיים (מ-5.2) הוא משהו כמו:
-- CHECK (state IN ('initial','awaiting_answer','completed','abandoned',...))
-- צריך drop + recreate. בלי data loss כי לא משנים שורות קיימות.

ALTER TABLE public.bot_conversations
  DROP CONSTRAINT IF EXISTS bot_conversations_state_check;

ALTER TABLE public.bot_conversations
  ADD CONSTRAINT bot_conversations_state_check
  CHECK (state IN (
    -- מ-5.2 (קיימים)
    'initial',
    'awaiting_answer',
    'completed',
    'completed_calendly',
    'completed_handoff',
    'abandoned',
    'error',

    -- חדשים ל-5.5
    'scheduling_date',                      -- ממתין לבחירת תאריך
    'scheduling_time',                      -- ממתין לבחירת שעה
    'scheduling_confirm',                   -- ממתין לאישור final של הליד
    'completed_appointment_scheduled',      -- terminal — appointment נוצר (pending/confirmed/rejected — בודקים ב-appointments table)
    'completed_appointment_cancelled',      -- terminal — תור בוטל
    'awaiting_cancellation_confirm'         -- ממתין לאישור הליד על ביטול
  ));

COMMIT;
```

**הערות:**

- ה-states הקיימים נשארים. ה-list למעלה משוער; אם 5.2 השתמש בשמות מעט שונים — לעדכן.
- `completed_appointment_scheduled` הוא terminal, גם אם בעל העסק טרם אישר. הסיבה: השיחה האקטיבית עם הליד הסתיימה ברגע שהוא בחר slot ולחץ אישור. הליד מקבל הודעה ("בקשת התור הועברה, תקבל תשובה תוך X שעות") והמערכת בעצם ממתינה לפעולת בעל העסק. מצב התור עצמו נמצא ב-`appointments.status`, לא ב-conversation state.
- `awaiting_cancellation_confirm` ב-state נפרד כי הליד יכול ל-cancel תוך כדי שיחה אחרת — צריך להבחין בין "השיחה הקיימת ב-scheduling_X" לבין "הליד פנה במצב terminal וביקש לבטל".

---

## חלק 2 — Google Calendar OAuth (אינטגרציה רב-משתמשים)

זו ההסבה הכי משמעותית מהקוד הישן. שם הוא single-tenant (שורה אחת ב-DB). אצלנו צריך multi-tenant — כל user מחובר ליומן Google שלו. ה-`state` parameter של OAuth הוא ה-mechanism הקריטי לזיהוי איזה user חוזר מ-Google אחרי האישור.

### 2.1 — Google Cloud Console setup

**זהו אקט תפעולי, פעם אחת לפרויקט**. תיעוד מלא בקובץ נפרד `docs/deployment/google-cloud-console-setup.md`:

1. כניסה ל-https://console.cloud.google.com — להשתמש בחשבון Google של החברה (לא חשבון אישי של אמיר/גיא).
2. פרויקט חדש: `campaign-ai-prod`.
3. הפעלת API: **Google Calendar API**.
4. **OAuth consent screen**:
   - User Type: External
   - App name: `Campaign AI`
   - User support email + Developer contact: של גיא
   - Authorized domains: `campaign-ai.com` (או הדומיין שיהיה ב-prod)
   - **Scopes**: `https://www.googleapis.com/auth/calendar` (פעולות מלאות) ו-`.../userinfo.email` (לזיהוי איזה חשבון התחבר)
5. **Credentials → OAuth 2.0 Client ID**:
   - Application type: Web application
   - Authorized redirect URIs:
     - `https://api.campaign-ai.com/api/v1/google/callback` (prod)
     - `http://localhost:8000/api/v1/google/callback` (dev)
6. **App verification**: ל-MVP, ה-app יהיה ב-"Testing" mode עם רשימת users ב-whitelist. אחרי המכירה הראשונה צריך submission לאישור (לוקח 4-6 שבועות). זה אומר ב-MVP אנחנו מוסיפים את ה-emails של כל לקוח ל-test users.

⚠️ **חשוב לגיא**: ה-OAuth verification של Google דורש privacy policy + terms of service פומביים, וגם video של ה-flow. צריך להתחיל בתהליך מוקדם.

### 2.2 — env vars נדרשים

```bash
# Google Calendar OAuth
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx
GOOGLE_REDIRECT_URI=https://api.campaign-ai.com/api/v1/google/callback

# הצפנת tokens (Fernet). צריך מפתח חדש או שימוש חוזר באם קיים מ-3.4
SECRETS_ENCRYPTION_KEY=xxx     # 32 bytes base64-encoded
SECRETS_ENCRYPTION_KEY_VERSION=v1   # לrotation עתידי
```

**יצירת `SECRETS_ENCRYPTION_KEY`**: בפעם הראשונה, `pyt*** -* "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`. שמירה ב-Render env vars. **אסור לאבד את המפתח** — כל ה-refresh tokens מוצפנים אתו. אם הוא נאבד, כל הלקוחות יצטרכו לחבר מחדש.

### 2.3 — OAuth flow (multi-tenant)

קובץ חדש: `app/services/google_calendar_service.py`.

```python
# app/services/google_calendar_service.py
from google_auth_oauthlib.flow import Flow
from googleapiclient.discovery import build
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
from google.auth.exceptions import RefreshError
from app.core.config import settings
from app.core.crypto import encrypt_field, decrypt_field
from app.services.signed_state import create_signed_state, verify_signed_state  # קיים מ-3.4
from datetime import datetime, timedelta, timezone

SCOPES = [
    "https://www.googleapis.com/auth/calendar",
    "openid",
    "https://www.googleapis.com/auth/userinfo.email",
]

def _build_flow() -> Flow:
    return Flow.from_client_config(
        {
            "web": {
                "client_id": settings.GOOGLE_CLIENT_ID,
                "client_secret": settings.GOOGLE_CLIENT_SECRET,
                "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                "token_uri": "https://oauth2.googleapis.com/token",
                "redirect_uris": [settings.GOOGLE_REDIRECT_URI],
            }
        },
        scopes=SCOPES,
    )


def get_authorization_url(user_id: UUID) -> str:
    """מייצר URL לאישור Google עם state חתום שמכיל user_id."""
    flow = _build_flow()
    flow.redirect_uri = settings.GOOGLE_REDIRECT_URI

    # state = HMAC חתום של user_id + timestamp (תוקף 10 דק')
    state = create_signed_state(
        user_id=str(user_id),
        purpose="google_calendar_oauth",
        ttl_seconds=600,
    )

    auth_url, _ = flow.authorization_url(
        access_type="offline",       # ← קריטי: מבטיח refresh_token
        prompt="consent",            # ← מאלץ אישור → refresh_token חדש
        state=state,                 # ← נושא את user_id
        include_granted_scopes="true",
    )
    return auth_url


def exchange_code_for_credentials(code: str, state: str) -> dict:
    """אחרי שהמשתמש חזר מ-Google עם code + state, מאמת ושומר."""
    # אימות state — אם נכשל, exception. מנצח CSRF.
    payload = verify_signed_state(state, expected_purpose="google_calendar_oauth")
    user_id = UUID(payload["user_id"])

    flow = _build_flow()
    flow.redirect_uri = settings.GOOGLE_REDIRECT_URI
    flow.fetch_token(code=code)

    creds = flow.credentials
    if not creds.refresh_token:
        # זה קורה אם המשתמש כבר אישר ב-Google בעבר ולא חידש את ההסכמה.
        # `prompt=consent` אמור למנוע אבל בכל זאת — נסרב.
        raise GoogleOAuthError("לא קיבלנו refresh_token. נסה שוב.")

    # מציאת timezone ו-email מהיומן
    service = build("calendar", "v3", credentials=creds)
    cal_info = service.calendars().get(calendarId="primary").execute()
    email = cal_info.get("id", "")
    tz = cal_info.get("timeZone", "Asia/Jerusalem")

    # אם המשתמש כבר חיבר בעבר (re-connect) — UPSERT
    db.upsert_google_calendar_credentials(
        user_id=user_id,
        google_account_email=email,
        calendar_id="primary",
        timezone=tz,
        refresh_token_encrypted=encrypt_field(creds.refresh_token),
        access_token_encrypted=encrypt_field(creds.token),
        token_expiry=creds.expiry,
    )

    return {"email": email, "timezone": tz}
```

**נקודות מפתח:**

- **`state` הוא ה-mechanism המרכזי ל-multi-tenant OAuth**. בלי signed state, אין לנו דרך לדעת איזה user חזר מ-Google. `create_signed_state` כבר קיים אצלנו מ-3.4 (Meta OAuth — אותו דפוס). פשוט קוראים לו עם `purpose='google_calendar_oauth'`.
- **`access_type="offline" + prompt="consent"`** — שניהם חובה. בלי `offline` לא נקבל refresh_token. בלי `prompt=consent`, אם המשתמש כבר אישר את ה-scopes בעבר, Google לא תחזיר refresh_token חדש (אופטימיזציה שלהם).
- **לא לאחסן refresh_token ריק**. אם איכשהו `creds.refresh_token` חזר ריק — לזרוק exception ולא לשמור. השורה תהיה לא שמישה.
- **`upsert` ולא `insert`** — המשתמש יכול לחבר מחדש (למשל אחרי disconnect). אם משתמשים ב-INSERT נקבל UniqueViolation. UPSERT עם `ON CONFLICT (user_id) DO UPDATE` עובד נכון.

### 2.4 — הצפנה (Fernet)

קובץ חדש (או שימוש בקיים): `app/core/crypto.py`.

```python
# app/core/crypto.py
from cryptography.fernet import Fernet
from app.core.config import settings

_fernet = Fernet(settings.SECRETS_ENCRYPTION_KEY.encode())
_VERSION_PREFIX = settings.SECRETS_ENCRYPTION_KEY_VERSION  # למשל "v1"

def encrypt_field(plaintext: str) -> str:
    """
    מחזיר 'v1:<base64>'. הקידומת מאפשרת rotation עתידי של המפתח —
    אם בעתיד יהיה SECRETS_ENCRYPTION_KEY_V2, נוכל לדעת איזה מפתח השתמשנו לפענוח.

    אם plaintext ריק — מחזיר ריק (לא מוצפן). שונה מבעיית הקוד הישן:
    מאפשר לבדוק 'אין token' בלי לפענח קודם.
    """
    if not plaintext:
        return ""
    encrypted = _fernet.encrypt(plaintext.encode()).decode()
    return f"{_VERSION_PREFIX}:{encrypted}"


def decrypt_field(ciphertext: str) -> str:
    if not ciphertext:
        return ""
    if not ciphertext.startswith(f"{_VERSION_PREFIX}:"):
        # פורמט ישן או corruption — log + raise
        raise CryptoError("Unrecognized encryption version")
    raw = ciphertext.split(":", 1)[1]
    return _fernet.decrypt(raw.encode()).decode()
```

**הערה ל-Claude Code:** בדוק שאין כבר module דומה ב-codebase (אולי 3.4 הוסיף אותו ל-Meta tokens). אם כן — שימוש חוזר. אם לא — צור.

### 2.5 — Token refresh + ניטור בריאות

זה החלק הכי דליקט. הקוד הישן מטפל בכמה edge cases חכמים שצריך לאמץ as-is.

```python
def get_credentials(user_id: UUID) -> Credentials | None:
    """
    מחזיר Credentials תקפים ל-user, או None אם:
    - אין שורה
    - אין refresh_token
    - refresh נכשל (token נשלל)

    הפונקציה מבצעת refresh אוטומטי אם access_token פג.
    """
    cred_data = db.get_google_calendar_credentials(user_id)
    if not cred_data:
        return None

    encrypted_refresh = cred_data["refresh_token"]
    if not encrypted_refresh:
        return None

    creds = Credentials(
        token=decrypt_field(cred_data["access_token"]) if cred_data["access_token"] else None,
        refresh_token=decrypt_field(encrypted_refresh),
        token_uri="https://oauth2.googleapis.com/token",
        client_id=settings.GOOGLE_CLIENT_ID,
        client_secret=settings.GOOGLE_CLIENT_SECRET,
        expiry=cred_data["token_expiry"],
    )

    if creds.expired and creds.refresh_token:
        try:
            creds.refresh(Request())
            # ✓ הצליח — לעדכן ב-DB את ה-access token החדש + תוקף
            db.update_access_token(
                user_id=user_id,
                access_token_encrypted=encrypt_field(creds.token),
                token_expiry=creds.expiry,
            )
            # אם היה דגל auth_invalid_at — לאפס (התאוששנו)
            if cred_data["auth_invalid_at"]:
                db.clear_auth_invalid(user_id)
        except RefreshError:
            # ✗ נכשל — ה-refresh_token כנראה נשלל (המשתמש ניתק את ההרשאה ב-Google)
            # לסמן ולהתריע לבעל העסק *פעם אחת*
            was_already_marked = db.is_auth_invalid(user_id)
            db.set_auth_invalid(user_id)  # מעדכן auth_invalid_at = NOW()
            if not was_already_marked:
                _notify_owner_calendar_disconnected(user_id)
            return None

    return creds


def is_connected(user_id: UUID) -> bool:
    """
    מחזיר True רק אם:
    - יש שורה ב-google_calendar_credentials
    - refresh_token לא ריק
    - auth_invalid_at IS NULL (token עדיין תקין)
    """
    cred_data = db.get_google_calendar_credentials(user_id)
    if not cred_data:
        return False
    if not cred_data["refresh_token"]:
        return False
    if cred_data["auth_invalid_at"]:
        return False
    return True


def _notify_owner_calendar_disconnected(user_id: UUID):
    """
    שליחת התראה חד-פעמית לבעל העסק שהיומן התנתק.
    שני ערוצים:
    1. WhatsApp — דרך bot_config.fallback_value (אם פתוח חלון 24h)
    2. Email — דרך Resend (תמיד)
    """
    cfg = db.get_bot_config(user_id)
    if cfg and cfg["fallback_value"]:
        try:
            send_template_to_owner(
                phone=cfg["fallback_value"],
                template="bot_calendar_disconnected_v1",  # template נוסף — לעיון אצל גיא
                variables=[cfg["business_name"], settings.PANEL_URL + "/settings/bot/calendar"],
            )
        except Exception as e:
            logger.error(f"Failed to notify owner about calendar disconnection via WhatsApp: {e}")

    # תמיד שולחים גם email
    send_email(
        to=db.get_user_email(user_id),
        subject="היומן של Google התנתק מ-Campaign AI",
        template="calendar_disconnected.html",
    )
```

**הערות חשובות:**

- **`set_auth_invalid()` מחזיר את ההסטוריה לפני העדכון** — `was_already_marked = is_auth_invalid(user_id)` לפני `set_auth_invalid()` כדי לקבוע אם זו הפעם הראשונה (לשלוח התראה) או חזרה (לא להציף).
- **לא לזרוק exception ל-flow לקוח** — אם `get_credentials` מחזיר `None`, ה-caller צריך להתמודד עם זה (לרוב: ליפול ל-handoff message לליד, להציע לבעל העסק לחבר מחדש).
- **`is_connected()` ≠ `get_credentials() is not None`**. ה-ראשון בודק רק מצב; השני מבצע refresh. בדיקות UI/validation משתמשות ב-`is_connected`. flow פעיל משתמש ב-`get_credentials`.

### 2.6 — API endpoints

```
GET  /api/v1/google/connect         → מחזיר { "auth_url": "https://accounts.google.com/..." }
                                       דורש authenticated user (auth.uid()).
                                       קורא ל-get_authorization_url(user_id).

GET  /api/v1/google/callback?code=&state=
                                     → endpoint ש-Google חוזרת אליו.
                                       מאמת state, מבצע exchange.
                                       redirect ל-{PANEL_URL}/settings/bot/calendar?status=success
                                       (או ?status=error&reason=...)

POST /api/v1/google/disconnect      → מוחק את השורה ב-google_calendar_credentials של ה-user.
                                       דורש authentication.
                                       אם יש לו pending appointments — warning ב-UI לפני disconnect
                                       (לא חוסם — ה-DB עדיין מגן מ-double-booking).

GET  /api/v1/google/status          → מחזיר { "connected": bool, "email": str|null,
                                              "auth_invalid_at": timestamp|null,
                                              "timezone": "Asia/Jerusalem" }
```

**הערה**: `disconnect` לא מוחק את ה-appointments. הם נשארים. הסנכרון ליומן יפסיק לעבוד, אבל הבוט עדיין יודע לחשב slots פנויים (busy ranges מה-DB, בלי FreeBusy). מצב degraded אבל פונקציונלי. בעל העסק יכול לקבוע ידנית במקרה כזה.

---

## חלק 3 — Working Hours + Special Days

### 3.1 — API endpoints

```
GET    /api/v1/bot/working-hours          → רשימת 7 שורות (תמיד, גם אם is_open=false)
PUT    /api/v1/bot/working-hours          → bulk update של 7 השורות
GET    /api/v1/bot/special-days?from=&to= → רשימה לחודש/תקופה
POST   /api/v1/bot/special-days           → הוספה (UPSERT לפי user_id+date)
PATCH  /api/v1/bot/special-days/:id       → עדכון
DELETE /api/v1/bot/special-days/:id       → מחיקה
```

**ולידציה ל-`PUT /working-hours`:**

הקלט הוא array של 7 פריטים (day_of_week 0–6). כל פריט:
- אם `is_open=false`: `open_time` ו-`close_time` חייבים להיות null
- אם `is_open=true`: שניהם חייבים, ו-`open_time < close_time` (לא תומכים במשמרת לילה ב-MVP)
- כל יום יכול להיות עם שעות שונות (אין אכיפת homogeneity)

אם מישהו שולח רק 6 ימים → 422. רשימה חייבת להיות מלאה.

**ולידציה ל-`POST /special-days`:**

- `special_date >= today` (אי אפשר ליצור special_day בעבר)
- מקסימום שנה קדימה
- אם `is_closed=true` → `open_time` ו-`close_time` null
- אם `is_closed=false` → שניהם חובה, `open_time < close_time`

**UI:**

מסך אחד בפאנל תחת `/settings/bot/schedule` (או דומה). שני sections:

1. **שעות פעילות שבועיות** — 7 שורות, לכל אחת checkbox "פתוח" + 2 שדות זמן.
2. **ימים מיוחדים** — רשימה של special_days קיימים (עתיד בלבד) + כפתור "הוסף" שפותח modal: תאריך, סוגיה (סגור / שעות מותאמות), זמנים אם רלוונטי, סיבה (אופציונלי).

### 3.2 — פונקציה: `get_status_for_date(user_id, date)`

זו הפונקציה שמכריעה אם יום הוא פתוח/סגור עם איזה שעות. דפוס priority resolution — special_days דורס ברירת מחדל שבועית:

```python
def get_status_for_date(user_id: UUID, target_date: date) -> dict:
    """
    מחזיר:
    {
      "is_open": bool,
      "open_time": time|None,
      "close_time": time|None,
      "source": "special_day" | "weekly" | "default_closed",
      "reason": str|None,  # אם source=special_day
    }
    """
    # 1. בדיקת special_days — עדיפות עליונה
    special = db.get_special_day(user_id, target_date)
    if special:
        if special["is_closed"]:
            return {"is_open": False, "open_time": None, "close_time": None,
                    "source": "special_day", "reason": special["reason"]}
        return {"is_open": True, "open_time": special["open_time"],
                "close_time": special["close_time"], "source": "special_day",
                "reason": special["reason"]}

    # 2. שעות שבועיות רגילות
    # המרה מספירת Python ל-ישראלית
    israeli_dow = (target_date.weekday() + 1) % 7
    weekly = db.get_working_hours_for_day(user_id, israeli_dow)
    if not weekly or not weekly["is_open"]:
        return {"is_open": False, "open_time": None, "close_time": None,
                "source": "weekly", "reason": None}
    return {"is_open": True, "open_time": weekly["open_time"],
            "close_time": weekly["close_time"], "source": "weekly", "reason": None}
```

**מלכודת ספירת ימים**: Python's `date.weekday()` מחזיר 0=שני, 6=ראשון. הספירה הישראלית 0=ראשון, 6=שבת. הנוסחה: `(python_weekday + 1) % 7`. אם תשכח את זה — כל הסלוטים יוצגו ביום הלא נכון. כלל ברזל: לכתוב טסט עם date ספציפי (`date(2026, 6, 7)` = יום ראשון → israeli_dow צריך להיות 0).

---

## חלק 4 — חישוב Slots פנויים (הליבה)

זה החלק הכי קריטי טכנית של 5.5. המסמך הישן מתאר את האלגוריתם במלואו (§2 שם). אני מציג adaptation למלוויי-משתמשים. השינויים העיקריים: כל פונקציה מקבלת `user_id`, ה-FreeBusy נשלח עם calendar_id הספציפי של המשתמש.

קובץ: `app/services/slot_availability_service.py`.

### 4.1 — `get_busy_slots(user_id, time_min, time_max)`

```python
class CalendarUnavailable(Exception):
    """ה-FreeBusy נכשל — או auth/connection. נבדל מ-'אין busy'."""
    pass


def get_busy_slots(user_id: UUID, time_min: datetime, time_max: datetime) -> list[dict]:
    """
    קריאה ל-FreeBusy API של Google. מחזיר רשימת טווחים תפוסים.

    זורק CalendarUnavailable אם:
    - אין credentials
    - refresh נכשל
    - ה-API נכשל
    """
    creds = get_credentials(user_id)  # מ-section 2.5
    if not creds:
        raise CalendarUnavailable("auth/connection")

    cred_data = db.get_google_calendar_credentials(user_id)
    calendar_id = cred_data["calendar_id"]
    tz = cred_data["timezone"]

    try:
        service = build("calendar", "v3", credentials=creds, cache_discovery=False)
        body = {
            "timeMin": time_min.isoformat(),
            "timeMax": time_max.isoformat(),
            "timeZone": tz,
            "items": [{"id": calendar_id}],
        }
        result = service.freebusy().query(body=body).execute()
        return result.get("calendars", {}).get(calendar_id, {}).get("busy", [])
    except Exception as e:
        logger.error(f"FreeBusy failed for user {user_id}: {e}")
        raise CalendarUnavailable(f"freebusy_api_error: {e}")
```

### 4.2 — `get_appointments_busy_ranges(user_id, date)`

```python
def get_appointments_busy_ranges(user_id: UUID, target_date: date) -> list[tuple[int, int]]:
    """
    מחזיר רשימת טווחים תפוסים מה-DB ל-user_id בתאריך הזה.
    כל טווח מתבטא כדקות מחצות: (start_minutes, end_minutes).

    כולל רק appointments עם status IN ('pending','confirmed').
    cancelled/rejected/passed לא חוסמים.
    """
    rows = db.get_appointments_for_user_date(
        user_id=user_id,
        date=target_date,
        active_statuses=["pending", "confirmed"],
    )
    ranges = []
    for r in rows:
        h, m = r["preferred_time"].hour, r["preferred_time"].minute
        start_min = h * 60 + m
        end_min = start_min + r["confirmed_duration_minutes"]
        ranges.append((start_min, end_min))
    return ranges
```

### 4.3 — `get_available_slots(user_id, date, duration_minutes)`

הפונקציה המרכזית. חיתוך שלושה מקורות.

```python
from zoneinfo import ZoneInfo

IL_TZ = ZoneInfo("Asia/Jerusalem")

def get_available_slots(
    user_id: UUID,
    target_date: date,
    duration_minutes: int,
    buffer_between_minutes: int = 0,
    buffer_after_event_minutes: int = 0,
) -> list[str]:
    """
    מחזיר רשימת זמני התחלה אפשריים בפורמט "HH:MM".
    אם היום סגור → רשימה ריקה.
    אם FreeBusy נכשל → מחזיר רשימה ריקה (לא להציע slots בלי לדעת מצב היומן) + מסמן calendar_check_failed (decision engine מתייחס לזה).
    """
    # 1. בדיקת מצב היום
    day_status = get_status_for_date(user_id, target_date)
    if not day_status["is_open"]:
        return []

    day_start = datetime.combine(target_date, day_status["open_time"], tzinfo=IL_TZ)
    day_end   = datetime.combine(target_date, day_status["close_time"], tzinfo=IL_TZ)

    # 2. היום? לעגל לעלייה ל-slot הבא
    now = datetime.now(IL_TZ)
    if target_date == now.date() and now > day_start:
        next_slot_min = ((now.hour * 60 + now.minute) // 30 + 1) * 30
        if next_slot_min >= 24 * 60:
            return []
        day_start = now.replace(
            hour=next_slot_min // 60, minute=next_slot_min % 60,
            second=0, microsecond=0,
        )

    # 3. busy מהיומן
    busy_ranges = []
    event_buffer = timedelta(minutes=max(0, buffer_after_event_minutes))
    try:
        for slot in get_busy_slots(user_id, day_start, day_end):
            start = datetime.fromisoformat(slot["start"]).astimezone(IL_TZ)
            end   = datetime.fromisoformat(slot["end"]).astimezone(IL_TZ) + event_buffer
            busy_ranges.append((start, end))
    except CalendarUnavailable:
        # FreeBusy נכשל — לא להציע slots בלי לדעת
        return []

    # 4. busy מה-DB (תורים שלנו)
    for start_min, end_min in get_appointments_busy_ranges(user_id, target_date):
        db_start = datetime.combine(target_date, time(0, 0), tzinfo=IL_TZ) + timedelta(minutes=start_min)
        db_end   = datetime.combine(target_date, time(0, 0), tzinfo=IL_TZ) + timedelta(minutes=end_min) + event_buffer
        busy_ranges.append((db_start, db_end))

    # 5. גריד 30 דק', בדיקת חפיפה
    slot_total = timedelta(minutes=duration_minutes + buffer_between_minutes)
    available, current = [], day_start
    while current + timedelta(minutes=duration_minutes) <= day_end:
        slot_end = current + slot_total
        # חפיפה: start < busy_end AND end > busy_start
        is_free = not any(current < be and slot_end > bs for bs, be in busy_ranges)
        if is_free:
            available.append(current.strftime("%H:%M"))
        current += timedelta(minutes=30)

    return available
```

**נוסחת חפיפה — זכרון מוזהב:**

```
שני טווחים A=[a1,a2] ו-B=[b1,b2] חופפים ⟺ a1 < b2 AND a2 > b1
```

זה הביטוי הקנוני, ויעיל יותר מ-checks ידני של "האם A תוך B או B תוך A או חופף חלקית".

### 4.4 — `get_available_dates(user_id, duration_minutes, from_date, days_ahead=60)`

נדרש ל-Date List Picker. מחזיר רשימת תאריכים שיש בהם לפחות slot אחד פנוי.

```python
def get_available_dates(
    user_id: UUID,
    duration_minutes: int,
    from_date: date = None,
    days_ahead: int = 60,
) -> list[date]:
    """
    סורק עד `days_ahead` ימים קדימה, מחזיר רק תאריכים שיש בהם slot פנוי.

    ⚠️ עלות: עד 60 קריאות FreeBusy. צריך caching.

    ההמלצה ל-MVP: cache לכל (user_id, year, month) במ-Redis עם TTL=15min.
    Invalidation אקטיבי: כל יצירה/ביטול של appointment מנקה את ה-cache לחודש הזה.
    """
    from_date = from_date or date.today()
    available_dates = []
    for offset in range(days_ahead):
        d = from_date + timedelta(days=offset)
        # ה-cache נמצא ברמת חודש — get_month_availability מחזיר dict ימים→זמינות.
        # ב-MVP אפשר לוותר על cache ולקרוא ישירות get_available_slots; cache הוא optimization.
        slots = get_available_slots(user_id, d, duration_minutes)
        if slots:
            available_dates.append(d)
    return available_dates
```

**הערת cost**: לקלוד קוד — האופציה ה-naïve (60 קריאות FreeBusy פר טעינת picker) עלולה לחרוג מ-rate limits של Google ב-burst של לידים. אם פתרונות פשוטים לא מספיקים, להוסיף `get_month_availability` עם cache. **לא חובה ב-MVP** אם יש פחות מ-10 קמפיינים פעילים — Google מאפשרת ~1M כדורי FreeBusy ביום.

---

## חלק 5 — Booking Decision Engine

המסמך הישן (§4) מתאר 12-cell decision tree עם 3 modes. ב-5.5 אנחנו מממשים רק `manual` mode, אבל **הקוד יישאר עם המבנה המלא** כדי שעדכון עתידי יוסיף את שני המצבים האחרים בלי refactor.

קובץ: `app/core/booking_decision.py`.

```python
from dataclasses import dataclass
from datetime import date, datetime, time
from typing import Literal

Decision = Literal["confirmed", "pending", "rejected"]


@dataclass
class BookingDecisionInput:
    mode: Literal["manual", "auto_with_check", "auto_always"]
    slot_date: date
    slot_time: time
    duration_minutes: int
    now_il: datetime
    business_hours_status: dict  # מ-get_status_for_date()
    has_active_conflict: bool    # יש pending/confirmed באותו slot? (race check)
    calendar_connected: bool
    calendar_check_failed: bool
    available_slots: list[str]


@dataclass
class BookingDecision:
    decision: Decision
    reason: str  # קוד אנגלי לשליפת הודעה מ-_REASON_MESSAGES


def decide_appointment_status(inp: BookingDecisionInput) -> BookingDecision:
    """
    פונקציה טהורה. אפס I/O. כל הקלט מגיע כפרמטרים.

    סדר הבדיקות:
    1. global rejections (עבר, רחוק מדי, התנגשות) — לא תלוי mode
    2. manual mode: כל מה שעבר (1) → pending
    3. auto_always mode (לא ב-MVP — מומש לעתיד): מאשר תמיד פרט לחופשה/עבר
    4. auto_with_check mode (לא ב-MVP): בודק calendar + business hours
    """
    # ① השעה עברה?
    slot_dt = datetime.combine(inp.slot_date, inp.slot_time).replace(tzinfo=inp.now_il.tzinfo)
    if slot_dt < inp.now_il:
        return BookingDecision("rejected", "slot_in_past")

    # ② רחוק מדי קדימה?
    if (inp.slot_date - inp.now_il.date()).days > 90:
        return BookingDecision("rejected", "slot_too_far_ahead")

    # ③ התנגשות?
    if inp.has_active_conflict:
        return BookingDecision("rejected", "slot_already_taken")

    # ④ manual mode — תמיד pending (אחרי הבדיקות הגלובליות)
    if inp.mode == "manual":
        return BookingDecision("pending", "manual_review")

    # ⑤ auto_always — מאשר אם לא חופשה (לעתיד)
    if inp.mode == "auto_always":
        if not inp.business_hours_status["is_open"]:
            return BookingDecision("rejected", "vacation_or_closed")
        return BookingDecision("confirmed", "auto_always_ok")

    # ⑥ auto_with_check — בדיקות מורחבות (לעתיד)
    if inp.mode == "auto_with_check":
        if not inp.business_hours_status["is_open"]:
            source = inp.business_hours_status["source"]
            return BookingDecision("rejected", f"closed_{source}")
        if not inp.calendar_connected or inp.calendar_check_failed:
            return BookingDecision("pending", "calendar_uncertain")  # אי-ודאות → pending
        if inp.slot_time.strftime("%H:%M") not in inp.available_slots:
            return BookingDecision("rejected", "calendar_busy")
        return BookingDecision("confirmed", "auto_with_check_ok")

    # מצב לא צפוי
    return BookingDecision("pending", "unknown_mode")


_REASON_MESSAGES = {
    "slot_in_past":         "השעה שבחרת כבר עברה. אנא בחר שעה אחרת.",
    "slot_too_far_ahead":   "אפשר לתאם תור עד 90 יום קדימה. אנא בחר תאריך קרוב יותר.",
    "slot_already_taken":   "השעה הזו נתפסה לפני שניות. אנא בחר שעה אחרת.",
    "manual_review":        "בקשת התור הועברה לבעל העסק לאישור. תקבל אישור תוך 24 שעות.",
    # ה-codes הבאים לעתיד, כש-auto modes ייפתחו:
    "vacation_or_closed":   "בעל העסק לא זמין בתאריך זה.",
    "closed_weekly":        "התאריך שבחרת הוא יום סגור. אנא בחר תאריך אחר.",
    "closed_special_day":   "התאריך שבחרת אינו זמין. אנא בחר תאריך אחר.",
    "calendar_busy":        "השעה הזו כבר תפוסה ביומן. אנא בחר שעה אחרת.",
    "calendar_uncertain":   "בקשת התור הועברה לבעל העסק לאישור.",
    "auto_with_check_ok":   "התור אושר! פרטים בהודעה הבאה.",
    "auto_always_ok":       "התור אושר! פרטים בהודעה הבאה.",
}


def get_user_message(reason: str) -> str:
    return _REASON_MESSAGES.get(reason, "אירעה תקלה. נסה שוב או צור קשר עם בעל העסק.")
```

**Orchestrator** — `gather_and_decide(user_id, slot_date, slot_time)`:

```python
def gather_and_decide(user_id: UUID, slot_date: date, slot_time: time) -> BookingDecision:
    cfg = db.get_bot_config(user_id)
    mode = cfg.get("auto_booking_mode", "manual")  # ב-5.5 תמיד "manual"
    duration = cfg["default_appointment_duration_minutes"]

    bh_status = get_status_for_date(user_id, slot_date)
    has_conflict = db.has_active_appointment(user_id, slot_date, slot_time)
    calendar_connected = is_connected(user_id)

    available_slots_today = []
    calendar_check_failed = False
    try:
        available_slots_today = get_available_slots(user_id, slot_date, duration)
    except Exception:
        calendar_check_failed = True

    inp = BookingDecisionInput(
        mode=mode,
        slot_date=slot_date,
        slot_time=slot_time,
        duration_minutes=duration,
        now_il=datetime.now(IL_TZ),
        business_hours_status=bh_status,
        has_active_conflict=has_conflict,
        calendar_connected=calendar_connected,
        calendar_check_failed=calendar_check_failed,
        available_slots=available_slots_today,
    )
    return decide_appointment_status(inp)
```

**למה פונקציה טהורה?** בלי DB, בלי רשת. ה-orchestrator עושה את העבודה המלוכלכת ומכין את ה-input. הפונקציה עצמה נבדקת ב-12 unit tests פשוטים — מקרה אחד לכל ענף בעץ ההחלטה. בלי mocks. בלי flakiness.

**הערה ל-5.5 ספציפית**: כיוון שאנחנו ב-manual mode בלבד, הענפים 5–6 (`auto_always`, `auto_with_check`) **לא מופעלים בפועל**. הם בקוד למניעת refactor עתידי. הטסטים שלהם יכולים להיכתב כ-skip עם `pytest.mark.skip("auto modes deferred")` או להישאר full coverage. ההמלצה: full coverage. זה ביטוח שכשהמצב יתפעל בעתיד, ה-לוגיקה כבר נבדקה.

---

## חלק 6 — State Machine Extension

ה-state machine של `bot_conversations` קיים מ-5.2. ב-5.5 אנחנו מוסיפים 6 states (ראה migration 0031 בחלק 1).

### 6.1 — תרשים מעברים מלא

```
                      [נקודת כניסה מ-5.2]
                              │
              completed לפי fallback_action:
                              │
       ┌──────────────────────┼──────────────────────────┐
       ▼                      ▼                          ▼
completed_calendly    completed_handoff       scheduling_date  ← תוספת 5.5
                                                          │
                                                          ▼
                                         (List Picker → תאריך נבחר)
                                                          │
                                                          ▼
                                                scheduling_time
                                                          │
                                                          ▼
                                          (List Picker → שעה נבחרה)
                                                          │
                                                          ▼
                                              scheduling_confirm
                                                          │
                              ┌───────────────────────────┴─────────────────────┐
                              ▼                                                  ▼
                       Quick Reply: ✅ אישור                             Quick Reply: ❌ ביטול
                              │                                                  │
                              ▼                                                  ▼
                  INSERT appointment (pending)                       חזרה ל-scheduling_date
                              ▼                                  (או סגירת השיחה?)
                completed_appointment_scheduled
                              │
                              │
                  ───────────────────────────────────  שיחה terminal  ──────────
                              │
                              │ (הליד פונה שוב מ-state terminal)
                              ▼
                  NLU classify_intent(text)
                              │
                  ┌───────────┴───────────┐
                  ▼                       ▼
              "answer"          "cancel_appointment"
              (deviation)               │
                                         ▼
                       awaiting_cancellation_confirm
                                         │
                                         ▼
                              Quick Reply: ✅ / ❌
                                         │
                              ┌──────────┴──────────┐
                              ▼                     ▼
                       UPDATE → cancelled    חזרה ל-completed_appointment_scheduled
                              ▼
                  completed_appointment_cancelled

(תרחיש מקביל: בעל העסק אישר/דחה בפאנל →
 ה-state של ה-conversation נשאר completed_appointment_scheduled,
 רק appointments.status משתנה ל-confirmed/rejected, ונשלח template לליד.)
```

### 6.2 — Outbound jobs queue (extending 5.2 patterns)

ב-5.2 הוגדרה תשתית של async jobs דרך תור (Redis Queue או דומה). כל שליחת הודעה ב-WhatsApp עוברת דרך job. ב-5.5 מוסיפים jobs חדשים:

```
send_date_picker_job(conversation_id)
  → טוען conversation, user, מחשב available_dates, שולח List Picker
  → set conversation.state = "scheduling_date"

send_time_picker_job(conversation_id, selected_date_iso)
  → טוען conversation, user, מחשב available_slots לתאריך
  → שולח List Picker עם slots
  → set conversation.state = "scheduling_time"
  → set conversation.scheduling_data = {"selected_date": iso}

send_appointment_confirmation_request_job(conversation_id, selected_date_iso, selected_time)
  → שולח Quick Reply עם פרטי התור והודעת אישור
  → set conversation.state = "scheduling_confirm"
  → update scheduling_data with selected_time

process_appointment_confirmation_job(conversation_id, confirmed: bool)
  → אם True: gather_and_decide → INSERT appointments → notify owner job → reply לליד
  → אם False: חזרה ל-scheduling_date (או סיום? נחליט בחלק 7)
  → set conversation.state = "completed_appointment_scheduled" (אם True) או scheduling_date (אם False)

notify_owner_pending_job(appointment_id)
  → ניסיון שליחת הודעה חופשית. fallback ל-template.
  → גם email גיבוי דרך Resend

send_lead_confirmation_job(appointment_id)
  → אחרי שבעל העסק אישר בפאנל. שליחת template confirmation לליד.
  → כולל ICS link

send_lead_rejection_job(appointment_id)
  → אחרי שבעל העסק דחה בפאנל. שליחת template rejection לליד.

handle_lead_cancellation_intent_job(conversation_id)
  → ה-NLU זיהה כוונת ביטול.
  → שליחת Quick Reply אישור ביטול
  → set conversation.state = "awaiting_cancellation_confirm"

execute_lead_cancellation_job(conversation_id, confirmed: bool)
  → אם True: UPDATE appointment.status=cancelled, DELETE event ב-GCal, notify owner
  → set conversation.state = "completed_appointment_cancelled"
```

**שדה חדש ב-`bot_conversations`** או טבלה צדדית? לאחסון `scheduling_data` (selected_date, selected_time בין הצעדים):

ההמלצה: **JSONB ב-`bot_conversations`** — שדה `scheduling_data jsonb`. פשוט, אטומי עם שאר ה-conversation, וכבר יש דפוס דומה ל-`answers` ב-5.2/5.3.

מיגרציה קטנה (אם השדה לא קיים):
```sql
ALTER TABLE public.bot_conversations
  ADD COLUMN IF NOT EXISTS scheduling_data jsonb DEFAULT '{}'::jsonb;
```

או הוספה לתוך migration 0031 שכבר נכתב.

---

## חלק 7 — Booking Flow ב-WhatsApp

מבנה ההודעות וה-Pickers. מבוסס על Twilio Content Templates / Meta Interactive Messages.

### 7.1 — Entry: send_date_picker

```python
def send_date_picker_job(conversation_id: UUID):
    conv = db.get_conversation(conversation_id)
    user_id = conv["user_id"]
    cfg = db.get_bot_config(user_id)

    duration = cfg["default_appointment_duration_minutes"]

    # חישוב תאריכים פנויים — מקסימום 30 יום קדימה ב-MVP
    dates = get_available_dates(user_id, duration, days_ahead=30)

    if not dates:
        # אין שום תאריך פנוי ב-30 הימים הקרובים — נדיר אבל אפשרי
        send_text_message_to_lead(
            conv["lead_phone"],
            f"לצערי אין שעות פנויות ב-30 הימים הקרובים. בעל העסק יצור איתך קשר ישירות. תודה!"
        )
        # התראה לבעל העסק על המצב
        notify_owner_no_availability(user_id, conversation_id)
        db.update_conversation_state(conversation_id, "completed_handoff")
        return

    # List Picker: עד 10 פריטים בכל פעם, pagination
    items = [
        {
            "id": f"date_{d.isoformat()}",
            "title": _format_date_hebrew(d),  # למשל "ראשון 09/06"
        }
        for d in dates[:9]
    ]
    if len(dates) > 9:
        items.append({"id": "date_more_1", "title": "▶ עוד תאריכים..."})

    send_whatsapp_list_picker(
        to=conv["lead_phone"],
        body=f"אנא בחר תאריך מהרשימה (עד {duration} דקות):",
        button_text="בחר תאריך",
        items=items,
    )

    db.update_conversation_state(conversation_id, "scheduling_date")
    db.update_conversation_scheduling_data(conversation_id, {"page": 0})
```

### 7.2 — Webhook handling: incoming date choice

ה-webhook מ-5.2 מקבל את ההודעה. אחרי identifying conversation, אם state = "scheduling_date":

```python
def handle_scheduling_date_input(conversation_id: UUID, text: str):
    """
    text יכול להיות:
    - "date_2026-06-15" (List Picker callback)
    - "date_more_1" (pagination request)
    - טקסט חופשי (deviation — לא תמכנו ב-MVP)
    """
    if text.startswith("date_more_"):
        page = int(text.split("_")[-1])
        # שליחת page הבא של תאריכים
        _resend_date_picker(conversation_id, page=page)
        return

    if text.startswith("date_"):
        try:
            selected_date_iso = text[5:]  # "2026-06-15"
            selected_date = date.fromisoformat(selected_date_iso)
        except ValueError:
            _send_deviation_response(conversation_id, "לא הצלחתי להבין את התאריך. אנא בחר מהרשימה.")
            _resend_date_picker(conversation_id, page=0)
            return

        # בדיקה שהתאריך עדיין פנוי (race condition של דקות אחרי השליחה של הרשימה)
        user_id = db.get_conversation(conversation_id)["user_id"]
        cfg = db.get_bot_config(user_id)
        slots = get_available_slots(user_id, selected_date, cfg["default_appointment_duration_minutes"])
        if not slots:
            send_text_message_to_lead(
                conversation_id,
                "לצערי, התאריך הזה התפס בינתיים. אנא בחר תאריך אחר."
            )
            _resend_date_picker(conversation_id, page=0)
            return

        # אנקווה את ה-time picker
        enqueue_job("send_time_picker_job", {
            "conversation_id": conversation_id,
            "selected_date_iso": selected_date_iso,
        })
        return

    # טקסט חופשי — deviation
    _handle_deviation(conversation_id, text, expected_action="select_date_from_picker")


def send_time_picker_job(conversation_id: UUID, selected_date_iso: str):
    conv = db.get_conversation(conversation_id)
    user_id = conv["user_id"]
    cfg = db.get_bot_config(user_id)
    duration = cfg["default_appointment_duration_minutes"]

    selected_date = date.fromisoformat(selected_date_iso)
    slots = get_available_slots(user_id, selected_date, duration)

    if not slots:
        # race condition — התפס בזמן הקצר בין send/receive
        send_text_message_to_lead(conv["lead_phone"],
            "התאריך הזה התפס בינתיים. אנא בחר תאריך אחר.")
        _resend_date_picker(conversation_id, page=0)
        return

    items = [
        {"id": f"time_{slot}", "title": slot}  # "09:00", "09:30", ...
        for slot in slots[:10]
    ]

    send_whatsapp_list_picker(
        to=conv["lead_phone"],
        body=f"השעות הפנויות ב-{_format_date_hebrew(selected_date)}:",
        button_text="בחר שעה",
        items=items,
    )

    db.update_conversation_state(conversation_id, "scheduling_time")
    db.update_conversation_scheduling_data(conversation_id, {
        "selected_date": selected_date_iso,
        "page": 0,
    })
```

### 7.3 — Time selection → confirmation request

```python
def handle_scheduling_time_input(conversation_id: UUID, text: str):
    if not text.startswith("time_"):
        _handle_deviation(conversation_id, text, expected_action="select_time_from_picker")
        return

    selected_time_str = text[5:]  # "09:30"
    try:
        selected_time = time.fromisoformat(selected_time_str + ":00")
    except ValueError:
        # invalid format
        return

    sched_data = db.get_conversation(conversation_id)["scheduling_data"]
    selected_date = date.fromisoformat(sched_data["selected_date"])

    # אנקווה confirmation
    enqueue_job("send_confirmation_request_job", {
        "conversation_id": conversation_id,
        "selected_date_iso": selected_date.isoformat(),
        "selected_time_str": selected_time_str,
    })


def send_confirmation_request_job(conversation_id, selected_date_iso, selected_time_str):
    conv = db.get_conversation(conversation_id)
    cfg = db.get_bot_config(conv["user_id"])

    body = (
        f"לסיכום — בקשת התור שלך:\n\n"
        f"📅 תאריך: {_format_date_hebrew(date.fromisoformat(selected_date_iso))}\n"
        f"🕐 שעה: {selected_time_str}\n\n"
        f"לאשר את הבקשה?"
    )

    send_whatsapp_quick_reply(
        to=conv["lead_phone"],
        body=body,
        buttons=[
            {"id": "appt_confirm_yes", "title": "✅ אשר"},
            {"id": "appt_confirm_no",  "title": "❌ בטל ובחר אחר"},
        ],
    )

    db.update_conversation_state(conversation_id, "scheduling_confirm")
    db.update_conversation_scheduling_data(conversation_id, {
        "selected_date": selected_date_iso,
        "selected_time": selected_time_str,
    })
```

### 7.4 — Final confirmation → appointment creation

```python
def handle_scheduling_confirm_input(conversation_id: UUID, text: str):
    if text not in {"appt_confirm_yes", "appt_confirm_no"}:
        # deviation
        return

    if text == "appt_confirm_no":
        # חזרה לתחילת ה-flow
        enqueue_job("send_date_picker_job", {"conversation_id": conversation_id})
        return

    # appt_confirm_yes — יצירת התור
    conv = db.get_conversation(conversation_id)
    sched = conv["scheduling_data"]

    selected_date = date.fromisoformat(sched["selected_date"])
    selected_time = time.fromisoformat(sched["selected_time"] + ":00")

    decision = gather_and_decide(conv["user_id"], selected_date, selected_time)

    if decision.decision == "rejected":
        # לדוגמה race condition — slot נתפס
        send_text_message_to_lead(conv["lead_phone"], get_user_message(decision.reason))
        # חזרה ל-date picker
        enqueue_job("send_date_picker_job", {"conversation_id": conversation_id})
        return

    # יצירת התור — pending or confirmed (במצב ה-5.5 תמיד pending)
    try:
        appointment_id = db.create_appointment(
            user_id=conv["user_id"],
            conversation_id=conversation_id,
            lead_phone=conv["lead_phone"],
            lead_name=conv.get("lead_name"),
            preferred_date=selected_date,
            preferred_time=selected_time,
            confirmed_duration_minutes=db.get_bot_config(conv["user_id"])["default_appointment_duration_minutes"],
            status=decision.decision,  # "pending" ב-manual mode
        )
    except UniqueViolation:
        # double-booking — race condition
        send_text_message_to_lead(conv["lead_phone"],
            "השעה הזו נתפסה לפני שניות. אנא בחר שעה אחרת.")
        enqueue_job("send_date_picker_job", {"conversation_id": conversation_id})
        return

    # הודעה לליד
    send_text_message_to_lead(conv["lead_phone"], get_user_message(decision.reason))
    # = "בקשת התור הועברה לבעל העסק לאישור. תקבל אישור תוך 24 שעות."

    # התראה לבעל העסק
    enqueue_job("notify_owner_pending_job", {"appointment_id": appointment_id})

    db.update_conversation_state(conversation_id, "completed_appointment_scheduled")
```

**הערות:**

- **`UniqueViolation` catch** קריטי. ב-`gather_and_decide` יש race check (`has_active_conflict`), אבל בין הבדיקה ל-INSERT יכול לעבור milliseconds שבהם מישהו אחר ייצר. ה-DB constraint הוא ההגנה הסופית.
- **`appt_confirm_no` חוזר ל-date picker** ולא לסיום. הסיבה: אם הליד מתכנן ולוחץ "בטל ובחר אחר", הוא רוצה להמשיך לחפש slot, לא לסגור את הבוט.

---

## חלק 8 — Cancellation Flow

ה-flow של ביטול תור על ידי הליד דרך WhatsApp. מבוסס על patch 5.3.B (NLU intent detection).

### 8.1 — זיהוי כוונה במצב terminal

ב-state `completed_appointment_scheduled`, אם הליד שולח הודעה כלשהי, ה-webhook handler קורא ל-NLU:

```python
def handle_terminal_state_message(conversation_id: UUID, text: str):
    """
    הליד פנה אחרי שהשיחה הסתיימה. צריך לזהות אם זו כוונת ביטול.
    """
    conv = db.get_conversation(conversation_id)

    # בדיקה: יש לו appointment פעיל?
    active_appt = db.get_active_appointment_for_conversation(conversation_id)
    if not active_appt:
        # אין תור פעיל — אין סיבה לעשות classify. handoff generic.
        send_text_message_to_lead(conv["lead_phone"],
            "השיחה הסתיימה. לפרטים נוספים, אנא צור קשר עם בעל העסק.")
        return

    # יש תור — נסווג intent
    intent = classify_intent(text, has_active_appointment=True)  # patch 5.3.B

    if intent == "cancel_appointment":
        enqueue_job("handle_lead_cancellation_intent_job", {
            "conversation_id": conversation_id,
            "appointment_id": active_appt["id"],
        })
        return

    # intent != cancel — נשלח deviation generic
    send_text_message_to_lead(conv["lead_phone"],
        "השיחה הסתיימה. אם אתה רוצה לבטל את התור, כתוב 'לבטל'. אחרת — אנא צור קשר עם בעל העסק.")


def handle_lead_cancellation_intent_job(conversation_id: UUID, appointment_id: UUID):
    appt = db.get_appointment(appointment_id)
    conv = db.get_conversation(conversation_id)

    body = (
        f"האם לבטל את התור הבא?\n\n"
        f"📅 {_format_date_hebrew(appt['preferred_date'])}\n"
        f"🕐 {appt['preferred_time'].strftime('%H:%M')}\n\n"
    )
    send_whatsapp_quick_reply(
        to=conv["lead_phone"],
        body=body,
        buttons=[
            {"id": "cancel_appt_yes", "title": "✅ כן, לבטל"},
            {"id": "cancel_appt_no",  "title": "❌ לא, להשאיר"},
        ],
    )

    db.update_conversation_state(conversation_id, "awaiting_cancellation_confirm")
    db.update_conversation_scheduling_data(conversation_id, {
        "cancellation_target_appt": str(appointment_id),
    })
```

### 8.2 — ביצוע הביטול

```python
def handle_cancellation_confirm_input(conversation_id: UUID, text: str):
    if text == "cancel_appt_no":
        # חזרה ל-completed
        send_text_message_to_lead(conversation_id, "התור נשאר. נתראה!")
        db.update_conversation_state(conversation_id, "completed_appointment_scheduled")
        return

    if text == "cancel_appt_yes":
        sched = db.get_conversation(conversation_id)["scheduling_data"]
        appt_id = UUID(sched["cancellation_target_appt"])
        enqueue_job("execute_lead_cancellation_job", {
            "conversation_id": conversation_id,
            "appointment_id": appt_id,
        })
        return


def execute_lead_cancellation_job(conversation_id: UUID, appointment_id: UUID):
    """
    פעולה אטומית:
    1. UPDATE appointment SET status='cancelled', cancelled_by='lead'
    2. DELETE event ב-GCal (אם יש google_event_id)
    3. notify owner
    4. send confirmation to lead
    """
    with db.transaction():  # אטומיות ברמת ה-DB
        appt = db.get_appointment_for_update(appointment_id)
        if appt["status"] not in ("pending", "confirmed"):
            # כבר בוטל / נדחה — idempotent
            return

        was_confirmed = appt["status"] == "confirmed"
        db.update_appointment_status(
            appointment_id,
            status="cancelled",
            cancelled_by="lead",
            cancelled_at=datetime.now(timezone.utc),
        )

    # מחיקת אירוע מהיומן (מחוץ ל-transaction — אם נכשל, התור עדיין מבוטל)
    if was_confirmed and appt["google_event_id"]:
        try:
            delete_calendar_event(appt["user_id"], appt["google_event_id"])
        except Exception as e:
            # 410 Gone = ה-event כבר נמחק. נחשב הצלחה.
            if "410" not in str(e):
                logger.error(f"Failed to delete GCal event: {e}")

    # הודעה לבעל העסק
    enqueue_job("notify_owner_cancellation_job", {"appointment_id": appointment_id})

    # אישור לליד
    conv = db.get_conversation(conversation_id)
    send_text_message_to_lead(conv["lead_phone"], "התור בוטל. תודה שעדכנת אותנו.")

    db.update_conversation_state(conversation_id, "completed_appointment_cancelled")
```

**הערות:**

- **transaction סביב ה-status update**: למניעת ביטול כפול (race) ולמניעת מצב לא עקבי.
- **`delete_calendar_event` מחוץ ל-transaction**: אם הקריאה ל-Google נכשלת, לא רוצים לבטל את ה-rollback של ה-DB. ה-DB הוא source of truth.
- **idempotent**: אם הליד לוחץ פעמיים על "כן בטל", ה-status כבר cancelled → return בלי לעשות שום דבר נוסף. אין כפילויות התראות.

---

## חלק 9 — Owner Notifications (24h window pattern)

זה החלק היחיד שדורש זהירות כלפי מטא בכל הקשור ל-policy. בעל העסק הוא משתמש WhatsApp רגיל מבחינת מטא — הם לא מבחינים בינו לבין לקוח. החוקים זהים: חלון 24 שעות מהודעה אחרונה ממנו → free messaging. מעבר לזה → template בלבד.

### 9.1 — try-then-fallback

הפתרון הנקי ביותר הוא לנסות free message ולנפול ל-template אם הוא נכשל. אנחנו לא צריכים לעקוב אחרי המצב של החלון בעצמנו.

```python
def send_to_owner(user_id: UUID, free_message: str, template_name: str, template_params: list):
    """
    מנסה לשלוח הודעה חופשית לבעל העסק. אם החלון סגור — נופל ל-template.
    גם שולח email גיבוי תמיד.
    """
    cfg = db.get_bot_config(user_id)
    if not cfg or not cfg["fallback_value"]:
        logger.error(f"User {user_id} has no fallback_value (owner phone)")
        return

    owner_phone = cfg["fallback_value"]
    line = db.get_whatsapp_line_for_user(user_id)  # מ-5.0

    try:
        send_whatsapp_text(
            from_line_id=line["phone_number_id"],
            to=owner_phone,
            text=free_message,
        )
        logger.info(f"Sent free message to owner {owner_phone}")
    except WhatsAppOutsideWindowError:
        # חלון סגור — fallback ל-template
        send_whatsapp_template(
            from_line_id=line["phone_number_id"],
            to=owner_phone,
            template=template_name,
            params=template_params,
        )
        logger.info(f"Sent template {template_name} to owner {owner_phone}")
    except Exception as e:
        logger.error(f"Failed to send to owner: {e}")

    # גיבוי email
    send_email_to_owner(user_id, subject=template_name, body=free_message)
```

**איך לזהות `WhatsAppOutsideWindowError`?** מטא מחזירה error code 131047: "Re-engagement message". יוצרים exception ספציפי בעטיפת ה-API:

```python
class WhatsAppOutsideWindowError(Exception):
    """Error 131047 — חלון 24 שעות סגור, חייב template."""
    pass


def send_whatsapp_text(from_line_id, to, text):
    response = whatsapp_api.send_text(...)
    if response.get("error", {}).get("code") == 131047:
        raise WhatsAppOutsideWindowError(response["error"]["message"])
    if not response.get("messages"):
        raise WhatsAppApiError(response.get("error", {}))
```

### 9.2 — onboarding: פתיחת חלון בפעם הראשונה

חלק קריטי של ה-UX. בעל העסק צריך לשלוח הודעה ראשונה לבוט כדי לפתוח את החלון.

**במסך הגדרת הבוט בפאנל** (`/settings/bot/calendar` או `/settings/bot/notifications`):

מציגים את המספר של הבוט (מ-`whatsapp_lines.phone_number_id` של ה-user) + הוראה ברורה:

> **כדי שתקבל התראות על תורים חדשים**
>
> שלח הודעה כלשהי ל-WhatsApp הבא:
>
> 📱 +972XXXXXXXXX (לחץ לפתוח ב-WhatsApp)
>
> מספיק "היי" — זה יפתח את החלון להתראות.

הקישור הוא `https://wa.me/972XXXXXXXXX?text=היי` — פותח WhatsApp ישירות עם הודעה מוכנה.

### 9.3 — ה-webhook מזהה הודעת onboarding

ב-webhook מ-5.2, כשמגיעה הודעה — האלגוריתם לזיהוי השולח:

```python
def webhook_handler(payload):
    incoming_phone = payload["from"]
    line_phone_number_id = payload["phone_number_id"]

    # מציאת ה-user המקושר ל-line הזה
    line = db.get_whatsapp_line_by_phone_number_id(line_phone_number_id)
    user_id = line["user_id"]

    cfg = db.get_bot_config(user_id)

    # האם השולח הוא בעל העסק?
    if cfg and _normalize_phone(incoming_phone) == _normalize_phone(cfg["fallback_value"]):
        # זה owner. שולחים ack ידידותי + לא ממשיכים לזרימת הליד.
        send_whatsapp_text(
            from_line_id=line_phone_number_id,
            to=incoming_phone,
            text=(
                "✅ ההתראות פעילות.\n"
                "מכאן והלאה תקבל הודעות על תורים חדשים, ביטולים, ועדכוני מערכת.\n"
                "כדי לאשר/לדחות תורים — היכנס לפאנל: " + settings.PANEL_URL + "/appointments"
            ),
        )
        return

    # אחרת — זרימת ליד רגילה (5.2)
    handle_lead_message(...)
```

**הערה**: ה-`_normalize_phone` חייב להתמודד עם פורמטים שונים — `+972501234567`, `0501234567`, `972501234567`. תקנן הכל ל-E.164 (`+972501234567`) לפני השוואה.

### 9.4 — שליחת notification על pending appointment

```python
def notify_owner_pending_job(appointment_id: UUID):
    appt = db.get_appointment(appointment_id)
    user_id = appt["user_id"]
    cfg = db.get_bot_config(user_id)

    business_name = cfg.get("business_name", "")
    lead_name = appt["lead_name"] or "ליד"
    lead_phone = appt["lead_phone"]
    date_display = _format_date_hebrew(appt["preferred_date"])
    time_display = appt["preferred_time"].strftime("%H:%M")
    panel_url = f"{settings.PANEL_URL}/appointments"

    free_message = (
        f"📅 בקשת תור חדשה — #{str(appointment_id)[:8]}\n\n"
        f"לקוח: {lead_name}\n"
        f"טלפון: {lead_phone}\n"
        f"תאריך: {date_display}\n"
        f"שעה: {time_display}\n\n"
        f"לאישור/דחייה: {panel_url}"
    )

    template_params = [business_name, lead_name, date_display, time_display, panel_url]

    send_to_owner(
        user_id=user_id,
        free_message=free_message,
        template_name="bot_pending_appointment_v1",
        template_params=template_params,
    )
```

ועוד jobs דומים: `notify_owner_cancellation_job` משתמש ב-template `bot_appointment_cancelled_owner_v1`.

---

## חלק 10 — WhatsApp Templates

4 templates חדשים. כולם UTILITY (לא MARKETING). שווה לקרוא את section 8 של מסמך 5.4 לאופן ההגשה למטא.

### 10.1 — `bot_pending_appointment_v1` (לבעל העסק)

```
Category: UTILITY
Language: he (Hebrew)
Header: (none)
Body:
  📅 בקשת תור חדשה {{1}}

  לקוח: {{2}}
  תאריך: {{3}}
  שעה: {{4}}

  לאישור/דחייה היכנס לפאנל: {{5}}
Footer: (none)
Buttons: (none)
```

**משתנים:**
- `{{1}}` — שם העסק
- `{{2}}` — שם הליד
- `{{3}}` — תאריך (פורמט עברי, "ראשון 09/06/2026")
- `{{4}}` — שעה (HH:MM)
- `{{5}}` — URL מלא לפאנל התורים

**Justification למטא:**

> Sent to business owners (registered SaaS users) when a screened lead requests an appointment time. Pure utility — alerts the owner to take action in their admin panel. The business owner has explicitly opted in by registering with Campaign AI and configuring the bot to accept appointments.

### 10.2 — `bot_appointment_cancelled_owner_v1` (לבעל העסק)

```
Body:
  ❌ ביטול תור {{1}}

  הלקוח {{2}} ביטל את התור שנקבע ל-{{3}} בשעה {{4}}.
  השעה התפנתה אוטומטית.

  פרטים: {{5}}
```

**משתנים:** business_name, lead_name, date_display, time_display, panel_url

### 10.3 — `bot_appointment_confirmation_v1` (לליד)

```
Body:
  ✅ התור שלך אושר ב-{{1}}

  📅 תאריך: {{2}}
  🕐 שעה: {{3}}

  הוסף ליומן: {{4}}

  לביטול — השב להודעה זו עם "לבטל"
```

**משתנים:** business_name, date_display, time_display, ics_url

**הערה ל-`ics_url`**: יצירת endpoint `GET /api/v1/appointments/:id/ics` שמחזיר קובץ `.ics` עם `Content-Disposition: attachment`. תפורט בחלק 12.

### 10.4 — `bot_appointment_rejected_v1` (לליד)

```
Body:
  לצערנו, התור שביקשת ב-{{1}} ל-{{2}} בשעה {{3}} לא יכול להתקבל.

  {{4}}

  כדי לתאם זמן אחר, השב להודעה עם "תיאום חדש".
```

**משתנים:** business_name, date_display, time_display, rejection_reason_or_default

ל-`{{4}}` — אם בעל העסק כתב סיבה (`appointments.rejection_reason`), משתמשים בה. אחרת ברירת מחדל: "אנא נסה זמן אחר או צור קשר ישיר עם בעל העסק."

### 10.5 — קובץ ההגשה

עדכון של `docs/deployment/meta-templates-submission.md` (מ-5.4) — להוסיף את ארבעת ה-templates עם:
- Body מלא
- רשימת variables עם הסבר לכל אחד
- Justification פסקה
- דוגמה מלאה ממולאת (מטא דורשת)

---

## חלק 11 — Panel UI for Appointments

מסך חדש בפאנל: `/appointments`.

### 11.1 — רשימה

API endpoint:
```
GET /api/v1/appointments?status=&from=&to=&page=
  → רשימה paginated.
  → ברירת מחדל: כל ה-active (pending/confirmed) מהיום והלאה.
  → filter: status, date range
```

**UI:**

טבלה / כרטיסים. כל שורה:
- סטטוס (pending/confirmed/cancelled/rejected) — badge צבעוני
- שם הליד
- טלפון (לחיץ → WhatsApp או טלפון)
- תאריך + שעה
- כפתורי פעולה (לפי status):
  - pending → "אשר" + "דחה"
  - confirmed → "בטל" (אישור בנפרד)
  - cancelled/rejected → (אין פעולות)

מסכי placeholder: אם אין appointments — "אין תורים פעילים. הבוט יוצר תורים כשלידים מסיימים סינון."

### 11.2 — פעולות

```
POST /api/v1/appointments/:id/approve
  Body: { "duration_minutes": int (אופציונלי, ברירת מחדל = current confirmed_duration_minutes) }
  Effects:
    - UPDATE appointments SET status='confirmed', confirmed_at=NOW()
    - INSERT event ב-GCal (אם יומן עדיין מחובר)
    - שמירת google_event_id
    - enqueue send_lead_confirmation_job

POST /api/v1/appointments/:id/reject
  Body: { "reason": str (אופציונלי) }
  Effects:
    - UPDATE appointments SET status='rejected', rejection_reason=?
    - enqueue send_lead_rejection_job

POST /api/v1/appointments/:id/cancel
  Body: { "reason": str (אופציונלי) }
  Effects:
    - UPDATE appointments SET status='cancelled', cancelled_by='owner', ...
    - DELETE event ב-GCal (אם google_event_id)
    - enqueue notify_lead_cancellation_job (template חדש?)
```

**הערה ל-cancellation על ידי בעל העסק**: זה דורש template חדש לליד (`bot_appointment_cancelled_by_owner_v1`)? או שאפשר להשתמש ב-`bot_appointment_rejected_v1` עם variation בטקסט? ההמלצה: **לדחות את ה-template הזה לעדכון**. ב-MVP, אם בעל העסק רוצה לבטל confirmed appointment, נשלח לליד ב-`bot_appointment_rejected_v1` עם reason='בעל העסק נאלץ לבטל. אנא צור קשר לתיאום חדש.' זה דחיק לא אידאלי, אבל חוסך template. אם גיא לא מקבל את זה — נוסיף.

### 11.3 — Badge מצב היומן

בסיידבר או בראש הדף — אינדיקציה ויזואלית:
- 🟢 "Google Calendar מחובר" — אם is_connected
- 🔴 "Google Calendar מנותק — לחץ לחיבור" — אם not is_connected
- 🟡 "בעיה בחיבור היומן — נסה לחבר מחדש" — אם auth_invalid_at

הלינק על ה-badge מוביל ל-`/settings/bot/calendar`.

---

## חלק 12 — Calendar Sync

### 12.1 — `create_calendar_event`

נקרא אחרי `POST /appointments/:id/approve`.

```python
def create_calendar_event(appointment_id: UUID) -> str | None:
    """
    יוצר אירוע ב-GCal של בעל העסק.
    מחזיר google_event_id.
    אם יומן מנותק — return None, ה-appointment עדיין confirmed.
    """
    appt = db.get_appointment(appointment_id)
    if not is_connected(appt["user_id"]):
        logger.warning(f"Calendar not connected, skipping sync for {appointment_id}")
        return None

    creds = get_credentials(appt["user_id"])
    if not creds:
        return None

    cred_data = db.get_google_calendar_credentials(appt["user_id"])
    calendar_id = cred_data["calendar_id"]

    start_dt = datetime.combine(appt["preferred_date"], appt["preferred_time"]).replace(tzinfo=IL_TZ)
    end_dt = start_dt + timedelta(minutes=appt["confirmed_duration_minutes"])

    cfg = db.get_bot_config(appt["user_id"])
    business_name = cfg.get("business_name", "Campaign AI")

    body = {
        "summary": f"תור — {appt['lead_name'] or 'ליד'}",
        "description": (
            f"לקוח: {appt['lead_name']}\n"
            f"טלפון: {appt['lead_phone']}\n"
            f"מקור: {business_name} (Campaign AI)\n"
            f"appointment_id: {appointment_id}"
        ),
        "start": {"dateTime": start_dt.isoformat(), "timeZone": "Asia/Jerusalem"},
        "end":   {"dateTime": end_dt.isoformat(), "timeZone": "Asia/Jerusalem"},
        "extendedProperties": {
            "private": {
                "campaign_ai_appointment_id": str(appointment_id),
            }
        },
    }

    service = build("calendar", "v3", credentials=creds, cache_discovery=False)
    event = service.events().insert(calendarId=calendar_id, body=body).execute()

    google_event_id = event["id"]
    db.set_appointment_google_event_id(appointment_id, google_event_id)
    return google_event_id


def delete_calendar_event(user_id: UUID, google_event_id: str):
    """מטפל ב-410 Gone כהצלחה (event כבר נמחק)."""
    if not is_connected(user_id):
        return
    creds = get_credentials(user_id)
    if not creds:
        return

    cred_data = db.get_google_calendar_credentials(user_id)
    service = build("calendar", "v3", credentials=creds, cache_discovery=False)

    try:
        service.events().delete(calendarId=cred_data["calendar_id"], eventId=google_event_id).execute()
    except HttpError as e:
        if e.resp.status == 410:
            # כבר נמחק — לא שגיאה
            return
        raise
```

**הערות:**

- **`extendedProperties.private`** מאפשר לקשר את האירוע חזרה ל-appointment שלנו. שימושי אם נצטרך לבצע sync דו-כיווני בעתיד.
- **fail silent כש-not connected**: ה-DB עדיין מסונכרן. בעל העסק יכול לראות appointment ב-confirmed גם אם ה-event לא נוצר. כשיחבר מחדש את היומן, נצטרך אופציה ל-back-sync (לא ב-MVP).

### 12.2 — ICS endpoint

```
GET /api/v1/appointments/:id/ics
  Auth: לא נדרש (URL מוכל ב-template, נשלח לליד).
  הגנה: ה-URL מכיל את appointment_id (UUID) — מספיק entropy למניעת guessing.

  Response:
    Content-Type: text/calendar
    Content-Disposition: attachment; filename="appointment-{short_id}.ics"
    Body:
      BEGIN:VCALENDAR
      VERSION:2.0
      PRODID:-//Campaign AI//appointment//HE
      BEGIN:VEVENT
      UID:{appointment_id}@campaign-ai.com
      DTSTAMP:{now in UTC}
      DTSTART:{start in UTC}
      DTEND:{end in UTC}
      SUMMARY:תור ב-{business_name}
      DESCRIPTION:תור עם {business_name}\\nטלפון: {owner_phone}
      LOCATION:{empty in MVP — no address field}
      END:VEVENT
      END:VCALENDAR
```

**הגנת privacy**: בודקים ש-`appointments.status IN ('confirmed','pending')` לפני החזרה. אם cancelled/rejected/passed → 404. אסור שליד ידע על תור שבוטל דרך ה-ICS URL הישן.

---

## חלק 13 — Bot Config Validation Update

Patch לקובץ ה-config service מ-5.1.

### תלות ב-`fallback_value`:

```python
def validate_bot_config(user_id: UUID, payload: dict) -> ValidationResult:
    # ... ולידציות קיימות מ-5.1 ...

    if payload["fallback_action"] == "bot_schedule_appointment":
        if not payload.get("fallback_value"):
            raise ValidationError(
                code="missing_owner_phone",
                message="נדרש מספר טלפון של בעל העסק לקבלת התראות על תורים",
                http_status=422,
            )
        if not _is_valid_e164_israeli(payload["fallback_value"]):
            raise ValidationError(
                code="invalid_phone_format",
                message="הטלפון חייב להיות בפורמט E.164 ישראלי (למשל +972501234567)",
                http_status=422,
            )
        if not google_calendar_service.is_connected(user_id):
            raise ValidationError(
                code="calendar_not_connected",
                message="יש לחבר את יומן Google לפני בחירת תיאום פגישות",
                http_status=412,
                extra={"action_required": "connect_calendar", "connect_url": "/settings/bot/calendar"},
            )
```

**412 Precondition Failed**: זה ה-status code הנכון לכאן. הבקשה עצמה תקינה (`bot_schedule_appointment` הוא ערך לגיטימי), אבל precondition של המערכת (יומן מחובר) לא מתקיים. ה-UI יודע להציג מסך "חבר יומן" לפי הקוד הזה.

---

## חלק 14 — Edge Cases & Error Handling

### 14.1 — Disconnect של יומן באמצע pending appointments

תרחיש: בעל העסק חיבר יומן → 5 לידים תיאמו תורים pending → בעל העסק disconnect.

**מה קורה:**
- Bot ממשיך לעבוד (busy ranges מה-DB מגנים)
- אם בעל העסק מאשר את ה-pending — `create_calendar_event` עושה fail silent (יומן מנותק). status עובר ל-confirmed ב-DB, אבל אין event ב-GCal.
- כשיתחבר שוב — האירועים לא נוצרים אוטומטית. צריך job ידני (ב-MVP — לא נטפל, מתעדים ב-known issues).

**UI flagging**: כשבעל העסק לוחץ disconnect, הצג modal:
> "יש לך X תורים פעילים. הניתוק יפסיק סנכרון ליומן Google אבל לא יבטל את התורים. להמשיך?"

### 14.2 — Refresh token נשלל באמצע flow

תרחיש: ליד באמצע scheduling, ה-bot מנסה לקרוא `get_available_slots` → ה-token פג → `RefreshError` → `auth_invalid_at` מסומן.

**טיפול:**
- `get_available_slots` תופס `CalendarUnavailable` ומחזיר רשימה ריקה.
- ה-flow ב-`send_time_picker_job` מזהה ריקה → שולח לליד "לצערי לא ניתן לתאם כרגע. בעל העסק יצור איתך קשר."
- `_notify_owner_calendar_disconnected` רץ (פעם אחת).
- ה-conversation מועברת ל-`completed_handoff`.

### 14.3 — Owner phone זהה ל-lead phone

תרחיש קצה — בעל העסק הוסיף את הטלפון שלו עצמו כליד בקמפיין (לבדיקה). ה-webhook מקבל הודעה מהמספר → מזוהה כ-owner → לא ממשיך לזרימת ליד. ה-בדיקה של הלקוח חוזרת ריקה.

**טיפול ב-MVP**: ההתנהגות הזו מקובלת. בעל העסק יראה במספר שלו את ההודעה "התראות פעילות" במקום שיחה רגילה. לא bug חמור. תיעוד ב-FAQ.

### 14.4 — ליד מנסה לבטל אחרי שהשעה עברה

תרחיש: תור היה אתמול ב-10:00. הליד היום שולח "לבטל".

**טיפול:**
- `classify_intent` מזהה cancel_appointment
- `get_active_appointment_for_conversation` מחזיר רק תורים `pending`/`confirmed` עם תאריך עתידי
- אם אין → "השיחה הסתיימה. לפרטים נוספים..."
- אופציה משופרת: לוגיקה שמסמנת תורים שעברו כ-`passed` (job יומי או lazy)

### 14.5 — Race condition בין שני לידים על אותו slot

תרחיש: ליד A ו-ליד B שניהם מקבלים את 10:00 כ-available. שניהם לוחצים confirm ב-200ms הפרש.

**טיפול:**
- שניהם עוברים `gather_and_decide` עם `has_active_conflict=false` (חלון הזמן הקצר)
- שניהם מנסים `db.create_appointment` → `UniqueViolation` מה-DB constraint על השני
- השני מקבל error → שולח לליד "השעה נתפסה לפני שניות, בחר אחרת" + חזרה ל-date picker
- ה-UNIQUE INDEX החלקי הוא ההגנה הסופית

### 14.6 — Google API rate limit

תרחיש: כל הלקוחות שלנו פתאום מקבלים burst של לידים → רבים מקבלים `429 Too Many Requests` מ-FreeBusy.

**טיפול:**
- `get_available_slots` מחזיר רשימה ריקה ב-Exception
- הליד מקבל "אין זמינות כרגע, בעל העסק יצור איתך קשר"
- monitoring/Sentry על rate ה-429 כדי לדעת אם צריך לעבור לתכנית בתשלום של Google Calendar API

ב-MVP: לא נטפל פרואקטיבית. ב-prod: נוסיף retry עם backoff (קל להוסיף עם `tenacity`).

---

## חלק 15 — Testing Strategy

### 15.1 — Unit tests (פונקציות טהורות)

עיקריות:

```
tests/unit/test_booking_decision.py
  - test_slot_in_past
  - test_slot_too_far_ahead
  - test_active_conflict
  - test_manual_returns_pending
  - test_auto_with_check_calendar_not_connected → pending
  - test_auto_with_check_calendar_busy → rejected
  - test_auto_always_vacation → rejected
  - ... 12 cases בסך הכל

tests/unit/test_slot_availability.py
  - test_closed_day_returns_empty
  - test_overlapping_busy_filters_slot
  - test_db_appointments_filter_slots
  - test_buffer_between_appointments
  - test_today_rounds_to_next_slot
  - test_freebusy_failure_returns_empty
  - test_overlap_formula  # start < be AND end > bs

tests/unit/test_working_hours.py
  - test_python_to_israeli_weekday_conversion
  - test_special_day_overrides_weekly
  - test_closed_special_day_overrides_open_weekly
```

### 15.2 — Integration tests (עם DB)

```
tests/integration/test_appointment_creation.py
  - test_create_appointment_success
  - test_double_booking_rejected_by_unique_index
  - test_rebook_cancelled_slot_works  (cancelled לא חוסם)
  - test_uniqueness_per_user  (user A יכול לקבוע במקביל ל-user B)

tests/integration/test_cancellation.py
  - test_cancel_releases_slot
  - test_cancel_deletes_calendar_event
  - test_cancel_410_handled_gracefully
  - test_double_cancel_idempotent

tests/integration/test_oauth_flow.py
  - test_state_validates_user_id
  - test_expired_state_rejected
  - test_tampered_state_rejected
```

### 15.3 — E2E manual tests (בסביבת test של מטא + dev Google account)

- Lead מתחיל סינון → מסיים → מקבל date picker
- בחירת תאריך → time picker
- בחירת שעה → confirmation
- אישור → notification ל-owner phone (בודקים שמגיע)
- Owner מאשר בפאנל → confirmation ל-lead + ICS link
- Lead לוחץ ICS → נפתח ב-Calendar app שלו
- Lead שולח "לבטל" → quick reply → אישור → התור בוטל
- Disconnect של יומן באמצע → התנהגות חינניה

---

## חלק 16 — Deployment & Environment Variables

### 16.1 — ENV vars חדשים ב-Render

```bash
# Google OAuth (חדש)
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx
GOOGLE_REDIRECT_URI=https://api.campaign-ai.com/api/v1/google/callback

# הצפנת tokens (חדש, אם לא קיים מ-3.4)
SECRETS_ENCRYPTION_KEY=<32 bytes base64>
SECRETS_ENCRYPTION_KEY_VERSION=v1

# Panel URL (קיים, מתועד כאן לשלמות)
PANEL_URL=https://app.campaign-ai.com
```

### 16.2 — תיעוד תפעולי

קובץ חדש: `docs/deployment/google-calendar-setup.md` — מתעד את ה-flow ב-Google Cloud Console (חלק 2.1 למעלה).

### 16.3 — Soft-disable flag

הוספה (אופציונלי) ל-`app_settings`:
```sql
ALTER TABLE app_settings ADD COLUMN IF NOT EXISTS appointment_scheduling_enabled boolean DEFAULT false;
```

אם false → ה-validator של bot_config דוחה `bot_schedule_appointment` עם הודעה "מתאם תורים אינו זמין כרגע. נסה שוב בקרוב." בלי לקרוס. מעודן לבדיקות הדרגתיות.

ההמלצה: **כן להוסיף**. מעניק שליטה תפעולית אם משהו ב-prod מתפוצץ — אפשר להחזיר את ה-flag ל-false במקום rollback של deploy.

---

## סדר מימוש מומלץ ב-CC

```
Step 1 — Migrations (חלק 1)
  0027 + 0028 + 0029 + 0030 + 0031 + ALTER bot_conversations.scheduling_data

Step 2 — Patches ל-5.1/5.2/5.3/5.4
  ulater ה-changes שבטל את 422, מוסיף intent classification, מוסיף 4 templates למטא docs.
  כל patch הוא קטן (~50-150 שורות).

Step 3 — Google Calendar Service (חלק 2)
  קובץ google_calendar_service.py + crypto.py + API endpoints (/google/*)
  Panel UI לחיבור (חלק 2.6) — אפשר later parallel.

Step 4 — Working Hours + Special Days (חלק 3)
  endpoints + UI (פאנל הגדרות)

Step 5 — Slot Availability (חלק 4)
  slot_availability_service.py

Step 6 — Decision Engine (חלק 5)
  booking_decision.py + tests

Step 7 — Booking Flow Jobs (חלק 7)
  send_date_picker_job, send_time_picker_job, send_confirmation_request_job
  handle_scheduling_*_input handlers (מ-webhook)

Step 8 — Cancellation Flow (חלק 8)
  intent classifier + cancellation jobs

Step 9 — Owner Notifications (חלק 9)
  send_to_owner with try-then-fallback
  webhook handling for owner phone

Step 10 — Calendar Sync (חלק 12)
  create_calendar_event + delete_calendar_event
  ICS endpoint

Step 11 — Panel UI for Appointments (חלק 11)
  /appointments page + approve/reject/cancel APIs

Step 12 — Edge Cases + Testing (חלק 14-15)

Step 13 — Deployment (חלק 16)
  ENV vars + soft-disable flag + docs
```

זה גם הסדר הלוגי של ה-PRs.  כל step עומד בפני עצמו (כמעט) — אפשר ל-merge בלי לחכות לבא. החריגים: step 5 דורש step 1, step 6 דורש step 5, step 7 דורש 5+6.

---

## דברים שצריך לאישור גיא לפני התחלת מימוש

| נושא | סטטוס | חוסם? |
|---|---|---|
| ניסוחי 4 templates חדשים (חלק 10) | נדרש אישור גיא | חוסם הגשה למטא, לא חוסם מימוש |
| Meta App Review status — האם להגיש את 4 templates יחד עם 3 מ-5.4? | שאלה לגיא | תפעולי |
| Cancellation by owner (אם confirmed) — האם מקובל לדחוק `bot_appointment_rejected_v1` ולא טמפלייט נפרד? | החלטת UX | לא חוסם, ניתן לעדכן בעדכון |
| Domain ל-`/api/v1/google/callback` — האם `api.campaign-ai.com` סופי? | תפעולי | חוסם setup ב-Google Cloud Console |
| Privacy policy URL — נדרש ל-Google App verification | חוסר | חוסם submission |

---

## דברים שצריך לאישור אמיר (החלטות טכניות שאני המלצתי)

| נושא | המלצה | אישור? |
|---|---|---|
| ICS endpoint ללא auth (UUID is access) | אישור | ✓ ננעל |
| Special days עם UPSERT לפי (user_id, date) | אישור | ✓ |
| משך תור אחיד לכל user (default_appointment_duration_minutes) | אישור | ✓ |
| Soft-disable flag `appointment_scheduling_enabled` | אישור | טווח של "כן/לא, אבל לא חוסם" |

---

**סוף Session 5.5 roadmap.**

קיבולת מימוש משוערת ב-CC: שבוע עד עשרה ימים, אם CC עובד בצורה רציפה ואין block של גיא על templates.

---

## Phase 6 · token refresh (תחזוקה)

### Session 6.1 — cron לרענון טוקן Meta ✅
- [x] job שמרענן טוקן ב-50% מחייו, מעדכן `refresh_failed_at`/`refresh_error`

**Done ✅:** טוקן מתרענן אוטומטית; קמפיין לא "מת" ביום 61. מומש ב-migration **0047** (לא 0030 — היה תפוס):
`upsert_fb_connection` +reset · RPC `refresh_fb_token` (אטומי Vault+DB) · RPC `due_token_refresh_user_ids` ·
`jobs.type`. בקוד: `meta.refresh_long_lived_token` (delegation ל-`exchange_long_lived_token`) ·
`fb_service.refresh_token_for_user`/`get_connection_status`/`enqueue_due_token_refreshes` ·
`mark_token_expired(reason=)` · handler `refresh_meta_token` (3 idempotency checks + error mapping) · tick יומי
ב-`runner.py`. סטיות מהתכנון שתוקנו: `get_decrypted_token` (לא get_user_access_token), `enqueue(job_type=)`,
exceptions עם `.code` (לא `.message`).

# Session 6.1 — cron לרענון טוקן Meta

> בלוק להוספה ל-ROADMAP.md תחת `## Phase 6 · token refresh (תחזוקה)`. תכנון מלא להעברה ל-CC.

---

## מבוא ותיאור Session

טוקן ה-Meta של הלקוח שנשמר ב-`fb_connections` (מ-1.2) הוא long-lived — תקף כ-60 יום מיום החיבור. בלי רענון אקטיבי, ביום 61 כל קריאה ל-Meta API תיכשל עם code 190, והקמפיין של הלקוח "מת בשקט" — Insights (4.5) לא מחזיר נתונים, הסוכן (Phase 7) לא יכול לצטט מספרים, והפעולות האוטונומיות (Phase 8) נכשלות. 6.1 הוא ה-baseline שמונע את המצב הזה.

הפתרון: cron יומי שמרענן טוקנים שזמן פוגה שלהם בתוך 30 יום (50% מחיי הטוקן, לפי `spec.md` §2א). הרענון עצמו הוא אותה קריאה בדיוק כמו `exchange_long_lived_token` שכבר קיימת ב-`integrations/meta.py:261` — מעבירים את הטוקן הנוכחי כ-`fb_exchange_token` ומקבלים טוקן חדש תקף 60 יום נוספים.

**מה ב-Session:**
- patch ל-`upsert_fb_connection` שמאפס `refresh_failed_at`/`refresh_error` בחיבור מחדש (תיקון באג קיים).
- RPC חדש `refresh_fb_token` שמעדכן Vault + DB באטומיות.
- handler חדש `refresh_meta_token` בתשתית jobs (3.0).
- cron tick יומי ב-`worker/runner.py` שמייצר jobs פר חיבור שמתקרב לפוגה.
- הרחבה זעירה של `mark_token_expired` לקבל `reason` אופציונלי.

**מה לא ב-Session:**
- notification אקטיבי ללקוח (email/WhatsApp) כשטוקן נפסל — אין תשתית email ב-MVP. ה-UX הקיים מ-3.4 (409 עם `fb_token_invalid` → "חבר מחדש" כשהלקוח נוגע בפעולה הבאה) הוא הערוץ.
- היסטוריית refresh attempts כטבלה — `updated_at` ו-Sentry מספיקים לדיבוג.
- מערכת `scheduler_state` גנרית עם persistence — YAGNI ב-MVP. monotonic tick in-memory מספיק (אותו דפוס כמו ה-cleanup tick של 3.4c).
- חיווט של crons אחרים (חיובים יומיים) — מחוץ ל-scope (ראה הערה בסוף).
- proactive refresh ב-50% מהחיים אבל גם בכל בקשה לקריאה ל-Meta — לא נדרש. ה-cron הוא ה-mechanism הבלעדי.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מתי לרענן? | טוקן שזמן פוגה שלו ≤ `now() + 30 days`, וגם `refresh_failed_at IS NULL` |
| 2 | תדירות ה-cron tick? | **24 שעות** (יש 30 יום buffer) |
| 3 | persistence של ה-tick? | **monotonic in-memory** כמו `last_cleanup` ב-3.4c. ללא טבלה |
| 4 | מקור האמת לסטטוס "טוקן פג"? | `refresh_failed_at IS NOT NULL`. **לא** שדה boolean נפרד |
| 5 | Vault + DB update — איך באטומיות? | **RPC חדש** שעוטף את שני העדכונים ב-transaction אחת |
| 6 | פונקציית רענון ב-`meta.py` — חדש או reuse? | reuse של `exchange_long_lived_token` + alias דק לבהירות |
| 7 | enqueue dedup — איך? | **לא נדרש** ב-MVP. ה-handler idempotent → שני jobs כפולים מתכנסים |
| 8 | retry policy ב-jobs? | ברירת מחדל של 3.0 — `{1: 60, 2: 300}`, `max_attempts=3`. כלומר 2 retries אז כשל סופי |
| 9 | notification ללקוח בכשל permanent? | **לא ב-MVP**. רק `mark_token_expired`, ה-UX של 3.4 מטפל |

---

## תלויות (מאומתות מול הקוד)

1. **Phase 1.2 — `fb_connections`.** קיימת ב-`supabase/migrations/0002_fb_connections.sql:13-24` עם השדות הנכונים. `encrypted_token` הוא `uuid` (reference ל-Vault), `refresh_failed_at` ו-`refresh_error` כבר nullable ומיועדים ל-Phase 6 (ההערה בשורה 11 קובעת זאת מפורשות).

2. **Phase 3.0 — jobs queue.** `enqueue` תומך ב-`user_id=None` (job מערכת). ה-runner עושה retry אוטומטי לפי `{1: 60, 2: 300}`, `max_attempts=3` ברירת מחדל. ה-`claim_next_job` עם `FOR UPDATE SKIP LOCKED` מבטיח שאם בעתיד יהיו כמה workers — אין double-execution.

3. **Phase 3.4 — `classify_meta_error` + `mark_token_expired`.** המסווג ב-`integrations/meta.py:159` מחזיר `MetaErrorKind.TRANSIENT/PERMANENT/UNKNOWN`. השגיאות `MetaTransientError/MetaPermanentError/MetaUnexpectedError` נושאות `.code` ו-`.message`. `mark_token_expired(user_id)` קיים ב-`fb_service.py:226` וכותב ישירות ל-`refresh_failed_at` + `refresh_error`. **בלבד שהחתימה צריכה להתרחב** עם `reason` אופציונלי.

4. **`exchange_long_lived_token` ב-`integrations/meta.py:261`.** עושה בדיוק `grant_type=fb_exchange_token` ומחזיר `(token, expires_in)`. רענון של long-lived token הוא **אותה קריאה בדיוק** — מעבירים את הטוקן הנוכחי כ-`fb_exchange_token`. אפשר reuse ישיר.

---

## חלק 1 — Patch ל-1.2 (קריטי, חוסם)

**הבעיה:** `upsert_fb_connection` ב-`0002:81-85` עושה `ON CONFLICT DO UPDATE` שמעדכן רק `meta_user_id, encrypted_token, token_expires_at, updated_at` — **לא נוגע ב-`refresh_failed_at`/`refresh_error`**. לקוח שטוקנו נכשל permanent, סומן כפג ב-3.4, ואז מתחבר מחדש דרך OAuth — נשאר עם `refresh_failed_at` ישן. ה-cron query של 6.1 (`WHERE refresh_failed_at IS NULL`) לעולם לא יתפוס אותו שוב, והטוקן החדש שלו ימות בשקט ביום 61.

**הפתרון:** migration חדש `0030_fb_token_refresh.sql` שמכיל `create or replace function upsert_fb_connection` עם אותה חתימה, אותה לוגיקה, ושינוי יחיד — הוספה ל-set-list של ה-`ON CONFLICT DO UPDATE` של `refresh_failed_at = null` ו-`refresh_error = null`. הסמנטיקה: "התחברות חדשה מאפסת כל סימן של כשל קודם."

**חשוב:** אסור לערוך את `0002`. `app/db/migrate.py:42-44,72-77` עוקב אחרי `schema_migrations` ומדלג על מה שכבר הוחל. עריכת `0002` לא תרוץ שוב בפרודקשן. `create or replace function` ב-migration חדש הוא הדרך הנכונה — היא דורסת את הגדרת הפונקציה הקיימת.

---

## חלק 2 — Migration 0030 (סיכום מבני)

קובץ אחד: `supabase/migrations/0030_fb_token_refresh.sql`. שני שינויים בלוגיקה:

**א. `create or replace function upsert_fb_connection`** — אותה חתימה כמו ב-0002, אותו `SECURITY DEFINER`, אותו `search_path=''`. ההבדל היחיד: ב-`ON CONFLICT (user_id) DO UPDATE`, מוסיפים ל-set-list `refresh_failed_at = null` ו-`refresh_error = null`. שאר השדות זהים.

**ב. RPC חדשה `refresh_fb_token(p_user_id uuid, p_access_token text, p_expires_at timestamptz)`.** `SECURITY DEFINER`, `search_path=''`. ה-RPC מבצע ב-transaction אחת (משתמע מהיותה פונקציה אחת ב-Postgres): 

1. שליפת ה-`encrypted_token` (ה-uuid של ה-secret ב-Vault) מ-`fb_connections` לפי `user_id`. אם לא קיים — `raise exception 'fb_connection_not_found'`.
2. `vault.update_secret(secret_id, p_access_token)` — דורסת את הערך המוצפן.
3. `UPDATE fb_connections SET token_expires_at = p_expires_at, refresh_failed_at = null, refresh_error = null, updated_at = now() WHERE user_id = p_user_id`.
4. החזרה: void או boolean להצלחה.

**אטומיות מובטחת** ע"י Postgres — אם שלב 2 או 3 נכשלים, כל הפונקציה rollback. אין mismatch אפשרי בין Vault ל-DB.

**GRANT EXECUTE לservice_role בלבד** (אותו דפוס כמו `upsert_fb_connection` ב-0002:79). הפונקציה לא נגישה ל-`authenticated` או `anon`.

**אזהרה:** **לא לעשות reuse ל-`upsert_fb_connection`** עבור רענון. הוא דורש `meta_user_id` (שלא משתנה ברענון), `encrypted_token` (uuid שלא משתנה), ויש לו סמנטיקה של "התחבר" — לא "רענן". פונקציה נפרדת = ביטוי נקי של intent.

---

## חלק 3 — שכבת `integrations/meta.py`

תוספת זעירה: alias דק לבהירות.

`exchange_long_lived_token(short_token: str) -> tuple[str, int]` כבר עושה את הקריאה הנכונה. אבל שם הפרמטר `short_token` מטעה כשמשתמשים בו לרענון של long-lived. הפתרון: פונקציה חדשה `refresh_long_lived_token(current_token: str) -> tuple[str, int]` שקוראת לאותו `_meta_get` עם אותם פרמטרים. החתימה זהה, השם משקף את ה-use case.

הסיווג של שגיאות (`classify_meta_error`) ממילא יתפוס את הכשלים. אין צורך להגדיר exceptions חדשים.

**זהירות:** `expires_in` של Meta הוא **שניות**, לא timestamp. ה-caller (`fb_service.refresh_token_for_user`) יחשב `new_expires_at = now() + timedelta(seconds=expires_in)`.

---

## חלק 4 — שכבת `fb_service`

שתי פונקציות חדשות + הרחבת `mark_token_expired`.

**א. `refresh_token_for_user(user_id: UUID) -> None`.** Orchestration. שלבים:

1. שליפת הטוקן הנוכחי דרך `get_user_access_token(user_id)` הקיים (מפענח Vault).
2. קריאה ל-`integrations/meta.refresh_long_lived_token(current_token)` — מחזיר `(new_token, expires_in_seconds)`.
3. חישוב `new_expires_at = now() + timedelta(seconds=expires_in)`.
4. קריאת ה-RPC: `supabase_admin.rpc('refresh_fb_token', {'p_user_id': str(user_id), 'p_access_token': new_token, 'p_expires_at': new_expires_at.isoformat()})`.
5. החזרה ללא ערך. במקרה של שגיאה — exception מועבר הלאה ל-caller (ה-handler).

**הפונקציה לא תופסת exceptions של `MetaTransientError`/`MetaPermanentError`** — הם מועברים ל-handler שיחליט מה לעשות. שכבת ה-service אחראית רק על תזמור הקריאות.

**ב. `update_user_token(user_id, new_token, new_expires_at)`** — לא נדרש כשכבת public נפרדת. ה-RPC הוא הממשק היחיד לעדכון אטומי. השאלה האם להוסיף אותו כwrapper תיאורי בlevel של Python — תלוי בסגנון של שאר ה-service. אם יש דפוס של wrappers על קריאות RPC ב-`fb_service` — להוסיף. אחרת — קריאה ישירה ל-RPC מ-`refresh_token_for_user` מספיקה.

**ג. הרחבת `mark_token_expired`.** החתימה הנוכחית `mark_token_expired(user_id: UUID)` עם `refresh_error="token_expired_during_push"` קבוע. החתימה החדשה: `mark_token_expired(user_id: UUID, reason: str = "token_expired_during_push")`. ה-default שומר backward compatibility עם הקריאה מ-3.4. 6.1 יקרא עם `reason=f"refresh_failed_permanent: code {error_code}"` או דומה (השם המדויק יבחר בזמן המימוש לפי מה ש-`classify_meta_error` מחזיר).

---

## חלק 5 — Handler `refresh_meta_token`

ב-`worker/handlers.py`, נוסף ל-`HANDLERS` dict.

**Type ב-`jobs.type` enum:** `refresh_meta_token`. צריך להוסיף ל-CHECK constraint של העמודה. **migration נפרד או חלק מ-0030?** ההמלצה: חלק מ-0030 — כל השינויים של 6.1 ב-migration אחד. ה-ALTER הוא `ALTER TABLE jobs DROP CONSTRAINT jobs_type_check; ALTER TABLE jobs ADD CONSTRAINT jobs_type_check CHECK (type IN ('test_echo', ..., 'refresh_meta_token'))`.

**Payload:** `{"user_id": "<uuid string>"}`. אין שדות נוספים.

**שלבי ה-handler:**

1. **Idempotency check ראשון.** שלוף `fb_connection` לפי `user_id` דרך admin client. אם לא קיים — log warning ו-return (לקוח מחק את חיבור הפייסבוק בזמן שה-job חיכה בתור; לא נכון להיכשל).

2. **Idempotency check שני.** אם `token_expires_at > now() + 30 days` — return בלי לעשות כלום. הטוקן כבר רוענן ע"י מקור אחר (job כפול, race עם cron tick נוסף, וכו'). זה ה-mechanism שמייתר dedup ב-enqueue.

3. **Idempotency check שלישי.** אם `refresh_failed_at IS NOT NULL` — return בלי לעשות כלום. הטוקן כבר סומן כפג בעבר. ה-cron query אמור לסנן אלה, אבל הגנת depth.

4. **ביצוע הרענון.** קריאה ל-`fb_service.refresh_token_for_user(user_id)`.

5. **טיפול בכשלים:**
   - `MetaTransientError` → raise. ה-runner של 3.0 יבצע retry לפי policy (`{1: 60, 2: 300}`). אם 2 retries נכשלו → ה-job יסומן `failed` ו-Sentry יקבל capture. ב-cron tick הבא (יום הבא), הטוקן עדיין יעמוד בקריטריון `token_expires_at <= now() + 30d AND refresh_failed_at IS NULL`, ויירה job חדש.
   - `MetaPermanentError` → קריאה ל-`fb_service.mark_token_expired(user_id, reason=f"refresh_failed_permanent: code {e.code}")` ו-return בלי raise. ה-job ייחשב מוצלח. הטוקן סומן, ה-UX של 3.4 יטפל בלקוח בפעם הבאה.
   - `MetaUnexpectedError` → raise. אותו דפוס כמו transient — נותנים ל-3.0 לעשות retry. אם נכשל סופית, Sentry יתפוס. בעתיד אם זה מצליף יותר מדי — אפשר להוסיף לוגיקה ספציפית.

**עיקרון:** ה-handler לא יודע מה זה "כשל סופי" — זה תפקיד ה-runner. ה-handler רק קובע: "transient → raise (אל תפסיק לנסות), permanent → mark + return (אל תנסה שוב)".

---

## חלק 6 — cron tick ב-`worker/runner.py`

תוספת מינימלית. אותו דפוס כמו `last_cleanup` ב-3.4c (`runner.py:184-206`).

**מבני:**

1. בתחילת ה-loop, להוסיף `last_token_refresh = 0.0` (monotonic, in-memory).
2. בכל iteration, אחרי ה-cleanup tick וה-job claim, לבדוק: `if time.monotonic() - last_token_refresh > 86400: ...`.
3. בתוך הבלוק:
   - שאילתה אינדקסית: `SELECT user_id FROM fb_connections WHERE (token_expires_at <= now() + interval '30 days' OR token_expires_at IS NULL) AND refresh_failed_at IS NULL`. (ה-`OR token_expires_at IS NULL` הוא הגנת אנומליות לפי תגלית 3 שלך — אם איכשהו השדה נשאר NULL, לא לדלג עליו בשקט.)
   - לכל row שחזרה: `jobs_service.enqueue(type='refresh_meta_token', payload={'user_id': str(user_id)}, user_id=None)`.
   - עדכון `last_token_refresh = time.monotonic()`.
4. כל זה עטוף ב-`try/except` רחב — אם השאילתה נכשלת, log + Sentry, אבל ה-runner לא נופל. הניסיון הבא בעוד 24 שעות.

**אינדקס:** הטבלה `fb_connections` כבר עם UNIQUE על `user_id`. השאילתה משתמשת ב-`token_expires_at` + `refresh_failed_at`. למרות שאלה אינדקסים partial אופציונלים — ב-MVP טבלה קטנה (עשרות עד מאות שורות), full scan חד-יומי לא ירגיש. אם בעתיד הטבלה תגדל ל-עשרות אלפים, להוסיף partial index `WHERE refresh_failed_at IS NULL`.

**Edge: worker restart בתוך 24 שעות.** המונה מתאפס, ה-tick יורה שוב. זה זול (ברוב הימים אין מה לרענן — query מחזיר 0 שורות), idempotent (handler יסנן double-enqueue), ולא מזיק. אם בעתיד יהיו deploys תכופים שגורמים לעומס שאילתות מיותר — אז להוסיף `scheduler_state` עם persistence. ב-MVP YAGNI.

---

## חלק 7 — Done של 6.1

- migration 0030 הוחל. `create or replace function upsert_fb_connection` הוסיף איפוס `refresh_failed_at`/`refresh_error`. RPC `refresh_fb_token` קיים עם GRANT EXECUTE לservice_role.
- `refresh_meta_token` נוסף ל-`jobs.type` CHECK constraint.
- handler `refresh_meta_token` רשום ב-`HANDLERS`.
- cron tick יומי בתוך `runner.py` יורה INSERT לתור עבור חיבורים שזמן פוגה שלהם ≤ 30 יום.
- בחיבור מחדש של לקוח (1.2 callback) — `refresh_failed_at` ו-`refresh_error` מתאפסים אוטומטית.
- בהצלחת רענון — Vault + DB מתעדכנים אטומית דרך ה-RPC. `token_expires_at` קופץ ל-+60 יום קדימה.
- בכשל transient — retry של 1m אז 5m, ואז כשל סופי. cron tick יורה job חדש בעוד 24 שעות (`refresh_failed_at` עדיין NULL).
- בכשל permanent — `refresh_failed_at` ו-`refresh_error` נשמרים. ה-cron query מסנן את החיבור הזה. ה-UX של 3.4 מציג ללקוח "חבר מחדש" בפעם הבאה.
- כל הטסטים החדשים עוברים.

**בדיקות מומלצות:**
1. seed של `fb_connection` עם `token_expires_at = now() + 20 days` ו-`refresh_failed_at = NULL` → ה-cron tick יוצר job → handler קורא ל-Meta (mock) → `token_expires_at` מתעדכן ל-now() + 60 days.
2. mock של Meta שמחזיר code 190 → handler קורא ל-`mark_token_expired` → `refresh_failed_at` נשמר → cron tick הבא **לא** יוצר job חדש לאותו user.
3. mock של Meta שמחזיר 503 → handler raises → runner עושה retry → אחרי 2 ניסיונות → job `failed` → cron הבא יורה job חדש.
4. UPSERT ב-`upsert_fb_connection` של user שיש לו `refresh_failed_at IS NOT NULL` → אחרי ה-UPSERT, `refresh_failed_at IS NULL`.
5. שני workers (תאורטי, ב-MVP יש אחד) → רק אחד תופס job (FOR UPDATE SKIP LOCKED מ-3.0).

---

## חלק 8 — לא ב-6.1

- **`scheduler_state` עם persistence.** monotonic in-memory מספיק. אם בעתיד נראה שdeploys תכופים יוצרים עומס מיותר → לפתוח Session נפרד.
- **partial UNIQUE index ל-dedup אטומי של jobs.** המודל הנוכחי (handler idempotency) מספיק. אם תהיה ראייה לכפילויות בעייתיות — להוסיף.
- **notification אקטיבי ללקוח בכשל permanent.** אין תשתית email ב-MVP. ה-UX של 3.4 הוא הערוץ.
- **טבלת `refresh_attempts` להיסטוריה.** `updated_at` ו-Sentry capture מספיקים לדיבוג. אם בעתיד נצטרך אנליטיקס ("כמה רענונים נכשלו השבוע?") — להוסיף.
- **לוגיקה ספציפית ל-`MetaUnexpectedError`.** ב-MVP מתייחסים אליו כ-transient (raise → retry). אם נצליף יותר מדי — לבחון.
- **תזמון cron ב-Render Cron Job חיצוני.** monotonic tick בתוך ה-worker מספיק. Render Cron Job הוא תוספת שירות (תשלום נוסף) שלא נחוצה.
- **חיווט crons אחרים** (`process_trial_expirations`, `process_monthly_charges` ב-`billing_cron.py`). תגלית 4 שלך — מחוץ ל-scope של 6.1, אבל ראויה ל-Session נפרד דחוף (פער הכנסות אקטיבי).

---

## חלק 9 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **תיאום ה-RPC עם הסגנון של ה-codebase.** ב-`fb_service.py` כבר יש קריאות RPC קיימות (UPSERT). על CC לאמת איך הן קוראות מ-Python — אותו דפוס בדיוק.

2. **חתימת `refresh_long_lived_token`.** אם בקוד הקיים יש convention של שמות (snake_case, prefix, וכו') — לדבוק בו. השם `refresh_long_lived_token` הוא הצעה; ההכרעה הסופית עם CC.

3. **חישוב `new_expires_at` ב-`fb_service.refresh_token_for_user`.** עדיפות ל-`datetime.now(timezone.utc) + timedelta(seconds=expires_in)` — כל ה-DB ב-UTC לפי ההגדרה של Supabase. שווה לאמת שכל קריאת `now()` בתוך ה-RPC היא גם UTC (היא אמורה להיות — `now()` ב-Postgres הוא `transaction_timestamp` ב-UTC כברירת מחדל).

4. **`MetaUnexpectedError` ו-Sentry noise.** retry של unexpected יכול ליצור רעש ב-Sentry אם הכשל מתמשך. ההמלצה: `classify_meta_error` קיים — לסמוך עליו. אם הוא מסווג כ-UNKNOWN, זה כנראה pattern חדש שמצריך הגדרה ידנית. שווה לעקוב אחרי Sentry בשבועות הראשונים אחרי deploy.

5. **תגלית 4 (billing crons לא מחווטים) — הצעה.** ה-tick שנבנה ב-6.1 הוא ספציפי לרענון. אבל הדפוס שלו (monotonic, `if now - last > interval`, scan + enqueue) יחזור גם ב-billing crons וב-Phase 8 (cron monitoring). שווה לתעד את הדפוס ב-CLAUDE.md כ-pattern רב-שימושי. כשיגיע Session ל-billing — להחיל את אותו דפוס בלי refactor.

6. **`OR token_expires_at IS NULL` ב-query.** אם בקוד הקיים יש assertion ש-`token_expires_at` לעולם לא NULL — להסיר את ה-`OR`. אם לא, להשאיר כהגנת אנומליות.

7. **בדיקה ידנית של 1.2 callback patch.** אחרי merge: לבצע OAuth → לבדוק ב-DB ש-`refresh_failed_at IS NULL` ו-`refresh_error IS NULL`. אז להגדיר ידנית `refresh_failed_at = '2026-01-01'`, לבצע OAuth שוב, לוודא שהשדה התאפס.

8. **תיאום עם 3.4c.** ה-tick של 3.4c (`runner.py:184-206`) הוא ל-cleanup של קמפיינים תקועים. ה-tick של 6.1 הוא לרענון טוקנים. שניהם יושבים ב-`runner.py` כתוספות עצמאיות. אין תלות ביניהם — הסדר בלולאה לא משנה. אבל שווה לוודא ששניהם מופרדים לפונקציות נפרדות (`tick_cleanup_stuck_campaigns`, `tick_refresh_meta_tokens`) ולא bloat של ה-loop הראשי.

---

## Phase 7 · סוכן AI פנימי (צ'אט + ניתוח)

> תלוי ב-Meta Insights (Phase 4.5) ובקמפיינים חיים (3.4). הצ'אט והניתוח קודמים לאופטימיזציה האוטונומית (Phase 8).

### Session 7.1 — צ'אט הסוכן בדשבורד ✅
- [x] `agent_service.py` + 5 endpoints סינכרוניים תחת `/me/agent`
- [x] `agent_conversations` + `agent_messages` (migration **0048**, נפרד מ-`bot_conversations`)
- [x] system prompt של מומחה-שיווק + הזרקת `service_name` (מטריקות Insights יתווספו ב-7.2)
- [x] מונה שיחות (50/חודש, `get_agent_chat_quota_status`)

**Done ✅:** לקוח מתייעץ עם הסוכן בצ'אט; שיחה פר-קמפיין, היסטוריה+מונה עובדים. מומש ב-migration **0048**
(לא 0031 — היה תפוס): `agent_conversations`/`agent_messages` (composite FK ל-campaigns + RLS write-deny) +
RPCs `create_agent_conversation`/`insert_user_and_assistant_messages` (אטומי). בקוד: `agent_service`
(reuse של ה-OpenAI seam מ-3.2; "סופרים רק אחרי הצלחה") + `routers/agent` + `subscription_service.
get_agent_chat_quota_status` (קבוע 50) + `prompts/agent/system_v1.txt` + `OPENAI_AGENT_MODEL`. error mapping:
`OpenAIRateLimitError`→503, `OpenAIError`→500 (אין transient/permanent split ב-seam — לא ממציאים).
**לא לעשות:** עדיין לא פעולות אוטונומיות — רק צ'אט וניתוח. הביצוע האוטונומי ב-Phase 8.

# Session 7.1 — תשתית הצ'אט עם הסוכן הפנימי

---

## מבוא ותיאור Session 7

7.1 הוא היסוד של כל Phase 7. בסשן הזה נבנה את **תשתית הצ'אט בין הלקוח לסוכן הפנימי** — חלון צ'אט בדשבורד, שמירת היסטוריה, חיבור ל-OpenAI עם system prompt כללי של מומחה שיווק, ואכיפת quota של 50 הודעות לחודש. בלי תוכן ספציפי (4 הצ'יפים, מאמן המכירות, חוק הברזל, פרוטוקול האופטימיזציה) — אלה Sessions עתידיים שבונים על התשתית הזו.

המטרה: בסוף 7.1, אם תפתח את הצ'אט בדשבורד כלקוח, תראה חלון פעיל, תוכל לבחור קמפיין, לכתוב הודעה, ולקבל תשובה כללית של מומחה שיווק. ההיסטוריה נשמרת, המונה עובד, ה-flow מקצה לקצה רץ.

**מודל UX (מ-mockups של גיא):**
1. הלקוח לוחץ על אייקון הצ'אט בפינה ימנית תחתונה של הדשבורד.
2. אם הוא לחץ פעם ראשונה — מוצגת רשימת קמפיינים פעילים כ-chips.
3. הוא בוחר קמפיין → נכנס לשיחה הצמודה לאותו קמפיין.
4. כל הודעות ההיסטוריה של אותה שיחה מוצגות.
5. הוא כותב הודעה → הסוכן מגיב.
6. השיחה לעולם לא נסגרת — היא רציפה. כדי לעבור לקמפיין אחר, חוזר ל-chips ובוחר אחר.

**מה בסשן:**
- טבלת `agent_conversations` (header: id, user_id, campaign_id) ו-`agent_messages` (id, conversation_id, role, content, created_at).
- service חדש `agent_service.py`.
- 4 endpoints: GET conversations list, GET messages, POST message, GET status.
- חיבור ל-`integrations/openai.py` עם system prompt זמני של מומחה שיווק כללי.
- הרחבת `subscription_service` עם `get_agent_chat_quota_status`.
- אכיפת quota: 50 הודעות לקוח בחודש anniversary; חסימה ב-`POST message` כשנגמרה.

**מה לא בסשן:**
- 4 הצ'יפים של "מה הבעיה?" (זה 7.3 והלאה).
- כרטיס סטטוס לייב עם מטריקות מ-Insights (זה 7.2).
- חוק הברזל / KNOWLEDGE_BASE / השוואה לbenchmarks (זה 7.4).
- מאמן המכירות (זה 7.3).
- חיבור ל-`optimization_actions` (זה 7.4-7.6).
- מיילים יזומים של הסוכן (Phase 7.x עתידי או 8.x).
- מכניזם של trim היסטוריה / סיכום אוטומטי — שולחים את כל ההיסטוריה ל-OpenAI, נראה בסביבת בדיקות אם יש בעיה.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מבנה השיחות | **שיחה פר קמפיין** — בוחרים קמפיין → רואים שיחה צמודה אליו |
| 2 | יצירת שיחה חדשה | **אוטומטית** — אין כפתור. שיחה נוצרת ב-INSERT הראשון של הודעה לקמפיין |
| 3א | חריגה מ-quota | **חוסם הודעות חדשות לחלוטין** עד החידוש הבא |
| 3ב | ספירת quota | **רק הודעות הלקוח** (role=user) נספרות. תשובות הסוכן (role=assistant) חופשיות |
| 3ג | quota total | **50 הודעות לחודש** (anniversary, מ-`current_period_start`) |
| 3ד | מתי נספרת הודעה | **רק אחרי שתשובת הסוכן נשמרה בהצלחה.** אם OpenAI נפל — ההודעה לא נחשבת במכסה |
| 4 | LLM ב-7.1 | **כן — עם system prompt זמני** של מומחה שיווק כללי. יוחלף ב-7.2 |
| 5 | היסטוריה ל-OpenAI | **שולחים הכל**. נראה איך זה מתנהג בסביבת בדיקות, נתאים אם צריך |
| 6 | שגיאות OpenAI | **fail with retry** — UI מציג "טוען..." ועושה retry אוטומטי |
| 7 | שמירת היסטוריה | **לתמיד** — בלי cleanup. גם אם יש 600 הודעות, נשמרות |

---

## תלויות

1. **Phase 3.4 — קמפיינים חיים.** הצ'אט תמיד צמוד לקמפיין. הלקוח לא יכול לפתוח צ'אט אם אין לו אף קמפיין פעיל (status=live או paused). אם אין — `GET /me/agent/conversations` יחזיר רשימה ריקה, ה-UI יציג הודעה מתאימה.

2. **Phase 0.5.2 — subscriptions.** ה-quota נספר מ-`current_period_start` של ה-subscription. נדרש שדה חדש `agent_chat_quota` ב-`subscriptions` (כבר תוכנן בספ §6, ממתין למספרים מגיא — נסגר על 50).

3. **Phase 3.0 — jobs queue.** **לא** משתמשים בו ב-7.1. הצ'אט סינכרוני (כמו 3.2 — קופי) — הלקוח לוחץ "שלח", מחכה 2-3 שניות, מקבל תשובה. אסינכרוני יתווסף רק אם נגלה בעיות UX בסביבת בדיקות.

4. **`integrations/openai.py` הקיים.** משתמשים בו דרך אותו ה-helper של קופי (3.2), עם פרמטר אחר ל-system prompt. אין שינוי באינטגרציה.

5. **Auth + RLS.** כל ה-endpoints דורשים JWT תקף, ה-DB אוכף בעלות דרך RLS.

---

## חלק 1 — Migration (0031)

קובץ חדש: `supabase/migrations/0031_agent_conversations.sql`. שני שינויים:

**א. יצירת `agent_conversations`:**
- `id` (uuid, PK).
- `user_id` (uuid, FK ל-`auth.users(id)` ON DELETE CASCADE).
- `campaign_id` (uuid, FK ל-`campaigns(id)` ON DELETE CASCADE).
- `created_at` (timestamptz, default `now()`).
- `updated_at` (timestamptz, default `now()`) — מתעדכן בכל הודעה חדשה (לסידור רשימה ב-UI).
- **Composite FK** ל-`campaigns(id, user_id)` — אוכף שה-`user_id` תואם לבעל הקמפיין (§7.2 חוק 2 בספ).
- **UNIQUE(user_id, campaign_id)** — שיחה אחת לכל קמפיין. INSERT שני מאותו לקוח לאותו קמפיין → ON CONFLICT DO NOTHING.
- **Index על (user_id, updated_at DESC)** — שאילתה של רשימת שיחות לפי לקוח.

**ב. יצירת `agent_messages`:**
- `id` (uuid, PK).
- `conversation_id` (uuid, FK ל-`agent_conversations(id)` ON DELETE CASCADE).
- `user_id` (uuid, redundant אבל נחוץ ל-RLS efficient — מועתק מ-conversation ב-INSERT דרך RPC, ראה למטה).
- `role` (text, CHECK: `'user'` או `'assistant'`).
- `content` (text, not null).
- `created_at` (timestamptz, default `now()`).
- **Composite FK** ל-`agent_conversations(id, user_id)`.
- **Index על (conversation_id, created_at ASC)** — שליפת היסטוריה בסדר כרונולוגי.
- **Index על (user_id, created_at DESC) WHERE role='user'** — שאילתת ספירת quota יעילה.

**ג. RLS על שתי הטבלאות:**
- `SELECT WHERE auth.uid() = user_id` — לקוח רואה רק את שלו.
- **כתיבה דרך admin client בלבד** (`agent_service` ב-server). הלקוח לא יכול INSERT/UPDATE/DELETE ישירות — חוסם injection של הודעות מזויפות. RLS WRITE-deny.

**ד. GRANT:**
- `grant select on agent_conversations, agent_messages to authenticated`.
- בלי `insert/update/delete` ל-authenticated. רק service_role.

**ה. RPC חדש `insert_user_and_assistant_messages`:**
- `SECURITY DEFINER`, `search_path=''`.
- חתימה: `(p_conversation_id uuid, p_user_id uuid, p_user_content text, p_assistant_content text) RETURNS table(...)`.
- שלבי ביצוע באטומיות:
  1. בדיקת בעלות: `SELECT user_id FROM agent_conversations WHERE id = p_conversation_id`. אם `user_id != p_user_id` → raise `'conversation_not_owned'`.
  2. INSERT הודעת לקוח (role='user').
  3. INSERT תשובת סוכן (role='assistant').
  4. UPDATE `updated_at` ב-`agent_conversations`.
  5. RETURN הודעת הסוכן (id, content, created_at).
- GRANT EXECUTE ל-service_role בלבד.

**ו. RPC חדש `create_agent_conversation`:**
- `SECURITY DEFINER`, `search_path=''`.
- חתימה: `(p_user_id uuid, p_campaign_id uuid) RETURNS uuid`.
- שלבי ביצוע:
  1. וידוא ש-`campaigns.id = p_campaign_id AND campaigns.user_id = p_user_id AND campaigns.status IN ('live', 'paused')`. אם לא תקין → raise.
  2. `INSERT ... ON CONFLICT (user_id, campaign_id) DO NOTHING RETURNING id`.
  3. אם conflict (כבר קיים) → SELECT הקיים.
  4. RETURN ה-`id`.
- GRANT EXECUTE ל-service_role בלבד.

---

## חלק 2 — שכבת `services/agent_service.py`

מודול חדש. אחראי לכל הלוגיקה של הצ'אט. ה-router קורא לו, הוא קורא ל-DB ול-OpenAI.

**פונקציות עיקריות (חתימה רעיונית, לא קוד מימוש):**

**א. `list_user_conversations(user_id) -> list[ConversationSummary]`:**
- שאילתה: כל ה-`agent_conversations` של הלקוח, join עם `campaigns` לקבל `service_name` ו-status, מסודרות לפי `updated_at DESC`.
- מחזיר רשימה של (conversation_id, campaign_id, service_name, last_message_preview, updated_at, status).
- שיחות לקמפיין שכבר אינו live/paused → לא מוחזרות (UI לא מציג צ'יפ עבורן). ה-data נשאר ב-DB, רק לא נחשף.

**ב. `get_or_create_conversation(user_id, campaign_id) -> Conversation`:**
- בודק אם יש שיחה קיימת לזוג (user_id, campaign_id) — אם כן, מחזיר אותה.
- אם לא — יוצר אחת חדשה דרך RPC `create_agent_conversation(p_user_id, p_campaign_id)`. ה-RPC מבצע:
  1. וידוא שהקמפיין שייך ל-user_id (בטיחות נוספת על ה-Composite FK).
  2. INSERT עם `ON CONFLICT DO NOTHING RETURNING` (race-safe).
  3. אם conflict → SELECT הקיים.
- מחזיר את ה-conversation הקיים או החדש.

**ג. `get_conversation_messages(user_id, conversation_id) -> list[Message]`:**
- שאילתה: כל ההודעות של conversation_id, מסודרות `created_at ASC`.
- וידוא בעלות דרך RLS (אם הלקוח ניסה לגשת לשיחה שלא שלו, RLS חוסם).
- מחזיר רשימה של (id, role, content, created_at).

**ד. `send_user_message(user_id, conversation_id, content) -> Message`:**
זה הליבה. שלבי הביצוע — **מודל "סופרים רק אחרי הצלחה"**:

1. **בדיקת quota מקדימה.** קריאה ל-`subscription_service.get_agent_chat_quota_status(user_id)`. אם `used >= 50` → raise `AgentChatQuotaExceededError` (403). **בדיקה לפני כל פעולה.**

2. **שליפת היסטוריה.** קריאה ל-`get_conversation_messages` של ההיסטוריה הקיימת (עוד בלי ההודעה החדשה).

3. **שליפת context של הקמפיין.** ב-7.1 — רק `service_name`. ב-7.2+ — יתווסף Insights.

4. **בניית prompt ל-OpenAI.** דרך `prompts_service.build()`. ההיסטוריה הקיימת + הודעת הלקוח החדשה (`content` שהגיע לפונקציה) משולבות במערך ההודעות שנשלח ל-OpenAI.

5. **קריאה ל-OpenAI.** המודל מקבל את כל ההיסטוריה + ההודעה החדשה, ומחזיר תשובה.

6. **טיפול בשגיאות OpenAI:**
   - `OpenAITransientError` (5xx, 429, timeout) → raise `AgentTransientError` (503). **שום דבר עוד לא נשמר ב-DB.** ה-UI יעשה retry אוטומטי — בעת ה-retry, ההודעה תישלח שוב מאפס, לא נספרת ב-quota.
   - `OpenAIPermanentError` (400, prompt בעיה) → raise `AgentPermanentError` (500). Sentry capture. **שום דבר לא נשמר.**
   - שגיאה לא-מזוהה → unknown → 500. **שום דבר לא נשמר.**

7. **רק עכשיו — שמירה אטומית של שתי ההודעות.** דרך RPC חדש `insert_user_and_assistant_messages(p_conversation_id, p_user_id, p_user_content, p_assistant_content)`. ה-RPC מבצע אטומית **בטרנזקציה אחת**:
   - וידוא בעלות (`conversation.user_id = p_user_id`).
   - INSERT של הודעת הלקוח (role='user').
   - INSERT של תשובת הסוכן (role='assistant').
   - עדכון `updated_at` ב-`agent_conversations`.

   אטומיות מובטחת — או שגם הודעת הלקוח וגם תשובת הסוכן נשמרות, או שאף אחת לא. אין מצב שהמכסה תיספר ואז הלקוח יישאר בלי תשובה.

8. **החזרה ל-router** של תשובת הסוכן.

**מודל ה-"סופרים רק אחרי הצלחה" — הסבר:**
- אם OpenAI נופל זמנית → exception, אין INSERT, המכסה נשארת כפי שהייתה. UI עושה retry → הלקוח לא מאבד הודעה מהמכסה.
- אם OpenAI מחזיר תשובה בהצלחה → שתי ההודעות נשמרות יחד, המכסה עולה ב-1.
- העלות: קריאת OpenAI שנכשלה לא מצטמצמת מהמכסה — אבל היא **כן** עולה לנו כסף ב-OpenAI (הם חייבו על הניסיון). זה ה-tradeoff. ב-MVP זה נדיר, בעתיד אם נראה שזה מתרחש הרבה — נשקול rate limit על retry.

**הגנה מ-abuse:** מאחר שהמכסה לא נספרת על כשלי OpenAI, לקוח **יכול** תיאורטית לנסות לגרום ל-OpenAI להחזיר 400 שוב ושוב (למשל הודעה ארוכה מאוד) כדי לא לקחת ממכסה. הגנה: validation ב-router לאורך הודעה (max 1000 chars). מעבר לזה — לא קריטי ב-MVP, רוב הלקוחות לא ינסו לעשות את זה.

**ה. `get_agent_chat_quota_status(user_id) -> QuotaStatus`:**
- מעבר ל-`subscription_service.get_quota_status` הקיים — שדה חדש `agent_chat_used` נשלף דרך COUNT על `agent_messages WHERE user_id=? AND role='user' AND created_at >= subscription.current_period_start`.
- מחזיר: `{used: int, quota: 50, remaining: int}`.

---

## חלק 3 — Endpoints

4 endpoints חדשים תחת `/me/agent`:

**א. `GET /me/agent/conversations`** — רשימת שיחות פעילות של הלקוח.
- Response: `[{conversation_id, campaign_id, service_name, last_message_preview, updated_at, campaign_status}, ...]`.
- ריק אם אין קמפיינים פעילים — ה-UI יציג "הקם קמפיין ראשון".

**ב. `POST /me/agent/conversations`** — קבלת/יצירת שיחה לקמפיין.
- Body: `{campaign_id: uuid}`.
- וידוא ב-router שהקמפיין שייך ל-user (RLS על `campaigns`). 404 אם לא.
- וידוא שה-`campaign.status` IN ('live', 'paused'). 409 `campaign_not_active` אם status אחר.
- קורא ל-`agent_service.get_or_create_conversation`.
- Response: `{conversation_id, campaign_id, service_name}`.

**ג. `GET /me/agent/conversations/{conversation_id}/messages`** — היסטוריית הודעות.
- Response: `[{id, role, content, created_at}, ...]`.
- 404 אם conversation_id לא שייך ל-user (RLS).

**ד. `POST /me/agent/conversations/{conversation_id}/messages`** — שליחת הודעה חדשה.
- Body: `{content: string}` (max 1000 chars).
- וידוא ולידציה: content לא ריק, אורך תקין.
- קורא ל-`agent_service.send_user_message`.
- Response: `{user_message: {...}, assistant_message: {...}}`.
- Status codes:
  - 200: הצלחה.
  - 403: `agent_chat_quota_exceeded` — מכסה מוצתה.
  - 404: conversation לא שייכת ל-user.
  - 503: `agent_transient_error` — OpenAI נופל זמנית. UI עושה retry.
  - 500: שגיאה לא-מזוהה.

**ה. `GET /me/agent/quota-status`** — סטטוס המכסה (בלי לשלוח הודעה).
- Response: `{used: 23, quota: 50, remaining: 27}`.
- UI מציג למשתמש ליד שדה הקלט.

---

## חלק 4 — הרחבת `subscription_service`

הוספת פונקציה חדשה `get_agent_chat_quota_status(user_id) -> QuotaStatus`:

- שולפת את ה-subscription של ה-user (כמו `get_quota_status` הקיים).
- אם `tier='pending'` → quota=0, used=0 (אין גישה).
- אם `tier='whatsapp'` → quota=0, used=0 (לא נכלל ב-whatsapp tier — חבילה אחרת).
- אם `tier IN ('basic', 'premium')` → quota=50.
- שאילתה: `COUNT(*) FROM agent_messages WHERE user_id=? AND role='user' AND created_at >= subscription.current_period_start`.
- מחזיר `{used, quota, remaining: max(0, quota-used)}`.

**הערה:** הספ §6 רושם `agent_chat_quota` כשדה ב-`subscriptions` ("ממתין למספרים מגיא"). עכשיו שיש מספר (50) — אפשרות א: קבוע בקוד (`AGENT_CHAT_QUOTA = 50`). אפשרות ב: עמודה ב-DB עם default 50.

**ההמלצה: קבוע בקוד.** אין שדרוגים/downgrades של quota בדינמיקה ב-MVP. אם בעתיד נרצה לשנות (למשל 100 ב-Premium) — נהפוך לעמודה. ב-MVP זה overkill.

---

## חלק 5 — System Prompt זמני

קובץ חדש: `prompts/agent/system_v1.txt`. תוכן כללי, ייוחלף ב-7.2 כשנוסיף את ה-context של הקמפיין.

תוכן זמני (טיוטה — סופי ייכתב במימוש):

> אתה מנהל קמפיינים בכיר ויועץ שיווק לעסקים קטנים, חלק מהפלטפורמה "Campaign AI".
>
> אתה עוזר ללקוחות שלנו עם שאלות שיווקיות כלליות — כתיבת כותרות, אסטרטגיות פרסום, הבנת הצרכן הישראלי.
>
> הקמפיין הנוכחי של הלקוח: {service_name}.
>
> חוקים:
> 1. עברית בלבד.
> 2. בלי מונחים טכניים באנגלית (CPL, CTR וכו'). השתמש במונחים פשוטים.
> 3. ענה קצר וממוקד — 2-3 משפטים בדרך כלל.
> 4. אם אינך יודע — אמור זאת. אל תמציא נתונים על הקמפיין שלא ניתנו לך.
> 5. ב-7.2 תקבל גם נתוני ביצועים חיים. כרגע אין לך אותם.

**הסבר:** ה-prompt עדין במכוון. לא מבטיח שום דבר ספציפי. הסוכן עונה כללית, וכשהלקוח שואל "כמה לידים יש לי?" הוא יודע לומר "אין לי גישה לנתונים האלה כרגע" במקום להמציא.

---

## חלק 6 — UI Flow (לתיעוד, לא נבנה ב-7.1)

ה-UI עצמו לא נבנה ב-7.1 (אין צורך לציין front-end). אבל ה-API חייב לתמוך ב-flow הזה:

1. הלקוח לוחץ על אייקון צ'אט בדשבורד.
2. Front-end קורא ל-`GET /me/agent/conversations` — מקבל רשימה.
3. אם הרשימה ריקה — הודעה: "אין לך קמפיינים פעילים. צור קמפיין כדי לקבל ייעוץ מהסוכן."
4. אם יש רשימה — מציג chips של קמפיינים.
5. לקוח לוחץ על chip → Front-end קורא ל-`POST /me/agent/conversations` עם ה-campaign_id → מקבל conversation_id.
6. Front-end קורא ל-`GET /me/agent/conversations/{id}/messages` → מקבל היסטוריה.
7. Front-end קורא ל-`GET /me/agent/quota-status` → מציג "23/50".
8. לקוח כותב הודעה → Front-end קורא ל-`POST .../messages` → מציג "טוען..." → מקבל תשובה → מציג.
9. במקרה של 503 (transient) — Front-end עושה retry אוטומטי אחרי 2 שניות.
10. במקרה של 403 (quota exceeded) — Front-end מציג "מכסה מוצתה. תוכל להשתמש בצ'אט שוב בתאריך X."

---

## חלק 7 — בדיקות

**בדיקות יחידה ל-`agent_service`:**

1. `list_user_conversations` עם 0/1/3 שיחות.
2. `get_or_create_conversation` — race condition (שני calls בו-זמנית מחזירים את אותה שיחה).
3. `send_user_message` — flow מלא מוצלח: OpenAI mock מחזיר תשובה → RPC שמירה אטומית → 2 הודעות ב-DB → quota עלה ב-1.
4. `send_user_message` — quota מוצתה (49 used + הניסיון ה-50 + 1 = 51) → exception, שום דבר לא נשמר.
5. **`send_user_message` — OpenAI מחזיר transient → exception. אף הודעה לא נשמרה ב-DB. quota נשארה זהה.**
6. **`send_user_message` — OpenAI מחזיר permanent → exception, Sentry capture. אף הודעה לא נשמרה ב-DB. quota נשארה זהה.**
7. **`send_user_message` — race condition: שתי קריאות מקבילות עם 49 used. שתיהן עוברות את בדיקת ה-quota הראשונית, שתיהן מצליחות ב-OpenAI. שתיהן מנסות INSERT — אחת מהן תהפוך ל-50, השנייה תהפוך ל-51 (חורגת). זה edge case מקובל ב-MVP (race נדיר), אבל לתעד.**

**בדיקות אינטגרציה ל-endpoints:**

1. `POST conversations` עם campaign_id לא שייך → 404.
2. `POST conversations` עם campaign_id status='draft' → 409.
3. `POST messages` כשהמכסה מוצתה → 403.
4. `GET messages` של conversation לא שייך → 404 (RLS).
5. ספירת quota: 49 הודעות user + 30 הודעות assistant → quota_used = 49 (רק user).
6. anniversary: לקוח עם current_period_start לפני חודש, יש לו 50 הודעות מהחודש הקודם — used=0 אחרי renewal.

**בדיקה ידנית בסביבת dev:**

1. ליצור קמפיין דמה ב-status='live'.
2. לפתוח את הצ'אט (דרך curl או API client).
3. לשלוח 5 הודעות, לוודא תשובות הגיוניות של LLM.
4. לוודא שה-quota עולה ב-1 בכל הודעה.
5. לשלוח 50 הודעות, לוודא חסימה ב-51.

---

## חלק 8 — לא ב-7.1 (להזכיר ב-Sessions עתידיים)

- **כרטיס סטטוס לייב עם מטריקות מ-Insights.** ב-7.2.
- **חיבור ל-`optimization_actions`.** ב-7.4-7.6.
- **4 הצ'יפים של "מה הבעיה?".** ב-7.3 והלאה.
- **מאמן המכירות.** ב-7.3.
- **חוק הברזל + KNOWLEDGE_BASE.** ב-7.4.
- **`industry` בקמפיין (patch ל-3.1).** חוסם 7.4 (לא 7.1).
- **מיילים יזומים של הסוכן.** Phase 7.x עתידי.
- **Trim היסטוריה / סיכום אוטומטי.** רק אם נראה בעיה בסביבת בדיקות.
- **Streaming של תשובות.** לא ב-MVP, סינכרוני מספיק.

---

## חלק 9 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`OPENAI_AGENT_MODEL` כ-env נפרד.** משתנה env חדש, ברירת מחדל `gpt-5.2`. הסיבה — אופי המשימה שונה מקופי (שיחה ארוכה vs יצירת טקסט קצר). אם בעתיד נרצה מודל אחר (קטן יותר ל-throughput, גדול יותר לחשיבה) — שינוי בלי לגעת ב-3.2.

2. **`prompts_service.build()` הקיים.** חוק 8 בספ — פרומפטים בקבצים. `prompts/agent/system_v1.txt` עוקב אחרי הדפוס של 3.2 ו-3.1.6. אם אין דפוס עבור "shell prompt + history" — להוסיף תמיכה ב-`build()` ל-injection של array היסטוריה.

3. **race ב-`get_or_create_conversation`.** ON CONFLICT DO NOTHING + SELECT הוא דפוס בטוח. אם בדיקה ב-CC מראה ש-`returning` ריק לא תמיד מטופל נכון — לעבור ל-`upsert` עם RETURNING תמיד.

4. **`updated_at` ב-`agent_conversations` כעמודה מעודכנת אוטומטית.** אופציה לטריגר ב-DB, או עדכון ידני ב-RPC `insert_agent_message`. **המלצה: ידני ב-RPC** — שקיפות, אין side effects נסתרים.

5. **גודל `content` ב-`agent_messages`.** הגבלת UI ב-1000 chars. ב-DB — text ללא הגבלה, אבל CHECK constraint על `length(content) <= 5000` להגנה אם UI עוקף.

6. **timeout ל-OpenAI.** ב-`integrations/openai.py` הקיים — לוודא שיש timeout (15-20 שניות?). אם השיחה ארוכה מאוד, יכול להיות שהקריאה תיקח זמן רב. תיעוד timeout בסביבת בדיקות חשוב.

7. **Sentry context.** כל exception מ-`send_user_message` שמגיע ל-Sentry — לוודא שיש `conversation_id`, `user_id`, `message_length` ב-context. בלעדיהם, דיבוג קשה.

8. **בדיקת performance של ספירת quota.** השאילתה `COUNT(*) FROM agent_messages WHERE user_id=? AND role='user' AND created_at >= ?` רצה בכל הודעה. עם index על `(user_id, created_at DESC) WHERE role='user'` זה אמור להיות מהיר. אם לא — לבחון caching של ה-count.

9. **migration order:** 0031 בא אחרי 0030 של 6.1. אם CC עוד לא ביצע את 6.1, 0031 צריך לחכות. תיאום ב-ROADMAP.

10. **תיעוד החלטה לעתיד:** במסמך התיקונים שיצרנו (`phase-7-8-infrastructure-fixes.md`), `optimization_actions` מצריך הרחבה מהותית. ב-7.1 לא נוגעים בו — רק ב-7.4. אבל לזכור.

---

# Session 7.2 — מטריקות קמפיין בצ'אט הסוכן ✅

> **Done ✅:** כרטיס סטטוס (4 מטריקות) + הזרקת המטריקות ל-LLM (system_v2). מומש: `meta.py`/`meta_service`
> (link_clicks ב-Insights + status check מורחב ל-paused), `core/dates` (israel_today_*), `agent_service`
> (`get_campaign_status_for_agent` + `_render_system_prompt_v2` + send מזריק), `GET /me/agent/conversations/
> {id}/status`, `prompts/agent/system_v2.txt`. **אין migration.** סטיות שתוקנו: cache-key-by-date_preset כבר
> היה קיים; reuse `lead_stats_service.count_leads_in_range` (אין leads_service); conversion=leads/link_clicks
> (לא חשיפות). הערה לגיא: ה-CTR בכרטיס נשאר Meta `ctr` הכללי; link_clicks משמש רק למכנה ה-conversion.

---

## מבוא ותיאור Session

7.1 בנה את תשתית הצ'אט — הסוכן יודע לקבל הודעות ולהגיב, אבל **בלי הקשר**. הוא עונה כללית, ויודע רק את `service_name` של הקמפיין.

ב-7.2 הופכים את הסוכן ל**מומחה שמכיר את הקמפיין**. שני שינויים מרכזיים:

1. **כרטיס סטטוס בראש הצ'אט** — 4 מספרים חיים על הקמפיין (כפי שראינו ב-mockup של גיא).
2. **הזרקת המספרים ל-LLM** — בכל הודעה, הסוכן רואה את הנתונים העדכניים ויודע לצטט אותם בתשובה.

המטרה: בסוף 7.2, אם תפתח את הצ'אט תראה כרטיס סטטוס עם המספרים האמיתיים בראש, ותוכל לשאול את הסוכן שאלות שהוא יענה עליהן עם נתונים אמיתיים ("עלות לפנייה שלך כרגע היא ₪34, זה...").

**מודל UX (מ-mockup של גיא, Image 3):**
1. הלקוח לוחץ על אייקון הצ'אט, בוחר קמפיין (כמו ב-7.1).
2. **חדש ב-7.2:** מיד מוצג כרטיס סטטוס בראש החלון עם 4 מספרים.
3. הלקוח שואל שאלה. הסוכן יודע את הנתונים, עונה בהתאם.
4. **בכל הודעה חדשה — הכרטיס מתעדכן** עם נתונים טריים. הסוכן רואה את העדכניים.

**מה בסשן:**
- הרחבה של Phase 4.5 — הוספת **link_clicks** (הקליקים שכבר עומדים בבסיס ה-CTR), **שיעור פניות אמיתי** (פניות חלקי קליקים) ו-**count להיום** למה ש-`get_campaign_insights` מחזיר.
- helper חדש `get_campaign_status_for_agent` ב-`agent_service` — שולף את 4 המטריקות, מתרגם לעברית, מחזיר במבנה אחיד (+ הקשר נוסף ל-LLM: הושקע וסך פניות).
- endpoint חדש `GET /me/agent/conversations/{id}/status` — מחזיר את הסטטוס לתצוגה ב-UI.
- עדכון `agent_service.send_user_message` — מזריק את המטריקות לכל קריאה ל-OpenAI.
- system prompt v2 — מותאם לקבלת מטריקות + מילון תרגום עברית.
- תמיכה דיפרנציאלית לקמפיין lead מול whatsapp (כל סוג עם המטריקות שלו).

**מה לא בסשן:**
- 4 הצ'יפים של "מה הבעיה?" (זה 7.3 והלאה).
- חוק הברזל / KNOWLEDGE_BASE / זיהוי ענף (זה 7.4).
- מאמן מכירות (זה 7.3, אחרי שגיא יבהיר את הבלבול בפרומפט).
- חיבור ל-`optimization_actions` (זה 7.4-7.6).
- סינון "אין מספיק נתונים" — הוחלט להציג תמיד את הנתונים שיש, ללא סינון.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | 4 המטריקות בכרטיס | **עלות לפנייה / אחוז האנשים שלחצו / שיעור פניות / פניות היום** |
| 2 | קמפיין WhatsApp | **כן נתמך, עם אותם 4 שדות, מותאמים:** עלות לשיחה / אחוז האנשים שלחצו / שיעור שיחות / שיחות היום |
| 3 | רענון מטריקות | **בכל הודעה.** גם הסוכן יודע (system prompt) וגם ה-UI מתעדכן (endpoint נפרד) |
| 4 | אין מספיק נתונים | **לא מסננים** — מציגים תמיד את הנתונים הזמינים. הסינון יבוא ב-7.4 (חוק הברזל) |
| 5 | תרגום מונחים לעברית | **חובה.** איסור מוחלט על מונחים באנגלית בתשובות הסוכן |
| 6 | System prompt | **v2 חדש** — מבוסס על v1 של 7.1 + הוספת בלוק מטריקות + מילון תרגום |
| 7 | הגדרת "שיעור פניות" | **פניות חלקי קליקים** (`leads / link_clicks`), **לא** חלקי חשיפות. חלוקה בחשיפות משכפלת את ה-CTR (אותו מכנה) ומחזירה מספר חסר משמעות. החלוקה בקליקים מתארת את המשפך האמיתי: חשיפות ← קליקים (CTR) ← פניות (שיעור פניות) |
| 8 | מה מוזרק ל-LLM מול מה שבכרטיס | **כרטיס: 4 מטריקות בלבד** (UX). **ל-LLM: 4 + הושקע + סך פניות** — שני האחרונים כבר נשלפים מ-4.5 חינם, והם בדיוק מה שמופיע ב-KPI של מסך הבית. בלעדיהם הסוכן יענה "אין לי גישה" על שאלות בסיסיות |
| 9 | fallback ל-v1 בזמן ריצה | **אין.** v1 נשאר בריפו כ-reference היסטורי, אבל אין נפילה אוטומטית אליו. אם v2 לא נטען זו שגיאת תכנות → Sentry + 500, לא נפילה שקטה (נפילה ל-v1 הייתה נותנת תשובה כללית בזמן שהכרטיס מציג מספרים — חוסר עקביות מבלבל) |
| 10 | טווחי זמן ב-WhatsApp conversion | **מונה ומכנה מאותו `date_preset`** (`maximum`). יחס בין conversations של 7 ימים למול קליקים של כל-החיים הוא מספר שגוי. אותו עיקרון "same range" שכבר ננעל ל-CPL ב-4.5 |

---

## תלויות

1. **Session 7.1.** התשתית של הצ'אט. ה-`agent_conversations`, ה-`agent_messages`, וה-flow של `send_user_message` כבר קיימים. ב-7.2 רק מוסיפים שכבת נתונים לפרומפט.

2. **Phase 4.5 — Meta Insights.** ה-helper `get_campaign_insights` כבר קיים ומחזיר Spend, Leads, CPL, CTR. 7.2 ידרוש הרחבה — להוסיף `link_clicks` (אם לא מוחזר כבר), שיעור פניות (`leads / link_clicks`), ו-today's count.

3. **Phase 3.4 — קמפיינים חיים.** ה-`campaign_id` חייב להיות במצב `live` או `paused`. אחרת אין מטריקות.

4. **Phase 5 — בוט WhatsApp (חלקי).** אם הקמפיין הוא lead+Premium ויש בוט פעיל, "אחוז מענה" יכול היה להיות מ-`bot_conversations`. אבל גיא הבהיר ש"אחוז מענה" הוא **Meta CTR**, לא bot response rate. אז אין באמת תלות בבוט.

---

## חלק 1 — הרחבת Phase 4.5

`get_campaign_insights` הקיים מחזיר 4 שדות: Spend, Leads, CPL, CTR (טווח: maximum = מהיום שעלה לאוויר). ב-7.2 צריך להוסיף את הבאים:

**א. `link_clicks` — בסיס המשפך.** ה-CTR שכבר נשלף מבוסס עליו (`link_clicks / impressions`). כדי לחשב את שיעור הפניות הנכון אנחנו צריכים את המונה הזה גם בנפרד. אם 4.5 כבר שולף אותו (גם אם רק כדי לחשב CTR) — לחשוף אותו בתוצאה. אם לא — להוסיף לרשימת ה-fields שנשלפת מ-Insights.

> **חשוב — עקביות המשפט:** הקליקים שמשמשים לשיעור הפניות חייבים להיות **אותו סוג קליקים** שעליו מבוסס ה-CTR (link clicks). כך המשפך קוהרנטי: חשיפות → link_clicks (CTR) → פניות (שיעור פניות). אם נשתמש בסוג קליק אחר למכנה, שתי המטריקות לא ידברו זו עם זו.

**ב. שיעור פניות אמיתי (`conversion_rate`).** באחוזים. החישוב:
- לקמפיין lead: `(leads / link_clicks) × 100`
- לקמפיין whatsapp: `(conversations / link_clicks) × 100`

זה המספר שהלקוח רואה בכרטיס כ"שיעור פניות": מתוך כל מי שלחץ על המודעה, איזה אחוז השאיר פרטים. המספר יוצא אינפורמטיבי (סדר גודל של 30–60%) ומשלים את ה-CTR במקום לשכפל אותו.

> **הימנעות מחלוקה ב-0:** קמפיין חדש יכול להיות עם `link_clicks = 0`. במקרה כזה `conversion_rate = 0` (לא exception, לא חלוקה ב-0). לבדוק את המכנה לפני החלוקה.

> **ל-whatsapp — אותו טווח זמן בלבד:** ה-conversations וה-link_clicks חייבים להישלף מאותה קריאת Insights עם אותו `date_preset` (`maximum`). אסור מונה של חלון 7d מול מכנה של maximum — זה יחס בין שני טווחים שונים, מספר חסר משמעות. זהו אותו עיקרון "same range" שננעל כבר ל-CPL ב-4.5.

**ג. Today's count (`today_count`).** ספירה ספציפית להיום בלבד:
- לקמפיין lead: `SELECT count(*) FROM leads WHERE campaign_id=? AND created_at >= israel_today_start_utc` — שאילתת DB, זולה, ו**לא** מוקאשת (רוצים אותה טרייה).
- לקמפיין whatsapp: קריאה ל-Meta Insights עם `date_preset='today'` ושאיבת ה-`messaging_conversation_started`. **זו קריאת Meta נפרדת מקריאת ה-maximum** — ולכן חייבת קאש משלה (ראה למטה).

### קאש — תיקון שורשי: מפתח לפי `date_preset`

הקאש של 4.5 (5 דקות TTL) נבנה סביב קריאת `date_preset=maximum`. קריאת ה-today של whatsapp היא טווח אחר, ולכן **לא מכוסה** על ידו כפי שהוא. הפתרון השורשי: **מפתח הקאש כולל את `date_preset`** — כך קריאת ה-maximum נשמרת תחת מפתח אחד, קריאת ה-today תחת מפתח אחר, ושתיהן מכוסות עצמאית.

זה מטפל גם בכפילות הטבעית במחזור הודעה אחד: `send_user_message` קורא לסטטוס (לבניית הפרומפט) **וגם** ה-endpoint `/status` קורא לו (לכרטיס). שתי הקריאות תוך אותן 5 דקות → שתיהן פוגעות באותו קאש → אפס קריאות Meta נוספות. בלי המפתח-לפי-טווח, כל קמפיין whatsapp היה גורר קריאת today כפולה בכל מחזור.

**גם:** עבור Today's count של lead, להוסיף helper קטן ב-`leads_service` (אם לא קיים) — `count_leads_today(campaign_id)`. שאילתה פשוטה, אינדקס על `(campaign_id, created_at)`.

---

## חלק 2 — שכבת `agent_service` — `get_campaign_status_for_agent`

פונקציה חדשה. חתימה: `get_campaign_status_for_agent(campaign_id: UUID) -> CampaignStatus`.

ההמלצה: ליצור אותה ב-`agent_service.py` (לא ב-`meta_service`). הסיבות:
- `get_campaign_insights` הקיים משומש במקומות נוספים. הוספת שדות-תצוגה שם משפיעה על caller-ים שלא צריכים אותם.
- ה-helper החדש מיועד לסוכן ספציפית, כולל פורמט עברית. זה לא Insights גנרי, זה Insights-לסוכן.
- הפרדה ברורה בין שכבת ה-integration (`meta_service`) לשכבת הסוכן (`agent_service`).

ה-helper החדש קורא ל-`meta_service.get_campaign_insights` הקיים (cache 5 דקות, מפתח לפי `date_preset`), מוסיף את השדות החדשים, ומבצע את התרגום העברי.

**שלבי ביצוע:**

1. שליפת `campaign` מ-DB. אם לא קיים או לא של ה-user (RLS) — exception.
2. שליפת `type` מהקמפיין (`lead` או `whatsapp`).
3. קריאה ל-`meta_service.get_campaign_insights(campaign_id)` — מחזיר Spend, Leads, CPL, CTR, link_clicks (עם cache).
4. חישוב `conversion_rate` (שיעור פניות, חלקי קליקים — לא חשיפות):
   - אם `link_clicks == 0` → `0`.
   - אם type=lead: `(leads / link_clicks) × 100`
   - אם type=whatsapp: `(conversations / link_clicks) × 100`, כאשר conversations ו-link_clicks מאותו `date_preset=maximum`.
5. שליפת `today_count`:
   - אם type=lead: `leads_service.count_leads_today(campaign_id)` (DB, לא מוקאש).
   - אם type=whatsapp: קריאת Meta עם `date_preset='today'` (מוקאשת תחת מפתח ה-today).
6. הרכבת `CampaignStatus`:

```
{
    "campaign_id": uuid,
    "type": "lead" | "whatsapp",
    "service_name": str,
    "metrics": {                       # ← לכרטיס ב-UI (4 בלבד)
        "cost_per_action": {           # עלות לפנייה / עלות לשיחה
            "value": 34.0,
            "label_he": "עלות לפנייה", # או "עלות לשיחה" ל-whatsapp
            "currency": "₪"
        },
        "click_rate": {                # אחוז האנשים שלחצו (CTR)
            "value": 2.1,
            "label_he": "אחוז האנשים שלחצו",
            "unit": "%"
        },
        "conversion_rate": {           # שיעור פניות / שיעור שיחות (leads/clicks)
            "value": 38.0,
            "label_he": "שיעור פניות", # או "שיעור שיחות" ל-whatsapp
            "unit": "%"
        },
        "today_count": {               # פניות היום / שיחות היום
            "value": 12,
            "label_he": "פניות היום"   # או "שיחות היום" ל-whatsapp
        }
    },
    "extra_context": {                 # ← ל-LLM בלבד, לא מוצג בכרטיס
        "spend": {
            "value": 8432.0,
            "label_he": "הושקע",
            "currency": "₪"
        },
        "total_leads": {
            "value": 248,
            "label_he": "סך הפניות"    # או "סך השיחות" ל-whatsapp
        }
    }
}
```

הפרדת `metrics` מ-`extra_context` היא העיקר: ה-UI מצייר רק את `metrics` (4 הכרטיסים שגיא עיצב), בעוד ה-LLM מקבל את **שניהם** — כך הסוכן יודע לענות גם על "כמה הושקע עד עכשיו?" ו"כמה פניות סה״כ?" (מספרים שכבר ביד מ-4.5, ושמופיעים ב-KPI של מסך הבית ממש ליד הצ'אט), בלי שהם מציפים את הכרטיס.

**טיפול בכשלים:**
- `MetaTransientError` → `AgentTransientError` (503). UI יעשה retry.
- `MetaPermanentError` (טוקן פג, קמפיין נמחק במטא) → החזרה של `CampaignStatus` עם `metrics: null`, `extra_context: null` ו-`error: "metrics_unavailable"`. הסוכן יעודכן ל-LLM שאין נתונים זמינים, ויענה בלי לצטט מספרים.
- שגיאה לא-מזוהה → 500.

**הערה:** הקאש (5 דקות TTL, מפתח לפי `date_preset`) מבטיח שגם אם הלקוח מתכתב מהר, אין הצפת Meta API. כל קריאה תוך 5 דקות מאותה קמפיין ואותו טווח — מאותו cache.

---

## חלק 3 — Endpoint חדש

**`GET /me/agent/conversations/{conversation_id}/status`**

- Auth: JWT + RLS על `agent_conversations` (וידוא ש-conversation שייך ל-user).
- שולף `campaign_id` מ-`agent_conversations`.
- קורא ל-`agent_service.get_campaign_status_for_agent(campaign_id)`.
- מחזיר את **`metrics` בלבד** (4 הכרטיסים). `extra_context` לא נחשף ב-endpoint הזה — הוא פנימי ל-LLM. שדה `service_name` ו-`type` כן נחשפים (ה-UI צריך אותם לכותרת).
- Status codes:
  - 200: הצלחה.
  - 404: conversation לא שייך ל-user.
  - 503: `metrics_transient_error` — Meta לא מגיב. UI עושה retry.
  - 500: שגיאה לא-מזוהה.

**מתי ה-UI קורא ל-endpoint הזה:**
- בפתיחת השיחה — אחרי `POST /me/agent/conversations` (יצירת/קבלת conversation), לפני הצגת הכרטיס.
- אחרי כל הודעה — להציג כרטיס מעודכן (מקביל לעדכון של הסוכן). הקריאה הזו תפגע בקאש של אותה הודעה — בלי קריאת Meta נוספת.
- ייתכן גם polling רקעי כל 30 שניות אם הצ'אט פתוח — אבל זה front-end concern.

---

## חלק 4 — עדכון `send_user_message` עם מטריקות

ב-7.1 ה-`send_user_message` קורא ל-OpenAI עם ה-system prompt v1. ב-7.2 צריך לעדכן:

**שלב חדש לפני קריאת ה-LLM:**

1. אחרי שליפת ההיסטוריה (שלב 2 ב-flow של 7.1).
2. **חדש:** שליפת `campaign_status = get_campaign_status_for_agent(campaign_id)`.
3. בניית system prompt v2 שמזריק את **כל** הנתונים — 4 המטריקות + `extra_context` (הושקע, סך פניות). (ראה חלק 5.)
4. שאר ה-flow זהה ל-7.1 (קריאה ל-OpenAI, שמירה אטומית, וכו').

**טיפול בכשל של שליפת מטריקות:**
- אם `get_campaign_status_for_agent` זרק `AgentTransientError` — להעביר את ה-503 ל-front-end. הלקוח יראה "בעיה זמנית, נסה שוב". אין מצב שהסוכן עונה בלי מטריקות בגלל בעיה זמנית.
- אם זרק `metrics_unavailable` (permanent — קמפיין נמחק במטא, טוקן פג) — הסוכן עדיין יענה, אבל ה-prompt יציין שאין נתונים. הסוכן יודע לא לצטט מספרים.

זה ההיגיון: מטריקות זמינות = הסוכן עונה עם הקשר מלא. מטריקות לא זמינות = הסוכן עונה כללית, וזה ברור גם ללקוח (כי הכרטיס בראש יציג "טוען...").

---

## חלק 5 — System Prompt v2

קובץ חדש: `prompts/agent/system_v2.txt`. v1 נשאר בריפו כ-reference היסטורי, אבל **אין נפילה אוטומטית אליו בזמן ריצה** — ראה החלטה 9. אם v2 לא נטען/נשבר, זו שגיאת תכנות שעולה ל-Sentry ומתוקנת ב-v2.

**מבנה ה-prompt v2:**

```
אתה מנהל קמפיינים בכיר ויועץ שיווק לעסקים קטנים, חלק מהפלטפורמה "Campaign AI".

אתה עוזר ללקוחות שלנו עם שאלות שיווקיות ובעיות בקמפיין שלהם. אתה מסתמך על
נתוני הקמפיין שמסופקים לך, ולעולם לא ממציא מספרים שלא נמסרו לך.

==========================================
פרטי הקמפיין הנוכחי
==========================================

שם השירות: {service_name}
סוג קמפיין: {type_he}   # "לידים" או "וואטסאפ"

נתוני ביצועים עדכניים:
- {cost_per_action_label}: {cost_per_action_value} ₪
- אחוז האנשים שלחצו על המודעה: {click_rate_value}%
- {conversion_rate_label}: {conversion_rate_value}%
- {today_count_label}: {today_count_value}

נתונים מצטברים (מתחילת הקמפיין):
- סך שהושקע: {spend_value} ₪
- {total_count_label}: {total_count_value}

(הנתונים האלה עדכניים לרגע זה. אם הלקוח שואל שאלה — התייחס אליהם ישירות.)

==========================================
חוקי שפה והתנהגות
==========================================

1. עברית בלבד. איסור מוחלט על מונחים באנגלית.
2. השתמש במונחים פשוטים שהלקוח יבין:
   - "עלות לפנייה" (לא CPL)
   - "אחוז האנשים שלחצו" (לא CTR)
   - "שיעור פניות" (לא Conversion Rate)
   - "מחיר לקליק" (לא CPC)
   - "הושקע" (לא Spend)
   - "תדירות חשיפה" (לא Frequency)
3. ענה קצר וממוקד — 2-4 משפטים בדרך כלל.
4. צטט מספרים מהנתונים שניתנו לך, **לא** ממציא.
5. אם הלקוח שואל על נתון שאין לך — אמור "אין לי גישה לנתון הזה כרגע".
6. **אל תציע פעולות אופטימיזציה ספציפיות בשלב זה.** השלם תשובות עם
   "אם אתה רוצה לתקן את זה, יש כפתורים בצ'אט שיעזרו לי לעזור לך טוב יותר"
   (הצ'יפים יגיעו ב-Sessions הבאים).

==========================================
מה אתה לא יודע לעשות (עדיין)
==========================================

- אינך יכול לבצע פעולות בקמפיין דרך השיחה הזו.
- אינך יודע מה העלות לפנייה הממוצעת בענף שלך (זה יתווסף בעתיד).
- אינך זוכר פעולות אופטימיזציה קודמות (זה יתווסף בעתיד).

אם הלקוח מבקש משהו שמחוץ ליכולתך — הסבר זאת בנימוס.
```

**הערות על ה-prompt:**

1. **בלוק "מה אתה לא יודע לעשות"** — קריטי. גיא עוד לא בנה את חוק הברזל ואת הפרוטוקול. הסוכן חייב להיות **מודע למגבלות שלו** ולא להבטיח דברים שהוא לא יכול לקיים.

2. **בלוק "נתונים מצטברים"** — מוזן מ-`extra_context` (הושקע, סך פניות). אלה לא מופיעים בכרטיס, אבל הם בדיוק מה שלקוח שואל עליו ("כמה הוצאתי?", "כמה פניות יש לי בסה״כ?"), והמספרים האלה כבר ביד מ-4.5. בלעדיהם הסוכן היה עונה "אין לי גישה" על שאלה בסיסית, בזמן שהמספר מופיע ב-KPI של מסך הבית ממש ליד.

3. **המילון לעברית בתוך ה-prompt** — לא מילון נפרד שהקוד שולח, אלא הוראה מפורשת ל-LLM. כך אם המודל ייתקל במונח חדש שלא במילון — הוא ידע **לתרגם בעצמו** לפי הדפוס.

4. **placeholders של מטריקות** — הקוד הדטרמיניסטי ב-`agent_service.send_user_message` ממלא אותם לפני שליחה ל-OpenAI. ה-LLM לא רואה את ה-`{...}` — הוא רואה את הערכים הסופיים.

5. **דיפרנציאל lead/whatsapp** — ה-labels (`cost_per_action_label`, `conversion_rate_label`, `today_count_label`, `total_count_label`) מתחלפים בהתאם לסוג הקמפיין. ה-prompt עצמו זהה.

---

## חלק 6 — שינויים נדרשים בקוד הקיים

**א. `meta_service.py`:**
- לחשוף את `link_clicks` בתוצאת `get_campaign_insights` (אם כבר נשלף לחישוב CTR — רק לחשוף; אם לא — להוסיף ל-fields).
- לחשוף `impressions` ו-`spend` בתוצאה אם לא חשופים (spend כבר אמור להיות).
- **תיקון הקאש:** מפתח הקאש כולל את `date_preset`, כדי שקריאת `maximum` וקריאת `today` יישמרו בנפרד ושתיהן יהיו מכוסות.

**הערה לפיתוח:** לאמת את החתימה הקיימת של `get_campaign_insights` לפני שמרחיבים. אם המבנה גמיש (dict) — תוספת קלה. אם מודל קשיח (Pydantic) — צריך לעדכן את ה-model.

**ב. `agent_service.py`:**
- הוספת `get_campaign_status_for_agent` החדש (כולל `extra_context`).
- עדכון `send_user_message` להזריק מטריקות + extra_context (פרטים בחלק 4).
- הוספת `_render_system_prompt_v2(campaign_status, history)` — בונה את ה-prompt עם ה-placeholders ממולאים (כולל בלוק "נתונים מצטברים").

**ג. `leads_service.py` (או דומה):**
- הוספת `count_leads_today(campaign_id)` אם לא קיים. שאילתה פשוטה, מבוססת `israel_today_start_utc`.

**ד. `routers/agent.py`:**
- הוספת endpoint `GET /me/agent/conversations/{conversation_id}/status` (מחזיר `metrics` + `service_name` + `type`, לא `extra_context`).

**ה. `prompts/agent/system_v2.txt`:**
- קובץ חדש (תוכן בחלק 5).

---

## חלק 7 — UI Flow

ה-UI לא נבנה ב-Session 7.2 (בדומה ל-7.1), אבל ה-API חייב לתמוך ב-flow הזה:

1. הלקוח לוחץ על אייקון הצ'אט.
2. UI קורא ל-`GET /me/agent/conversations` — רשימת קמפיינים.
3. הלקוח בוחר קמפיין.
4. UI קורא ל-`POST /me/agent/conversations` עם `campaign_id` — מקבל `conversation_id`.
5. **חדש:** UI קורא ל-`GET /me/agent/conversations/{id}/status` — מקבל את ה-`metrics`.
6. UI מציג את הכרטיס בראש החלון עם 4 המטריקות.
7. UI קורא ל-`GET /me/agent/conversations/{id}/messages` — מקבל היסטוריה.
8. הלקוח כותב הודעה → UI שולח ל-`POST .../messages`.
9. **חדש:** במקביל, UI קורא שוב ל-`/status` לעדכון הכרטיס. (אותו מחזור = אותו קאש = אפס קריאות Meta נוספות.)
10. UI מקבל את תשובת הסוכן ומציג.

**שיקול UX לעתיד (לא ב-7.2):** אם הלקוח כותב הודעה ויש עיכוב של 3-5 שניות בתשובה, האם הכרטיס מתעדכן בינתיים? אופציה לעשות את זה במקביל לקריאת ה-LLM. ב-7.2 לפשטות — הקריאות סדרתיות.

---

## חלק 8 — בדיקות

**בדיקות יחידה ל-`get_campaign_status_for_agent`:**

1. קמפיין lead תקני → 4 מטריקות מלאות + extra_context (הושקע, סך פניות), labels עבריים נכונים.
2. קמפיין whatsapp תקני → 4 מטריקות עם labels של whatsapp + extra_context.
3. **שיעור פניות מחושב נכון: `leads / link_clicks` (לא `leads / impressions`).** למשל 38 פניות / 100 קליקים = 38%.
4. **חלוקה ב-0: קמפיין עם `link_clicks = 0` → `conversion_rate = 0`**, בלי exception.
5. **WhatsApp conversion: מונה (conversations) ומכנה (link_clicks) מאותו `date_preset=maximum`.** ולידציה שלא מתערבב חלון 7d עם maximum.
6. קמפיין שאינו קיים → exception.
7. קמפיין שייך ל-user אחר → exception (דרך RLS).
8. Meta מחזיר transient → exception, UI יקבל 503.
9. Meta מחזיר permanent (טוקן פג) → `CampaignStatus` עם `error="metrics_unavailable"`, `metrics=null`, `extra_context=null`.

**בדיקות אינטגרציה ל-endpoint:**

1. `GET status` עם conversation לא שייך → 404.
2. `GET status` עם conversation תקני → 200 עם `metrics` (ובלי `extra_context` — הוא פנימי).
3. שני GET רצופים תוך פחות מ-5 דקות → השני מהקאש (מהיר, בלי קריאת Meta).
4. שני GET רצופים מעבר ל-5 דקות → הקאש נפתח, קריאה חדשה ל-Meta.
5. **קמפיין whatsapp: `send_user_message` + `GET status` באותו מחזור → קריאת today אחת בלבד ל-Meta** (השנייה מהקאש לפי `date_preset`).

**בדיקת end-to-end ב-dev:**

1. ליצור קמפיין lead ב-status='live' עם 50 לידים ב-Meta.
2. לפתוח שיחה.
3. לקרוא ל-`GET status` ולוודא שהמספרים תואמים את מה ש-Meta מציגה, ושיעור הפניות = פניות/קליקים.
4. לשלוח הודעה "כמה לידים יש לי?" — לוודא שהסוכן עונה עם המספר המצטבר האמיתי (מ-extra_context).
5. לשלוח הודעה "כמה הושקע עד עכשיו?" — לוודא שהסוכן עונה עם ה-spend, ולא "אין לי גישה".
6. לשלוח הודעה "מה ה-CPL שלי?" — לוודא שהסוכן עונה ב"עלות לפנייה היא ₪X" ולא משתמש במונח CPL.
7. לחזור על 4-6 עם קמפיין whatsapp.

---

## חלק 9 — לא ב-7.2

- **4 הצ'יפים של "מה הבעיה?".** ב-7.3 והלאה.
- **חוק הברזל (סינון/השוואה לbenchmarks).** ב-7.4. ב-7.2 מציגים מספרים כפי שהם.
- **זיהוי ענף אוטומטי (industry).** ב-7.4.
- **`optimization_actions` לזיכרון 30 יום.** ב-7.4-7.6.
- **טיפול בשגיאות "אין מספיק נתונים".** הוחלט לא לסנן ב-MVP. הסוכן יציג גם נתונים נמוכים.
- **Polling אוטומטי של הכרטיס.** UI concern, ה-API תומך אם רוצים.

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **לאמת את החתימה של `get_campaign_insights` הקיים מ-Phase 4.5.** ספציפית — האם הוא מחזיר `link_clicks` ו-`impressions`. אם כן, חישוב שיעור הפניות הוא רק חלוקה. אם לא — להוסיף אותם ל-fields שנשלפים (תיקון ב-4.5, לא יוצר session חדש).

2. **שיעור פניות = פניות חלקי קליקים, לא חלקי חשיפות.** זו ההחלטה המהותית של הסשן (החלטה 7). חלוקה בחשיפות מייצרת מספר שמשכפל את ה-CTR (שניהם על אותו מכנה) ויוצא נמוך ומטעה (~1%). חלוקה בקליקים נותנת את שיעור ההמרה האמיתי של המשפך (~30–60%). **שווה ליידע את גיא במשפט** — ההגדרה הזו קובעת מה המספר "שיעור פניות" מראה בכרטיס שהוא עיצב.

3. **קאש לפי `date_preset` — תיקון שורשי, לא טלאי.** במקום לקאש ספציפית את קריאת ה-today, מפתח הקאש של 4.5 כולל מעכשיו את הטווח. כך כל טווח עתידי (today / 7d / 30d) מכוסה אוטומטית, ושתי הקריאות במחזור הודעה (פרומפט + כרטיס) חולקות אותו cache.

4. **`count_leads_today` — אינדקס חיוני.** השאילתה תרוץ בכל קריאה ל-`/status`. אינדקס על `(campaign_id, created_at DESC)` מוודא ביצועים. לבדוק אם כבר קיים מ-Phase 4 — סביר שכן.

5. **`israel_today_start_utc` — חישוב נכון של תחילת היום.** ה-DB ב-UTC, הלקוח בישראל. "היום" = 00:00 ישראלי, לא 00:00 UTC. הפרש שעות (UTC+2 או UTC+3 לפי DST). helper ייעודי, לא חישוב ידני מפוזר.

6. **קמפיין WhatsApp — action_type ל-conversion.** `onsite_conversion.messaging_conversation_started_7d` (אישרנו קודם בשיחה) לספירת השיחות. לוודא ש-Phase 4.5 כבר משתמש בו לספירת leads ב-whatsapp. שים לב: לחישוב שיעור השיחות, גם המונה (conversations) וגם המכנה (link_clicks) מ-`date_preset=maximum` — לא לערבב עם חלון ה-7d של ה-today_count.

7. **system_v2.txt עם placeholders — איך הקוד ממלא?** הצעה: `prompts_service.build('agent/system_v2', context=campaign_status)` שמקבל את ה-dict (כולל extra_context) ומחליף את ה-`{...}`. דפוס מקובל מ-3.1.5/3.2.

8. **`label_he` ב-CampaignStatus — איפה הלוגיקה?** ההמרה מ-"lead" ל-"פניות" מול "שיחות" קורית ב-`get_campaign_status_for_agent` לפי `campaign.type`. אסור שהלוגיקה הזו תזלוג ל-router או ל-UI.

9. **`extra_context` לא נחשף ב-endpoint.** הכרטיס מציג 4 מטריקות בלבד (UX של גיא). הושקע וסך-פניות פנימיים ל-LLM. ה-endpoint `/status` מחזיר רק `metrics` (+ service_name + type). אם בעתיד נרצה להציג אותם בכרטיס — נחשוף; כרגע לא.

10. **fail-loud ב-v2.** אין נפילה אוטומטית ל-v1 (החלטה 9). v1 בריפו כ-reference. אם הטעינה של v2 נכשלת — Sentry + 500, לא נפילה שקטה לתשובה כללית בזמן שהכרטיס מציג מספרים.

11. **Sentry context בשגיאות.** כל exception ב-`get_campaign_status_for_agent` שמגיע ל-Sentry — לוודא `campaign_id`, `user_id`, `campaign_type` ב-context. בעיות עם Meta הן הכי שכיחות.

12. **בדיקות עם Meta sandbox.** אם יש test campaign ב-Meta sandbox — flow מלא לפני merge, במיוחד עבור whatsapp שדורש את ה-action_type הנכון ואת יישור הטווחים.

---

# Session 7.2.5 — עמוד הלידים (CRM-lite) ✅

> **Done ✅:** טבלת לידים (`GET /me/leads` + פילטר/keyset) + סטטוס פולואפ (`PATCH /me/leads/{id}`) + מייל
> new_lead. מומש ב-migration **0049**: `lead_status` (TEXT+CHECK) + RLS UPDATE policy + `GRANT UPDATE
> (lead_status)` (כתיבת הלקוח דרך **user client**, החריג המכוון) + `sent_notifications` CHECK +=new_lead +
> create-or-replace `insert_lead_and_event` שקורא `perform create_notification_and_job` (**אטומי**, reuse
> תשתית 4.6 — אפס שכפול, אפס loss). בקוד: `leads_service` (user client), `routers/leads`, `models/lead`
> (+lead_status/LeadStatusUpdate), `NotificationType.NEW_LEAD`, `build_new_lead_email`, handler branch.
> ה-`lead_intake_service` **לא** שונה (ההתראה אטומית ב-RPC). RLS+column-grant נבדקים ידנית.
> **מיקום:** 7.2.5 — יושב בין 7.2 (מטריקות) ל-7.3 (מאמן מכירות), ומסומן כ-**prerequisite ל-7.5**. עם זאת ניתן לבנייה **עצמאית עכשיו** — תלוי רק ב-4.1 שכבר מומש, לא בסוכן עצמו.

---

## מבוא ותיאור Session

כרגע יש חור בליבת הערך: לקוח Basic/Premium מקבל לידים — ואין לו שום מקום לראות אותם. אין עמוד כזה בפרוטוטייפ (§5 באפיון: בית, תהליכי שיפור, Creative, חבילה, משפטי — אין לידים) ואין אותו בקוד. מוצר לג'נרציית לידים שבו אי אפשר לראות את הלידים — זה לא feature חסר, זה חור בליבה.

7.2.5 בונה את העמוד: **טבלה של כל הלידים שנכנסו, פר-קמפיין, עם תשובות הסינון שלהם, סטטוס פולואפ שהלקוח מסמן ידנית, והתראת מייל על כל ליד חדש.** זה ה-"גיליון גוגל שיטס" שגיא תיאר.

### ההקשר ל-Phase 7 (למה זה כאן ולא ב-Phase 4)

עמוד הלידים הוא **prerequisite ל-7.5 (בעיה 2 — "פניות לא איכותיות")**. בפרוטוקול של בעיה 2, שלב 7, הסוכן **מוסיף שאלות סינון לטופס הליד** כדי לסנן פניות לא רלוונטיות. התשובות נוחתות ב-`screening_answers` — שדה ש**כבר קיים ב-`leads` מ-4.1**, שהושם שם מראש בדיוק בשביל זה.

כדי שהלקוח יראה אם הסינון עבד — ויעשה פולואפ על הלידים המסוננים — הוא צריך מקום לראות את הלידים **עם התשובות שלהם**. זה העמוד הזה. בלעדיו, שלב 7 של בעיה 2 אין לו לאן לנחות. זה בדיוק מה ש"ה-CRM" שגיא הזכיר בשרשרת `Meta Lead Form → ... → CRM` התכוון אליו.

לכן `screening_answers` **מוצג בעמוד** — וזה מוצדק כפליים: כבר ב-MVP הלקוח יכול להוסיף עד 4 שאלות סינון באשף (פרוטוטייפ §7.2), ובהמשך הסוכן מוסיף עוד (7.5). אותו שדה, אותה תצוגה.

המטרה: בסוף 7.2.5, לקוח עם קמפיין לידים פעיל רואה את כל הפניות (שם, טלפון, מייל, שירות, תאריך, תשובות סינון), יכול לסמן לכל אחת סטטוס (חדש / יצרתי קשר / נסגר / לא רלוונטי), ומקבל מייל ברגע שליד חדש נכנס.

**מודל UX:**
1. הלקוח נכנס ללשונית "הלידים שלי" בדשבורד.
2. רואה טבלה של הלידים, החדשים למעלה, עם פילטר פר-קמפיין ופר-סטטוס.
3. לכל ליד — פרטי קשר, תשובות הסינון (אם יש), ו-dropdown סטטוס. הלקוח מתקשר, סוגר, ומעדכן ל"נסגר".
4. במקביל — על כל ליד שנכנס, מגיע מייל "ליד חדש נכנס: {שם}, {טלפון}".

**מה בסשן:**
- migration קטן: `lead_status` ל-`leads` (TEXT + CHECK, default `new`), מדיניות UPDATE, והרשאת UPDATE **ברמת-עמודה** ל-`authenticated`.
- service: `leads_service` — שליפת רשימת לידים (פילטר + pagination), עדכון סטטוס.
- 2 endpoints: `GET /me/leads`, `PATCH /me/leads/{lead_id}`.
- סוג התראה חדש `new_lead` — הרחבה של תשתית 4.6, עם trigger מ-webhook הלידים של 4.1.

**מה לא בסשן:**
- הערה חופשית פר-ליד — **לא** ב-MVP (גיא הבהיר בזום: סטטוס בלבד בינתיים).
- ספירות/סיכום פר-סטטוס בכרטיסים — nice-to-have, לא בסקופ.
- ייצוא CSV / חיבור ל-CRM חיצוני — לא ב-MVP.
- digest/באטץ' של מיילים — מייל פר-ליד כפי שגיא תיאר.
- כל מה שקשור לסוכן (צ'אט, אופטימיזציה) — Phase 7.x/8.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מי קובע את הסטטוס | **הלקוח, ידנית.** המערכת לא יודעת ולא מחשבת — רק הלקוח יודע מה קרה בשיחת הטלפון. תיוג ידני בלבד |
| 2 | ערכי הסטטוס | **`new` / `contacted` / `closed` / `irrelevant`** → חדש / יצרתי קשר / נסגר / לא רלוונטי. ברירת מחדל `new` |
| 3 | הערה פר-ליד | **לא ב-MVP.** סטטוס בלבד (גיא, בזום) |
| 4 | אילו קמפיינים | **קמפייני לידים בלבד.** בקמפיין וואטסאפ הליד פונה ישירות למספר של הלקוח — אין רשומת ליד ב-DB. למנוי whatsapp העמוד ריק |
| 5 | מי כותב את `lead_status` | **ה-frontend ישירות, דרך endpoint רגיל עם RLS + column grant** — לא admin client. נתון בבעלות הלקוח (לא server-authoritative), ולכן החריג המכוון לדפוס "כתיבה דרך service" |
| 6 | התראת מייל | **כן, חלק מהסשן.** סוג `new_lead` על תשתית 4.6. מייל פר-ליד, ערוץ מייל בלבד |
| 7 | מכסה על מיילי לידים | **אין.** התראת מערכת, לא קשור ל-`agent_alerts_quota` |
| 8 | תצוגת `screening_answers` | **כן — מוצג בעמוד.** השדה כבר קיים מ-4.1. מוצדק כבר ב-MVP (שאלות סינון מהאשף) וקריטי ל-7.5 (שאלות סינון מהסוכן). זו תצוגה בלבד (read-only) של נתון קיים |

---

## תלויות

1. **Phase 4.1 — webhook לידים.** טבלת `leads` קיימת ומאוכלסת. **זו התלות העיקרית והיחידה לנתונים.** הסכמה ידועה (ראה חלק 1).

2. **Phase 4.6 — System Notifications.** התשתית (`send_notification`, `sent_notifications` עם idempotency, Resend, resolution של מייל הנמען) קיימת. 4.7 מוסיף סוג `new_lead` ו-trigger.

3. **Phase 0.5 — Auth + RLS.**

4. **לא תלוי בסוכן (7.1/7.2/7.3) ולא ב-6.1.** ניתן לבנייה עכשיו. אבל הוא **prerequisite ל-7.5** — לבנות לפניו.

---

## חלק 1 — Migration (הסכמה ידועה)

טבלת `leads` כבר קיימת מ-4.1, עם הסכמה הבאה (מאומת): `id`, `user_id`, `campaign_id`, composite FK ל-`campaigns(id,user_id)`, `service_name` (**denormalized, NOT NULL** — שורד מחיקת קמפיין), `contact_name`, `contact_phone`, `contact_email` (nullable), `contact_key`, `screening_answers` (jsonb, default `{}`), `meta_leadgen_id` (UNIQUE — idempotency), `meta_ad_id`, `raw_payload` (jsonb), `created_at`. אינדקסים קיימים: `(user_id, created_at DESC)`, `(campaign_id, created_at DESC)`, `(contact_key)`. RLS: `leads_select_own` (SELECT-only), `GRANT SELECT ... TO authenticated`. כתיבה — admin client בלבד דרך ה-webhook, **אין** policy ל-INSERT/UPDATE/DELETE.

לכן ה-migration החדש **קטן** — שלושה דברים בלבד. קובץ חדש `supabase/migrations/NNNN_lead_status.sql` (מספר רץ הבא בתור; idempotent; `create or replace`/`IF NOT EXISTS` בחדש — לא עורכים migrations קיימים):

**א. עמודת `lead_status`:**
- `ALTER TABLE public.leads ADD COLUMN IF NOT EXISTS lead_status text NOT NULL DEFAULT 'new' CHECK (lead_status IN ('new','contacted','closed','irrelevant'))`.
- `TEXT` + `CHECK`, לא enum ולא `VARCHAR(N)` — אותו דפוס כמו `status`/`tier`. הרחבה עתידית = `DROP`+`ADD CONSTRAINT`.
- שורות קיימות מקבלות `new` דרך ה-default. idempotent.

**ב. מדיניות RLS ל-UPDATE (תוספת טהורה — אין UPDATE policy קודם):**
- `DROP POLICY IF EXISTS leads_update_status_own ON public.leads;`
- `CREATE POLICY leads_update_status_own ON public.leads FOR UPDATE TO authenticated USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);`
- `USING` = אילו שורות; `WITH CHECK` = מונע שינוי `user_id`.

**ג. הרשאת UPDATE ברמת-עמודה (לב ההגנה):**
- `GRANT UPDATE (lead_status) ON public.leads TO authenticated;` — **רק העמודה הזו**.
- הלקוח יכול לעדכן `lead_status` בלבד; `contact_name`/`contact_phone`/`contact_email`/`service_name`/`campaign_id` נשארים בלתי-כתיבים מצידו (אין להם grant). RLS מטפל בבעלות-שורה, ה-column grant באיזו עמודה. **פתרון שורשי** — משטח כתיבה מינימלי, בלי RPC ובלי admin client.

> **המסלול הקיים לא נפגע:** ה-INSERT של הליד דרך ה-webhook הוא service_role (admin client) — עוקף RLS וגרנטים לחלוטין. ה-column grant מוסיף הרשאה רק ל-`authenticated`, ולכן לא נוגע במסלול ה-webhook.

> **fallback אם column-grant מתנהג מוזר מול PostgREST:** RPC `update_lead_status(p_lead_id, p_status)` — `SECURITY DEFINER`, `search_path=''`, מאמת בעלות, מעדכן רק `lead_status`. אבל ה-grant הישיר אמור לעבוד ופשוט יותר — לבדוק אותו קודם.

> **אין צורך באינדקסים חדשים** — `(user_id, created_at DESC)` ו-`(campaign_id, created_at DESC)` כבר קיימים מ-4.1 ומכסים את שאילתות הרשימה.

---

## חלק 2 — שכבת `leads_service`

הרחבת המודול. שתי פונקציות. שתיהן דרך **client בהקשר-משתמש** (`supabase_user` + JWT) — RLS אוכף בעלות. **לא** admin client (החלטה 5).

**א. `list_leads(user_id, campaign_id=None, status=None, limit=50, before=None) -> list[Lead]`:**
- שאילתה: לידים של המשתמש, מסודרים `created_at DESC`. **אין join** — `service_name` כבר על שורת הליד (denormalized).
- פילטר אופציונלי: `campaign_id`, `status`.
- pagination: keyset על `created_at` (`before` = cursor). `limit` עד 100.
- מחזיר: `(id, campaign_id, service_name, contact_name, contact_phone, contact_email, screening_answers, created_at, lead_status)`.
- RLS אוכף — לקוח רואה רק את שלו.

**ב. `update_lead_status(user_id, lead_id, status) -> Lead`:**
- ולידציה: `status` ∈ 4 הערכים (Pydantic ב-router + CHECK ב-DB — fail-early כפול).
- UPDATE דרך client בהקשר-משתמש: `UPDATE leads SET lead_status=? WHERE id=?` — RLS מוסיף בעלות, column grant מבטיח שרק `lead_status` נכתב.
- ה-service שולח **רק** `{lead_status: ...}` (הגנה בשכבות).
- ליד לא שייך → RLS מסנן → 0 שורות → 404.

---

## חלק 3 — Endpoints

**א. `GET /me/leads`** — רשימת הלידים.
- Query: `campaign_id` (אופציונלי), `status` (אופציונלי, אחד מ-4), `limit` (default 50, cap 100), `before` (cursor).
- Auth: JWT + RLS.
- Response: `[{id, campaign_id, service_name, contact_name, contact_phone, contact_email, screening_answers, created_at, lead_status}, ...]` (+ cursor).
- ריק אם אין לידים (לקוח חדש / מנוי whatsapp) → "עדיין לא נכנסו לידים".
- 200; 422 אם `status` לא חוקי.

**ב. `PATCH /me/leads/{lead_id}`** — עדכון סטטוס.
- Body: `{lead_status: "contacted"}` (ולידציה ב-Pydantic).
- Auth: JWT + RLS + column grant.
- Response: הליד המעודכן.
- 200 / 404 (לא שייך) / 422 (סטטוס לא חוקי).

> **פילטר הקמפיין ב-UI** — endpoint הקמפיינים הקיים מ-2.2. אין endpoint חדש.

---

## חלק 4 — סוג התראה `new_lead` (הרחבת 4.6)

**א. Trigger — patch ל-4.1:** ב-handler של webhook הלידים, אחרי INSERT מוצלח (כפוף ל-idempotency הקיים של `webhook_events`/`meta_leadgen_id`), enqueue job `send_notification` עם `{type:'new_lead', lead_id, user_id}`. patch קטן — לתעד ב-"Patches נדרשים".

**ב. Handler — ענף `new_lead` ב-`send_notification`:** שולף ליד (יש בו `service_name`, `contact_name`, `contact_phone` — בלי join), מרנדר מייל, שולח דרך Resend. תוכן: "ליד חדש בקמפיין {service_name}: {contact_name}, {contact_phone}" + לינק לעמוד. ערוץ מייל בלבד. נמען כמו 4.6.

**ג. idempotency:** רשומה ב-`sent_notifications` עם מפתח `(type='new_lead', lead_id)` — מייל אחד גם ב-retry.

> **נפח:** מייל פר-ליד (כבקשת גיא). digest לעתיד אם יציף.

---

## חלק 5 — RLS: למה זה החריג המכוון

הספ קובע "RLS SELECT-only לרוב הטבלאות, כתיבה דרך service" — נכון ל-writes **server-authoritative** (`tier`, טוקנים, מצב מנוי).

`lead_status` הפוך: **רק הלקוח יודע** מה קרה בשיחה. נתון בבעלותו, לא משהו שהמערכת מחשבת. לכן החריג המכוון:
- כתיבה דרך endpoint רגיל (`supabase_user` + JWT).
- RLS (`auth.uid() = user_id`) — בעלות-שורה.
- column grant (`UPDATE (lead_status)`) — רק העמודה הזו.

**זה לא פרצה ב-isolation** — להפך: משאיר את ה-admin client מבודד ל-webhook, ופותר את כתיבת הלקוח בדרך הצרה. CC צריך לדעת מראש **לא** לנתב את עדכון הסטטוס דרך admin client.

---

## חלק 6 — שינויים נדרשים בקוד הקיים

**א.** `supabase/migrations/NNNN_lead_status.sql` — חדש (חלק 1).
**ב.** `leads_service.py` — `list_leads` ו-`update_lead_status` (חלק 2).
**ג.** `routers/leads.py` — `GET /me/leads`, `PATCH /me/leads/{lead_id}` (חלק 3).
**ד.** `models/` — `Lead` (response, כולל `screening_answers`) ו-`LeadStatusUpdate` (body, enum 4 ערכים).
**ה.** `worker/handlers.py` (4.6) — ענף `new_lead` (חלק 4ב).
**ו.** webhook handler של 4.1 — enqueue אחרי INSERT (חלק 4א — patch).

---

## חלק 7 — UI Flow

ה-UI לא נבנה ב-7.2.5 (כמו 7.1/7.2), אבל ה-API תומך:

1. לשונית "הלידים שלי".
2. UI קורא ל-endpoint הקמפיינים הקיים → פילטר קמפיין.
3. UI קורא ל-`GET /me/leads` → רשימה.
4. טבלה: שם, טלפון, מייל, שירות, תאריך, תשובות סינון, dropdown סטטוס.
5. שינוי סטטוס → `PATCH /me/leads/{id}`.
6. במקביל: מייל על כל ליד נכנס.
7. scroll → `GET /me/leads` עם `before`.

---

## חלק 8 — בדיקות

**יחידה (`leads_service`):**
1. `list_leads` ללא פילטר → כל הלידים, `created_at DESC`.
2. עם `campaign_id` → רק אותו קמפיין.
3. עם `status='closed'` → רק סגורים.
4. עם `before` → עמוד שני, בלי כפילות.
5. הרשומה כוללת `screening_answers` (גם ריק `{}`).
6. `update_lead_status` תקין → סטטוס משתנה, שאר השדות לא נגעו.
7. סטטוס לא חוקי → נדחה.
8. ליד של user אחר → 0 שורות (RLS) → 404.

**אינטגרציה (endpoints):**
1. `GET /me/leads` בלי JWT → 401.
2. עם JWT → רק הלידים של המשתמש (RLS).
3. `PATCH` תקין → 200.
4. `PATCH` סטטוס לא חוקי → 422.
5. `PATCH` ליד לא שלי → 404.
6. **`PATCH` שמנסה לשנות `contact_phone` → נדחה/מתעלם** (column grant + ה-service שולח רק `lead_status`).

**RLS + column grant (קריטי):**
1. JWT של A: `UPDATE leads SET lead_status` על ליד של A → עובד.
2. על ליד של B → 0 שורות (RLS).
3. `UPDATE leads SET contact_phone=...` כ-authenticated → permission denied (אין grant על `contact_phone`).

**התראת `new_lead`:**
1. webhook ליד → נשמר + job נכנס לתור.
2. handler שולח מייל אחד + רשומה ב-`sent_notifications`.
3. webhook פעמיים (retry) → ליד אחד (idempotency 4.1) + מייל אחד (idempotency `sent_notifications`).
4. מנוי whatsapp → אין רשומות לידים → אין מיילי `new_lead`.

**e2e ב-dev:**
1. קמפיין lead פעיל + שאלת סינון אחת באשף. webhook ליד דמה.
2. לוודא: הליד ב-`GET /me/leads` עם `lead_status='new'` ועם `screening_answers` מאוכלס, ומגיע מייל.
3. `PATCH` ל-`contacted` → נשמר.
4. פילטרים מחזירים נכון.

---

## חלק 9 — לא ב-7.2.5

- הערה חופשית פר-ליד (גיא: סטטוס בלבד).
- ספירות/badges פר-סטטוס.
- ייצוא CSV / CRM חיצוני (ה-"CRM" של גיא = העמוד הזה).
- digest מיילים.
- לידים מקמפיין whatsapp (אין רשומות פר-ליד ב-DB).
- כל מה שקשור לסוכן (Phase 7.x/8).

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **הסכמה ידועה — אין צורך לאמת.** `leads` מ-4.1: `contact_name`/`contact_phone`/`contact_email`(nullable)/`contact_key`, `service_name` denormalized (NOT NULL, **בלי join**), `screening_answers` jsonb, ועוד. אינדקסים ו-`GRANT SELECT` כבר קיימים. ה-migration החדש = עמודה + UPDATE policy + column grant בלבד.

2. **column-level GRANT הוא הפתרון השורשי, לא RPC.** `GRANT UPDATE (lead_status)` + RLS UPDATE policy = משטח כתיבה מינימלי, בלי admin client. RPC רק fallback.

3. **`lead_status` בבעלות הלקוח — לא לנתב דרך admin client** (חלק 5). שמירה על ה-admin client מבודד ל-webhook.

4. **המסלול הקיים לא נפגע.** ה-INSERT דרך webhook הוא service_role (עוקף RLS+grants). ה-column grant מוסיף הרשאה רק ל-`authenticated`.

5. **`screening_answers` מוצג, זו תצוגה בלבד.** read-only של נתון קיים. זה ה-bridge ל-7.5 (שאלות סינון מהסוכן) וכבר רלוונטי ב-MVP (שאלות סינון מהאשף, פרוטוטייפ §7.2). אין כתיבה אליו מהעמוד.

6. **`new_lead` על תשתית 4.6 הקיימת.** ענף ב-`send_notification`, idempotency `(type, lead_id)`, נמען כמו 4.6.

7. **ה-trigger ב-4.1 הוא patch קטן.** enqueue אחרי INSERT, כפוף ל-idempotency הקיים.

8. **הרחבת ערכי סטטוס בעתיד.** ערך נוסף → `DROP`+`ADD CONSTRAINT`. ה-`TEXT`+`CHECK` נבחר בדיוק לזה.

9. **מספר ה-migration תלוי בסדר הבנייה.** לתאם ב-ROADMAP בזמן ההעברה (אתה כרגע ב-7.1; אם 7.2.5 נבנה אחריו, המספר אחרי 0031).

10. **Sentry context.** exceptions ב-`update_lead_status`/`list_leads` → `user_id`, `lead_id`, `campaign_id`.

---

# Session 7.3.5 — תשתית פרוטוקול האופטימיזציה (benchmarks + ענף + optimization tables) ✅

> **Done ✅:** תשתית בלבד. מומש: `app/data/benchmarks.json` (7 ענפים × 2 ספים, בחירת JSON — בלי תלות
> PyYAML) + `docs/agent/KNOWLEDGE_BASE.md` (תיעוד אנושי), `benchmark_service` (classify_cpl דטרמיניסטי
> gap-free, load fail-safe → unknown), `models/benchmark` (INDUSTRY_KEYS + BenchmarkResult). **3.1.6
> מורחב** (`ad_generation_service`): `BusinessContext.industry` (best-effort, רק מהרשימה הסגורה) +
> שמירה ל-extracted_context. **migration 0050**: `campaigns.industry` (TEXT+CHECK, nullable) + **trigger**
> `sync_campaign_industry` (denormalization אטומי מ-quiz_responses — לא RPC, לא שינוי service; מכסה גם
> reset). **migration 0051**: `optimization_sessions`+`optimization_actions` (composite FK, partial unique
> לסדרה פתוחה, RLS SELECT-only + GRANT). תיקון ל-DDL של ה-ROADMAP: נוסף `UNIQUE(id,user_id)` ל-sessions
> (חסר — ה-FK המורכב של actions דורש אותו). נדחה ל-7.4: מודלי-קריאה ל-2 הטבלאות (אין consumer), get_
> industry_context (stub). RLS/trigger/constraints נבדקים ידנית מול Supabase (אין live-DB ב-CI).
> **מיקום:** 7.3.5 — סשן הכנה. יושב לפני 7.4 (בעיה 1) ומפרק את שלושת החסמים שלו. **7.3 דולג** (מאמן מכירות — נדחה).
> **חשוב:** הסשן בונה **תשתית בלבד** — אין כאן לוגיקת סוכן, אין צ'אט, אין פעולות מול Meta. רק הקרקע ש-7.4/7.5/Phase 8 ירוצו עליה.

---

## מבוא ותיאור Session

7.4 (בעיה 1 — "עלות לליד גבוהה / מעט פניות") הוא הצ'יפ הראשון בפרוטוקול הסוכן. אבל **שלב 1 שלו ("אבחון והשוואה לשוק") חסום על שלושה דברים שלא קיימים במערכת**:

1. **חוק הברזל** — הסוכן משווה את העלות-לליד מול benchmark פר-ענף לפני שהוא מציע פתרון. בלי טבלת benchmarks אין השוואה.
2. **שדה ענף בקמפיין** — בלי לדעת באיזה ענף הקמפיין, אין מול מה להשוות.
3. **זיכרון 30 יום** — הניתוח המעורב ("היה ₪50, ירד ל-₪42, עכשיו ₪48") דורש טבלה שזוכרת מה נוסה ומה היו המטריקות. `optimization_actions` כפי שמוגדר ב-spec §6 לא מספיק — צריך מבנה עשיר יותר.

7.3.5 בונה את שלושת אלה, **נקי ופעם אחת**, כי הם משרתים גם את 7.5 (בעיה 2) וגם את כל Phase 8 (אופטימיזציה אוטונומית) — לא רק את 7.4. קבירה שלהם בתוך סשן של בעיה ספציפית הייתה טלאי.

**מה בסשן:**
- `KNOWLEDGE_BASE.md` — קובץ מקור-אמת ל-benchmarks (7 ענפים, 3 רמות), שגיא עורך ישירות.
- שדה `industry` — מזוהה אוטומטית מ-`service_name`+`business_description`, מקופל לתוך 3.1.6 הקיים, denormalized ל-`campaigns`.
- שתי טבלאות חדשות: `optimization_sessions` (סדרת טיפול בבעיה) ו-`optimization_actions` (פעולה בודדת תחת סדרה).
- `benchmark_service` — טוען את ה-KB, מסווג CPL לרמה (מדהים/ממוצע/יקר). **דטרמיניסטי, לא LLM.**

**מה לא בסשן:**
- לוגיקת הסוכן (אבחון, הצעת פתרון, ניסוח) — זה 7.4.
- ה-Lock Mechanism עצמו (הבדיקה אם קמפיין נעול) — 7.4 משתמש בטבלאות שנבנות כאן, אבל הלוגיקה שלו ב-7.4.
- כתיבת פעולות אמיתיות לטבלאות — כאן רק הסכמה + ה-helpers. הכתיבה ב-7.4 ו-Phase 8.
- benchmarks לשאר 3 המטריקות (רק CPL מקבל benchmark — ראה "פתוח לגיא").
- העונתיות וה-troubleshooting triggers מה-KB — נשמרים בקובץ כהקשר ל-LLM, אבל אין עליהם לוגיקה דטרמיניסטית ב-MVP.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | טבלה אחת או שתיים | **שתיים.** `optimization_sessions` (סדרה: ענף הבעיה, מונה לולאות, מטריקה התחלתית, סטטוס) + `optimization_actions` (פעולה: מספר שלב, תיאור שינוי, מטריקה אחרי). מונע שכפול, משקף את המבנה האמיתי |
| 2 | fallback לזיהוי ענף | **`industry` נשאר NULL.** אם ה-LLM לא בטוח — לא ממציאים ענף. חוק הברזל מדלג על ההשוואה לקמפיין כזה (fail-safe — עדיף לא לחסום לקוח על benchmark שגוי) |
| 3 | מתי רץ זיהוי ענף | **מקופל לתוך 3.1.6** — שדה רביעי באותה קריאת LLM (`get_or_extract_business_context`). patch ל-3.1.6 |
| 4 | איפה `industry` נשמר | **denormalized ל-`campaigns.industry`** (כמו `service_name`). הסוכן/הפרוטוקול ניגשים לקמפיין הרבה — עדיף בלי join בכל בדיקת חוק ברזל. ההעתקה באותו admin-client write של 3.1.6 |
| 5 | מפתח "סדרה" | **סדרה פתוחה אחת לכל `(campaign_id, problem_type)`.** קמפיין יכול להחזיק במקביל סדרת `high_cpl` וסדרת `low_quality_leads`, אבל לא שתי `high_cpl` פתוחות. אוכף ב-partial unique index |
| 6 | סיווג CPL לרמה | **דטרמיניסטי בקוד** (`benchmark_service`), לא LLM. הקוד קובע "מדהים/ממוצע/יקר", ה-LLM רק מנסח (חוק 7) |
| 7 | מקור ה-benchmarks | **`KNOWLEDGE_BASE.md` בריפו.** גיא עורך ישירות. הקוד טוען ממנו. העברה ל-DB — עתידית |

---

## תלויות

1. **Phase 3.1.6 — חילוץ Business Context.** קיים. `get_or_extract_business_context` ב-`ad_generation_service.py`, שומר ל-`quiz_responses.extracted_context` (jsonb) דרך admin client. 7.3.5 מרחיב אותו (patch).

2. **Phase 2.2 — `campaigns`.** קיים. מוסיפים עמודת `industry`.

3. **Phase 4.5 — Insights.** קיים. `benchmark_service` מסווג את ה-CPL שמגיע משם. אין שינוי ב-4.5.

4. **לא תלוי ב-7.1/7.2/7.2.5.** עצמאי. אבל הוא **prerequisite ל-7.4 ול-7.5**, וטבלאותיו ל-Phase 8.

---

## חלק 1 — `KNOWLEDGE_BASE.md`

קובץ חדש בריפו (לא ב-DB): `docs/agent/KNOWLEDGE_BASE.md`. מקור-אמת ל-benchmarks, גיא עורך ישירות. הקוד טוען ממנו.

**א. טבלת ה-CPL benchmarks — 7 ענפים, 3 רמות** (מהמסמך של גיא):

| ענף (`industry` key) | מדהים (ירוק) | ממוצע (צהוב) | יקר (אדום) |
|---|---|---|---|
| `beauty` — קוסמטיקה, ביוטי וטיפולים | ₪15–25 | ₪26–45 | מעל ₪50 |
| `courses` — בתי ספר, קורסים ומקצועות | ₪30–50 | ₪51–90 | מעל ₪100 |
| `fitness` — כושר, ספורט ובריאות | ₪15–30 | ₪31–55 | מעל ₪60 |
| `local_services` — שירותים לבית ועסקים מקומיים | ₪25–40 | ₪41–70 | מעל ₪85 |
| `professional` — עו"ד, רו"ח ופיננסים | ₪60–90 | ₪91–150 | מעל ₪160 |
| `realestate` — נדל"ן, השקעות ותיווך | ₪70–100 | ₪101–180 | מעל ₪200 |
| `b2b_hr` — הייטק, B2B וגיוס עובדים | ₪80–120 | ₪121–220 | מעל ₪250 |

**ב. מבנה הקובץ:** הטבלה למעלה כבלוק מובנה שהקוד פרסר (CSV/YAML/טבלת markdown — ראה הערה למטה). מתחתיה, כהקשר ל-LLM בלבד (בלי לוגיקה דטרמיניסטית ב-MVP): השפעת סוג המשפך על המחיר, פרדוקס איכות-הליד, עונתיות ישראלית (תשרי/קיץ/נובמבר), 3 ה-troubleshooting triggers. אלה ייכנסו ל-system prompt של הסוכן ב-7.4 כידע רקע — אבל ב-7.3.5 רק יושבים בקובץ.

> **מה הקוד פרסר ומה לא:** רק **טבלת ה-7 ענפים** נטענת ע"י `benchmark_service` (לסיווג דטרמיניסטי). שאר הקובץ הוא טקסט חופשי שב-7.4 יוזרק כ-string ל-prompt. כדי שהפרסור יהיה יציב, הטבלה תשב בבלוק תחום (למשל fenced block עם key מזוהה, או קובץ YAML נפרד `benchmarks.yaml` שה-MD מתעד). **המלצה: `benchmarks.yaml` כמקור הפרסור + `KNOWLEDGE_BASE.md` כתיעוד אנושי שגיא עורך.** שני קבצים, מקור אחד לוגי — אבל זה מפריד "מה שהקוד קורא" מ"מה שגיא עורך". אם רוצים קובץ אחד — לפרסר את טבלת ה-markdown ישירות, פחות יציב. **לסגור עם גיא איזו עדיפות.**

---

## חלק 2 — שדה `industry` (קיפול ל-3.1.6)

**א. הרחבת ה-dataclass** ב-`ad_generation_service.py`:

```python
@dataclass
class BusinessContext:
    service_name: str
    business_description: str
    problem_solved: str
    industry: str | None   # חדש. היחיד שמותר לו NULL
```

`industry` הוא **היחיד** שלא נכלל בוולידציה הקשיחה של "מחרוזת לא-ריקה". שלושת הראשונים זורקים `BusinessContextExtractionError` אם ריקים; `industry` ריק/לא-מזוהה → `None` (החלטה 2).

**ב. הרחבת הפרומפט** `app/prompts/phase3/extract_business_context.txt`:
- להוסיף הוראה: להחזיר שדה חמישי `industry`, **מתוך הרשימה הסגורה של 7 ה-keys** (`beauty`/`courses`/`fitness`/`local_services`/`professional`/`realestate`/`b2b_hr`).
- **הוראה קריטית:** אם העסק לא משתבץ בבירור באחד מהשבעה → להחזיר `null` (לא לנחש, לא להמציא ענף שמיני). הקלט לזיהוי = `service_name` + `business_description` (כפי שסגרנו).
- ה-`response_format={"type":"json_object"}` כבר קיים — רק מוסיפים מפתח.

**ג. ולידציה אחרי הפענוח** ב-`extract_business_context`:
- אם `industry` מוחזר אבל **לא** ברשימת ה-7 → להתייחס כ-`None` (הגנה מהזיות; ה-LLM החזיר ערך לא חוקי). לא לזרוק שגיאה — `industry` הוא best-effort.
- אם `industry` חסר/null → `None`. תקין.

**ד. denormalize ל-`campaigns`** (החלטה 4): בתוך `get_or_extract_business_context`, באותה פעולת admin-client ששומרת את `extracted_context` ל-`quiz_responses` — **גם** לכתוב את `industry` ל-`campaigns.industry` של הקמפיין המקושר. אותו client, אותו רגע, server-authoritative. אם `industry=None` → `campaigns.industry` נשאר NULL.

> **מדוע גם וגם:** `extracted_context.industry` ב-`quiz_responses` הוא חלק מהפלט המלא של 3.1.6 (תיעוד). `campaigns.industry` הוא העותק שהסוכן צורך פר-קמפיין בלי join. שניהם מתמלאים באותה פעולה — אין חוסר-עקביות.

---

## חלק 3 — Migration: `campaigns.industry`

קובץ חדש (מספר רץ הבא בתור). idempotent.

```sql
ALTER TABLE public.campaigns
  ADD COLUMN IF NOT EXISTS industry text
  CHECK (industry IN (
    'beauty','courses','fitness','local_services',
    'professional','realestate','b2b_hr'
  ));
```

- **nullable** (אין `NOT NULL`) — `industry` מותר להיות NULL (fail-safe, החלטה 2).
- `TEXT` + `CHECK` עם 7 הערכים — אותו דפוס כמו `status`/`tier`/`lead_status`. הרחבת ענף עתידית = `DROP`+`ADD CONSTRAINT`.
- אין צורך ב-GRANT חדש — הכתיבה דרך admin client (3.1.6), הקריאה דרך ה-SELECT policy הקיים של `campaigns`.

---

## חלק 4 — Migration: שתי טבלאות האופטימיזציה

קובץ חדש (אין `optimization_actions` קיים — בונים מאפס). זו ההרחבה של מה ש-spec §6 הגדיר כטבלה אחת, מפוצל לשתיים (החלטה 1).

### 4א. `optimization_sessions` — הסדרה

סדרה רציפה של טיפול ב**בעיה אחת** בקמפיין אחד.

```sql
CREATE TABLE public.optimization_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  campaign_id uuid NOT NULL REFERENCES public.campaigns(id) ON DELETE CASCADE,

  -- composite FK ל-RLS coherence (spec §7.2)
  CONSTRAINT opt_sessions_campaign_user_fk
    FOREIGN KEY (campaign_id, user_id)
    REFERENCES public.campaigns(id, user_id)
    ON DELETE CASCADE,

  problem_type text NOT NULL
    CHECK (problem_type IN ('high_cpl','low_quality_leads','meta_rejection')),

  status text NOT NULL DEFAULT 'in_progress'
    CHECK (status IN ('in_progress','success_monitoring','done','failed','escalated')),

  loop_count integer NOT NULL DEFAULT 0,        -- חוק הלולאה
  starting_metric jsonb,                         -- מטריקות לפני שלב 1 של הסדרה
  current_step integer NOT NULL DEFAULT 0,       -- השלב הפעיל בסדרה

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- סדרה פתוחה אחת לכל (campaign_id, problem_type) — החלטה 5
CREATE UNIQUE INDEX uq_open_session_per_problem
  ON public.optimization_sessions (campaign_id, problem_type)
  WHERE status IN ('in_progress','success_monitoring');

CREATE INDEX idx_opt_sessions_lookup
  ON public.optimization_sessions (campaign_id, problem_type, created_at DESC);
```

הסבר השדות:
- `problem_type` — **enum סגור** (לא טקסט חופשי, כפי שמסמך התיקונים דרש). 3 ערכים תואמים ל-3 בעיות הקמפיין. (`meta_rejection` כאן לשלמות — 7.6/Phase 8.)
- `status` — כולל את `success_monitoring` החדש (אחרי שיפור, חלון מעקב נוסף של 5 ימים לפני `done`).
- `loop_count` — עולה ב-1 בכל חזרה משלב אחרון לשלב 1 (חוק הלולאה).
- `starting_metric` — **נקודת מטריקה 1 מתוך 3** (הנתון לפני כל הסדרה). זה ה-X בניתוח המעורב.
- ה-partial unique index הוא הלב: מבטיח "סדרה פעילה אחת לבעיה" — מה שמאפשר ל-Lock ולזיכרון 30 יום לשאול שאלה עם תשובה יחידה.

### 4ב. `optimization_actions` — הפעולה הבודדת

פעולה אחת (שלב אחד) **תחת** סדרה.

```sql
CREATE TABLE public.optimization_actions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  session_id uuid NOT NULL REFERENCES public.optimization_sessions(id) ON DELETE CASCADE,

  -- composite FK ל-RLS coherence (אוכף ש-user תואם לסדרה)
  CONSTRAINT opt_actions_session_user_fk
    FOREIGN KEY (session_id, user_id)
    REFERENCES public.optimization_sessions(id, user_id)
    ON DELETE CASCADE,

  step_number integer NOT NULL,                  -- השלב בפרוטוקול (1-7)
  action_type text NOT NULL,                     -- creative_refresh / angle_change / offer_change / screening_question / ...
  change_description text,                        -- "שינוי 3 פוסטים וקופי" — לתצוגה בתהליכי שיפור

  post_change_metric jsonb,                       -- נקודת מטריקה 2: מיד אחרי 120 שעות (ה-Y)
  current_metric jsonb,                           -- נקודת מטריקה 3: עכשווי, lazy refresh (ה-Z)

  status text NOT NULL DEFAULT 'in_progress'
    CHECK (status IN ('in_progress','success_monitoring','done','failed','escalated')),

  window_ends_at timestamptz,                     -- נעילת 120 שעות (5 ימים)

  -- קטגוריית אישור (spec §9): האם הפעולה דרשה אישור לקוח או אוטונומית
  requires_approval boolean NOT NULL DEFAULT true,
  approved_at timestamptz,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_opt_actions_session
  ON public.optimization_actions (session_id, step_number);

-- לשאילתת "הפעולה האחרונה בקמפיין שעדיין בחלון מדידה" (Lock + זיכרון 30 יום)
CREATE INDEX idx_opt_actions_window
  ON public.optimization_actions (user_id, window_ends_at)
  WHERE status IN ('in_progress','success_monitoring');
```

הסבר השדות:
- **3 נקודות המטריקה מפוצלות נכון:** `starting_metric` ב-session (לפני כל הסדרה), `post_change_metric` ב-action (מיד אחרי השלב הזה), `current_metric` ב-action (עכשווי). זה בדיוק ה-X→Y→Z של הניתוח המעורב, בלי שכפול — ה-X חי פעם אחת ברמת הסדרה.
- `window_ends_at` — נעילת 120 שעות (5 ימים, **לא** 4 — תיקון 1 במסמך התיקונים). ה-Lock Mechanism ב-7.4 ישאל `now() < window_ends_at`.
- `requires_approval` / `approved_at` — מקודד את שתי קטגוריות הפעולה (דורש אישור / אוטונומי) מ-spec §9. ב-7.4 רלוונטי, כאן רק הסכמה.

### 4ג. RLS לשתי הטבלאות

- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` על שתיהן.
- מדיניות **SELECT**: `USING (auth.uid() = user_id)` — הלקוח רואה את ההיסטוריה שלו (תצוגת "תהליכי שיפור").
- **כתיבה — admin client בלבד.** הסוכן (server) כותב, לא הלקוח. אין policy ל-INSERT/UPDATE/DELETE → ברירת מחדל deny. הכתיבה דרך service ייעודי ב-7.4/Phase 8.
- `GRANT SELECT ON optimization_sessions, optimization_actions TO authenticated`. בלי insert/update/delete ל-authenticated.

> **למה כתיבה דרך admin/service ולא הלקוח:** בניגוד ל-`lead_status` (בבעלות הלקוח), פעולות אופטימיזציה הן server-authoritative לחלוטין — הסוכן מחליט, מבצע, ומתעד. הלקוח רק **רואה**. זה התרחיש הקלאסי של "RLS SELECT-only, כתיבה דרך service" (לא החריג של עמוד הלידים).

---

## חלק 5 — `benchmark_service` (דטרמיניסטי)

מודול חדש `app/services/benchmark_service.py`. **אין כאן LLM** — סיווג טהור (חוק 7).

**פונקציות:**

**א. `load_benchmarks() -> dict`:**
- טוען את טבלת ה-7 ענפים מ-`benchmarks.yaml` (או הבלוק ב-KB — לפי החלטה עם גיא).
- ממופה ל-`{industry_key: {amazing: (min,max), average: (min,max), expensive_above: N}}`.
- caching בזיכרון (הקובץ סטטי; טעינה פעם אחת).

**ב. `classify_cpl(industry: str | None, cpl: float) -> BenchmarkResult`:**
- אם `industry is None` → מחזיר `level='unknown'`. **חוק הברזל מדלג** (fail-safe, החלטה 2). הסוכן לא יעצור את הלקוח ולא יציג השוואה.
- אחרת, משווה את `cpl` לטווחי הענף ומחזיר `level ∈ {amazing, average, expensive}`.
- מבנה: `{level, industry, cpl, range_for_level}` — הסוכן ב-7.4 מקבל את **המסקנה** (לא את הטבלה), ורק מנסח.

> זה בדיוק חוק 7: הקוד מחליט "מדהים/ממוצע/יקר/לא-ידוע", ה-LLM של 7.4 רק עוטף את זה בניסוח משכנע ("אתה בטופ של הענף, לא לגעת").

**ג. `get_industry_context(industry) -> str`** (אופציונלי, ל-7.4):
- מחזיר את הטקסט החופשי הרלוונטי מה-KB (עונתיות, פרדוקס איכות) להזרקה ל-prompt. ב-7.3.5 רק ה-stub; השימוש ב-7.4.

---

## חלק 6 — שינויים נדרשים בקוד הקיים

**א.** `docs/agent/KNOWLEDGE_BASE.md` + `benchmarks.yaml` — חדשים (חלק 1).
**ב.** `ad_generation_service.py` — הרחבת `BusinessContext` dataclass + `extract_business_context` (ולידציה רכה ל-industry) + `get_or_extract_business_context` (כתיבה ל-`campaigns.industry`). **patch ל-3.1.6.**
**ג.** `app/prompts/phase3/extract_business_context.txt` — הוספת שדה `industry` + הוראת null. **patch ל-3.1.6.**
**ד.** migration `campaigns.industry` (חלק 3).
**ה.** migration שתי טבלאות (חלק 4).
**ו.** `benchmark_service.py` — חדש (חלק 5).
**ז.** `models/` — `BenchmarkResult`, ומודלים ל-2 הטבלאות (לקריאה; כתיבה ב-7.4).

---

## חלק 7 — בדיקות

**זיהוי ענף (3.1.6 מורחב):**
1. `service_name="טיפולי פנים"` → `industry='beauty'`.
2. `service_name="שיעורי נהיגה"` → `industry='courses'`.
3. `service_name="ייעוץ אישי"` (עמום) → `industry=None`, בלי שגיאה.
4. ה-LLM מחזיר ערך לא ברשימה (הזיה) → מטופל כ-`None`.
5. שלושת השדות הראשונים ריקים → `BusinessContextExtractionError` (כמו קודם). `industry` ריק → לא זורק.
6. `get_or_extract_business_context` → `extracted_context.industry` **וגם** `campaigns.industry` מתמלאים באותה פעולה. `None` → שניהם NULL.
7. caching: קריאה שנייה → לא מריצה LLM, `industry` כבר ב-`extracted_context`.

**`benchmark_service`:**
1. `classify_cpl('beauty', 22)` → `amazing`.
2. `classify_cpl('beauty', 35)` → `average`.
3. `classify_cpl('beauty', 60)` → `expensive`.
4. `classify_cpl(None, 60)` → `unknown` (חוק הברזל מדלג).
5. גבולות: `classify_cpl('beauty', 25)` → `amazing`; `26` → `average` (לוודא שאין חור בין הטווחים).
6. `load_benchmarks` טוען את כל 7 הענפים.

**טבלאות (סכמה + constraints):**
1. INSERT session תקין → מצליח.
2. שתי sessions פתוחות עם אותו `(campaign_id, problem_type='high_cpl')` → השנייה נכשלת (unique index).
3. session `high_cpl` + session `low_quality_leads` לאותו קמפיין, שתיהן פתוחות → מצליח (problem_type שונה).
4. session שעברה ל-`done` → אפשר לפתוח חדשה לאותה בעיה (ה-index חל רק על פתוחות).
5. `problem_type` לא חוקי → CHECK דוחה.
6. action עם `session_id` של user אחר → composite FK / RLS חוסם.
7. RLS: authenticated רואה רק sessions/actions שלו (SELECT). ניסיון INSERT כ-authenticated → deny.

---

## חלק 8 — לא ב-7.3.5

- **לוגיקת הסוכן** (אבחון, הצעת פתרון) — 7.4.
- **Lock Mechanism** (הבדיקה עצמה) — 7.4 (משתמש ב-`window_ends_at` + ה-index שנבנו כאן).
- **כתיבת פעולות אמיתיות** — 7.4/Phase 8. כאן רק סכמה + helpers לקריאה.
- **benchmarks לשאר 3 המטריקות** — רק CPL (ראה פתוח לגיא).
- **לוגיקה על עונתיות/troubleshooting triggers** — נשמרים כטקסט ל-prompt, בלי קוד.
- **מילוי `industry` רטרואקטיבי לקמפיינים קיימים** — קמפיינים שנוצרו לפני 7.3.5 יישארו `industry=NULL` עד שירוצו שוב דרך 3.1.6. אם צריך backfill — סשן נפרד. (ב-MVP אין הרבה קמפיינים — לא חוסם.)

---

## חלק 9 — פתוח לגיא (לפני 7.4, לא חוסם את 7.3.5)

1. **benchmarks רק ל-CPL?** הסוכן מקבל 4 מטריקות, רק CPL מקבל benchmark. האם שאר השלוש (אחוז לחיצה, שיעור פניות, פניות יום) צריכות סף כלשהו, או ש-CPL מספיק לחוק הברזל? (מסמך הפירוק סימן את זה כפתוח.)
2. **`benchmarks.yaml` נפרד או טבלה ב-KB?** איזו עדיפות — קובץ פרסור נפרד + MD לתיעוד, או פרסור ישיר של טבלת ה-markdown.
3. **אחרי "אני בכל זאת רוצה לשנות"** — סגרנו override פשוט (אופציה א), אבל מה קורה מיד אחריו: הסוכן נכנס לפרוטוקול הרגיל (שלב 2)? זה משפיע על 7.4, לא על 7.3.5.

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **אין `optimization_actions` קיים — בונים מאפס.** spec §6 הגדיר טבלה אחת; כאן מפצלים לשתיים (החלטה 1). זה מחליף את ההגדרה ב-spec — לעדכן את spec §6 בהתאם אחרי המימוש.

2. **`industry` הוא best-effort, היחיד שמותר NULL.** ב-`BusinessContext` הוא לא בוולידציה הקשיחה. בכל מקום שצורך אותו — לטפל ב-`None` (חוק הברזל מדלג, לא קורס).

3. **denormalize ל-`campaigns` קורה ב-`get_or_extract_business_context`, באותו admin-write.** לא join. עקבי עם `service_name`.

4. **3 נקודות המטריקה מפוצלות בין הטבלאות:** `starting_metric` ב-session (פעם אחת לסדרה), `post_change_metric`+`current_metric` ב-action. זה מונע שכפול של ה-X בכל שורת פעולה.

5. **ה-partial unique index הוא הליבה.** בלעדיו, "מה הסדרה הפעילה לבעיה הזו" יכול להחזיר כמה שורות, וה-Lock נשבר. לוודא שהוא על `status IN ('in_progress','success_monitoring')` בלבד.

6. **120 שעות, לא 96.** `window_ends_at` מחושב כ-5 ימים (תיקון 1 במסמך התיקונים). לא לכתוב 4 בשום מקום.

7. **כתיבה דרך admin/service, קריאה דרך RLS.** שתי הטבלאות הן server-authoritative (בניגוד ל-`lead_status`). אין column grant לכתיבה ללקוח.

8. **`benchmark_service` דטרמיניסטי לחלוטין.** אם מוצאים את עצמנו שולחים benchmark ל-LLM כדי "להחליט" רמה — זו טעות. הקוד מסווג, ה-LLM מנסח.

9. **Sentry context.** exceptions ב-`extract_business_context` (כולל industry) → `quiz_response_id`, `campaign_id`.

10. **מספרי migration תלויים בסדר הבנייה** (אתה ב-7.1). לתאם ב-ROADMAP בזמן ההעברה.

---


# Session 7.4א — בעיה 1: חוק הברזל, Lock ושלב האבחון ✅

> **Done ✅:** orchestrator דטרמיניסטי (חוק 7 כקוד) — ה-endpoint לא קורא ל-LLM ישירות. מומש: `lock_service`
> (check_lock, 2 שאילתות), `optimization_service` (open_session race-safe + get_active_or_last + build_
> mixed_analysis), `agent_orchestrator.diagnose_problem_1` (מטריקות → Lock → benchmark → היסטוריה →
> פתיחת סדרה → LLM לניסוח → עצירה לפני פתרון), `POST /me/agent/conversations/{id}/diagnose`,
> `prompts/agent/problem_1.txt` (הפרומפט המלא של גיא), models (`Chip`/`DiagnoseRequest`/`DiagnoseResponse`),
> ו-`benchmark_service.format_range_he`. הרחבות: `agent_service` (`industry` ב-`_campaign_for_chat`+
> `AgentCampaignStatus`, `save_assistant_message`). **migration 0052** (סטייה מתוכננת מה-ROADMAP "אין
> migration"): 2 RPCs בלבד — `open_optimization_session` (advisory lock, כי ה-index חלקי) +
> `insert_agent_assistant_message` (assistant בלבד, בלי quota). 3 תוצאות: lock / benchmark_stop / diagnosis
> (+ unavailable כ-fail-safe). נדחה ל-7.4ב: שלבים 2-7, כתיבת optimization_actions, מעקב 120ש'. בדיקות:
> 23 חדשות (lock/optimization/orchestrator e2e/endpoint/prompt-render). RLS+RPCs נבדקים ידנית מול Supabase.
> **מיקום:** 7.4א — החצי הראשון של בעיה 1 ("עלות לליד גבוהה / מעט פניות"). 7.4ב (מנוע הפרוטוקול המלא) בא אחריו.
> **תלוי ב-7.3.5** (benchmarks + `industry` + 2 הטבלאות) ו-**7.2** (כרטיס מטריקות). בונה עליהם את שכבת ההחלטה הראשונה של הסוכן.

---

## מבוא ותיאור Session

ב-7.2 הסוכן עונה כללית עם מטריקות. ב-7.4א הוא מתחיל **לנהל** — הצ'יפ הראשון, "עלות לפנייה גבוהה", מפעיל את שלב האבחון של בעיה 1.

7.4א בונה את **שלוש שכבות ההחלטה הדטרמיניסטיות** שרצות לפני שה-LLM בכלל נכנס לתמונה:
1. **Lock Mechanism** — אם הקמפיין כבר בטיפול לבעיה הזו (בחלון 120 שעות) — עצור, אל תאפשר תהליך חדש.
2. **חוק הברזל** — השווה את ה-CPL ל-benchmark הענף. אם "מדהים"/"ממוצע" — עצור את הלקוח ("אתה בטופ, לא לגעת").
3. **שלב האבחון** — אם באמת חורג, הצג ניתוח (פשוט בתלונה ראשונה, מעורב X→Y→Z אם יש היסטוריה) ו**פתח סדרה** — אבל **עצור לפני** הצעת פתרון (זה 7.4ב).

המטרה: בסוף 7.4א, לקוח שלוחץ "עלות גבוהה" מקבל אחת משלוש תגובות — "נעול, חכה" / "אתה בטופ, אל תיגע" / "אכן חורג, הנה הניתוח, ממשיכים" — והכל מבוסס נתונים אמיתיים מ-Insights ומהטבלאות. **יצירת הקופי וההמשך לשלבים 2-7 — ב-7.4ב.**

**העיקרון הארכיטקטוני המרכזי (חוק 7 כקוד):**
ה-endpoint של הצ'אט (מ-7.1) **לא קורא ישר ל-LLM יותר** כשהצ'יפ נלחץ. הוא עובר דרך **orchestrator דטרמיניסטי** שמריץ את סדר החשיבה הקשיח: בדיקת Lock → סיווג benchmark → שליפת היסטוריה → ורק אז, עם כל המסקנות ביד, בונה prompt וקורא ל-LLM **לניסוח בלבד**. ה-LLM הוא התחנה האחרונה, לא הראשונה.

> **על הפרומפט:** מאחר שאנחנו לא בפרודקשן, מזריקים את **החלק המלא של בעיה 1 מהפרומפט של גיא** כמו שנכתב — בלי לגזור אותו לחתיכות "זמניות" ובלי הסתייגויות-ביניים. הסוכן יכול "לדבר" על כל 7 השלבים; הקוד פשוט לא יאכוף את שלבים 2-7 עד 7.4ב. פרומפט מלא, קוד מדורג.

**מה בסשן:**
- `agent_orchestrator` — מנתב הצ'יפ דרך סדר החשיבה (Lock → benchmark → היסטוריה → LLM).
- `lock_service` — בדיקת Lock פר-`(campaign_id, problem_type)`.
- שילוב `benchmark_service` (7.3.5) — סיווג ה-CPL.
- `optimization_service` — פתיחת סדרה (`optimization_sessions`) + שליפת היסטוריה (הניתוח המעורב).
- endpoint לטיפול בצ'יפ: `POST /me/agent/conversations/{id}/diagnose`.
- system prompt: החלק של בעיה 1 מהפרומפט של גיא + הזרקת המסקנות הדטרמיניסטיות.

**מה לא בסשן (7.4ב):**
- שלבים 2-7: יצירת 3 קופי/כותרות/זוויות, אישור, העלאה ל-Meta.
- מעקב 120 שעות + מדידה + מעבר שלב.
- חוק הלולאה (loop_count).
- כתיבת `optimization_actions` (פעולות בודדות) — כאן רק `optimization_sessions` נפתחת.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | תלונה ראשונה (אין היסטוריה) | **אבחון פשוט**, בלי ניתוח מעורב. "העלות ₪X, מעל ממוצע השוק בענף". פותח סדרה חדשה. הניתוח המעורב (X→Y→Z) רץ רק כשיש סדרה קודמת |
| 2 | Lock לפי בעיה או קמפיין | **פר-`problem_type`.** סדרת `high_cpl` פתוחה בחלון → נעול ללחיצת "עלות גבוהה". סדרת `low_quality_leads` פתוחה → **לא** נעול (בעיות בלתי-תלויות). תואם ל-partial unique index מ-7.3.5 |
| 3 | מי מחליט (LLM מול קוד) | **קוד מחליט, LLM מנסח.** Lock/benchmark/היסטוריה/מצב — דטרמיניסטי. ה-LLM מקבל מסקנות ועוטף בניסוח. orchestrator מריץ את הסדר |
| 4 | `industry=NULL` | **חוק הברזל מדלג.** אין benchmark → הסוכן לא עוצר ולא משווה, עובר ישר לאבחון/המשך. (fail-safe מ-7.3.5) |
| 5 | override ("בכל זאת רוצה לשנות") | **פשוט** (אופציה א). הלחיצה מסירה את העצירה וממשיכה. *מה* קורה אחריה (כניסה לשלב 2) — 7.4ב |
| 6 | סנכרון מול "הסוכן היוזם" | ב-MVP אין סוכן יוזם (זה Phase 8). ה-Lock בודק רק סדרות שנפתחו דרך הצ'אט. כשייבנה Phase 8 — אותו Lock, אותן טבלאות, בלי שינוי |

---

## תלויות

1. **7.3.5** — `benchmark_service`, `campaigns.industry`, `optimization_sessions`+`optimization_actions`. **הליבה.**
2. **7.2** — `get_campaign_status_for_agent` (CPL חי מ-Insights). ה-benchmark מסווג את המספר הזה.
3. **7.1** — תשתית הצ'אט, `agent_conversations`, `send_user_message`.
4. **3.4** — קמפיין חי עם `meta_*_id` (לפעולות עתידיות ב-7.4ב; כאן רק קריאת מטריקות).

---

## חלק 1 — `lock_service` (בדיקת Lock)

מודול חדש `app/services/lock_service.py`. **דטרמיניסטי לחלוטין.**

**`check_lock(campaign_id, problem_type) -> LockStatus`:**
- שאילתה: סדרה פתוחה ל-`(campaign_id, problem_type)` שעדיין בחלון מדידה.

```sql
SELECT s.id, a.window_ends_at
FROM optimization_sessions s
JOIN optimization_actions a ON a.session_id = s.id
WHERE s.campaign_id = :campaign_id
  AND s.problem_type = :problem_type
  AND s.status IN ('in_progress', 'success_monitoring')
  AND a.window_ends_at IS NOT NULL
  AND now() < a.window_ends_at
ORDER BY a.created_at DESC
LIMIT 1;
```

- אם יש תוצאה → `{locked: true, window_ends_at, remaining_hours}`.
- אם אין → `{locked: false}`.
- **החריגים** (6 מהפרומפט): `meta_rejection`, תקלה טכנית, טופס לא עובד, חיבור לא תקין, קמפיין עצר, חשבון חסום — **לא נעולים גם בחלון**. ב-7.4א רלוונטי רק החריג של `problem_type` שונה (כבר מטופל ע"י המפתח). שאר החריגים (תקלות טכניות) — Phase 8/7.6. החריג ה-7 (override של הלקוח) — מטופל ב-orchestrator (החלטה 5).

> **על המפתח:** ה-Lock פר-`(campaign_id, problem_type)` — בדיוק מה שה-partial unique index מ-7.3.5 מבטיח (סדרה פתוחה אחת לכל צירוף). לכן השאילתה תמיד מחזירה לכל היותר סדרה אחת.

---

## חלק 2 — `optimization_service` (סדרה + היסטוריה)

מודול חדש `app/services/optimization_service.py`. כתיבה דרך **admin client** (server-authoritative, 7.3.5).

**א. `get_active_or_last_session(campaign_id, problem_type) -> Session | None`:**
- שולף את הסדרה הפתוחה ל-`(campaign_id, problem_type)`, או — אם אין פתוחה — את האחרונה שנסגרה ב-30 הימים האחרונים (לניתוח המעורב).
- מחזיר את הסדרה + הפעולה האחרונה שלה (עם `starting_metric`, `post_change_metric`, `current_metric`).

**ב. `open_session(campaign_id, user_id, problem_type, starting_metric) -> Session`:**
- יוצר סדרה חדשה ב-`optimization_sessions` (`status='in_progress'`, `current_step=1`, `starting_metric` = ה-CPL הנוכחי + מטריקות).
- **race-safe:** ה-partial unique index מבטיח שלא ייפתחו שתי סדרות פתוחות לאותה בעיה. `INSERT ... ON CONFLICT DO NOTHING` + SELECT הקיים אם conflict.
- מחזיר את הסדרה.
- **חוזה ה-`starting_metric` (עודכן ב-migration 0055, תיקון Cursor#1):** אם לסדרה פתוחה קיימת **עדיין אין** שורות ב-`optimization_actions` (ה-X "מרחף") — ה-RPC מרענן את `starting_metric` ל-CPL העכשווי שעבר בקריאה. אם כבר יש action — ה-baseline ננעל (שינוי ישבור את ההשוואה של ה-monitor 120h שמסיק improved/regressed מול ה-X שהיה בזמן ה-push). הבדיקה והעדכון בתוך אותו `pg_advisory_xact_lock` (אטומי, חוק 1).

**ג. `build_mixed_analysis(session) -> MixedAnalysis | None`:**
- אם לסדרה (או לסדרה קודמת) יש פעולה עם `starting_metric`/`post_change_metric`/`current_metric` → מרכיב את ה-X→Y→Z.
- `current_metric` — **lazy refresh**: נשלף מ-Insights ברגע הזה (CPL עכשווי) ונשמר.
- מחזיר `{x: starting, y: post_change, z: current}` או `None` (תלונה ראשונה — החלטה 1).

---

## חלק 3 — `agent_orchestrator` (סדר החשיבה כקוד)

מודול חדש `app/services/agent_orchestrator.py`. **זה הלב של הסשן** — מתרגם את סדר החשיבה הקשיח של גיא לרצף קוד דטרמיניסטי, ורק בסוף קורא ל-LLM.

**`diagnose_problem_1(user_id, conversation_id, campaign_id) -> AgentResponse`:**

הרצף (סדר החשיבה: אבחון → השוואה → זיכרון → דעה → הסבר):

1. **שליפת מטריקות.** `get_campaign_status_for_agent(campaign_id)` (7.2) → CPL חי + שאר.
2. **Lock.** `lock_service.check_lock(campaign_id, 'high_cpl')`.
   - אם `locked` → **עצור.** מחזיר תגובת Lock (הנוסח הקבוע מהפרומפט + `remaining_hours`). **לא קורא ל-LLM** — הנוסח קבוע (אפשר template, או LLM שמנסח מסר קבוע; המלצה: קבוע, חוק 7). מציג כפתור "חזרה לתפריט".
3. **benchmark.** `benchmark_service.classify_cpl(campaign.industry, cpl)`.
   - אם `level='unknown'` (industry NULL) → דילוג על חוק הברזל (החלטה 4). ממשיך ל-(5) כאילו חורג.
   - אם `level ∈ {amazing, average}` → **תרחיש א'.** הלקוח בטופ. בונה prompt עם המסקנה "עצור אותו" → LLM מנסח את העצירה (הנוסח מהפרומפט) → מציג כפתורים `[אני מקבל את ההמלצה]` / `[אני בכל זאת רוצה לשנות]`. **לא פותח סדרה.**
   - אם `level='expensive'` → **תרחיש ב'.** ממשיך.
4. **היסטוריה.** `optimization_service.get_active_or_last_session(campaign_id, 'high_cpl')` + `build_mixed_analysis`.
5. **פתיחת סדרה.** אם אין סדרה פתוחה → `open_session(...)` עם ה-CPL הנוכחי כ-`starting_metric`.
6. **ניסוח (LLM).** בונה prompt עם המסקנות: ה-CPL, הרמה (`expensive`), והניתוח (מעורב X→Y→Z אם יש, פשוט אם אין) → LLM מנסח את האבחון ללקוח.
7. **עצירה לפני פתרון.** מחזיר את האבחון, **בלי** להציע קופי. ההמשך (שלב 2) — 7.4ב. ב-7.4א הסוכן מסיים ב"זיהיתי שהעלות חורגת, [ניתוח]. אני מכין הצעת שיפור" — וכאן 7.4א נעצר.

**טיפול ב-override (החלטה 5):** אם הלקוח לחץ `[אני בכל זאת רוצה לשנות]` (מתרחיש א') → ה-orchestrator מדלג על חוק הברזל ונכנס ל-(4)→(5)→(6) (פתיחת סדרה + אבחון), כאילו `expensive`. פשוט, בלי אזהרה נוספת.

> **למה ה-LLM אחרון:** ב-(2) ו-(3) ההחלטה (`locked`/`amazing`/`expensive`) היא **של הקוד**. ה-LLM מקבל אותה כעובדה ורק עוטף בניסוח. הוא לא רואה את הטבלה, לא מחשב רמה, לא מחליט לעצור. זה חוק 7 כקוד.

---

## חלק 4 — Endpoint

**`POST /me/agent/conversations/{conversation_id}/diagnose`**

- Body: `{problem_type: "high_cpl", override: false}`.
  - `problem_type` — ב-7.4א רק `high_cpl` נתמך. (`low_quality_leads`/`meta_rejection` → 7.5/7.6, יחזיר 422 "not_implemented" עד אז.)
  - `override` — true אם הלקוח לחץ "בכל זאת רוצה לשנות".
- Auth: JWT + RLS (conversation שייך ל-user, campaign שייך ל-user).
- שולף `campaign_id` מה-conversation, קורא ל-`agent_orchestrator.diagnose_problem_1`.
- Response: `{type: "lock"|"benchmark_stop"|"diagnosis", message, chips: [...], session_id?}`.
  - `lock` → הודעת המתנה + chip "חזרה לתפריט".
  - `benchmark_stop` → עצירת חוק הברזל + chips `[מקבל]`/`[בכל זאת לשנות]`.
  - `diagnosis` → האבחון + (ב-7.4ב יתווסף chip להמשך).
- שמירת ההודעות: כמו 7.1 — תגובת הסוכן נשמרת ב-`agent_messages` (role=assistant). **מכסת הצ'אט (`agent_chat_quota`) לא נספרת על לחיצת צ'יפ** — זה לא הודעת לקוח חופשית. (לאמת מול החלטת 7.1 על מה נספר.)

---

## חלק 5 — System Prompt (בעיה 1)

קובץ חדש `prompts/agent/problem_1.txt`. זהו תוכן הקובץ **במלואו** — הטקסט הקבוע של הפרומפט (פרטי הסוכן, 5 הכובעים, חוק הברזל, חוקי הזיכרון, חוקי השפה, ופרוטוקול בעיה 1 על 7 שלביו) + בלוקים שהקוד ממלא בזמן ריצה (ה-`{...}`). אין מה להשלים — זה הקובץ שמועבר ל-CC כמות שהוא.

> **על ה-placeholders `{...}`:** רק אלה ממולאים ע"י הקוד (`agent_orchestrator`/`prompts_service.build`) לפני השליחה ל-OpenAI. כל השאר טקסט קבוע. ה-LLM לא רואה את ה-`{...}` — הוא רואה את הערכים הסופיים.

````
אתה "המוח" (Autopilot Agent) – העוזר האסטרטגי, מנהל הקמפיינים הבכיר ומאמן
המכירות הפסיכולוגי של פלטפורמת "Campaign AI". התפקיד שלך לשמש כעוזר
האינטראקטיבי של הלקוח, הניזון ומסתנכרן בזמן אמת עם "הסוכן היוזם" (המערכת
האוטומטית שמנהלת את המדיה ומבצעת שינויים בקמפיינים).

### פרופיל ויכולות מקצועיות:
אתה משלב יכולות של מנהל קמפיינים מומחה, קופירייטר בכיר, אסטרטג שיווק, יועץ
עסקי, מומחה לפסיכולוגיית צרכנים, משפכי שיווק ומכירה והמרת לידים ללקוחות.
המטרה שלך אינה להביא יותר לידים בכל מחיר, אלא להביא יותר לקוחות מתאימים
לעסק תוך שמירה על רווחיות גבוהה ככל האפשר.
בעת קבלת החלטות עליך לחשוב כמו מנהל קמפיינים המנתח נתונים, פסיכולוג המבין
מה מניע אנשים לפעולה, קופירייטר המבין כיצד מסרים משפיעים על קהל יעד, איש
מכירות המזהה התנגדויות וחששות, ויועץ עסקי המבין את מטרות העסק.

### 🌟 חוק הברזל: סמכות מקצועית ואינדיקציית שוק ("אתה לא בובה")
אתה לא כאן כדי לבצע פקודות באופן עיוור. אתה "חי" את הקמפיין ואת העסק של
הלקוח, ותמיד שואף להביא אותו לטופ.
**המערכת כבר הצליבה עבורך את העלות לליד מול ממוצעי השוק וקבעה את הרמה
(ראה "מסקנות המערכת" למטה). אל תחשב את הרמה מחדש — פעל לפיה.**
לידיעתך בלבד, אלה טווחי השוק שלפיהם המערכת סיווגה:
- ביוטי, קוסמטיקה וטיפולים: מדהים 15-25 ₪ | ממוצע 26-45 ₪ | יקר מעל 50 ₪.
- בתי ספר, קורסים ומקצועות: מדהים 30-50 ₪ | ממוצע 51-90 ₪ | יקר מעל 100 ₪.
- כושר, ספורט ובריאות: מדהים 15-30 ₪ | ממוצע 31-55 ₪ | יקר מעל 60 ₪.
- שירותים לבית ועסקים מקומיים: מדהים 25-40 ₪ | ממוצע 41-70 ₪ | יקר מעל 85 ₪.
- עו"ד, רו"ח ופיננסים: מדהים 60-90 ₪ | ממוצע 91-150 ₪ | יקר מעל 160 ₪.
- נדל"ן, השקעות ותיווך: מדהים 70-100 ₪ | ממוצע 101-180 ₪ | יקר מעל 200 ₪.
- הייטק, B2B וגיוס עובדים: מדהים 80-120 ₪ | ממוצע 121-220 ₪ | יקר מעל 250 ₪.
אם המערכת סיווגה את העלות כ"מדהימה" או "ממוצעת ותקינה" ביחס לענף, חובתך
המקצועית לעצור את הלקוח ולהסביר שהקמפיין עובד מצוין, וששינוי כעת עלול
לסכן את יציבות האלגוריתם ולהעלות את המחיר.

### 🧠 חוקי זיכרון היסטורי ולוגיקת החלטות:
**המערכת מנהלת עבורך את הזיכרון ואת מצב הפרוטוקול (ראה "מסקנות המערכת").
אתה מנסח על בסיס מה שנמסר לך — אינך מחשב בעצמך אילו ימים עברו או איזה שלב
כבר נוסה.**
1. זיכרון 30 יום: כשלקוח מתלונן מחדש על בעיה, המערכת שולפת עבורך את השינוי
   האחרון לבעיה הזו ואת שלוש נקודות המדידה — העלות לפני השינוי, מיד אחריו,
   והעכשווית.
2. חוק השיפור: שיפור — אפילו של שקל — פירושו לעצור ולהמתין למדידה נוספת,
   לא להתקדם לשלב הבא.
3. חוק חוסר שיפור: רק כשאין שום שיפור, מתקדמים לשלב הבא.
4. חוק הלולאה: בסיום כל השלבים בלי פתרון, חוזרים לשלב 1.

### חוקי שפה והתנהגות קבועים:
1. פעל תמיד לפי סדר החשיבה: אבחון והשוואה לשוק ➔ ניתוח השינוי האחרון
   ב-30 יום ➔ דעה מקצועית ➔ הסבר ללקוח ➔ יצירת פתרון (רק במידת הצורך) ➔
   אישור לקוח ➔ ביצוע.
2. לעולם אל תבטיח תוצאות, ואל תציע פתרונות שאין להם הצדקה בנתונים.
3. דבר עברית מקצועית, שיווקית, חדה ובגובה העיניים.
4. איסור מוחלט על מונחים באנגלית. השתמש: "עלות לליד / עלות לפנייה",
   "כמות פניות / לידים", "אחוז האנשים שלחצו על המודעה", "מחיר לקליק",
   "מגמת ביצועים".

#### 🔴 בעיה 1 – כמות לידים נמוכה / עלות לליד גבוהה
• שלב 1 - אבחון, הערכת שוק והצלבת עבר: אתה מנתח את העלות לליד, אחוז האנשים
  שלחצו, המחיר לקליק, כמות הלידים ומגמת הביצועים, ומסתמך על נתוני 30 הימים
  והתיקון האחרון — כולם מסופקים לך ב"מסקנות המערכת".
  - תרחיש א' (העלות "מדהים" או "ממוצע ותקין"): עצור את הלקוח מיד. אמור:
    "אני מבין את השאיפה לשפר ולהגיע לטופ, אבל כמנהל הקמפיינים שלך חובתי
    המקצועית להגיד לך שהמספרים הנוכחיים שלך מעולים ביחס לממוצע בשוק לענף
    שלך. שינוי אגרסיבי עכשיו הוא הימור מסוכן – הוא עלול לבלבל את האלגוריתם,
    לפגוע ביציבות ולהעלות את עלות הליד משמעותית. אני ממליץ בחום לא לגעת
    כרגע ולתת לקמפיין להמשיך לעבוד." (כפתורים: [אני מקבל את ההמלצה] /
    [אני בכל זאת רוצה לשנות]).
  - תרחיש ב' (העלות "יקר", או תלונה חוזרת אחרי תקופת שיפור): אם יש היסטוריה,
    הצג ניתוח מעורב: "אני רואה שלפני התיקון האחרון עלות הליד הייתה X,
    התיקון הצליח והוריד אותה ל-Y, אך כעת בנתוני הלייב היא עומדת על Z.
    מכיוון שהשינוי האחרון מיצה את עצמו, אנחנו מתקדמים לשלב הבא." אם זו
    תלונה ראשונה (אין היסטוריה): "אני רואה שעלות הליד שלך היא X, מעל
    ממוצע השוק בענף שלך. אני מתקדם להכין פתרון." והמשך לשלב 2.
• שלב 2 - יצירת פתרון (שלב א'): צור 3 כותרות חדשות, 3 קופי חדשים ו-3 זוויות
  שיווק חדשות (כאב, חלום, דחיפות, הוכחה חברתית, או סמכות).
• שלב 3 - אישור לקוח: הצג מה זוהה, מה הפתרון, ואילו מודעות חדשות נוצרו.
• שלב 4 - העלאה: הלקוח מאשר ואתה מעלה את המודעות.
• שלב 5 - מעקב והחלטה (120 שעות / 5 ימים): בתום הזמן — אם יש שיפור (אפילו
  ₪1), עצור והמלץ לא לגעת 5 ימים נוספים; המערכת שומרת את הנתון ההתחלתי
  ונתון ההצלחה. אם אין שיפור — התקדם לשלב 6.
• שלב 6 - שינוי זווית (שלב ב'): צור זווית שונה לחלוטין מהרשימה שלא נוצלה.
  אישור, העלאה, ושוב 120 שעות מעקב.
• שלב 7 - שינוי הצעה שיווקית (שלב ג'): "לדעתי הבעיה אינה רק במודעות אלא
  בהצעה השיווקית עצמה." הצע הטבה/בונוס/ייעוץ חינם/הצעת ניסיון/שינוי המסר,
  וצור פוסטים, קופי וכותרות סביב ההצעה. [אם גם שלב 7 נכשל — חזור לשלב 1].

==========================================
מסקנות המערכת (דטרמיניסטי — אל תחשב מחדש)
==========================================

הקמפיין: {service_name}
הענף: {industry_he}                  # או "לא זוהה" אם אין ענף
עלות לפנייה נוכחית: {cpl} ₪
רמת השוק: {benchmark_level_he}        # "מדהים" / "ממוצע ותקין" / "יקר — דורש תיקון" / "לא ידוע"
{benchmark_range_he}                  # הטווח לרמה (אם זוהה ענף)

{mixed_analysis_block}
# אם יש היסטוריה:
#   "לפני התיקון האחרון: {x} ₪ | מיד אחרי: {y} ₪ | עכשיו: {z} ₪"
# אם אין:
#   "זו הפנייה הראשונה לבעיה זו — אין היסטוריית תיקונים."

==========================================
ההנחיה שלך עכשיו
==========================================

{scenario_instruction}
# תרחיש א' (מדהים/ממוצע): "המספרים מעולים. עצור את הלקוח לפי הנוסח של תרחיש א'."
# תרחיש ב' (יקר / ענף לא זוהה): "העלות חורגת. הצג את הניתוח (מעורב אם יש
#   היסטוריה, פשוט אם לא) והסבר שאתה מתקדם להכין פתרון."
# בשני המקרים: **אל תיצור את הקופי עכשיו.** רק הצג את האבחון; יצירת הפתרון
#   (שלב 2) תופעל בנפרד.
````

**הערות:**
1. **המסקנות מוזרקות כעובדות.** הקוד כבר החליט `benchmark_level` (דרך `benchmark_service`). ה-LLM לא משווה ולא מחשב — הוא מנסח. אם בבדיקות נראה שה-LLM "מתווכח" עם הרמה או מחשב מחדש מהטבלה — לחזק את "אל תחשב מחדש".
2. **טבלת השוק נשארת בפרומפט כרקע בלבד.** ה-LLM פועל לפי `{benchmark_level_he}` שהוזרק, לא לפי הטבלה. הטבלה שם רק כדי שהסוכן יבין את ההיגיון אם הלקוח שואל.
3. **הניסוחים הקבועים של גיא** (עצירת חוק הברזל, הניתוח המעורב) — מודבקים מילה במילה, כדי שהסוכן ינסח אותם עקבי.
4. **הפרומפט מלא — כולל שלבים 2-7** — למרות ש-7.4א אוכף בקוד רק את שלב 1 (אבחון). זה מכוון: לא בפרודקשן, אז אין טעם לגזור. הקוד פשוט לא יפעיל את שלבים 2-7 עד 7.4ב.
5. **בלי בלוק "מה אתה לא יודע (עדיין)"** — בניגוד ל-system_v1/v2 של 7.1/7.2. אין לקוח אמיתי, אין סיכון הבטחה. הפרומפט מלא ומקצועי.
6. **`{industry_he}`** — תרגום ה-key לעברית לתצוגה (`beauty`→"קוסמטיקה וטיפולים" וכו'). המיפוי ב-`agent_orchestrator`, לא ב-prompt.

---

## חלק 6 — שינויים נדרשים בקוד הקיים

**א.** `lock_service.py` — חדש (חלק 1).
**ב.** `optimization_service.py` — חדש (חלק 2).
**ג.** `agent_orchestrator.py` — חדש (חלק 3, הלב).
**ד.** `routers/agent.py` — endpoint `diagnose` (חלק 4).
**ה.** `prompts/agent/problem_1.txt` — חדש (חלק 5).
**ו.** `agent_service.py` — ה-`send_user_message` הרגיל (טקסט חופשי) נשאר; ה-diagnose הוא מסלול נפרד (chip → orchestrator). לוודא ששניהם שומרים ל-`agent_messages` עקבי.
**ז.** `models/` — `LockStatus`, `BenchmarkResult` (מ-7.3.5), `MixedAnalysis`, `AgentResponse`.

---

## חלק 7 — בדיקות

**`lock_service`:**
1. אין סדרה פתוחה → `locked=false`.
2. סדרת `high_cpl` פתוחה, `window_ends_at` בעתיד → `locked=true` + `remaining_hours`.
3. סדרת `high_cpl` פתוחה אבל `window_ends_at` עבר → `locked=false`.
4. סדרת `low_quality_leads` פתוחה, נשאל על `high_cpl` → `locked=false` (בעיה שונה).

**`benchmark` באורקסטרטור:**
5. CPL ₪22, ענף `beauty` → `amazing` → תרחיש א' (עצירה), לא נפתחת סדרה.
6. CPL ₪65, ענף `beauty` → `expensive` → תרחיש ב', נפתחת סדרה.
7. `industry=NULL` → `unknown` → דילוג חוק ברזל, ממשיך כמו expensive.
8. CPL גבולי ₪25 ב-`beauty` → `amazing` (גבול עליון של הטווח).

**`optimization_service`:**
9. תלונה ראשונה → אין סדרה קודמת → `build_mixed_analysis` מחזיר `None`, אבחון פשוט.
10. יש סדרה קודמת עם 3 מטריקות → ניתוח מעורב X→Y→Z.
11. `open_session` פעמיים במקביל לאותה בעיה → רק אחת נפתחת (unique index), השנייה מקבלת את הקיימת.

**orchestrator (e2e):**
12. צ'יפ "עלות גבוהה", קמפיין נעול → תגובת `lock`, בלי קריאת LLM, בלי פתיחת סדרה.
13. צ'יפ, CPL מעולה → `benchmark_stop`, chips `[מקבל]`/`[בכל זאת]`.
14. `[בכל זאת לשנות]` (override) → מדלג חוק ברזל, פותח סדרה, מחזיר אבחון.
15. צ'יפ, CPL חורג, תלונה ראשונה → `diagnosis` פשוט, סדרה נפתחה.
16. צ'יפ, CPL חורג, יש היסטוריה → `diagnosis` עם X→Y→Z.

**endpoint:**
17. `diagnose` עם `problem_type='low_quality_leads'` → 422 not_implemented (עד 7.5).
18. `diagnose` על conversation/campaign לא שלי → 404 (RLS).

---

## חלק 8 — לא ב-7.4א (7.4ב)

- **שלבים 2-7 של בעיה 1:** יצירת 3 קופי/כותרות/זוויות (LLM), אישור לקוח, העלאה ל-Meta (אטומי, כמו 3.4).
- **כתיבת `optimization_actions`** (פעולה בודדת עם `window_ends_at`). ב-7.4א רק `optimization_sessions` נפתחת; הפעולה הראשונה נכתבת ב-7.4ב כשמבצעים שלב.
- **מעקב 120 שעות + מדידה** + מעבר שלב (חוק השיפור/אי-שיפור).
- **חוק הלולאה** (`loop_count++`).
- **קטגוריות אישור/אוטונומי** בפועל (`requires_approval`) — הסכמה קיימת מ-7.3.5, השימוש ב-7.4ב.

---

## חלק 9 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **ה-orchestrator הוא הלב — ה-LLM אחרון.** הסדר Lock → benchmark → היסטוריה → LLM הוא חוק 7 כקוד. אם מוצאים את עצמנו שולחים ל-LLM "תחליט אם נעול/חורג" — טעות. הקוד מחליט, ה-LLM מנסח.

2. **ה-Lock פר-`problem_type`, לא פר-קמפיין** (החלטה 2). תואם ל-partial unique index מ-7.3.5. סדרות בעיות שונות בלתי-תלויות.

3. **תלונה ראשונה ≠ תלונה חוזרת** (החלטה 1). `build_mixed_analysis` מחזיר `None` בראשונה. הסוכן מאבחן פשוט. אסור שה-LLM "ימציא" X→Y→Z כשאין היסטוריה.

4. **`industry=NULL` → חוק הברזל מדלג** (החלטה 4). לא לעצור לקוח על benchmark שלא קיים. fail-safe.

5. **נוסחי הביניים לא חשובים (לא פרודקשן).** הפרומפט מלא, בלי "מה אתה לא יודע (עדיין)". הקוד מדורג — שלבים 2-7 פשוט לא ייקראו עד 7.4ב.

6. **הניסוחים הקבועים של גיא** (Lock, עצירת חוק ברזל) — מילה במילה בפרומפט/template. עקביות חשובה גם בלי פרודקשן.

7. **מכסת הצ'אט לא נספרת על לחיצת צ'יפ.** לאמת מול 7.1 (שם רק role=user של טקסט חופשי נספר). לחיצת צ'יפ היא פעולה מובנית, לא הודעה.

8. **אין "סוכן יוזם" ב-MVP** (החלטה 6). ה-Lock בודק רק סדרות מהצ'אט. כש-Phase 8 ייבנה — אותו Lock, אותן טבלאות.

9. **Sentry context.** exceptions ב-orchestrator → `conversation_id`, `campaign_id`, `problem_type`, השלב ברצף שבו נפל.

10. **מספרי migration/סדר** — אין migration חדש ב-7.4א (משתמש בטבלאות מ-7.3.5). רק קוד ופרומפט.

---

# Session 7.4ב — בעיה 1: מנוע הפרוטוקול (פתרון, העלאה, מעקב, שלבים) — פוצל ל-3

> **פוצל ל-3 (אישור המשתמש — הסשן ענק ומסוכן).**
> **7.4ב-1 ✅ Done:** יצירת הפתרון — `solution_service` (3 וריאציות קופי, **רק טקסט**; 5 זוויות פרוטוקול
> נפרד מ-3.2 + prompt `solution_copy`), endpoint `POST .../propose`, models (Propose/SolutionVariation).
> reuse: ה-LLM seam + `get_or_extract_business_context`; הוספות admin: `quiz_service.fetch_quiz_for_
> campaign` + `optimization_service.get_session_by_id` + `agent_service.get_owned_campaign_id`. **אין
> migration, אין שינוי ל-3.2.** הוריאציות לא נכתבות כפעולה (נשלחות חזרה ב-approve).
>
> **עדכון (0093 — save-proposed):** ה-propose **שומר** את הוריאציות כ-action עם `push_status=NULL`
> (`approved_at=NULL` — "מוצע, טרם אושר") דרך RPC `save_proposed_action`, ומחזיר `action_id` ב-ProposeResponse;
> ה-approve תופס את ה-action הזה (`open_optimization_push_action` claim NULL→pushing) במקום ליצור חדש — כך
> עמוד-אישור עתידי שולף את הוריאציות לפי id (לא רק in-memory ב-frontend). `screening_question` נשמר בעמודה
> חדשה (low_quality שלב 4). monitor (`push_status='pushed'`) ו-Lock (`window>now`) מסננים NULL — ללא שינוי.
> **known-gap:** propose-action(step N) נשאר יתום אם ה-step התקדם בין propose↔approve (advance מתעלם, לא מזיק).
>
> **הוראה 2 (עמוד-אינטראקציה) ✅:** `GET /me/actions/{id}` (תצוגה — `approval_state` נגזר ב-server: `pending`=מצב-1
> אישור / `done`/`in_progress`/`failed`=מצב-3 תצוגה; כלל 6 — בלי push_status raw) + `GET /me/sessions/{id}`
> (SERIES_RESOLVED) + `POST /me/actions/{id}/approve` (אישור-פאנל; variations מה-action השמור — "רק אשר"). ה-push-flow
> חולץ ל-`optimization_approve_service.execute_approval` (chat+panel = 2 endpoints דקים מעל core אחד;
> advance→binding→dispatch→finish_offer; CAS-lease/idempotency נשמר). **AD_REJECTED=אישור (מצב 1)** — ה-worker יוצר
> action ב-`pushing`+`attempt_started_at=NULL` (ממתין-לאישור), ה-approve תופס את ה-lease (0079:77-84). RLS ל-GET
> (user-client, כמו leads) — **אין migration** (policy+grant קיימים ב-0051). frontend: deep-link
> `/action/{id}`+`/session/{id}` (SPA fallback) → `view-action`/`view-review` + auth-resume. router
> `optimization_views`, models Action/SessionDetailResponse.
>
> **הוראה 2.5 (ערוך+רענן) ✅:** עריכת draft inline (`PATCH /me/actions/{id}` → RPC `edit_proposed_variations`/0094,
> CAS על `push_status IS NULL` + owner-check — כי `optimization_actions` הוא SELECT-only ל-authenticated, אין UPDATE
> policy) + רענן (`POST /me/actions/{id}/refresh` → `generate_solution`, idempotent על אותו action). flags
> `editable`(=push_status NULL)/`refreshable`(=editable, לא meta_rejection, לא offer_change) ב-ActionDetailResponse (server-computed,
> כלל 6). **`editable`⊊`pending`** — rejection-fix של ה-worker הוא pending אך לא editable (אישור-בלבד, חוק 7). binding:
> אם cron קידם step בין הצגה לרענון (`result.step_number≠action`) → 409. frontend: ערוך per-card (reuse `wizEditCopy`,
> input/textarea עם `.value` — XSS-safe) + רענן page-level; flags קובעים הצגה. ה-אישור לא השתנה (קורא מ-action.variations).
>
> **הוראה 3 (מייל-מעקב + מצב 2) ✅:** `QUALITY_FOLLOWUP` — מייל 120ש' פר-שלב ל-low_quality. ה-cron שולח אותו
> כשהחלון נגמר (במקום מדידה — אין מטריקה אוטומטית) ומסמן `feedback_requested_at`+`window_ends_at=NULL` (CAS
> `mark_feedback_requested_cas`; **send-לפני-clear**, idempotent על UNIQUE anchor). `compute_approval_state` גוזר
> **`awaiting_feedback`** (pushed + requested + response NULL — API-only, לא ב-CHECK) → **מצב 2** בעמוד-האינטראקציה
> ("השתפר? כן/לא"). `POST /me/actions/{id}/feedback` (CAS `record_feedback_cas`, retry-safe): כן→`set_session_status('done')`;
> לא→`generate_solution` מייצר את השלב הבא (advanced+next_action_id, מצב 1) / `StepsExhausted` אחרי שלב 4 → exhausted.
> **לינק anchor-aware** (`_anchor_link`, opt-in): `quality_followup`→`/action/{id}`, `series_resolved`→`/session/{id}`;
> `step_advanced`/`ad_rejected` נשארים `/campaigns` (ה-anchor שלהם הוא ה-action הסגור). תיקון: `_NOTIFICATION_COLUMNS`
> בוחר `anchor_id`. push services (creative+screening) חזרו ל-window=120h. migration 0095 (2 עמודות + הרחבת CHECK).
> **COPY_PENDING_APPROVAL בוטל** — הצ'אט (mock) יקשר לעמוד-הייעודי כשיהפוך אמיתי (Phase 3.2); אין מייל/type לקופי.
>
> **הוראה 4 (חיווט צ'אט-האופטימיזציה) ✅:** הצ'אט החי (`index.html`) היה **mock דמו רב-לקוחות** (`CLIENTS={}`,
> `activeClient`, הצעות מזויפות) — חוּוט ל-backend האמיתי (כל ה-endpoints כבר היו בנויים). מקור הקמפיינים =
> `GET /me/agent/conversations` (שיחה פר-קמפיין live/paused; **אין `GET /campaigns`** — ה-frontend לא החזיק
> `campaign_id`). הזרימה: bootstrap (conversations → קמפיין/פיקר) → `diagnose {problem_type}` (high_cpl /
> low_quality דו-שלבי subcategory) → `propose` → **לינק שמפעיל `openActionView(action_id)`** → עמוד-הפעולה
> המחווט (אישור/עריכה/רענן — Phase 3.2 כפי שתוכנן, DRY). טקסט-חופשי → `messages` (+STAGE_C ג'-2 inline proposal).
> `quota-status` → כרטיס המכסה. **meta_rejection מחוץ לצ'אט** (מטופל מייל→view-action). 6 מתודות api.js חדשות;
> הוסר ~418ש' קוד-דמו מת (CLIENTS/chatCtx/incMsg/הרנדררים), נשמרו `addOptLog`/`checkFourDayRule` (הפניות חיות).
> frontend-only — ללא שינויי backend/migration/pytest (אימות: `node --check` + חילוץ ה-inline script + ידני).
>
> **הוראה 5 (חיווט הקופי ההתחלתי — Dream Wizard) ✅:** ה-wizard היה **mock מלא** (3 תבניות קופי קשיחות,
> edit/refresh/publish מזויפים, **בלי ליצור קמפיין/quiz/Meta**). חוּוט לקופי אמיתי (scope=**קופי בלבד**): סיום
> ה-wizard (`wizStartLoading`) → `POST /campaigns` (type + fb account/page מ-`APP.account/page` שמ-Meta) →
> `POST /campaigns/{id}/quiz` (`_buildQuizBody`: `WIZ.data`→`QuizResponseInput`, מיפוי tone עברית→enum, total→lifetime,
> name→area) → `generate-copy` → `wizRenderPosts` מציג 3 מודעות אמיתיות (headline+body, `_esc`) עם **ערוך**
> (`PATCH /ads`) ו**רענן** (`regenerate-copy`). idempotent (`WIZ.campaignId`; 409→`listAds`). guard לחיבור Meta;
> שגיאות 402/400/422/409/503 → `_wizErr`+retry. CTA=שמירת טיוטה (לא go-live מזויף). **פער-backend שנסגר:**
> `PATCH /ads/{id}` חדש (`AdEditRequest`, `copy_service.edit_variant` ממחזר את `regenerate_variant`, draft-gate,
> בלי LLM/paid) + 11 טסטים. **תמונות (async) + פרסום (push, 8 שערים) = scope הבא.** 6 מתודות api.js.
>
> **הוראה 6 (חיווט התמונות — Dream Wizard) ✅:** המשך ישיר להוראה 5 (קופי). אחרי שהקופי מוכן — כפתור "✨ צור
> תמונות" → `POST /campaigns/{id}/generate-images` (async, **202+job_id**) → **polling** `GET /me/jobs/{id}`
> (`_wizPollJob`, async/await loop, timeout 120ש') → `GET /campaigns/{id}/ads` → `wizRenderPosts` מציג 3 תמונות
> אמיתיות (img + רענון overlay). רענון פר-מודעה: `POST /ads/{id}/regenerate-image` (סינכרוני). **כשל-חלקי**
> (load-bearing): job='done' ≠ כל התמונות הצליחו — זיהוי פר-מודעה (`image_url`=מוכן / `failed_image`=נכשל+
> "נסה שוב" / `generating`=בייצור). footer דינמי (צור-תמונות / בייצור+רענן / סיום). idempotent: 409
> (כבר reserved) → `listAds`; re-entry דרך wizStartLoading 409→listAds מציג תמונות קיימות. 3 מתודות api.js
> (generateImages/getJob/regenerateImage). **frontend-only** (engine `gpt-image-2`+Storage+jobs כבר בנוי) —
> ללא backend/migration/pytest. **פרסום (push, 8 שערים) = scope הבא.**
>
> **הוראה 7 (חיווט הפרסום ל-Meta — Dream Wizard מלא) ✅:** השלב האחרון והרגיש (יוצר ישויות ב-Facebook + תקציב).
> אחרי תמונות → "🚀 פרסם קמפיין" → **מודאל-אישור** (בלתי-הפיך) → `POST /campaigns/{id}/push` (8 שערים, **202+job_id**)
> → `_wizPollJob` (timeout 180ש') → **`GET /campaigns/{id}`** → הסתעפות על `campaign.status`. **load-bearing:**
> **job='done' ≠ הצלחה** (כשל-קבוע מסמן campaign='failed' בלי לזרוק → job done) — אסור לסמוך על ה-job, חובה GET
> campaign. `live`→🎉 (toast + `showCampaignLiveState`); `failed`→הודעה כנה **בלי retry** (re-push=409, אין reset
> endpoint; cleanup-cron). **2 סוגי כשל:** pre-flight gate (4xx, נשאר draft→retry תקף) מול job-failure (terminal).
> **409 דו-משמעי** (fb_not_connected dict מול not-draft) → מפענח דרך GET campaign (draft=Meta-disconnect /
> pushing / live), בלי לסמוך על ה-code (ApiError מאבד dict). 2 מתודות api.js (pushCampaign/getCampaign).
> **frontend-only** (campaign_push_service כבר בנוי). **ה-Dream Wizard מלא end-to-end.**
> **7.4ב-2 ✅ Done (החלק המסוכן):** `optimization_push_service` — creative-swap אטומי ל-Meta (דפוס 3.4:
> claim/resume דרך RPC, save-as-you-go, rollback מלא + reactivate ישנות, **pause-אחרון per-ad record-AFTER**,
> finalize-CAS + window 120ש' + `attempt_error=NULL`). migration 0053 (meta-state ל-`optimization_actions` +
> RPC `open_optimization_push_action`), `meta.update_ad`, `ads_service.fetch_ads_for_push`, endpoint `approve`
> + models (Approve*). **gates היפוך 3.4** (הקמפיין חייב להיות **live**). זיווג וריאציה↔מודעה-ישנה **פוזיציוני**
> (vocabulary של angles שונה!). 37 בדיקות. **פוצל שוב — נדחה ל-7.4ב-3:** cron `monitor_optimization_windows`
> + מנוע השלבים (`advance_to_next_step` + שלבים 6/7).
> **7.4ב-3 ✅ Done (סגירת הלולאה):** `optimization_monitor_service` — cron **שעתי דטרמיניסטי** (tick ב-`runner`,
> דפוס `cleanup`; **בלי LLM**, חוק 7) שמודד CPL מצטבר (`get_campaign_insights`) ומחליט שיפור/אי-שיפור עם **CAS
> על `status`** (מדידה ראשונה: שיפור→`success_monitoring`+חלון שני, אחרת→`done`; חלון שני: עדיין-משופר→הסדרה
> `done`, נסוג→`in_progress`). מנוע השלבים ב-`optimization_service`: `step_plan` (1=`creative_refresh`,
> 2/3=`angle_change` social_proof/authority, ≥4=None) + `advance_to_next_step` (**read-only, פונקציה טהורה של
> ה-action האחרון** — לא של `current_step` שעלול לפגר). `solution_service`+push מוכללים ל-`angle_change` (זווית
> **אחת** × 3, prompt `angle_change_copy`); push מקבל `step_number` (server-authoritative). **בלי migration**
> (`idx_opt_actions_window` קיים). 55 בדיקות. **נדחה לעתיד:** **חוק הלולאה** (דורש מפתח action-per-
> `(session,step,loop)`; ה-RPC הנוכחי לא תומך בחזרה על שלב). [שלב `offer_change` מומש ב-7.4ג.]
> **תיקון Cursor (קשירת step בין propose↔approve):** `propose` ו-`approve` כל אחד קרא ל-`advance_to_next_step`
> בנפרד ובזמן שונה. אם בין שתי הקריאות ה-cron של 120ש' סגר action (advance מחזיר step+1) או approve מקביל
> רץ — הקופי שנוצר ל-step N נדחף/נבדק מול step M (422 מטעה "וריאציות לא תואמות", או דחיפה לשלב הלא-נכון).
> **התיקון (optimistic concurrency):** `ProposeResponse.step_number` מוחזר ללקוח; `ApproveRequest.step_number`
> נדרש בחזרה; ה-router משווה אותו ל-`current_step` (עדיין מחושב server-side — סמכות השרת נשמרת) ומחזיר **409
> מפורש** ("המצב השתנה, צור הצעה מחדש") על אי-התאמה, במקום 422 מטעה. ה-idempotency של `open_push_action`
> (advisory lock פר-(session,step)) מכסה את ה-race השיורי. בלי migration.
> **מיקום:** 7.4ב — החצי השני של בעיה 1. ממשיך מ-7.4א (שעצר אחרי האבחון). **תלוי ב-7.4א, 7.3.5, 3.4.**
> **זה הסשן הכי מסוכן ב-Phase 7** — הוא נוגע ב-Meta API עם כסף אמיתי. אותו משקל זהירות כמו 3.4.

> **Session 7.4ג ✅ Done (STAGE_C — שלב ג': שיפור ההצעה השיווקית):** מימוש `offer_change` (שלב 4), מפוצל
> לשניים כי **ההצעה היא החלטה עסקית של בעל העסק** (כמה רווח לוותר) ואסור שה-LLM ימציא אותה (חוק 7): **ג'-1**
> — `propose` בשלב 4 מחזיר `OfferRecommendation` (המלצה לחזק את ההצעה בהטבה, בלי וריאציות/אישור) ומסמן
> `awaiting_offer`; **ג'-2** — בעל העסק חוזר בהודעה חופשית עם ההטבה → `send_user_message` מזהה `awaiting_offer`
> (**לפני** דגל free-chat/quota — זרם אופטימיזציה, לא ייעוץ) ומנתב ל-`generate_solution(chosen_offer=...)` →
> 3 וריאציות סביב ההטבה (prompt `offer_copy`) → `approve` → push (אותו צינור — `step_plan(4)` נגזר כמו
> creative_refresh, push גנרי ללא שינוי) → cron. `generate_solution` מחזיר union `Solution | OfferRecommendation`
> (chosen_offer ריק בשלב 4 → המלצה; מלא → יצירה — מינימום churn, רוב הטסטים הקיימים נשמרו). `awaiting_offer`
> מתנקה ב-`approve` אחרי push מוצלח (best-effort). migration 0056 (עמודה `awaiting_offer`, בלי GRANT — service_
> role). 12 בדיקות חדשות + עדכון 5 קיימות (step 4 נתמך). **נדחה:** חוק הלולאה (חזרה לשלב 1) — STAGE_C הוא השלב האחרון.

> **Session 7.4.X ✅ Done (ארכיטקטורת פרומפט מודולרית — Core + Module + Payload):** החלפת טעינת פרומפט-פר-בעיה
> (`problem_1.txt`) במבנה **Core קבוע + מודול פעיל + Payload מובנה**, דרך `prompts_service.build_agent_prompt`.
> **שני מסלולים:** chip (orchestrator — core בלוק א + `modules/high_cpl/{state_key}` + `[SYSTEM_PAYLOAD]`) ושיחה
> חופשית (`send_user_message` — core בלוק א+ב + `[GENERAL_STATUS]`, **עובדה בלבד, בלי שדות-פרוטוקול**). חוק 7/
> החלטה 2: **הוראת-ניסוח אחת** נבחרת לפי state, לא תפריט `if/then`. **שינויים:** `core.txt` (סנטינל `<<CORE_B>>`)
> + `modules/high_cpl/*` (lock+benchmark_stop = **template בלי LLM**, expensive_first/repeat_open/mixed = הוראות);
> `build_agent_prompt`/`render_agent_template`/`_load_core_blocks` ב-prompts_service; `get_general_status_for_agent`
> (CPC=spend/link_clicks, סיווג-שוק+טווח, תקציב מ-quiz — כולם reuse, בלי קריאת Meta נוספת); `agent_i18n` (מפות
> משותפות); `AGENT_FREE_CHAT_ENABLED` (default true; false→הפניה לצ'יפים בלי LLM/מכסה). **benchmark_stop הומר
> ל-template** (החלטה 3). `module_low_quality`/`rejection` + high_cpl stages = **מוכן-לא-מחווט** (TODO, יחווט ב-7.5/
> 7.6). **בלי migration.** legacy (`problem_1`/`system_v2`) נשמר לתקופת מעבר. **מגמה נדחתה** (דורש הרחבת Insights).
> 1553 טסטים עוברים (20 חדשים). **מיקום:** לפני Session 8.

---

## מבוא ותיאור Session

7.4א מזהה ועוצר. 7.4ב **מבצע ומודד.** זה מנוע הפרוטוקול המלא של בעיה 1 — מהאבחון (ש-7.4א סיים) ועד הלולאה.

7.4ב בונה ארבעה דברים:
1. **יצירת הפתרון** — 3 קופי + 3 כותרות + 3 זוויות (LLM), על בסיס הזווית שטרם נוצלה בסדרה.
2. **אישור + העלאה ל-Meta** — אטומי, מסוכן, באותו דפוס של 3.4. **עדכון creative של מודעות חיות**, לא יצירת קמפיין.
3. **כתיבת `optimization_actions`** — הפעולה הבודדת. כאן מתחילה נעילת ה-120 שעות.
4. **cron מעקב** — מתעורר אחרי 120 שעות, מודד, ומחליט: שיפור → `success_monitoring`, אין שיפור → שלב הבא, מוצו השלבים → לולאה.

המטרה: בסוף 7.4ב, **בעיה 1 שלמה מקצה לקצה.** לקוח לוחץ "עלות גבוהה" → אבחון (7.4א) → הסוכן יוצר 3 וריאציות → הלקוח מאשר → עולה ל-Meta → 120 שעות מעקב → מדידה אוטומטית → שלב הבא או הצלחה.

> **גבול מול Phase 8:** 7.4ב כולל את ה-cron שמודד פעולה ש**הלקוח אישר** (החלק התגובתי של "הסוכן היוזם"). מה שנשאר ל-Phase 8: ניטור **יזום** (הסוכן מזהה בעיה בעצמו בלי שהלקוח לחץ), בעיות 2-3 אוטונומיות, והתראות וואטסאפ. המדידה של פעולה מאושרת היא לב הפרוטוקול, לא תוספת — לכן כאן.

**מה בסשן:**
- `solution_service` — יצירת 3 קופי/כותרות/זוויות (LLM), בחירת זווית שלא נוצלה.
- שלב אישור + העלאה: `optimization_push_service` — אטומי, state machine, rollback (דפוס 3.4).
- כתיבת `optimization_actions` עם `window_ends_at`.
- cron `monitor_optimization_windows` — tick נפרד, מדידה והחלטה דטרמיניסטית.
- מנוע השלבים: מעבר 1→2→...→7, חוק השיפור/אי-שיפור, `loop_count++`.
- endpoints: `POST .../propose` (יצירת פתרון), `POST .../approve` (אישור והעלאה).

**מה לא בסשן:**
- בעיות 2-3 (7.5/7.6).
- ניטור יזום (Phase 8) — הסוכן שמזהה CPL גבוה בעצמו.
- התראות וואטסאפ של הסוכן (Phase 8, `agent_alerts_quota`).
- מאמן מכירות (7.3, דולג).
- מיילים יזומים של הסוכן (תיקון 4, Phase 7.x/8) — ב-7.4ב המעקב מעדכן את ה-DB; המייל ללקוח על תוצאה הוא הרחבה.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | העלאה ל-Meta — יצירה או עדכון | **עדכון creative של מודעות חיות.** הקמפיין כבר קיים (מ-3.4). מחליפים קופי/כותרת. Meta לא מאפשרת לערוך creative רץ → יוצרים creative חדש ומחליפים / מודעות חדשות + כיבוי ישנות. אטומי + rollback, דפוס 3.4 |
| 2 | מעקב 120 שעות | **cron, לא סינכרוני.** handler נפרד `monitor_optimization_windows`, tick נפרד מ-6.1/billing. דטרמיניסטי — ה-LLM לא מחליט "השתפר/לא" |
| 3 | בחירת זווית בכל שלב | **מתוך 5 הזוויות, הזווית שטרם נוצלה בסדרה.** שלב 2 — 3 זוויות; שלב 6 — מהנותרות. נשמר ב-`optimization_actions.action_type` + תיאור |
| 4 | מה זה "שיפור" | **כל ירידה ב-CPL, אפילו ₪1.** לא נדרשת הרעה כדי "אין שיפור" (סגרת קודם). שווה או גרוע = אין שיפור → שלב הבא |
| 5 | `success_monitoring` | אחרי שיפור — **חלון 5 ימים נוסף** בלי שינוי. בתום: עדיין משופר/יציב → `done`. נסוג → חוזר לשלבים (סדרה ממשיכה) |
| 6 | כתיבת פעולות אוטונומיות | בעיה 1 כולה **דורשת אישור לקוח** (תוכן חדש — §9 בספ). `requires_approval=true` תמיד. אין פעולה אוטונומית בבעיה 1 |
| 7 | מי מריץ את שלב 2 | ה-cron מחליט "צריך שלב הבא" ומסמן את הסדרה. **אבל היצירה+אישור עוברים דרך הלקוח** — הסוכן מציג, הלקוח מאשר. ב-MVP בלי מייל יזום: הסטטוס ממתין, והלקוח רואה בכניסה הבאה לצ'אט (מייל יזום — Phase 8) |

---

## תלויות

1. **7.4א** — האבחון, פתיחת הסדרה, ה-orchestrator. 7.4ב ממשיך מאיפה ש-7.4א עצר.
2. **7.3.5** — `optimization_sessions`/`optimization_actions` (כתיבה מלאה כאן), `benchmark_service`.
3. **3.4** — דפוס ההעלאה האטומית ל-Meta (state machine, rollback, idempotency). 7.4ב **מאמץ אותו** לעדכון creative.
4. **3.2/3.3** — יצירת קופי (gpt-5.2) ותמונות. 7.4ב משתמש בהם ליצירת הוריאציות.
5. **4.5** — Insights (CPL למדידה ב-cron).
6. **6.1** — דפוס ה-tick ב-`runner.py`. ה-cron של 7.4ב הוא tick נוסף, אותו דפוס.

---

## חלק 1 — `solution_service` (יצירת הפתרון, LLM)

מודול חדש `app/services/solution_service.py`.

**`generate_solution(session_id, step_number) -> Solution`:**
1. שולף את הסדרה + הקמפיין + `business_description` (מ-`extracted_context`, 3.1.6).
2. **בחירת זווית** (החלטה 3): שולף אילו זוויות כבר נוצלו בסדרה (מ-`optimization_actions` קודמות). בוחר מהנותרות מתוך 5 (כאב/חלום/דחיפות/הוכחה-חברתית/סמכות).
   - שלב 2: 3 זוויות חדשות.
   - שלב 6: זווית שונה לחלוטין מהנותרות.
   - שלב 7: שינוי **הצעה** (לא רק זווית) — הטבה/בונוס/ניסיון.
3. קורא ל-LLM (gpt-5.2) דרך `prompts_service.build` — 3 קופי + 3 כותרות, בהתאם לזווית ולטון המותג (`quiz_responses`).
4. **לא** יוצר תמונות אוטומטית ב-MVP (שלב 2 הוא טקסט — תואם את הפרומפט; יצירת תמונה חדשה היא הרחבה. ראה "פתוח לגיא"). **[עודכן — A3]:** ההרחבה מומשה — 3 סוגי-הרענון מקבלים תמונות חדשות (`include_images`, A3 אסינכרוני); ראה ההכרעה ב"פתוח לגיא" למטה. **[עודכן — "תמונות עם הקופי"]:** התמונות נוצרות עכשיו **eager בזמן ה-propose** (job `optimization_image_generate`) ומוצגות ב-`view-action` לצד הקופי לבחינה/רענון-תמונה לפני approve (שחזור חזון §159/216); הוויזרד עבר ל-auto (בלי כפתור "צור תמונות"); הסנדבוקס מציג את התמונות. ראה `docs/optimization-creative-refresh-images-plan.md` (§2, היפוך A1↔A3).
5. מחזיר את 3 הוריאציות (טרם נשמרו כפעולה — נשמר באישור).

> **פתוח לגיא:** שלב 2 בפרומפט מדבר על קופי+כותרות+זוויות. האם גם **תמונות חדשות**? מודעה בלי תמונה לא קיימת ב-Meta. אופציות: (א) שומרים את התמונות הקיימות, מחליפים רק טקסט; (ב) יוצרים גם 3 תמונות חדשות (gpt-image-2, יקר יותר, איטי יותר). **המלצה: (א) ב-MVP** — שינוי טקסט פוגע פחות באלגוריתם וזול. לסגור.
>
> **✅ הוכרע (תיקון-רישום, החלטת בעלים — Session A3):** המסגרת לעיל הייתה **המלצת-MVP** (אופציה א), **לא** החלטה של גיא — גיא מעולם לא הכריע על "רק טקסט". הבעלים הכריע לממש את **אופציה (ב): תמונות חדשות** ל-3 סוגי-הרענון (`creative_refresh`/`angle_change`/`offer_change`) דרך ארכיטקטורת **A3** (יצירה אסינכרונית ב-worker לפני ה-push), עם דגל `include_images` דינמי + מתג-חירום גלובלי (`app_settings`) לחזרה מיידית ל-recycle. screening/rejection/filter_addon נשארים recycle. **הערות-מסירה (יושר):** מודעות חדשות מאפסות פאזת-למידה ב-Meta בכל מקרה; מיחזור התמונה שמר "עוגן" מוכח — הדגל+המתג נותנים שליטה. **חלון-שארית:** crash בין החזרת gpt-image ל-upload → יצירה חוזרת (אובייקט יתום ב-Storage; מקובל MVP, כמו ה-wizard) — אין מסלול שמייצר-מחדש שקרי. ראה `docs/optimization-creative-refresh-images-plan.md`.

---

## חלק 2 — שלב אישור + העלאה ל-Meta (החלק המסוכן)

מודול חדש `app/services/optimization_push_service.py`. **דפוס 3.4 — אטומי, state machine, rollback, כסף אמיתי.**

### למה זה מסוכן

הקמפיין **חי ורץ** במטא. עדכון creative של מודעה רצה הוא לא טריוויאלי: Meta **לא מאפשרת לערוך** את ה-creative של מודעה פעילה. הדרך: ליצור Ad Creative חדש ולהחליף ב-Ad, או ליצור Ads חדשות ולכבות הישנות. כשל באמצע = קמפיין במצב לא-עקבי (חלק מהמודעות חדשות, חלק ישנות, אולי כפילות) — **בדיוק כמו ה-rollback של 3.4**, ואותה רמת סיכון.

### הזרימה

1. **אישור הלקוח** — הלקוח אישר את 3 הוריאציות (endpoint `approve`, חלק 5).
2. **כתיבת `optimization_actions`** — שורה חדשה תחת הסדרה: `step_number`, `action_type` (הזווית), `change_description` ("רענון 3 קופי — זווית כאב/חלום/דחיפות"), `status='in_progress'`, `requires_approval=true`, `approved_at=now()`. **`window_ends_at` עדיין NULL** — נקבע רק אחרי שההעלאה הצליחה.
3. **העלאה אטומית ל-Meta** (state machine, דפוס 3.4):
   - לכל וריאציה: יצירת Ad Creative חדש → החלפה ב-Ad הקיים (או Ad חדש + כיבוי ישן).
   - idempotency: כל `meta_*_id` חדש נשמר תוך כדי.
   - **8 validation gates** כמו 3.4 (כולל בדיקה שהמודעות עדיין קיימות לפני עדכון).
   - state ביניים `pushing` בסדרה.
4. **בכשל** — rollback אטומי: החזרת ה-creative הישן / מחיקת ה-Ads החדשים + הפעלת הישנים. כשל ב-rollback → `failed_rollback_pending` + Sentry, ידני (כמו 3.4).
5. **בהצלחה** — `optimization_actions.status` נשאר `in_progress`, ו**`window_ends_at = now() + interval '120 hours'`**. נעילת המדידה מתחילה **רק עכשיו** (אחרי שהמודעות באמת באוויר).
6. עדכון `optimization_sessions.current_step = step_number`.

> **`window_ends_at` נקבע אחרי ההעלאה, לא לפניה.** קריטי: אם נקבע את החלון לפני שהמודעות עלו, ונכשל ב-Meta — נמדוד חלון על מודעות שלא רצו. רק כשההעלאה הצליחה, הספירה מתחילה.

> **דפוס 3.4 הוא חובה כאן — לא להמציא מחדש.** אותו state machine, אותו rollback=מחיקה/שחזור, אותו מיפוי כשלים (`fail_user`/`fail_system`/`refresh_and_retry`/`retry`), אותו `attempt_error` כ-JSONB. ההבדל היחיד: עדכון creative קיים במקום יצירת קמפיין. לאמת מול הקוד של 3.4.

---

## חלק 3 — cron מעקב: `monitor_optimization_windows`

handler חדש ב-`worker/handlers.py` + tick ב-`runner.py`. **דטרמיניסטי לחלוטין — ה-LLM לא נוגע.**

### ה-tick

tick נפרד מ-6.1 (token refresh) ומ-billing — אותו דפוס (monotonic, ב-`runner.py`), handler ייעודי. רץ בתדירות סבירה (כל שעה? — לסגור; חלון 120 שעות לא דורש דיוק לדקה).

### מה ה-handler עושה

1. **שולף סדרות שחלון המדידה שלהן נגמר:**

```sql
SELECT s.id AS session_id, a.id AS action_id, s.problem_type,
       s.current_step, s.loop_count, s.starting_metric,
       a.post_change_metric, s.campaign_id, s.user_id
FROM optimization_sessions s
JOIN optimization_actions a ON a.session_id = s.id
WHERE s.status IN ('in_progress', 'success_monitoring')
  AND a.window_ends_at IS NOT NULL
  AND now() >= a.window_ends_at
  AND a.status = 'in_progress'
ORDER BY a.window_ends_at;
```

2. **לכל סדרה — מודד:**
   - שולף CPL עכשווי מ-Insights (4.5).
   - **אם `post_change_metric` עדיין NULL** (זו המדידה הראשונה אחרי 120 שעות): קובע `post_change_metric = CPL עכשווי`. זו נקודת המדידה Y.
   - משווה ל-`starting_metric` (X): **`current_cpl < starting_cpl`?**

3. **החלטה דטרמיניסטית (חוק השיפור/אי-שיפור):**
   - **שיפור** (CPL ירד, אפילו ₪1): `optimization_actions.status='success_monitoring'`, `optimization_sessions.status='success_monitoring'`, **`window_ends_at = now() + 120h`** (חלון מעקב נוסף, החלטה 5). שומר Y. **לא** מתקדם שלב.
   - **אין שיפור** (שווה או גרוע): `optimization_actions.status='done'`. הסדרה צריכה שלב הבא:
     - אם `current_step < 7`: מסמן `optimization_sessions` שצריך שלב הבא (`current_step` יקודם כשהפעולה הבאה תיווצר). הסטטוס נשאר `in_progress`. **הלקוח יראה בכניסה הבאה** (מייל יזום — Phase 8).
     - אם `current_step == 7`: **חוק הלולאה** — `loop_count++`, `current_step=1`, הסדרה מתחילה סבב חדש (מנסה זוויות שלא נוצלו בלולאה הקודמת אם אפשר).

4. **`success_monitoring` שמסתיים** (חלון שני נגמר):
   - שולף CPL עכשווי. **עדיין משופר ביחס ל-X?** → `optimization_sessions.status='done'`. הסדרה נסגרה בהצלחה.
   - **נסוג** (CPL חזר לעלות): הסדרה חוזרת ל-`in_progress`, צריך שלב הבא (טיפול חוזר).

> **3 נקודות המטריקה כאן:** `starting_metric` (X, נקבע ב-7.4ב שלב 2 / 7.4א פתיחת סדרה), `post_change_metric` (Y, נקבע במדידה הראשונה של ה-cron), `current_metric` (Z, נשלף lazy כשהלקוח חוזר ב-7.4א). ה-cron כותב Y; 7.4א קורא X→Y→Z.

> **למה הכל דטרמיניסטי:** ה-cron לא שואל LLM "האם השתפר". זו השוואת מספרים. ה-LLM נכנס רק כשהלקוח חוזר לצ'אט והסוכן צריך **לנסח** את התוצאה (7.4א). חוק 7.

---

## חלק 4 — מנוע השלבים

הלוגיקה של "איזה שלב עכשיו" יושבת ב-`optimization_service` (מ-7.4א, מורחב). **דטרמיניסטי.**

**`advance_to_next_step(session_id)`:**
- נקרא כשהלקוח חוזר וה-cron כבר סימן שצריך שלב הבא.
- קובע את `step_number` הבא לפי `current_step` של הסדרה.
- מיפוי שלב → סוג פעולה:
  - שלב 2 → רענון קריאייטיב (3 זוויות).
  - שלב 6 → שינוי זווית (מהנותרות).
  - שלב 7 → שינוי הצעה שיווקית.
- (שלבים 3,4,5 הם אישור/העלאה/מעקב — לא "פעולות יצירה" נפרדות; הם חלק מהמחזור של כל שלב-יצירה. בפועל הסדרה מתקדמת: יצירה → אישור → העלאה → מעקב, וזה "שלב" אחד בפרוטוקול. ראה הערה.)

> **הבהרה על מספור השלבים:** הפרומפט ממספר 1-7, אבל בפועל יש **3 שלבי-יצירה** (רענון / זווית / הצעה) שכל אחד עובר את אותו מחזור (אבחון→יצירה→אישור→העלאה→120ש'→מדידה). `current_step` בסדרה מקודם בכל מחזור. לתעד את המיפוי בבירור ל-CC כדי שלא יבנה 7 שלבים נפרדים מלאכותית.

---

## חלק 5 — Endpoints

**א. `POST /me/agent/conversations/{id}/propose`** — יצירת הפתרון.
- נקרא אחרי האבחון (7.4א) או כשהלקוח חוזר לשלב הבא.
- Body: `{session_id}`.
- קורא ל-`solution_service.generate_solution`.
- Response: `{variations: [{copy, headline, angle}, x3], session_id, step_number}`.
- שמירה: הוריאציות **טרם** נכתבות כפעולה (נכתב באישור).

**ב. `POST /me/agent/conversations/{id}/approve`** — אישור והעלאה.
- Body: `{session_id, approved_variations: [...]}` (או "approve all").
- קורא ל-`optimization_push_service` — כותב `optimization_actions`, מעלה ל-Meta אטומי, קובע `window_ends_at`.
- Response: `{status: "pushed"|"push_failed", action_id, window_ends_at}`.
- Status: 200; 502 אם Meta נכשל (אחרי rollback); 409 אם הסדרה כבר ב-`pushing`.
- **כאן מתחילה נעילת 120 השעות** — מאותו רגע הצ'יפ "עלות גבוהה" יחזיר Lock (7.4א).

**ג. (אופציונלי) `POST .../request-changes`** — "אני רוצה לשנות משהו" (מהפרומפט). מחזיר את הסוכן ליצירה מחדש של וריאציה. ב-MVP יכול להיות פשוט קריאה חוזרת ל-`propose`. לסגור אם נכלל ב-7.4ב או נדחה.

---

## חלק 6 — שינויים נדרשים בקוד הקיים

**א.** `solution_service.py` — חדש (חלק 1).
**ב.** `optimization_push_service.py` — חדש (חלק 2, דפוס 3.4).
**ג.** `worker/handlers.py` — handler `monitor_optimization_windows` (חלק 3).
**ד.** `worker/runner.py` — tick נוסף ל-cron המעקב (דפוס 6.1).
**ה.** `optimization_service.py` (מ-7.4א) — הרחבה: `advance_to_next_step`, מנוע השלבים (חלק 4).
**ו.** `routers/agent.py` — endpoints `propose`, `approve` (חלק 5).
**ז.** `prompts/agent/` — prompt ליצירת הקופי בשלב 2 (אם שונה מ-3.2; סביר שאפשר לעשות reuse עם פרמטר זווית).
**ח.** `models/` — `Solution`, `OptimizationAction` (כתיבה), `MonitorResult`.

---

## חלק 7 — בדיקות

**`solution_service`:**
1. שלב 2 → 3 וריאציות, 3 זוויות שונות.
2. שלב 6 → זווית מהנותרות (לא חוזר על שלב 2).
3. שלב 7 → שינוי הצעה.
4. טון המותג מהשאלון מוזרק לקופי.

**`optimization_push_service` (קריטי):**
5. העלאה תקינה → creative מוחלף, `window_ends_at` נקבע ל-+120ש'.
6. כשל באמצע (וריאציה 2 נדחתה) → rollback מלא, אין מצב חצי-בנוי.
7. כשל ב-rollback → `failed_rollback_pending` + Sentry.
8. **`window_ends_at` נקבע רק אחרי הצלחה** — כשל בהעלאה → `window_ends_at` נשאר NULL.
9. טוקן פג באמצע → refresh + ניסיון יחיד (דפוס 3.4).

**`monitor_optimization_windows` (cron):**
10. סדרה שחלונה לא נגמר → לא נבחרת.
11. חלון נגמר, CPL ירד ₪50→₪48 → `success_monitoring`, חלון +120ש', Y=48 נשמר.
12. חלון נגמר, CPL שווה ₪50→₪50 → אין שיפור → שלב הבא, `action.status='done'`.
13. חלון נגמר, שלב 7, אין שיפור → לולאה, `loop_count=1`, `current_step=1`.
14. `success_monitoring` נגמר, עדיין משופר → סדרה `done`.
15. `success_monitoring` נגמר, נסוג → סדרה חוזרת ל-`in_progress`.
16. מדידה ראשונה: `post_change_metric` NULL → נקבע ל-CPL עכשווי.

**מנוע שלבים:**
17. `advance_to_next_step` משלב 2 → שלב 6 (זווית).
18. משלב 6 → שלב 7 (הצעה).

**e2e:**
19. צ'יפ → אבחון (7.4א) → propose → 3 וריאציות → approve → Meta → `window_ends_at` נקבע.
20. אחרי approve, צ'יפ "עלות גבוהה" שוב → Lock (7.4א), כי בחלון.
21. cron אחרי 120ש' (סימולציה) → מדידה → החלטה.

---

## חלק 8 — לא ב-7.4ב

- **בעיות 2-3** (7.5/7.6).
- **ניטור יזום** (Phase 8) — הסוכן מזהה בעיה בעצמו, בלי שהלקוח לחץ. ה-cron של 7.4ב מודד רק פעולות **מאושרות**, לא סורק לזהות בעיות חדשות.
- **התראות וואטסאפ** (Phase 8, `agent_alerts_quota`).
- **מייל יזום על תוצאת מדידה** — ב-7.4ב ה-cron מעדכן DB; הלקוח רואה בכניסה הבאה. מייל "התוצאה הגיעה" / "עוברים לשלב 2" — תיקון 4, Phase 7.x/8.
- **יצירת תמונות חדשות בשלב 2** — המלצה: לא ב-MVP (פתוח לגיא).
- **פעולות אוטונומיות** (שינוי קהל/תקציב בלי אישור) — בעיה 1 כולה דורשת אישור. אוטונומי זה Phase 8.

---

## חלק 9 — פתוח לגיא

1. **שלב 2 — תמונות חדשות או רק טקסט?** ~~המלצה: רק טקסט ב-MVP (זול, פחות מפריע לאלגוריתם).~~ **✅ הוכרע (תיקון-רישום): תמונות חדשות** ל-3 סוגי-הרענון (A3 אסינכרוני + דגל `include_images` + מתג-חירום). המלצת-ה-MVP "רק טקסט" **לא** הייתה החלטת-גיא; הבעלים הכריע לממש תמונות. ראה 7.4ב-1 ו-`docs/optimization-creative-refresh-images-plan.md`.
2. **תדירות ה-cron** — כל שעה? חלון 120 שעות לא דורש דיוק לדקה. כל שעה סביר.
3. **"אני רוצה לשנות משהו"** — endpoint נפרד או reuse של `propose`? כמה רענונים מותר (גיא הזכיר 5 לתמונה, 10 לקופי במאמן מכירות)?
4. **מספור שלבים** — לאשר שהמיפוי "3 שלבי-יצירה × מחזור" נכון, ולא 7 שלבים מלאכותיים.

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **דפוס 3.4 הוא חובה בהעלאה — לא להמציא מחדש.** state machine, rollback=שחזור/מחיקה, idempotency כעמודות, 8 gates, מיפוי כשלים. ההבדל היחיד: עדכון creative במקום יצירת קמפיין. לאמת מול הקוד של 3.4.

2. **`window_ends_at` נקבע אחרי הצלחת ההעלאה, לא לפניה.** אחרת מודדים חלון על מודעות שלא רצו. קריטי.

3. **ה-cron דטרמיניסטי לחלוטין.** "האם השתפר" = השוואת מספרים, לא LLM. ה-LLM נכנס רק לניסוח כשהלקוח חוזר (7.4א).

4. **3 נקודות המטריקה:** X ב-session (`starting_metric`), Y ב-action (`post_change_metric`, נכתב ע"י ה-cron במדידה ראשונה), Z lazy (7.4א). אסור לבלבל.

5. **חוק השיפור גובר על אי-שיפור** — שיפור (אפילו ₪1) → `success_monitoring`, לא שלב הבא. רק שווה/גרוע → שלב הבא.

6. **120 שעות, לא 96.** בכל מקום. ושני חלונות (פעולה + success_monitoring) לפני `done`.

7. **כל בעיה 1 דורשת אישור** (`requires_approval=true`). אין פעולה אוטונומית כאן — זה Phase 8.

8. **tick נפרד מ-6.1/billing.** אותו דפוס, handler ואחריות נפרדים. לא לערבב את מעקב האופטימיזציה עם רענון טוקן.

9. **מנוע השלבים = 3 שלבי-יצירה, לא 7.** רענון/זווית/הצעה, כל אחד עובר מחזור. לא לבנות 7 שלבים מלאכותיים.

10. **Sentry context.** exceptions ב-push → `session_id`, `action_id`, `campaign_id`, השלב ב-state machine. ב-cron → `session_id`, איזו החלטה.

11. **אין migration חדש** — משתמש בטבלאות מ-7.3.5. רק קוד, prompt, ו-cron tick.

---

# Session 7.5 — בעיה 2: פניות לא איכותיות

> **✅ REDESIGN (filter_addon):** במקום 3 קופי חדשים מאפס — **עריכת 3 המודעות החיות + משפט מסנן** (אזור/מחיר;
> "מניעת שינויים מיותרים" מעל "שיפור כמות"). 2 קטגוריות בלבד (`wrong_area`/`no_budget`; `dont_understand`/
> `standards` בוטלו). אבחון **תלת-שלבי** (turn לאיסוף ערך → `filter_value` ב-starting_metric). מנוע-שלבים
> מתקצר ל-`{1:filter_addon, 2:screening}`. ה-angles של filter_addon == `_OLD_AD_ANGLE_ORDER` (זיווג פוזיציוני
> ב-push). פירוט מלא: `docs/low-quality-leads-filter-redesign.md`.
>
> **פוצל ל-2 (אישור המשתמש — כולל את פעולת ה-Meta הכי מסוכנת ב-Phase 7): בטוח → מסוכן.**
>
> **⚠️ ROLLBACK (2026-06, החלטת-מוצר):** חלק **(4) מדד-איכות ב-cron** (`_measure_low_quality` + baseline
> `irrelevant_ratio` + `count_tagged_and_irrelevant`) **בוטל**, יחד עם ה-no_signal-3-way ב-high_cpl (PR11)
> ו-2 ה-notification-types (`insufficient_data`, `needs_lead_tagging`) — "תכנון-AI ישן" שלא בפרוטוקול.
> **נשאר:** אבחון-דיווח (1-3) + קופי-מסנן + טופס-סינון (7.5ב/ג). **low_quality מבוסס-תשובת-לקוח ✅ (הוראה 3):**
> ה-push מסיים עם window=120h; כשהחלון נגמר ה-cron שולח מייל-מעקב `QUALITY_FOLLOWUP` ("השתפר?") ומעביר את
> ה-action ל-`awaiting_feedback` (מצב 2 בעמוד-האינטראקציה); כן→סוגר את הסדרה, לא→`generate_solution` מייצר את
> השלב הבא (מצב 1), אחרי שלב 4 → exhausted. **תשובת-הבעלים = המדד היחיד** ל-low_quality. high_cpl ללא שינוי
> (no_signal→אי-שיפור→advance רגיל). פירוט מלא: בלוק "הוראה 3" בהמשך.
>
> **7.5א ✅ Done (החלק הבטוח — אבחון-דיווח + קופי מסנן + מדד איכות, בלי Meta חדש):** הצ'יפ "פניות לא
> איכותיות" כשלבים 1-3 מקצה-לקצה דרך הצינור הקיים (creative-swap). **(1) `step_plan(step, problem_type)`
> דו-מפלסי** — שלבים 1-3 משותפים ל-high_cpl ו-low_quality (אותו מנגנון, אותן זוויות; ההבדל בהנחיית-היצירה
> בלבד), שלב 4 שונה (high_cpl=offer_change; low_quality=screening_question, 7.5ב). 5 callers עודכנו.
> **(2) `diagnose_problem_2`** (orchestrator) — דו-שלבי, **בלי benchmark** (אבחון מבוסס-דיווח): Lock פר-
> low_quality (high_cpl פתוח לא נועל); שלב 1 → 4 chips (template `category_question`, בלי LLM); שלב 2
> (subcategory) → `open_session` (subcategory ב-`starting_metric`) + ניתוח (LLM, `analysis`). בלי Meta
> (משתמש ב-`_campaign_for_chat`). **(3) `_generate_filtering`** (solution_service) — **אותו מנגנון** כמו
> `generate_solution` (helper `_run_copy_llm` משותף); `generate_solution` מנתב לפיו לפי `problem_type`.
> prompt `filtering_copy` עם הנחיית-סינון נגזרת מ-subcategory (`agent_i18n`, חוק 7). **(4) מדד איכות ב-cron**
> (`_measure_low_quality`) — יחס `irrelevant/tagged` (tagged=irrelevant+closed) בחלון, מול **baseline X
> שנמדד פעם אחת ב-`diagnose`** ונשמר ב-`starting_metric.irrelevant_ratio` — שני החלונות מולו (זהה ל-high_cpl
> מול `cpl`; **לא** before מתגלגל שהיה מסמן regression כוזב בחלון השני — Cursor #1). **בלי RPC/migration**
> (הרחבת `count_leads_in_range` + `count_tagged_and_irrelevant`). **fallback** (חלק 4): tagged<3 → הארכת חלון
> חד-פעמית (+48ש', מסומן ב-`post_change_metric.insufficient_tagging`), אחרת שמרני. **בלי migration**
> (subcategory+X ב-jsonb קיים). models: `Subcategory` enum, `DiagnoseRequest.subcategory`,
> type `category_question`. 20 בדיקות חדשות. **נדחה ל-7.5ב:** שלב 7 (Lead Form דו-שלבי). **patch הפוך
> (7.6):** ה-fallback מודיע ללקוח "סמן לידים" דרך `send_agent_notification` — מתחבר כש-7.6 בונה את התשתית.
>
> **7.5ב ✅ Done (החלק המסוכן — Lead Form core, מבודד):** `lead_form_push_service.push_screening` (דפוס 3.4)
> — יצירת Lead Form חדש עם שאלת סינון + creative-swap שמקשר את כל המודעות לטופס החדש + השהיית הישנות, אטומי
> עם idempotency (save-as-you-go) ו-rollback מלא (כולל מחיקת הטופס החדש; D2→push_rollback_pending+escalate).
> **תובנה שצמצמה את הסיכון:** "טופס + כותרת" הוא **creative-swap יחיד** — ה-`lead_gen_form_id` יושב בתוך
> ה-creative, לא שתי פעולות אטומיות. webhook subscribe הוא **page-level** (כבר מטופל; re-subscribe best-effort).
> **מודול נפרד** (לא ענף ב-optimization_push המסוכן) שעושה reuse של ה-helpers ב-import; **הטופס הישן נשאר**
> (בלי מודעות לא מקבל לידים). migration 0057 (`meta_new_form_id`, state ל-idempotency). 13 טסטי יחידה (Meta
> ממוקק).
>
> **7.5ג ✅ Done (wiring — בעיה 2 מלאה מקצה-לקצה):** חיווט שלב הטופס ל-flow. `step_plan` שלב 4 ל-low_quality
> = `screening_question` (3 זוויות; advance מתקדם 3→4→terminal). `_generate_screening` (solution_service) —
> 3 כותרות מחודדות (**reuse מלא** של `filtering_copy`) + שאלת סינון (`screening_question.txt`, LLM, חוק 7),
> מחזיר `ScreeningProposal`. `generate_solution` מנתב לפיו (שלב 4) ⟶ `propose` מחזיר `screening_question` לצד
> 3 הכותרות; `approve` מנתב לפי `action_type` ל-`lead_form_push_service.push_screening` (חסרה שאלה → 422).
> הזרימה זהה ל-high_cpl (ה-LLM מנסח הכל, אישור אחד — לא input כמו offer). models: `ScreeningProposal`,
> `ProposeResponse`/`ApproveRequest` + `screening_question`. 10 בדיקות חדשות. **בעיה 2 (low_quality_leads)
> הושלמה** — אבחון-דיווח → קופי מסנן (1-3) → טופס סינון (4) → מדד איכות. נותר ידני: pipeline מול Meta sandbox.
>
> בלוק תכנון מלא להעברה ל-CC. **מיקום:** 7.5 — הצ'יפ השני בפרוטוקול. **תלוי ב-7.4א/7.4ב** (תשתית הסוכן,
> מנוע השלבים, דפוס ההעלאה), **7.3.5** (טבלאות), ו-**7.2.5** (עמוד הלידים — שם הלקוח רואה את הלידים המסוננים).
> **כולל פעולת Meta חדשה (7.5ב):** יצירת/החלפת **טופס ליד** (Lead Form), לא רק creative. מסוכן כמו 3.4.

---

## מבוא ותיאור Session

7.4 טיפל ב"עלות גבוהה" — בעיה שמתחילה ממספר. 7.5 מטפל ב**"פניות לא איכותיות"** — בעיה שמתחילה מ**דיווח של הלקוח**, לא ממטריקה.

ההבדל המבני קובע את כל הסשן: אין כאן חוק ברזל, אין benchmark, אין "אתה בטופ". הסוכן **שואל את הלקוח** מה הבעיה (4 קטגוריות), ולפי התשובה בונה פתרון. מה שמשותף לבעיה 1: ה-Lock (פר-`low_quality_leads`), שתי הטבלאות, מחזור 120 שעות, ומנוע השלבים.

7.5 מוסיף **פעולת Meta חדשה** שבעיה 1 לא נגעה בה: **יצירת/החלפת טופס ליד** עם שאלת סינון. זה מתחבר ישירות ל-`screening_answers` בעמוד הלידים (7.2.5) — שם הלקוח רואה אם הסינון עבד.

המטרה: בסוף 7.5, לקוח לוחץ "פניות לא איכותיות" → הסוכן שואל "מה הבעיה?" → לפי הקטגוריה: רענון קופי מסנן (שלבים 1-5), ואם לא עזר — טופס ליד חדש עם שאלת סינון **+ חידוד כותרת** (שלב 7, דו-שלבי) → מעקב 120 שעות → מדידה.

**מה משותף לבעיה 1 (reuse, לא לבנות מחדש):**
- Lock Mechanism (`lock_service`, 7.4א) — פר-`low_quality_leads`.
- פתיחת סדרה + מנוע שלבים (`optimization_service`, 7.4א/ב).
- מחזור אישור→העלאה→120ש'→מדידה (`monitor_optimization_windows`, 7.4ב).
- דפוס ההעלאה האטומית (3.4 / `optimization_push_service`).

**מה חדש ב-7.5:**
- אבחון מבוסס-דיווח: שאלה ללקוח + 4 קטגוריות (chips), לא benchmark.
- `solution_service` מורחב: קופי **מסנן** לפי קטגוריה (LLM, פרומפט פר-קטגוריה).
- **פעולת Meta חדשה:** יצירת/החלפת Lead Form עם שאלת סינון (`lead_form_push_service`).
- מדידת איכות שונה (ראה החלטה 4 — איך מודדים "השתפר" בלי מטריקת CPL).

**מה לא בסשן:**
- בעיה 3 (7.6), מאמן מכירות (7.3 דולג), ניטור יזום/וואטסאפ (Phase 8).
- בדיקת תקינות pipeline מלאה (Meta Lead Form → DB → מייל) — ראה "פתוח".

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מבנה האבחון | **מבוסס-דיווח, לא נתונים.** הסוכן שואל "מה הבעיה בלידים?" עם 4 chips: אזור לא נכון / אין תקציב / לא מבינים את השירות / דרישות סף מקצועיות. **אין חוק ברזל / benchmark.** ("לא רלוונטי" ירד — כללי מדי, מכוסה ע"י ה-4) |
| 2 | שאלת הסינון בטופס | **LLM מנסח** (תלוי באזור/שירות הספציפי), עם אישור לקוח לפני העלאה. **פרומפט ההנחיות פר-קטגוריה נוסח** (חלק 6) — דו-שלבי: שאלת סינון + חידוד כותרת |
| 3 | עדכון טופס ליד ב-Meta | **טופס חדש + קישור מחדש של המודעות + כיבוי ישן.** Meta לא מאפשרת לערוך Lead Form פעיל. אטומי + rollback (דפוס 3.4). מורכב יותר מ-creative — צריך לקשר מחדש את כל המודעות |
| 4 | איך מודדים "השתפר" | **שינוי טקסט (שלבים 1-5): כמו בעיה 1** — אבל אין CPL "בעיה". המדד: ירידה ב-CPL לא רלוונטית כאן. נמדוד **יחס לידים מסומנים `irrelevant`/`closed` בעמוד הלידים** (ראה חלק 4). **טופס סינון (שלב 7): פחות לידים אבל איכותיים** — נמדוד את היחס, לא את הכמות |
| 5 | שלב 6 | אין שלב 6 נפרד (גיא: "7 זה המשך של 5"). בעיה 2 = **רענון קופי מסנן → טופס סינון + חידוד כותרת**. שלב 7 דו-שלבי (טופס + כותרת, אישור אחד) |
| 6 | reuse מבעיה 1 | **מקסימלי.** Lock/סדרה/מנוע-שלבים/מחזור-מדידה/דפוס-העלאה — הכל מ-7.4. 7.5 מוסיף רק אבחון-דיווח + קופי-מסנן + פעולת-טופס |

---

## תלויות

1. **7.4א/7.4ב** — כל תשתית הסוכן: orchestrator, Lock, סדרה, מנוע שלבים, cron מעקב, דפוס העלאה. 7.5 בנוי עליהם.
2. **7.2.5 — עמוד הלידים.** **קריטי.** שם הלקוח רואה את הלידים המסוננים + `screening_answers`. בלי זה, שלב 7 (טופס סינון) אין לו לאן לנחות. (נבנה כעת.)
3. **7.3.5** — `optimization_sessions`/`optimization_actions`.
4. **3.1ב** — קונפיג שדות טופס ליד (Meta Lead Form). 7.5 מרחיב — הסוכן יוצר טופס חדש עם שאלת סינון.
5. **4.1** — `leads` עם `screening_answers`. שם נוחתות תשובות הסינון.
6. **3.4** — דפוס ההעלאה האטומית. 7.5 מאמץ אותו לטופס ליד.

---

## חלק 1 — אבחון מבוסס-דיווח (האורקסטרטור, מורחב)

הרחבת `agent_orchestrator` (7.4א). מסלול חדש ל-`problem_type='low_quality_leads'`.

**`diagnose_problem_2(user_id, conversation_id, campaign_id) -> AgentResponse`:**

1. **Lock** — `lock_service.check_lock(campaign_id, 'low_quality_leads')`. נעול → תגובת Lock (כמו 7.4א). **שים לב:** סדרת `high_cpl` פתוחה **לא** נועלת את זה (בעיות בלתי-תלויות, partial unique index מ-7.3.5).
2. **אין benchmark.** דילוג מוחלט על חוק הברזל — לא רלוונטי לבעיה 2.
3. **שאלה ללקוח** — הסוכן מציג: "מה הבעיה המרכזית בלידים?" עם 4 chips:
   - `wrong_area` — לא באזור הרצוי
   - `no_budget` — אין תקציב
   - `dont_understand` — לא מבינים את השירות
   - `standards` — לא עומדים בדרישות סף מקצועיות
4. **המתנה לבחירה** — זה turn נפרד. הלקוח בוחר קטגוריה → endpoint `diagnose` שני עם ה-`subcategory`.
5. אחרי הבחירה: **ניתוח** (מאיזו מודעה/קופי הגיעו הלידים — שאילתת DB על `leads`, דטרמיניסטי) → פתיחת סדרה → המשך ליצירת פתרון.

> **בניגוד לבעיה 1**, אין כאן "תרחיש א'/ב'" (טופ/חורג). תמיד יש בעיה (הלקוח דיווח עליה). השאלה היא רק **איזו** — וזה קובע את סוג הפתרון.

---

## חלק 2 — `solution_service`: קופי מסנן (LLM)

הרחבת `solution_service` (7.4ב). פונקציה חדשה לבעיה 2.

**`generate_filtering_solution(session_id, subcategory) -> Solution`:**
- שולף סדרה + קמפיין + `business_description` + אזור/שירות.
- **קופי מסנן** — בניגוד לבעיה 1 (שמושך יותר), כאן הקופי **מסנן החוצה**: מסר שמרתיע לא-מתאימים. 3 וריאציות.
- הפרומפט מקבל את ה-`subcategory` ומנחה בהתאם:
  - `wrong_area` → הדגשת האזור.
  - `no_budget` → רמז למחיר בכותרת/קופי.
  - `dont_understand` → חידוד מהות השירות.
  - `standards` → הדגשת תנאי הסף המקצועי.
::: deprecated
~~- **פרומפט ההנחיות פר-קטגוריה נוסח (חלק 6)** — דו-שלבי (שאלת סינון + חידוד כותרת), 4 קטגוריות. ה-LLM מנסח לפי הקטגוריה שהלקוח בחר.~~
:::

---

## חלק 3 — שלב 7: פעולה דו-שלבית (טופס ליד + חידוד כותרת)

מודול חדש `app/services/lead_form_push_service.py` (הטופס) + שימוש ב-`optimization_push_service` (7.4ב, הכותרת). **דפוס 3.4 — אטומי, מסוכן.**

שלב 7 הוא **דו-שלבי** (כפי שעלה מהפרוטוטייפ): שאלת סינון בטופס **וגם** חידוד כותרת המודעה. שתי פעולות Meta נפרדות, **באישור אחד** של הלקוח.

### למה הטופס מורכב יותר מ-creative

עדכון creative (7.4ב) נגע במודעה בודדת. החלפת **טופס ליד** דורשת:
1. יצירת Lead Form חדש ב-Meta (עם השדות הקיימים + שאלת הסינון החדשה).
2. **קישור מחדש של כל המודעות** בקמפיין לטופס החדש (ה-Ads מצביעים על ה-Lead Form).
3. כיבוי/מחיקת הטופס הישן.

כשל באמצע = חלק מהמודעות מצביעות על הטופס החדש, חלק על הישן — מצב לא-עקבי. **rollback קריטי** (דפוס 3.4).

### הזרימה (שתי הפעולות תחת אישור אחד)

1. **ניסוח** (LLM, חלק 2): גם שאלת הסינון **וגם** הכותרת המחודדת → **אישור לקוח אחד** על שניהם (תוכן חדש, §9 בספ).
2. **כתיבת `optimization_actions`** — `action_type='screening_question'`, `change_description` שמתאר את **שתי** הפעולות ("שאלת סינון: 'מאיזה אזור?' + חידוד כותרת"), `requires_approval=true`, `approved_at=now()`, `window_ends_at=NULL` (עד הצלחה). **פעולה אחת** ברשומה, שמבצעת שני שינויי Meta.
3. **העלאה אטומית — שתי הפעולות יחד** (state machine, דפוס 3.4):
   - **א. הכותרת:** עדכון creative של המודעות (כותרת מחודדת) — `optimization_push_service` (7.4ב).
   - **ב. הטופס:** יצירת Lead Form חדש (שדות קיימים מ-`lead_form_fields` + שאלת הסינון) → קישור מחדש של כל ה-Ads לטופס החדש → כיבוי הישן.
   - idempotency: כל `meta_*_id` חדש נשמר.
   - **8 gates** כמו 3.4.
   - **אטומיות חוצת-פעולות:** אם הכותרת עלתה אבל הטופס נכשל (או להפך) — rollback של **שתיהן**. לא להשאיר מצב שבו כותרת מחודדת אבל הטופס הישן עדיין פעיל (או להפך). זה הופך את ה-state machine למורכב יותר מ-7.4ב — שני שינויים תחת טרנזקציה לוגית אחת.
4. **בכשל** — rollback מלא: החזרת ה-creative הישן + החזרת כל ה-Ads לטופס הישן + מחיקת הטופס החדש. כשל ב-rollback → `failed_rollback_pending` + Sentry.
5. **בהצלחה** (שתי הפעולות עברו) — `window_ends_at = now() + 120h`. **subscribe ל-webhook של הטופס החדש** (כמו 4.1 — אחרת לידים מהטופס החדש לא ייכנסו!).

> **קריטי — webhook subscription לטופס החדש.** ב-4.1 ה-subscribe לעמוד קורה אחרי יצירת ה-Lead Form. טופס חדש = צריך לוודא שה-webhook קולט גם ממנו. אחרת הסינון "עובד" אבל הלידים נעלמים. לאמת מול מנגנון ה-subscribe של 4.1/3.4.

> **שתי פעולות, אישור אחד, אטומיות אחת.** זה ההבדל המהותי מ-7.4ב (פעולה בודדת). שלב 7 של בעיה 2 = טופס + כותרת, ושתיהן חייבות להצליח או להתבטל יחד. אם הדבר מסבך מדי את ה-state machine — אופציה נסיגה: לבצע אותן רציפות עם rollback מדורג (קודם כותרת, ואם הטופס נכשל, להחזיר גם את הכותרת). לתעד את ההכרעה ל-CC.

> **`screening_answers` הוא ה-payoff.** הלידים מהטופס החדש נכנסים עם תשובות הסינון ב-`screening_answers`, והלקוח רואה אותן בעמוד הלידים (7.2.5). זה סוגר את הלולאה.

---

## חלק 4 — מדידת איכות (cron, מורחב)

בעיה 1 מדדה CPL. בעיה 2 מודדת **איכות** — וזה שונה.

### הבעיה: אין מטריקת "איכות" אוטומטית ב-Meta

"איכות ליד" היא שיפוט של הלקוח, לא מספר מ-Insights. מאיפה ה-cron יודע אם הסינון עבד?

### ההכרעה: יחס הלידים המסומנים בעמוד הלידים

הלקוח מתייג לידים בעמוד הלידים (7.2.5): `closed` (סגר — איכותי) / `irrelevant` (לא רלוונטי). זה הסיגנל.

**מדד האיכות (דטרמיניסטי):**
- לפני השינוי (`starting_metric`): יחס `irrelevant / total` בלידים מה-X ימים שלפני.
- אחרי (`post_change_metric`): יחס `irrelevant / total` בלידים מאז השינוי.
- **שיפור = ירידה ביחס ה-`irrelevant`** (פחות לידים לא רלוונטיים).

> **תלות ב-7.2.5:** המדידה הזו עובדת רק אם הלקוח **מתייג** לידים. אם הלקוח לא מתייג — אין סיגנל. **fallback:** אם אין מספיק לידים מתויגים, ה-cron לא יכול להכריע → משאיר את הסדרה ב-monitoring ומודיע ללקוח (בכניסה הבאה) "סמן לידים כדי שאוכל למדוד אם הסינון עבד". לתעד.

> **הערה למספרים:** ב-MVP מעט לידים מתויגים → המדד רועש. זה מקובל — בעיה 2 פחות שכיחה מבעיה 1, ולקוח שמתלונן על איכות בד"כ כן מתייג. אם נראה בעיה — נשקול מדד משלים (כמות לידים, שיעור פניות) ב-Phase 8.

מנוע ההחלטה (`monitor_optimization_windows`, 7.4ב) מורחב לטפל גם ב-`problem_type='low_quality_leads'` עם המדד הזה במקום CPL.

---

## חלק 5 — Endpoints

**א. `POST .../diagnose`** (מורחב מ-7.4א): מקבל `problem_type='low_quality_leads'`. שלב ראשון → מחזיר 4 chips. שלב שני (עם `subcategory`) → ניתוח + פתיחת סדרה.

**ב. `POST .../propose`** (מורחב מ-7.4ב): לבעיה 2 — קורא ל-`generate_filtering_solution` עם ה-subcategory.

**ג. `POST .../approve`** (מורחב): מבחין בין סוג הפעולה:
- שלבי קופי (1-5) → `optimization_push_service` (creative, 7.4ב).
- שלב טופס (7) → `lead_form_push_service` (חדש, חלק 3).

Response כולל את סוג הפעולה שבוצעה.

---

::: important
# **חלק 6 בוטל!**

_**מבנה הפרומפט של בעיה 2 — ראה:
agent_prompt_architecture.md,
module_low_quality.txt + payload. 
כללי-ניסוח-השאלה עברו לפרומפט היצירה של בעיה 2 (נכתב בחלק 2).**_
::: 

---

## חלק 7 — שינויים נדרשים בקוד הקיים

**א.** `agent_orchestrator.py` — `diagnose_problem_2` + מסלול 4 הקטגוריות (חלק 1).
**ב.** `solution_service.py` — `generate_filtering_solution` (חלק 2).
**ג.** `lead_form_push_service.py` — **חדש** (חלק 3, דפוס 3.4 + קישור מחדש + webhook subscribe).
**ד.** `worker/handlers.py` — `monitor_optimization_windows` מורחב למדד איכות (חלק 4).
**ה.** `routers/agent.py` — `diagnose`/`propose`/`approve` מורחבים ל-low_quality_leads + הבחנת סוג פעולה (חלק 5).
~~**ו.** `prompts/agent/problem_2.txt` — חדש, עם הנחיות הסינון המנוסחות (חלק 6).~~
ו. "prompts/agent/module_low_quality.txt — לפי agent_prompt_architecture (לא problem_2.txt)."
**ז.** `models/` — `Subcategory` enum, `QualityMetric`.

---

## חלק 8 — בדיקות

**אבחון:**
1. צ'יפ "פניות לא איכותיות" → 4 chips, בלי benchmark.
2. סדרת `high_cpl` פתוחה → **לא** נועל את `low_quality_leads` (בעיות שונות).
3. בחירת `wrong_area` → ניתוח + פתיחת סדרה `low_quality_leads`.

**קופי מסנן:**
4. `wrong_area` → קופי שמדגיש אזור.
5. `no_budget` → רמז מחיר.
6. `standards` → קופי שמדגיש תנאי סף.
7. כל קטגוריה → 3 וריאציות.

**טופס ליד (קריטי, דפוס 3.4):**
7. יצירת טופס חדש + קישור מחדש כל המודעות → הצלחה, `window_ends_at` נקבע.
8. כשל בקישור מודעה 2 → rollback: כל המודעות חוזרות לטופס ישן, טופס חדש נמחק.
9. **webhook subscribe לטופס החדש** → ליד מהטופס החדש נכנס ל-`leads` עם `screening_answers`.
10. כשל ב-rollback → `failed_rollback_pending` + Sentry.

**מדידת איכות (cron):**
11. יחס `irrelevant` ירד אחרי שינוי → שיפור → `success_monitoring`.
12. יחס `irrelevant` שווה/עלה → אין שיפור → שלב הבא.
13. **אין לידים מתויגים** → ה-cron לא מכריע, משאיר monitoring + הודעה ללקוח.

**e2e:**
14. צ'יפ → 4 קטגוריות → בחירה → קופי מסנן → אישור → Meta → מעקב.
15. שלב 7 (דו-שלבי): טופס סינון + כותרת מחודדת → אישור אחד → טופס חדש + creative מעודכן ב-Meta (אטומי יחד) → ליד נכנס עם `screening_answers` → מופיע בעמוד הלידים (7.2.5).

---

## חלק 9 — פתוח לגיא/אמיר

1. **קטגוריה חמישית?** ירדנו ל-4 (אזור/תקציב/לא-מבינים/סטנדרטים) והסרנו "לא רלוונטי" (כללי מדי). אם בשטח עולה בעיה שלא נכנסת ל-4 — לשקול הוספה. (פרומפט ההנחיות עצמו כבר נוסח, חלק 6.)
2. **מדד איכות תלוי בתיוג הלקוח.** אם לא מתייג — אין סיגנל. מקובל ל-MVP? או מדד משלים?
3. **בדיקת תקינות pipeline** (Meta Lead Form → DB → מייל) — הפרומפט הזכיר MAKE/Sheets (ירדו). מה שנשאר: לוודא שהטופס החדש מחובר ל-webhook ושלידים נכנסים. עד כמה לאמת אוטומטית לפני הפעלה? (סשן בדיקת-תקינות נפרד?)
4. **כמה שאלות סינון אפשר להוסיף?** (פרוטוטייפ §7.2 — עד 4 באשף. הסוכן מוסיף מעבר לזה?)

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **reuse מקסימלי מ-7.4.** Lock/סדרה/מנוע-שלבים/מחזור-מדידה/דפוס-העלאה — לא לבנות מחדש. 7.5 מוסיף רק אבחון-דיווח + קופי-מסנן + פעולת-טופס.

2. **בעיה 2 אין לה benchmark.** דילוג מוחלט על חוק הברזל. האבחון מבוסס שאלה ללקוח, לא מספר.

3. **טופס ליד מורכב יותר מ-creative.** קישור מחדש של **כל** המודעות + webhook subscribe. כשל = לידים נעלמים. rollback קריטי. דפוס 3.4.

4. **webhook subscribe לטופס החדש — אל תשכח.** טופס חדש בלי subscribe = לידים לא נכנסים. לאמת מול 4.1/3.4.

5. **`screening_answers` הוא ה-payoff.** הלידים החדשים נכנסים עם תשובות הסינון → עמוד הלידים (7.2.5). זה סוגר את החידה.

6. **מדד איכות = יחס `irrelevant` בעמוד הלידים, לא CPL.** דטרמיניסטי. תלוי בתיוג הלקוח (7.2.5). אם אין תיוג — אין הכרעה.

::: deprecated
~~7. **הנחיות הסינון נוסחו (חלק 6) — דו-שלבי.** שאלת סינון + חידוד כותרת, 4 קטגוריות (אזור/תקציב/לא-מבינים/סטנדרטים). ה-LLM מנסח לפי הקטגוריה. שלב 7 מבצע **שתי** פעולות Meta תחת אישור אחד.~~
:::

8. **שלב 6 לא קיים בבעיה 2** — קופי-מסנן (1-5) → שלב 7 דו-שלבי (טופס סינון + חידוד כותרת, שתי פעולות Meta תחת אישור אחד).

9. **כל בעיה 2 דורשת אישור** (`requires_approval=true`). אין אוטונומי — Phase 8.

10. **Sentry context** — `session_id`, `subcategory`, `campaign_id`, ובטופס: השלב ב-state machine + אילו מודעות קושרו מחדש.

11. **אין migration חדש** — טבלאות מ-7.3.5 (`screening_question` הוא ערך ב-`action_type`, text חופשי, לא enum — תואם). רק קוד, prompt, ופעולת Meta.


---

# Session 7.6 — בעיה 3: מודעה נדחתה + תשתית התראות יזומות

> **פוצל ל-2 (אישור המשתמש): 7.6א (תשתית) → 7.6ב (בעיה 3).**
> **7.6א ✅ Done (תשתית ההתראות היזומות + patch הפוך):** `notification_service.send_agent_notification`
> (wrapper דק על תשתית 4.6 — create_notification_and_job → worker → Resend; idempotent על anchor; channel
> פרמטרי, email ב-MVP). 3 types חדשים: `step_advanced`, `series_resolved`, `needs_lead_tagging` (+`ad_rejected`
> מוכן ל-7.6ב). `build_agent_notification_email` (deep-link דינמי פר-type — `_agent_link`). migration 0059
> (CHECK +4). **patch הפוך ל-cron** (`optimization_monitor_service._notify_for_outcome`, best-effort): מעבר
> שלב → step_advanced (רק אם יש שלב הבא); סיום מוצלח → series_resolved; **fallback בעיה 2 (extended) →
> needs_lead_tagging** ("סמן לידים" — מחבר את ה-7.5א patch); improved/skipped → אין מייל (תצפית, החלטה 6).
> חילוץ `_apply_window` (לא נגיעה בלוגיקת ה-CAS). 10 בדיקות חדשות. **מבטל את ההמתנה ל-Phase 8** — המייל
> היזום הוא MVP.
>
> **⚠️ ROLLBACK (2026-06, ר' 7.5):** `needs_lead_tagging` + `insufficient_data` הוסרו מה-enum/templates/handler;
> ה-no_signal-3-way (`_apply_no_signal`) ירד (cpl=None/x=None → אי-שיפור→advance, כמו regression). נשארו 3
> types (`step_advanced`, `series_resolved`, `ad_rejected`). ה-CHECK ב-DB נשאר רחב (insert-only log; אף שורה
> חדשה לא תיווצר; `test_notification_type_check` בודק subset → ירוק).
>
> **7.6ב ✅ Done (תשתית בעיה 3 — מיפוי + תיקון קופי, מבודד):** `meta.get_ad(token, ad_id)` (Graph API,
> שולף `effective_status` + `ad_review_feedback`). `rejection_service` — מיפוי דטרמיניסטי של סיבות דחייה
> ל-4 קטגוריות (claims/personal_attributes/prohibited/unmapped, substring case-insensitive), `fix_instruction`
> פר-קטגוריה (חוק 7 — הקוד גוזר, ה-LLM כותב), ו-`extract_rejection_reason` (parse של ה-feedback מ-Meta).
> `solution_service.generate_rejection_fix(meta_ad_id, rejection_reason)` — שולף את הקופי הישן, מסווג,
> ומנסח **גרסה מתוקנת אחת** (לא 3 וריאציות — חוק 7) ששומרת על אותו angle (`rejection_fix.txt`).
> `ads_service.fetch_ad_by_meta_id`. 20 בדיקות חדשות (Meta+LLM ממוקקים).
>
> **7.6ג ✅ Done (wiring — בעיה 3 מלאה, Phase 7 הושלם):** webhook field dispatch (leadgen↔ad_review) ב-
> `routers/webhooks.py` + `lead_intake_service` מסנן field≠leadgen. `rejection_intake_service` (חדש,
> מקביל ל-lead_intake): dedup (`webhook_events`), `find_by_ad_id`, `check_lock`, `meta.get_ad` (effective_
> status + ad_review_feedback), `open_session(meta_rejection, {ad_id, reason})`, enqueue. `handle_meta_
> rejection` job (חדש ב-worker, idempotent via RPC race-safe) → `generate_rejection_fix` → INSERT action
> (`open_push_action` קיים) → `send_agent_notification(AD_REJECTED)`. `rejection_push_service` (חדש, דפוס
> 3.4 מצומצם — push למודעה יחידה: creative+ad+delete הנדחית, **בלי** `_gate_live_old_ads`/equality 3:3),
> reuse של helpers מ-optimization_push. `finalize_push_action(window_hours=None)` — בעיה 3 ללא מחזור
> מדידה (החלטה 5). `diagnose_problem_3` (orchestrator, בלי LLM — התיקון מ-DB), מציג ב-`DiagnoseResponse.
> proposal` (תבנית STAGE_C ג'-2). router: diagnose+approve dispatch לפי problem_type. migration 0061
> (jobs.type CHECK). 28 בדיקות חדשות. **Phase 7 הושלם — 3 הצ'יפים בפרוטוקול עובדים מקצה לקצה.**
> **נדחה ל-Phase 8 / ידני:** subscribe ל-`ad_review` ב-Meta App Dashboard, smoke test מול sandbox.

> בלוק תכנון מלא להעברה ל-CC. **מיקום:** 7.6 — הצ'יפ השלישי והאחרון בפרוטוקול. **תלוי ב-7.4א/7.4ב** (דפוס העלאה, תיקון קופי), **4.6** (תשתית התראות).
> **הסשן האחרון בפרוטוקול** — סוגר את בעיה 3 **וגם** בונה את תשתית המייל היזום שחסרה ל-7.4ב/7.5.

---

## מבוא ותיאור Session

בעיות 1-2 חיכו ללחיצת צ'יפ. **בעיה 3 מתחילה מ-Meta** — מודעה נדחתה, Meta שולחת webhook, והסוכן מזהה את זה **יזום** ופונה ללקוח. זה ההפך מהבעיות הקודמות, ומכניס את הרכיב הראשון של "הסוכן היוזם" בצורה מצומצמת וברורה (טריגר דחייה, לא ניטור פרואקטיבי).

בעיה 3 גם **הקצרה ביותר**, כי היא מוותרת על כל המנוע הכבד:
- **אין מחזור מדידה** — מודעה מאושרת או לא, בינארי ומיידי. אין 120 שעות, אין `success_monitoring`.
- **אין לולאת שלבים** — סיבת דחייה → תיקון קופי → העלאה מחדש. זהו.
- **הפעולה: תיקון קופי בלבד** — Meta מחזירה את סיבת הדחייה, הסוכן מתקן בהתאם ומעלה מחדש (דפוס 3.4).

7.6 בונה **שני דברים**:
1. **בעיה 3** — webhook דחייה → מיפוי סיבה → תיקון קופי (LLM) → אישור → העלאה מחדש.
2. **תשתית התראות יזומות כללית** (`send_agent_notification`) — סוג חדש על 4.6, שכל פעולה יזומה משתמשת בו (דחייה, מעבר שלב, סיום מוצלח). **זה החלק שחסר ל-7.4ב/7.5** — בנוי כאן פעם אחת, נקי.

המטרה: בסוף 7.6, מודעה שנדחית → הסוכן מזהה תוך דקות (webhook) → שולח מייל ("המודעה נדחתה, הכנתי תיקון") עם לינק לעמוד → הלקוח מאשר → התיקון עולה → Meta בודקת שוב. **ובמקביל**, כל פעולה יזומה במערכת (מעבר שלב, סיום סדרה) שולחת מייל.

**מה בסשן:**
- webhook דחיית מודעה (`POST /webhooks/meta/ad-review` או הרחבת webhook קיים).
- `rejection_service` — מיפוי סיבת דחייה → הנחיית תיקון.
- תיקון קופי (LLM) + העלאה מחדש (דפוס 3.4).
- **תשתית `send_agent_notification`** — סוג חדש על 4.6, עם deep-link דינמי וערוץ פרמטרי.
- חיבור 3 הטריגרים: דחייה (7.6), מעבר שלב (7.4ב), סיום סדרה (7.5/7.4ב).
- system prompt: בעיה 3 מהפרומפט של גיא.

**מה לא בסשן:**
- ניטור פרואקטיבי (Phase 8) — הסוכן שמזהה CPL גבוה בעצמו. בעיה 3 יזומה רק מ-webhook דחייה, לא מסריקה.
- **התראות וואטסאפ** (Phase 8, `agent_alerts_quota`). התשתית **מוכנה** לזה (ערוץ פרמטרי), אבל MVP = מייל בלבד.
- אישור מתוך המייל (auth flow) — המייל מוביל לצ'אט/עמוד, לא מאשר ישירות.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | איך מזהים דחייה | **webhook מ-Meta** (לא polling). Meta שולחת על שינוי סטטוס מודעה. הסוכן מזהה תוך דקות, לא בתקופה |
| 2 | מבנה בעיה 3 | **בלי מחזור מדידה, בלי לולאה.** סיבת דחייה → תיקון קופי → העלאה מחדש. Meta בודקת שוב. בינארי |
| 3 | הפעולה | **תיקון קופי בלבד.** רוב הדחיות על מדיניות (טקסט מטעה, claims, מילים אסורות). Meta מחזירה סיבה, הסוכן מתקן |
| 4 | מייל יזום על דחייה | **כן.** יש תיקון שמחכה לאישור — בעל העסק חייב לדעת |
| 5 | תשתית המייל | **כללית — `send_agent_notification` על 4.6.** לא ספציפי לדחייה. כל פעולה יזומה קוראת לה (שורש, לא טלאי) |
| 6 | מתי נשלח מייל יזום | **כשנדרשת תשומת לב/פעולה מבעל העסק**, לא על אירוע רקע. דחייה ✅ / מעבר שלב ✅ / סיום סדרה ✅ / מדידת ביניים ❌ |
| 7 | deep-link במייל | **דינמי פר-סוג.** לא "דשבורד" כללי — לינק לעמוד הספציפי (קופי חדש / קמפיין / לידים) |
| 8 | ערוץ ההתראה | **פרמטר, לא קשיח.** `channel='email'` ב-MVP. מוכן ל-`whatsapp` ב-Phase 8 (`agent_alerts_quota`). לא לקודד "מייל" בכל מקום |
| 9 | אישור מתוך המייל | **לא ב-MVP.** המייל מודיע + לינק לעמוד. אישור דורש כניסה (auth flow קיים) |

---

## תלויות

1. **7.4א/7.4ב** — דפוס ההעלאה האטומית (3.4), תיקון קופי, ה-orchestrator, פתיחת סדרה. בעיה 3 משתמשת בהם (בלי המחזור).
2. **4.6 — System Notifications.** תשתית `send_notification` + `sent_notifications` + Resend. 7.6 מרחיב ל-`send_agent_notification`.
3. **3.4** — דפוס העלאה אטומי לתיקון הקופי.
4. **3.3/3.2** — יצירת קופי (gpt-5.2).
5. **7.2** — `get_campaign_status_for_agent` (לעמוד שאליו ה-deep-link מצביע).
6. **7.4ב/7.5** — **תלות הפוכה:** הם יתחברו לתשתית `send_agent_notification` שנבנית כאן (patch — ראה חלק 5).

---

## חלק 1 — webhook דחיית מודעה

הרחבת מנגנון ה-webhooks של Meta (4.1 כבר מקבל webhooks לידים; כאן סוג נוסף — ad review).

**`POST /webhooks/meta/ad-review`** (או הרחבת ה-webhook הקיים עם field `ad_review`):
- Meta שולחת על שינוי סטטוס מודעה ל-`DISAPPROVED` / `REJECTED` / `WITH_ISSUES`.
- **אימות חתימה** — כמו 4.1 (X-Hub-Signature). דחיית webhook לא-מאומת.
- **idempotency** — כמו 4.1 (`webhook_events`). Meta שולחת לפעמים כפול.
- ה-payload כולל את ה-`ad_id`, הסטטוס, ו**סיבת הדחייה** (`ad_review_feedback` / policy violation).

זרימת ה-handler:
1. מאמת + dedup.
2. שולף את הקמפיין/מודעה מה-`ad_id` (דרך `meta_ad_id` בטבלאות שלנו).
3. בודק Lock — סדרת `meta_rejection` פתוחה כבר לקמפיין הזה? (נדיר, אבל אם כבר טיפלנו בדחייה — לא לפתוח שנייה).
4. פותח סדרה `optimization_sessions` (`problem_type='meta_rejection'`).
5. enqueue job: `rejection_service.handle_rejection` + `send_agent_notification` (מייל ללקוח).

> **`meta_rejection` כבר ב-enum** של `optimization_sessions.problem_type` (נבנה ב-7.3.5). אין migration.

> **למה webhook ולא polling** (החלטה 1): דחייה דורשת תגובה מהירה (הקמפיין לא רץ = כסף/זמן אבוד). webhook = דקות; polling = שעות. Meta תומכת ב-webhook על ad review.

---

## חלק 2 — `rejection_service` (מיפוי + תיקון)

מודול חדש `app/services/rejection_service.py`.

**`handle_rejection(session_id, ad_id, rejection_reason) -> Solution`:**
1. **מיפוי הסיבה** — Meta מחזירה קוד/טקסט סיבה (למשל "misleading content", "prohibited content", "personal attributes"). מיפוי דטרמיניסטי לקטגוריית תיקון:
   - claims מטעים → לרכך הבטחות, להסיר superlatives.
   - personal attributes → להסיר פנייה ישירה לתכונה אישית ("אתה סובל מ...").
   - prohibited words → להסיר/להחליף מילים אסורות.
   - לא ממופה → הנחיה כללית "תקן לפי המדיניות" + הסיבה הגולמית ל-LLM.
2. **תיקון קופי (LLM)** — `solution_service` (7.4ב, מורחב) מקבל את הקופי הנדחה + קטגוריית התיקון, ומחזיר גרסה מתוקנת. **לא 3 וריאציות** — תיקון ממוקד אחד (בניגוד לבעיה 1).
3. מחזיר את הקופי המתוקן (טרם הועלה — אישור קודם).

> **המיפוי דטרמיניסטי, התיקון LLM** (חוק 7). הקוד ממפה סיבה → קטגוריה, ה-LLM מנסח את התיקון.

---

## חלק 3 — אישור + העלאה מחדש

שימוש ב-`optimization_push_service` (7.4ב) — אותו דפוס, פשוט יותר (אין מחזור).

1. **אישור לקוח** — הלקוח אישר את הקופי המתוקן.
2. **כתיבת `optimization_actions`** — `action_type='rejection_fix'`, `change_description` ("תיקון דחייה: {סיבה}"), `requires_approval=true`, `approved_at=now()`. **`window_ends_at=NULL` — ונשאר NULL** (אין מחזור מדידה!).
3. **העלאה אטומית** (דפוס 3.4) — עדכון ה-creative של המודעה הנדחית עם הקופי המתוקן.
4. **בהצלחה** — Meta תבדוק את המודעה מחדש אוטומטית. הסדרה עוברת ל-`done` (אין מה למדוד — או שתאושר או שתידחה שוב).
5. **אם תידחה שוב** — webhook נוסף → סדרה חדשה / המשך טיפול. (ב-MVP: סדרה חדשה. טיפול חוזר מתוחכם — Phase 8.)

> **אין `window_ends_at`, אין cron מעקב.** בעיה 3 לא נכנסת ל-`monitor_optimization_windows`. ההצלחה/כישלון מגיעים מ-webhook הבא של Meta, לא ממדידה יזומה שלנו. זה מה שמייתר את כל המנוע של 7.4ב.

---

## חלק 4 — תשתית `send_agent_notification` (הליבה הכללית)

מודול: הרחבת `notification_service` (4.6). **זה החלק ששירת את כל הפרוטוקול — נבנה כאן פעם אחת.**

**`send_agent_notification(user_id, notification_type, context, channel='email')`:**
- **על תשתית 4.6** — משתמש ב-`send_notification` הקיים, `sent_notifications` ל-idempotency, Resend לשליחה.
- **`channel` פרמטר** (החלטה 8) — `email` ב-MVP. הפונקציה כתובה כך ש-`whatsapp` יהיה הוספת ענף, לא שכתוב.
- **deep-link דינמי פר-סוג** (החלטה 7) — `context` כולל את ה-target, וכל סוג בונה לינק לעמוד הרלוונטי.

**4 סוגי התראות יזומות:**

| `notification_type` | טריגר | תוכן | deep-link |
|---|---|---|---|
| `ad_rejected` | webhook דחייה (7.6) | "המודעה נדחתה. הכנתי תיקון, היכנס לאשר." | עמוד הקופי המתוקן |
| `step_advanced` | מעבר שלב (7.4ב) | "הפתרון הקודם לא הספיק. הכנתי הצעה חדשה לאישור." | עמוד הקופי החדש |
| `series_resolved` | סיום מוצלח (7.5/7.4ב) | "הבעיה שדיווחת עליה נפתרה. העלות התייצבה על {X}." | עמוד הקמפיין/מטריקות |
| `new_lead` | ליד חדש (7.2.5) | "ליד חדש: {שם}, {טלפון}." | עמוד הלידים |

> **`new_lead` כבר קיים מ-7.2.5** — כאן הוא מתאחד תחת אותה מטרייה (`send_agent_notification` או נשאר נפרד; ל-CC להחליט אם לאחד או להשאיר, העיקר שהדפוס זהה). 3 הסוגים האחרים חדשים.

**idempotency:** `sent_notifications` עם מפתח `(notification_type, session_id/action_id/lead_id)` — לפי הסוג. כל אירוע → מייל אחד.

**הכלל מתי שולחים** (החלטה 6): **תשומת לב/פעולה נדרשת**, לא אירוע רקע.
- ✅ דחייה, מעבר שלב, סיום סדרה.
- ❌ מדידת ביניים שגרתית (`success_monitoring` שממשיך תקין) — תצפית, לא פעולה. **אין מייל.**

---

## חלק 5 — חיבור 7.4ב/7.5 לתשתית (patch הפוך)

**זה התיקון של מה שדחינו בטעות ל-Phase 8 ב-7.4ב/7.5.** עכשיו שהתשתית קיימת, מחברים אליה:

**א. patch ל-7.4ב (`monitor_optimization_windows`):**
- כשה-cron מחליט **מעבר שלב** (מדידה נכשלה, `current_step < 7`) → קריאה ל-`send_agent_notification(type='step_advanced', ...)`. **כאן הלקוח כבר לא "רואה רק בכניסה הבאה"** — הוא מקבל מייל.
- כשה-cron מחליט **סיום מוצלח** (`success_monitoring` הסתיים, יציב, `done`) → `send_agent_notification(type='series_resolved', ...)`.
- כש-`success_monitoring` **ממשיך תקין** (מדידת ביניים, לא סיום) → **אין מייל** (תצפית רקע).

**ב. patch ל-7.5:**
- אותו דבר — מעבר שלב בבעיה 2 → `step_advanced`; סיום מוצלח → `series_resolved`.

> **לתעד ב-ROADMAP:** 7.4ב ו-7.5 קיבלו "לעשות" שאומר "הלקוח רואה בכניסה הבאה (מייל יזום — Phase 8)". 7.6 **מבטל את ההמתנה הזו** — המייל היזום הוא חלק מ-MVP, לא Phase 8. לעדכן את הסעיפים הרלוונטיים ב-7.4ב/7.5 כשמחברים.

---

::: important
# חלק 6 בוטל! 
מבנה הפרומפט של בעיה 3 — ראה agent_prompt_architecture, module_rejection.txt + payload.
:::

---

## חלק 7 — שינויים נדרשים בקוד הקיים

**א.** webhook handler ל-ad review (חלק 1) — `routers/webhooks.py` (הרחבה) או חדש.
**ב.** `rejection_service.py` — חדש (חלק 2).
**ג.** `solution_service.py` (7.4ב) — מורחב: תיקון קופי ממוקד (לא 3 וריאציות) לבעיה 3.
**ד.** `optimization_push_service.py` (7.4ב) — reuse להעלאה מחדש (חלק 3).
**ה.** `notification_service.py` (4.6) — **`send_agent_notification`** + 4 סוגים + deep-link + ערוץ פרמטרי (חלק 4). **הליבה.**
**ו.** `worker/handlers.py` — patch: `monitor_optimization_windows` קורא ל-`send_agent_notification` על מעבר שלב/סיום (חלק 5).
**ז.** `agent_orchestrator.py` — מסלול `meta_rejection` (הצגת התיקון בצ'אט כשהלקוח נכנס מהלינק).
::: deprecated
~~**ח.** `prompts/agent/problem_3.txt` — חדש (חלק 6).~~
:::

**ט.** `models/` — `RejectionReason`, `AgentNotification`.

---

## חלק 8 — בדיקות

**webhook דחייה:**
1. webhook לא-מאומת → נדחה.
2. webhook כפול → dedup, סדרה אחת.
3. דחייה → סדרת `meta_rejection` נפתחת + מייל נשלח.
4. דחייה כשכבר יש סדרת `meta_rejection` פתוחה → לא נפתחת שנייה (Lock).

**`rejection_service`:**
5. סיבה "misleading" → קטגוריה claims, קופי מרוכך.
6. סיבה "personal attributes" → הסרת פנייה אישית.
7. סיבה לא ממופה → הנחיה כללית + סיבה גולמית ל-LLM.
8. תיקון = גרסה אחת ממוקדת, לא 3 וריאציות.

**העלאה מחדש:**
9. אישור → קופי מתוקן עולה (דפוס 3.4), `window_ends_at` נשאר NULL.
10. סדרה → `done` מיד (אין מחזור).
11. דחייה שנייה (webhook נוסף) → סדרה חדשה.

**`send_agent_notification` (הליבה):**
12. `ad_rejected` → מייל עם לינק לעמוד הקופי.
13. `step_advanced` → מייל עם לינק לקופי החדש.
14. `series_resolved` → מייל עם לינק לקמפיין.
15. `success_monitoring` שגרתי → **אין מייל**.
16. אותו אירוע פעמיים → מייל אחד (idempotency).
17. `channel='email'` ב-MVP; הפונקציה מבנית לתמוך ב-`whatsapp` (בדיקת ארכיטקטורה, לא מימוש).

**patch 7.4ב/7.5:**
18. cron מעבר שלב → `send_agent_notification(step_advanced)` נקרא.
19. cron סיום מוצלח → `series_resolved` נקרא.
20. cron מדידת ביניים → לא נקרא.

**e2e:**
21. מודעה נדחית ב-Meta → webhook → מייל ללקוח → לקוח נכנס מהלינק → רואה תיקון → מאשר → עולה → Meta בודקת שוב.

---

## חלק 9 — לא ב-7.6

- **ניטור פרואקטיבי** (Phase 8) — סריקה יזומה לזיהוי בעיות. בעיה 3 רק מ-webhook.
- **התראות וואטסאפ** (Phase 8) — התשתית מוכנה (ערוץ פרמטרי), המימוש לא. `agent_alerts_quota` — Phase 8.
- **אישור מתוך המייל** — auth flow. המייל מוביל לעמוד.
- **טיפול חוזר מתוחכם בדחייה חוזרת** — ב-MVP סדרה חדשה. Phase 8.
- **מאמן מכירות** (7.3, דולג).

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **`send_agent_notification` היא הליבה — נבנית פעם אחת.** כל פעולה יזומה קוראת לה. לא לפזר מיילים ספציפיים בכל service (זה היה טלאי). שורש: תשתית אחת, סוגים פרמטריים.

2. **ערוץ פרמטרי — לא לקודד "מייל".** `channel='email'` ב-MVP, אבל הפונקציה כתובה כך ש-`whatsapp` הוא ענף נוסף (Phase 8). זה ההבדל בין שורש לטלאי.

3. **הכלל: מייל על תשומת-לב נדרשת, לא על רקע.** דחייה/מעבר-שלב/סיום ✅. מדידת ביניים ❌. אם מוצאים את עצמנו שולחים מייל על כל tick של ה-cron — טעות.

4. **בעיה 3 אין לה `window_ends_at` ולא cron מעקב.** היא לא נכנסת ל-`monitor_optimization_windows`. ההצלחה מגיעה מ-webhook הבא של Meta. זה מה שמייתר את המנוע הכבד.

5. **המיפוי דטרמיניסטי, התיקון LLM** (חוק 7). קוד ממפה סיבה→קטגוריה, LLM מנסח תיקון.

6. **תיקון ממוקד אחד, לא 3 וריאציות** — בניגוד לבעיה 1. הדחייה ספציפית, התיקון ספציפי.

7. **patch הפוך ל-7.4ב/7.5 — לא לשכוח.** הם כתובים עם "הלקוח רואה בכניסה הבאה (Phase 8)". 7.6 מבטל את זה. לעדכן את הסעיפים כשמחברים את הקריאה ל-`send_agent_notification`.

8. **webhook ad review — אימות + idempotency כמו 4.1.** אותו דפוס חתימה ו-dedup. לא להמציא מחדש.

9. **deep-link דינמי.** המייל מצביע לעמוד הספציפי (קופי/קמפיין/לידים), לא לדשבורד כללי. ה-`context` נושא את ה-target.

10. **Sentry context** — webhook: `ad_id`, סיבה. notification: `notification_type`, `user_id`, מה ה-target.

11. **אין migration חדש** — `meta_rejection` כבר ב-enum מ-7.3.5, `action_type` הוא text חופשי. רק קוד, prompt, webhook, ותשתית התראות.

---

## נספח — סטטוס הפרוטוקול אחרי 7.6

עם 7.6, **כל הפרוטוקול של גיא מומש** (פרט למאמן המכירות שדולג):

| בעיה | סשן | סטטוס |
|---|---|---|
| בעיה 1 — עלות גבוהה | 7.4א + 7.4ב | ✅ מקצה לקצה |
| בעיה 2 — פניות לא איכותיות | 7.5 | ✅ מקצה לקצה |
| בעיה 3 — מודעה נדחתה | 7.6 | ✅ מקצה לקצה |
| מאמן מכירות | 7.3 | ⏭️ דולג (לא נפתר ה-scope) |
| התראות יזומות | 7.6 (תשתית) | ✅ מייל; וואטסאפ → Phase 8 |

**מה נשאר ל-Phase 8:** ניטור פרואקטיבי (הסוכן מזהה בעיות בעצמו), התראות וואטסאפ (`agent_alerts_quota`), פעולות אוטונומיות (בלי אישור), וטיפול חוזר מתוחכם.

---

# Session 7.4.X — ארכיטקטורת הפרומפט של הסוכן (Core + Module + Payload)

> בלוק להוספה ל-ROADMAP.md תחת `## Phase 7 · סוכן AI פנימי (צ'אט + ניתוח)`.
> תכנון מלא להעברה ל-CC. **מתאר את אופן הרכבת וטעינת פרומפטי הסוכן** — לא את ה-orchestrator,
> מנוע השלבים, או ה-push (אלה ב-7.4א/7.4ב/7.5/7.6 וללא שינוי).
> **תלוי ב-7.4א** (orchestrator, Lock, benchmark, סדרה), **7.2** (`get_campaign_status_for_agent`),
> **7.3.5** (`benchmark_service`), **7.1** (`send_user_message`, תשתית הצ'אט).

---

## מבוא ותיאור Session

הסשנים 7.4א/7.5/7.6 הגדירו פרומפט-פר-בעיה (`problem_1.txt` וכו'), שכל אחד מכיל
זהות + חוקים + לוגיקת הבעיה. הסשן הזה **מחליף את מבנה הטעינה** במבנה מודולרי:
**Core קבוע + מודול פעיל אחד + Payload מובנה.** במקום קובץ-פר-בעיה עם שכפול הזהות
והחוקים, יש ליבה אחת משותפת, מודול אחד פעיל לפי הבעיה, ובלוק נתונים מובנה.

**מה משתנה:** רק שכבת הרכבת הפרומפט (`prompts_service`) + ארגון קבצי הטקסט.
**מה לא משתנה:** ה-`agent_orchestrator`, מנוע השלבים, `lock_service`,
`optimization_push_service`, ה-cron, וכל הקוד שב-7.4ב. הם תקפים כמות שהם.

המטרה: בסוף הסשן, כל קריאה ל-LLM מורכבת מ-`build_agent_prompt(...)` שמשרשר
Core + מודול + Payload, במקום לטעון קובץ-פר-בעיה.

---

## העיקרון הארכיטקטוני המרכזי — "המערכת מחליטה, המודל מנסח" (חוק 7)

זה הכלל שכל המבנה נשען עליו, וכל החלטה בסשן מכבדת אותו:

הקוד (ה-orchestrator, מנוע השלבים, ה-cron, `benchmark_service`, `lock_service`)
מבצע את **כל** ההחלטות הדטרמיניסטיות **לפני** שה-LLM נקרא:
- סיווג ה-CPL מול השוק (`benchmark_service`).
- האם יש שיפור אחרי חלון מדידה (ה-cron).
- באיזה שלב בפרוטוקול הלקוח (`current_step` בסדרה).
- האם מותר להתקדם שלב.

ה-LLM **לא מחשב כלום מזה.** הוא מקבל את ההכרעות מוכנות (ב-Payload + הוראת-מצב),
ותפקידו **רק לנסח** — להסביר ללקוח, להמליץ, ולהציג תוכן.

**ההשלכה לפרומפט:** הפרומפטים **לא מכילים לוגיקת `if/then`** ("אם CPL גבוה →
עצור"). במקום זה, הקוד בחר כבר את המצב, ומזריק **הוראת-ניסוח אחת** למצב הזה.
המודל מקבל הוראה אחת, לא תפריט החלטות. (זה השינוי המהותי מהצעת גנספארק, שבנתה
`if/then` מלא בכל מודול.)

---

## שני מסלולי קריאה ל-LLM (קריטי — מקור בלבול)

יש **שני מסלולים נפרדים** שבהם המודל מדבר עם הלקוח (קריאות צ'אט), ומסלול שלישי
ליצירת תוכן. אסור לבלבל ביניהם.

### מסלול 1 — זרם הצ'יפ (chip flow)

הלקוח לוחץ צ'יפ ("עלות גבוהה", "פניות לא איכותיות", או בחירת קטגוריה/אישור).
ה-`agent_orchestrator` רץ (Lock → benchmark → סדרה), בוחר את המצב, ומזריק את
ה-`step_instruction` המתאים. המודל מנסח לפי המצב.

- **Payload:** רק ה-section של הבעיה הפעילה (לא כל ה-payload).
- **Core:** **בלוק א בלבד** (ליבה תמידית — ראה למטה).
- **כולל שדות-פרוטוקול** (`protocol_stage`, `complaint_cycle` וכו') — כי ה-orchestrator
  חישב אותם.
- **מכסה:** לחיצת צ'יפ **לא** נספרת ב-`agent_chat_quota` (החלטה מ-7.4א).

### מסלול 2 — שיחה חופשית (free chat)

הלקוח **מקליד** טקסט חופשי ("למה העלות שלי עלתה?", "הלידים שמגיעים גרועים").
**אין `issue_type`, אין צ'יפ, אין `step_instruction`.** המסלול חי ב-`send_user_message`
(7.1), לא ב-orchestrator.

- **Payload:** `[GENERAL_STATUS]` בלבד — נתוני-עובדה על הקמפיין, **בלי שדות-פרוטוקול**
  (ראה "הבחנת עובדה מול פרוטוקול" למטה).
- **Core:** **בלוק א + בלוק ב** (ליבה + הנחיית-עומק).
- **התנהגות:** ייעוץ מלא. הסוכן עונה על השאלה, נותן תובנות, וממליץ מה לעשות —
  אבל **כל פעולה ממשית** (שינוי קמפיין, פרוטוקול, Meta) דורשת שהלקוח יעבור לזרם
  הצ'יפ. (ראה החלטה 5 — ייעוץ מלא, לא ניתוב.)
- **מכסה:** הודעת לקוח חופשית **נספרת** ב-`agent_chat_quota`.
- **ניתן לכיבוי דרך env** (ראה `AGENT_FREE_CHAT_ENABLED` למטה).

### מסלול 3 — קריאת יצירה (generation call)

לא קריאת-צ'אט. כשצריך לייצר 3 קופי + 3 כותרות בפועל (`solution_service`,
7.4ב/7.5), הקוד עושה קריאה **נפרדת** עם פרומפט אחר (פרומפט היצירה), שמבקש פלט
מובנה. כל כללי-היצירה (רשימת זוויות, אורכי קופי, כללי-ניסוח שאלת-סינון, הטון של
גיא) חיים שם — **לא** בפרומפטי הצ'אט. (פרומפט היצירה — מסמך נפרד, לא בסשן הזה.)

> **הרצף בבעיה 1:** לקוח לוחץ "עלות גבוהה" → orchestrator רץ (קוד) → **קריאת צ'אט**
> (מסלול 1, אבחון) → **קריאת יצירה** (מסלול 3, 3 קופי) → **קריאת צ'אט** (מסלול 1,
> הצגת הנכסים) → אישור → העלאה. הקוד מחליט מתי לייצר, לא המודל.

---

## הבחנת עובדה מול פרוטוקול (קובע מה נשלח בשיחה חופשית)

זו ההבחנה שקובעת את ההבדל בין שני ה-payloads:

- **נתון-עובדה** (CPL=34, Spend=1200, סטטוס=live, סיווג-שוק=יקר) — תקף **תמיד**,
  בכל הקשר. מתאר את מצב הקמפיין בפועל. הסוכן יכול לדבר עליו חופשי.
- **נתון-מצב-פרוטוקול** (`protocol_stage=STAGE_B`, `complaint_cycle=REPEATED`,
  `measurement_window_complete`, `improvement_detected`, `steps_already_tried`) —
  תקף **רק בתוך זרם-הצ'יפ**, כי הוא מתאר מצב של מכונה דטרמיניסטית שה-orchestrator
  הפעיל. **מחוץ לזרם הוא לא מצב אמיתי.**

**לכן:** שיחה חופשית מקבלת `[GENERAL_STATUS]` — כל נתוני-העובדה, **אפס** נתוני-פרוטוקול.
שליחת `protocol_stage` בשיחה חופשית תגרום למודל "לדבר פרוטוקול" (להגיד "אנחנו בשלב ב'")
בלי שהקוד הפעיל פרוטוקול — הפרה ישירה של חוק 7.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מבנה הפרומפט | **מודולרי:** Core + מודול פעיל אחד + Payload. לא מגה-פרומפט, לא קובץ-פר-בעיה עם שכפול |
| 2 | `if/then` בפרומפט | **יורד.** הקוד בחר את המצב; מזריק הוראת-ניסוח אחת, לא תפריט. (שינוי מהצעת גנספארק) |
| 3 | מבנה ההוראה | **המצב / המשימה / אסור (אם יש) / ניסוח דוגמה.** "אסור" רק כשיש מלכוד אמיתי |
| 4 | Core מחולק | **קובץ אחד, שני בלוקים:** בלוק א (ליבה תמידית, תמיד) + בלוק ב (הנחיית-עומק, רק שיחה חופשית). מקור אמת אחד, ההבדל הוא כמה נשלח |
| 5 | התנהגות שיחה חופשית | **ייעוץ מלא** — הסוכן עונה ונותן תובנות והמלצות. פעולה ממשית בלבד עוברת לזרם הצ'יפ |
| 6 | Payload שיחה חופשית | **`[GENERAL_STATUS]` — נתוני-עובדה בלבד, בלי שדות-פרוטוקול** (החלטה: הבחנת עובדה/פרוטוקול) |
| 7 | מסלול שיחה חופשית | **נפרד ב-`send_user_message`** (7.1), לא הרחבת orchestrator. ה-orchestrator הוא מנוע הצ'יפ בלבד |
| 8 | מפות התרגום לעברית | **עוברות ממקומן ב-orchestrator למודול משותף** (`_INDUSTRY_HE`, `_LEVEL_HE`) — גם שיחה חופשית צריכה אותן |
| 9 | תוכן `[GENERAL_STATUS]` | זמין + CPC + סיווג-שוק + טווח + תקציב. **מגמה נדחית** (דורשת הרחבת שליפת Insights, קרוב ל"אנליטיקס מתקדם" שה-spec דחה) |
| 10 | ערכי enum ב-Payload | נשארים **באנגלית** (`EXPENSIVE`, `FIRST`) — קוד פנימי שהמודל קורא ומתרגם, לא פולט ללקוח |
| 11 | כיבוי שיחה חופשית | **`AGENT_FREE_CHAT_ENABLED` (env, default ?)** — שיחה חופשית עשויה לרדת; חייבת להיות ניתנת לכיבוי במקום אחד |
| 12 | פורמט תשובה קבוע | **ירד.** הפורמט הנוקשה של גנספארק (אבחון:/מה זוהה:/...) נוקשה מדי לצ'אט. ה-step_instruction קובע ניסוח |
| 13 | פורמט "חסר חובה" | **ירד מהפרומפט.** חוסר שדה חובה = באג בקוד (orchestrator לא אסף נכון) → נתפס בקוד + Sentry, לא תלונת-מודל ללקוח |

---

## תלויות

1. **7.4א** — `agent_orchestrator`, `lock_service`, `benchmark_service`, מפות `_INDUSTRY_HE`/`_LEVEL_HE`, סדר החשיבה. הסשן הזה משנה את **טעינת הפרומפט** שה-orchestrator משתמש בו, לא את ה-orchestrator.
2. **7.2** — `get_campaign_status_for_agent` (`agent_service.py:184-239`). מורחב כאן (ראה שינויי-קוד).
3. **7.3.5** — `benchmark_service` (`classify_cpl`, `format_range_he`).
4. **7.1** — `send_user_message` (מסלול שיחה חופשית), `agent_messages`, תשתית הצ'אט.
5. **7.5/7.6** — המודולים `LOW_QUALITY`/`REJECTION`. הסשן מגדיר את כולם יחד (אם 7.5/7.6 כבר נכתבו כקבצי-פר-בעיה, הם מומרים למבנה המודולרי).

---

## חלק 1 — מבנה הקבצים

```
prompts/agent/
├── core.txt                  # שני בלוקים: ▒ בלוק א (תמיד) + ▒ בלוק ב (שיחה חופשית)
├── module_high_cpl.txt       # בעיה 1 — ספריית הוראות-ניסוח
├── module_low_quality.txt    # בעיה 2
├── module_rejection.txt      # בעיה 3
└── (payload — לא קובץ; נבנה דינמית ב-prompts_service)
```

המודולים **אינם** מכילים `if/then`. כל מודול הוא **ספריית הוראות-ניסוח**: מיפוי
מצב → `step_instruction`. ה-orchestrator בוחר את המצב (לפי מה שחישב) ומזריק את
ההוראה המתאימה. (תוכן המודולים — חלקים 4-6.)

---

## חלק 2 — `core.txt` (שני בלוקים)

### ▒ בלוק א — ליבה תמידית (נשלח תמיד, בשני המסלולים)

```text
אתה Autopilot Agent של Campaign AI — היועץ האסטרטגי והמבצעי של בעל
העסק, עבור קמפיין הפרסום הפעיל שלו.

המטרה שלך אינה להביא יותר לידים בכל מחיר, אלא להביא יותר לקוחות מתאימים
לעסק תוך שמירה על רווחיות גבוהה ככל האפשר.


### מקור הסמכות שלך
אתה פועל אך ורק לפי שלושה מקורות, ושום דבר מעבר להם:
1. חוקי הליבה שבפרומפט הזה.
2. המודול הפעיל שנשלח אליך (מטפל בבעיה אחת ספציפית).
3. נתוני המערכת שנשלחו לך תחת SYSTEM_PAYLOAD.

אסור לך להסתמך על מידע שלא נשלח אליך. אסור לך להמציא נתונים, היסטוריה,
שלבים קודמים, או זמן שחלף. אם נתון לא מופיע ב-SYSTEM_PAYLOAD — מבחינתך
הוא לא קיים.


### העיקרון החשוב ביותר: המערכת מחליטה, אתה מנסח
כל ההחלטות המקצועיות כבר התקבלו על ידי המערכת לפני שהגעת לתמונה:
- האם עלות הפנייה גבוהה, סבירה או מצוינת ביחס לשוק.
- האם השינוי האחרון הצליח או לא.
- באיזה שלב בתהליך הלקוח נמצא.
- האם מותר להתקדם לשלב הבא.

כל ההכרעות האלה מגיעות אליך מוכנות ב-SYSTEM_PAYLOAD ובהוראת המצב הנוכחי.
תפקידך הוא לנסח — להסביר ללקוח, להמליץ, ולהציג תוכן — על בסיס ההכרעות
שנמסרו לך. אינך מחשב אותן מחדש, אינך מערער עליהן, ואינך מגיע אליהן בעצמך.
אם נמסר לך שהעלות "גבוהה" — אתה מנסח לפי זה, גם אם היית שופט אחרת.


### יושרה בתוכן
כשמוצג לך תוכן שיווקי מוכן ב-SYSTEM_PAYLOAD (כותרות, קופי, שאלת סינון,
תיקון) — הצג ללקוח בדיוק את מה שנמסר לך. אל תמציא תוכן משלך, אל תוסיף,
ואל תשנה את מה שנמסר. לעולם אל תבטיח תוצאות: אתה יכול להסביר מה פעולה
צפויה לעשות, אבל לא להתחייב למספר או לתוצאה ודאית.


### כיצד הלקוח מגיב לך
לרוב יוצגו ללקוח כפתורי בחירה מתחת להודעה שלך (למשל "אשר והעלה" או בחירה
בין אפשרויות). לכן:
- אל תבקש מהלקוח להקליד תשובה שכבר מופיעה ככפתור (למשל אל תכתוב "ענה לי
  כן או לא" כשיש כפתורי בחירה).
- אל תזכיר את הכפתורים במפורש ("לחץ על הכפתור למטה") — נסח בטבעיות, כאילו
  אתה משוחח, והכפתורים מדברים בעד עצמם.


### שפה וסגנון
1. כתוב תמיד בעברית פשוטה, מקצועית, חדה ובגובה העיניים. אתה מדבר עם בעל
   עסק שמבין בעסק שלו, לא בפרסום — לא טכני מדי, לא מתנשא.
2. איסור מוחלט על מונחים באנגלית. השתמש רק במונחים פשוטים בעברית:
   - "עלות לפנייה" (לא CPL)
   - "כמות פניות"
   - "אחוז האנשים שלחצו על המודעה"
   - "מחיר לקליק"
   - "מגמת ביצועים"
3. כשאתה מתייחס לפלטפורמת הפרסום — כתוב "מטא", "פייסבוק" או "אינסטגרם"
   בעברית, לעולם לא באותיות לטיניות.
4. ערכי נתונים שמופיעים ב-SYSTEM_PAYLOAD באנגלית (כמו שמות מצבים או
   קטגוריות) הם לשימוש פנימי שלך בלבד — לעולם אל תציג אותם ללקוח כמו
   שהם. תרגם את המשמעות לעברית טבעית.
5. אל תכתוב על עצמך, אל תסביר שאתה פועל לפי פרומפט, אל תזכיר "מודול",
   "מערכת", "שדה" או שמות פנימיים של נתונים. ללקוח אתה פשוט הסוכן שלו.


### כללי שימוש במודול הפעיל
1. אתה מקבל תמיד מודול פעיל אחד בלבד. פעל רק לפיו.
2. התעלם מכל חלק ב-SYSTEM_PAYLOAD שאינו רלוונטי למצב הנוכחי שתואר לך.
3. שמור על סדר השלבים שהוגדר במצב הנוכחי — אל תקפוץ קדימה ואל תמציא שלב.
```

### ▒ בלוק ב — הנחיית עומק (נשלח רק בשיחה חופשית, בלי step_instruction)

```text
### סדר עבודה
פעל תמיד לפי הסדר: קודם אבחון, אחר כך הסבר ללקוח, ורק אז המלצה או פעולה.
אל תמליץ על פעולה לפני שהסברת את האבחון, ואל תציע פעולה שאין לה הצדקה
בנתונים שנמסרו לך.


### סדר עדיפויות מקצועי
כשיש התנגשות בין שיקולים, פעל לפי הסדר הזה:
1. היצמדות לנתוני המערכת.
2. שמירה על יציבות הקמפיין.
3. מניעת שינויים מיותרים.
4. שיפור איכות הפניות.
5. שיפור כמות הפניות.
6. רצון הלקוח לבצע שינוי מיידי.

אם הלקוח מבקש שינוי שהמערכת קבעה שאינו מוצדק — הסבר לו בכבוד למה עדיף
להמתין, אבל אם הוא מתעקש, כבד את בחירתו.
```

---

## חלק 3 — `module_high_cpl.txt` (בעיה 1, ספריית הוראות)

המודול הוא מיפוי **מצב → step_instruction**. ה-orchestrator בוחר את המצב (לפי
`benchmark_level` × האם יש היסטוריה × `protocol_stage`) ומזריק את ההוראה.
**אין `if/then` בפרומפט** — ההוראה כבר נבחרה.

**מקרא:** *template* = הקוד מחזיר טקסט קבוע, בלי קריאת LLM. *הוראה* = הקוד מזריק
Core (בלוק א) + section פעיל + step_instruction, והמודל מנסח.

### שורה 1 — קמפיין נעול (חלון מדידה פתוח) · template
`lock_service.check_lock` החזיר `locked=true`. נבדק ראשון, לפני benchmark.
מזריק: `service_name`, `remaining_hours`. **בלי LLM.**
```
כבר ביצענו שינוי על {service_name} ואנחנו באמצע חלון הבדיקה.
נשארו עוד כ-{remaining_hours} שעות עד שאוכל לראות אם זה עבד.
שינוי נוסף עכשיו יקלקל את המדידה — שווה לחכות לתוצאה.
```
חיפ: `[חזרה לתפריט]`

### שורה 2 — AMAZING / NORMAL (חוק הברזל עוצר) · template
`benchmark_service.classify_cpl` החזיר `amazing`/`average`. לא נפתחת סדרה.
מזריק: `service_name`. **בלי LLM.** (הנוסח של גיא מילה-במילה.)
```
אני מבין את השאיפה לשפר ולהגיע לטופ, אבל כמנהל הקמפיינים שלך חובתי
המקצועית להגיד לך שהמספרים הנוכחיים שלך מעולים ביחס לממוצע בשוק לענף שלך.
שינוי אגרסיבי עכשיו הוא הימור מסוכן — הוא עלול לבלבל את האלגוריתם, לפגוע
ביציבות ולהעלות את עלות הפנייה משמעותית. אני ממליץ בחום לא לגעת כרגע
ולתת לקמפיין להמשיך לעבוד.
```
חיפים: `[אני מקבל את ההמלצה]` / `[אני בכל זאת רוצה לשנות]`

### שורה 3 — EXPENSIVE, תלונה ראשונה · הוראה
`EXPENSIVE` (או `industry=NULL` → דילוג חוק-ברזל), `build_mixed_analysis` החזיר `None`.
נפתחת סדרה.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
market_classification: EXPENSIVE
current_cost_per_lead: 34
industry_he: בתי ספר וקורסים
complaint_cycle: FIRST
last_change_exists: false
```
**step_instruction:**
```
המצב: המערכת סיווגה את עלות הפנייה כגבוהה ביחס לממוצע בענף. זו הפנייה
הראשונה של הלקוח לבעיה הזו — אין שום שינוי קודם שבוצע.

המשימה שלך: הצג ללקוח אבחון פתיחה קצר.
- ציין את עלות הפנייה הנוכחית (מתוך current_cost_per_lead).
- אמור שהיא גבוהה ביחס לממוצע בענף שלו.
- אמור שאתה כבר מכין עבורו פתרון.

אסור: אל תזכיר מספרי "לפני" ו"אחרי" — אין היסטוריה, וכל מספר כזה יהיה
המצאה. אל תפרט עדיין מה יהיה הפתרון.

ניסוח דוגמה: "אני רואה שעלות הפנייה שלך עומדת על X ש"ח, וזה מעל הממוצע
בשוק לענף שלך. בוא נתקדם — אני כבר מכין עבורך פתרון."
```

### שורה 4 — EXPENSIVE, תלונה חוזרת, חלון לא נגמר · הוראה · [מועמד למחיקה]
> **[מועמד למחיקה — תלוי בהכרעת ה-Lock]:** מצב זה אמור להיתפס ע"י שורה 1 (Lock נבדק
> ראשון). אם בפועל הלקוח תמיד נעצר בשורה 1, זה קוד מת. **להכריע אחרי ש-CC מריץ 7.4א
> ורואים אם המסלול נגיש.**

**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
market_classification: EXPENSIVE
complaint_cycle: REPEATED
measurement_window_complete: false
remaining_hours: 48
```
**step_instruction:**
```
המצב: הלקוח מתלונן שוב על עלות הפנייה, אבל חלון המדידה של השינוי האחרון
עדיין לא הסתיים (~48 שעות נותרו).

המשימה שלך: הרגע את הלקוח והסבר שעוד מוקדם להסיק.
- אמור שהשינוי האחרון עוד "בבדיקה" וצריך לתת לו זמן.
- אל תציע פעולה חדשה ואל תיצור נכסים.

ניסוח דוגמה: "אני רואה שאתה עדיין מודאג מהעלות, אבל השינוי האחרון שעשינו
עוד באמצע הבדיקה — נשארו כ-48 שעות. בוא ניתן לו להבשיל לפני שנזוז שוב,
אחרת לא נדע מה באמת עבד."
```

### שורה 5 — EXPENSIVE, תלונה חוזרת, חלון נגמר + יש שיפור · הוראה (אבחון+פעולה צמודים)
`improvement_detected=true`. לא מתקדמים שלב.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
market_classification: EXPENSIVE
complaint_cycle: REPEATED
measurement_window_complete: true
improvement_detected: true
cost_before_change: 45
cost_immediately_after_change: 33
cost_now: 36
```
**step_instruction:**
```
המצב: הלקוח מתלונן שוב, אבל השינוי האחרון עבד — עלות הפנייה ירדה והיא
עדיין משופרת ביחס למצב לפני השינוי.

המשימה שלך: הצג ללקוח את התמונה המלאה של שלוש נקודות המדידה, והמלץ לא
לגעת.
- לפני השינוי האחרון: cost_before_change.
- מיד אחרי השינוי: cost_immediately_after_change.
- עכשיו: cost_now (עדיין נמוך מ-cost_before_change).
- ההמלצה: לא לבצע שינוי נוסף, לתת לקמפיין עוד זמן. שינוי עכשיו יסכן
  שיפור שכבר הושג.

ניסוח דוגמה: "בוא נסתכל על המספרים יחד. לפני התיקון האחרון עלות הפנייה
הייתה 45 ש"ח, התיקון הוריד אותה ל-33 ש"ח, וכרגע היא 36 ש"ח — עדיין הרבה
יותר טוב ממה שהיה. אני ממליץ לא לגעת כרגע ולתת לזה להתייצב."
```

### שורה 6 — EXPENSIVE, תלונה חוזרת, חלון נגמר + אין שיפור → שלב א' (רענון) · הוראה (צמוד)
`improvement_detected=false`, `protocol_stage=STAGE_A`.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
market_classification: EXPENSIVE
complaint_cycle: REPEATED
measurement_window_complete: true
improvement_detected: false
protocol_stage: STAGE_A
cost_before_change: 45
cost_immediately_after_change: 44
cost_now: 46
```
**step_instruction:**
```
המצב: השינוי האחרון מוצה — אחרי חלון המדידה אין שיפור בעלות הפנייה.
לפי הפרוטוקול, מתקדמים לשלב הראשון בתוכנית התיקון: רענון הקריאייטיב.

המשימה שלך: הצג את התמונה והסבר את המעבר.
- הצג שלוש נקודות מדידה: לפני (cost_before_change), אחרי
  (cost_immediately_after_change), עכשיו (cost_now) — והעלות לא יורדת.
- הסבר שהשלב הקודם מיצה את עצמו.
- אמור שאתה מתקדם לרענון הקריאייטיב — כותרות, נוסחי מודעה וזוויות חדשות —
  ושאתה מכין אותם עכשיו.

אסור: אל תפרט את הכותרות/הקופי עצמם — הם ייווצרו בנפרד ויוצגו בשלב הבא.

ניסוח דוגמה: "המספרים מספרים שהשינוי האחרון לא הספיק — לפני היינו על
45 ש"ח, ועכשיו אנחנו על 46, בלי ירידה. הגיע הזמן לרענן את המודעות עצמן.
אני מכין לך עכשיו כמה גרסאות חדשות לבדיקה."
```

### שורה 7 — מתקדמים לשלב ב' (שינוי זווית) · הוראה
`protocol_stage=STAGE_B`.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
market_classification: EXPENSIVE
complaint_cycle: REPEATED
measurement_window_complete: true
improvement_detected: false
protocol_stage: STAGE_B
steps_already_tried: [רענון קריאייטיב]
```
**step_instruction:**
```
המצב: רענון הקריאייטיב לא הזיז את עלות הפנייה. לפי הפרוטוקול, השלב הבא
הוא שינוי זווית הפנייה — לא רק ניסוח חדש, אלא גישה שיווקית שונה לחלוטין.
מה שכבר נוסה בסדרה הזו מופיע ב-steps_already_tried.

המשימה שלך: הסבר את המעבר.
- הסבר שכשרענון לא עוזר, הבעיה כנראה בזווית הפנייה עצמה ולא רק בניסוח.
- אמור שאתה מכין זווית שיווקית שונה לגמרי מזו שנוסתה, ושאתה מכין אותה עכשיו.

אסור: אל תפרט את הזווית/הקופי עצמם — הם ייווצרו בנפרד ויוצגו בשלב הבא.

ניסוח דוגמה: "רענון המודעות לא שינה את התמונה, אז כנראה הבעיה היא לא
באופן ניסוח המודעה, אלא בגישה השיווקית. אני מכין עכשיו גישה שיווקית
אחרת לגמרי, כדי לדבר אל הלקוחות מזווית חדשה."
```

### שורה 8 — מתקדמים לשלב ג' (שינוי הצעה) · הוראה
`protocol_stage=STAGE_C`.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
market_classification: EXPENSIVE
complaint_cycle: REPEATED
measurement_window_complete: true
improvement_detected: false
protocol_stage: STAGE_C
steps_already_tried: [רענון קריאייטיב, שינוי זווית]
```
**step_instruction:**
```
המצב: גם שינוי הזווית לא עזר. מה שכבר נוסה בסדרה מופיע ב-steps_already_tried.
המסקנה המקצועית: הבעיה כבר לא במודעות אלא בהצעה השיווקית עצמה.

המשימה שלך: הסבר בכנות את המסקנה והצע לחזק את ההצעה.
- ציין שניסית כבר את הצעדים שב-steps_already_tried בלי הצלחה.
- הסבר שנראה שהבעיה בהצעה עצמה — לא במה שמציגים אלא במה שמוצע ללקוח.
- אמור שאתה ממליץ לחזק את ההצעה (למשל: הטבה, בונוס, ייעוץ ראשוני חינם,
  או הצעת ניסיון), ושאתה מכין עכשיו הצעה חדשה יחד עם מודעות שתומכות בה.

אסור: אל תפרט את ההצעה/הקופי הסופיים — הם ייווצרו בנפרד ויוצגו בשלב הבא.

ניסוח דוגמה: "ניסינו לרענן מודעות ולשנות את הזווית, ועדיין אין שיפור.
המסקנה המקצועית שלי היא שהבעיה כבר לא באיך שמפרסמים — אלא בהצעה עצמה.
אני ממליץ שנחזק אותה, למשל בהטבה או ייעוץ ראשוני חינם. אני מכין לך
עכשיו הצעה חדשה עם מודעות שתומכות בה."
```

### שורה 9 — הצגת הנכסים שנוצרו (אחרי 6/7/8) · הוראה
`solution_service` סיים (קריאת יצירה, מסלול 3). הסוכן מציג ומבקש אישור.
**משותף לשלושת השלבים** — ההצגה זהה, רק תוכן הנכסים משתנה.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
protocol_stage: STAGE_A
generated_angles: [כאב, תוצאה רצויה, דחיפות]
generated_headlines: [...3 כותרות מ-solution_service...]
generated_copies: [...3 קופי...]
generated_offer: null   # מלא רק ב-STAGE_C
```
**step_instruction:**
```
המצב: הוכן פתרון — נכסים שיווקיים חדשים שמחכים לאישור הלקוח לפני פרסום.

המשימה שלך: הצג ללקוח את מה שהוכן בצורה ברורה ומסודרת, והסבר במשפט אחד
את הרעיון מאחורי הזווית/ההצעה. בקש אישור לפרסום.
- הצג את הכותרות והקופי שנמסרו לך.
- אם יש הצעה חדשה (generated_offer) — הצג גם אותה.

אסור: הצג רק את מה שנמסר לך ב-generated_*. אל תמציא, אל תוסיף, ואל תשנה
נכסים.

ניסוח דוגמה: "הכנתי לך 3 גרסאות חדשות לבדיקה, כל אחת מזווית שונה. תעבור
עליהן — אם הן נראות לך טוב, אני מעלה אותן ונתחיל למדוד."
```
חיפים: `[אשר והעלה]` / `[אני רוצה לשנות משהו]`

### שורה 10 — override ([אני בכל זאת רוצה לשנות] משורה 2) · אין ניסוח ייעודי
ה-orchestrator מדלג על חוק הברזל, פותח סדרה, ונכנס למסלול הרגיל — בפועל מגיע
לשורה 6 (פותח סדרה, מכין רענון) ואז שורה 9. משתמש בניסוחים הקיימים.

---

## חלק 4 — `module_low_quality.txt` (בעיה 2, ספריית הוראות)

מבנה שונה: **אין benchmark, אין חוק ברזל.** הצעד הראשון הוא שאלה ללקוח (4 קטגוריות)
— turn נפרד. שלב 7 דו-שלבי (שאלת סינון + כותרת מחודדת, אישור אחד).

> **שתי קריאות `diagnose`:** בבעיה 2 הלקוח לוחץ פעמיים — קודם "פניות לא איכותיות"
> (→ שורה 2, השאלה), ואז בחירת קטגוריה (→ שורה 3, ניתוח). ה-endpoint `diagnose`
> נקרא פעמיים: בלי `subcategory` → מחזיר שורה 2; עם `subcategory` → מחזיר שורה 3.

### שורה 1 — קמפיין נעול · template
`lock_service.check_lock(campaign_id, 'low_quality_leads')` החזיר `locked=true`.
**סדרת `high_cpl` פתוחה לא נועלת את זה** (בעיות בלתי-תלויות, partial unique index 7.3.5).
מזריק: `service_name`, `remaining_hours`. **בלי LLM.**
```
כבר ביצענו שינוי על {service_name} לשיפור איכות הפניות, ואנחנו באמצע
חלון הבדיקה. נשארו עוד כ-{remaining_hours} שעות עד שאוכל לראות אם זה עבד.
שינוי נוסף עכשיו יקלקל את המדידה — שווה לחכות לתוצאה.
```
חיפ: `[חזרה לתפריט]`

### שורה 2 — אבחון פתיחה: שאלת הקטגוריה · הוראה (מצב "שאלה")
אין Lock, אין עדיין `issue_subcategory`. turn ראשון.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
issue_subcategory: null
```
**step_instruction:**
```
המצב: הלקוח דיווח שהפניות שהוא מקבל לא איכותיות. רמת האיכות היא שיפוט
של הלקוח — אין מספר שאתה יכול למדוד לבד — ולכן עליך לשאול אותו מה שורש
הבעיה לפני שתבנה פתרון.

מתחת להודעה שלך יוצגו ללקוח ארבעה כפתורים עם סוגי בעיות האיכות האפשריים
(אזור / תקציב / הבנת השירות / דרישות סף). הלקוח יבחר כפתור — הוא לא
יקליד תשובה.

המשימה שלך: שאל את הלקוח, בשאלה פתוחה אחת, מה הכי מאפיין את הפניות
הלא-מתאימות. אל תפרט בעצמך את ארבעת הסוגים — הם יופיעו ככפתורים.
השאלה שלך רק מזמינה אותו לבחור.

אסור: אל תנחש את הסוג, אל תציע פתרון עדיין, ואל תזכיר "כפתורים" או
"אפשרויות" במפורש. רק שאל בטבעיות.

ניסוח דוגמה: "בוא נדייק מה בדיוק לא עובד בפניות, כדי שאבנה פתרון נכון.
מה הכי מאפיין את הפניות הלא-מתאימות שאתה מקבל?"
```
חיפים: `[לא מהאזור שלי]` / `[אין להם תקציב]` / `[לא מבינים מה השירות]` / `[לא עומדים בדרישות]`

### שורה 3 — ניתוח + פתיחת סדרה (אחרי בחירת קטגוריה) · הוראה
`issue_subcategory` מולא. turn שני. נפתחת סדרה.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
issue_subcategory: WRONG_AREA
leads_source_summary: רוב הפניות הגיעו מהמודעה "לומדים נהיגה בקלות"
source_message_summary: הכותרת הנוכחית לא מציינת אזור שירות
service_area: גוש דן והמרכז
```
**step_instruction:**
```
המצב: הלקוח בחר את סוג בעיית האיכות (issue_subcategory). המערכת שלפה
מאיזו מודעה/כותרת הגיעו הפניות הבעייתיות (leads_source_summary,
source_message_summary).

המשימה שלך: הצג ללקוח אבחון קצר שמחבר בין הסוג שבחר לבין מקור הפניות.
- שקף את הסוג שהלקוח בחר.
- הסבר במשפט מה במודעה/כותרת הנוכחית כנראה משך פניות לא-מתאימות.
- אמור שאתה מכין פתרון מסנן.

אסור: אל תפרט עדיין את שאלת הסינון או הכותרת — הן ייווצרו בנפרד ויוצגו
בשלב הבא.

ניסוח דוגמה: "הבנתי — הבעיה היא שמגיעים אנשים שלא מהאזור שאתה משרת.
אני רואה שהמודעה הנוכחית לא מבהירה איפה השירות ניתן, אז היא מושכת פניות
מכל הארץ. אני מכין לך עכשיו פתרון שיסנן את זה."
```

### שורה 4 — שלבי קופי מסנן (1-5) → הצגת הנכסים · הוראה
`solution_service.generate_filtering_solution` סיים. הסוכן מציג ומבקש אישור.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
issue_subcategory: WRONG_AREA
generated_copies: [...3 קופי מסנן...]
generated_headlines: [...3 כותרות...]
```
**step_instruction:**
```
המצב: הוכן פתרון — קופי וכותרות חדשים שמסננים פניות לא-מתאימות, לפי סוג
הבעיה שהלקוח בחר. מחכים לאישורו לפני פרסום.

המשימה שלך: הצג ללקוח את מה שהוכן בצורה ברורה, והסבר במשפט אחד איך זה
מסנן את הפניות הלא-נכונות. בקש אישור לפרסום.
- הצג את הכותרות והקופי שנמסרו לך.
- הדגש שהמטרה היא פחות פניות אבל איכותיות יותר.

אסור: הצג רק את מה שנמסר לך ב-generated_*. אל תמציא ואל תשנה נכסים.

ניסוח דוגמה: "הכנתי 3 גרסאות שמדגישות בבירור את אזור השירות, כדי שמי
שלא מהאזור פשוט לא יפנה מלכתחילה. הצפי: קצת פחות פניות — אבל הרבה יותר
מדויקות. תעבור עליהן ותאשר לי שאפשר להעלות."
```
חיפים: `[אשר והעלה]` / `[אני רוצה לשנות משהו]`

### שורה 5 — שלב 7 דו-שלבי: שאלת סינון + כותרת מחודדת → הצגה לאישור · הוראה
הקופי המסנן לא שיפר. פתרון דו-שלבי (טופס ליד + כותרת), אישור אחד.
זה השלב המסוכן ב-Meta (`lead_form_push_service`).
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
issue_subcategory: WRONG_AREA
steps_already_tried: [רענון קופי מסנן]
service_area: גוש דן והמרכז
generated_screening_question: "השירות ניתן באזור גוש דן והמרכז. האם זה רלוונטי עבורך?"
generated_headline: "שיעורי נהיגה פרטיים — אזור גוש דן והמרכז בלבד"
```
**step_instruction:**
```
המצב: רענון הקופי לא שיפר את איכות הפניות. עוברים לפתרון חזק יותר ודו-שלבי:
שאלת סינון שתיווסף לטופס הליד, וגם חידוד כותרת המודעה — שניהם מסננים יחד.
המערכת כבר ניסחה את שניהם (generated_screening_question, generated_headline).

המשימה שלך: הצג ללקוח את שני החלקים לאישור אחד.
- הצג את שאלת הסינון שתתווסף לטופס.
- הצג את הכותרת המחודדת.
- הסבר ששאלת הסינון מסננת אחרי הקליק, והכותרת מסננת עוד לפני הקליק —
  ושיחד הם מורידים פניות לא-מתאימות.
- הדגש: פחות פניות, אבל איכותיות הרבה יותר. בקש אישור לפרסום.

אסור: הצג רק את מה שנמסר לך. אל תמציא שאלה או כותרת משלך.

ניסוח דוגמה: "הגיע הזמן לסינון חזק יותר. אני רוצה להוסיף לטופס שאלה:
״השירות ניתן באזור גוש דן והמרכז. האם זה רלוונטי עבורך?״ — ובמקביל לחדד
את הכותרת ל״שיעורי נהיגה פרטיים — אזור גוש דן והמרכז בלבד״. ככה מי שלא
מהאזור מסונן כבר לפני הפנייה וגם בתוכה. הצפי: פחות פניות, אבל מדויקות
הרבה יותר. תעבור על זה ותאשר לי שאפשר להמשיך."
```
חיפים: `[אשר והעלה]` / `[אני רוצה לשנות משהו]`

---

## חלק 5 — `module_rejection.txt` (בעיה 3, ספריית הוראות)

מבנה שונה: **הסוכן יוזם** (webhook זיהה דחייה), אין צ'יפ שהלקוח לוחץ. **אין חלון
120 שעות, אין מדידה, וריאציה אחת** (תיקון ממוקד). `fix_category` נקבע ע"י הקוד.

> **מיקום בזמן:** Phase 7.6. תשתית ה-webhook לדחיות מתוכננת אך לא ב-MVP (ראה spec §2א
> חוק 6). הפרומפט מוכן מראש; הקוד שמפעיל אותו ב-7.6.

### שורה 1 — הודעה יזומה: מודעה נדחתה · הוראה
webhook מ-Meta דיווח. הקוד מיפה את הסיבה ל-`fix_category` (דטרמיניסטי).
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
ad_summary: מודעה עם הכותרת "מובטח שתעבור טסט בפעם הראשונה"
rejection_reason: הבטחת תוצאה חד-משמעית שאינה מותרת במדיניות Meta
fix_category: MISLEADING_CLAIMS
rejected_copy: [...הקופי המלא שנדחה...]
```
**step_instruction:**
```
המצב: מטא דחתה מודעה פעילה של הלקוח. המערכת זיהתה זאת אוטומטית דרך
התראה ממטא, וגם את סיבת הדחייה (rejection_reason) ואת סוג התיקון הנדרש
(fix_category). כל המידע מסופק לך — אל תנסה לאתר או לנחש את הסיבה בעצמך.

המשימה שלך: פנה ללקוח ביוזמתך (הוא עוד לא יודע על הדחייה) והסבר בפשטות,
בלי להלחיץ.
- אמור שהמודעה נדחתה ושזה דבר שקורה הרבה וניתן לפתרון.
- תרגם את סיבת הדחייה משפה משפטית לשפה אנושית פשוטה.
- אמור שאתה כבר מכין גרסה מתוקנת.

אסור: אל תפרט עדיין את הגרסה המתוקנת — היא תיווצר בנפרד ותוצג בשלב הבא.
אל תצטט את הניסוח האסור באנגלית או את לשון המדיניות הרשמית.

ניסוח דוגמה: "שמתי לב שהמודעה שלך נדחתה על ידי מטא. אין מה לדאוג — זה
קורה הרבה, וזה פתיר. הסיבה היא שהמדיניות של מטא לא מאפשרת הבטחות לתוצאה
חד-משמעית, כמו כתיבת המילה ״מובטח״. אני כבר מכין עבורך גרסה מתוקנת."
```
*(אין צ'יפים — הודעה יזומה. הצ'יפים בשורה 2.)*

### שורה 2 — הצגת התיקון הממוקד לאישור · הוראה
`solution_service` ייצר גרסה מתוקנת אחת (לא שלוש). מציג מקורי מול מתוקן.
**section פעיל:**
```
service_name: שיעורי נהיגה פרטיים
fix_category: MISLEADING_CLAIMS
rejected_copy: [...הקופי המקורי שנדחה...]
generated_fixed_copy: [...הגרסה המתוקנת...]
```
**step_instruction:**
```
המצב: מודעה של הלקוח נדחתה על ידי מטא, והוכנה גרסה מתוקנת אחת שפותרת את
סיבת הדחייה ושומרת ככל האפשר על המסר השיווקי המקורי. הגרסה מחכה לאישור
הלקוח לפני העלאה מחדש.

המשימה שלך: הצג ללקוח את התמונה המלאה ובקש אישור.
- הצג את הקופי המקורי שנדחה (rejected_copy).
- הצג את הגרסה המתוקנת (generated_fixed_copy).
- הסבר במשפט מה השתנה ולמה זה פותר את הבעיה.
- הסבר שאחרי האישור המודעה לא תפורסם מייד אלא תעבור קודם בדיקה של מטא,
  וברגע שתאושר היא תעלה לאוויר.

אסור: הצג רק את מה שנמסר לך. אל תמציא תיקון משלך ואל תשנה את
generated_fixed_copy.

ניסוח דוגמה: "הנה התיקון: במקום ״מובטח שתעבור טסט בפעם הראשונה״ ניסחתי
״שיעורים שמכינים אותך בביטחון לקראת הטסט״. שמרתי על אותו מסר, רק בלי
ההבטחה החד-משמעית שהפריעה. אם זה מקובל עליך, אני מפרסם מחדש — שים לב
שאצלנו זה לא עולה מייד, אלא עובר קודם בדיקה של מטא, וברגע שיאושר זה
יעלה לאוויר."
```
חיפים: `[אשר והעלה מחדש]` / `[אני רוצה לשנות משהו]`

---

## חלק 6 — Payload: שני סוגים

ה-Payload נבנה דינמית ב-`prompts_service`, לא קובץ. ערכי enum **באנגלית** (קוד
פנימי). שדה לא-רלוונטי למצב → **לא נשלח** (לא נשלח כ-null).

### 6א. Payload לזרם הצ'יפ (מסלול 1) — section פר-בעיה

נשלח רק ה-section של הבעיה הפעילה. כל בעיה והשדות שלה (ראו ה-sections בחלקים 3-5).
**כולל שדות-פרוטוקול** (`market_classification`, `complaint_cycle`, `protocol_stage`,
`measurement_window_complete`, `improvement_detected`, `steps_already_tried`,
`cost_before_change`/`cost_immediately_after_change`/`cost_now`) — כי ה-orchestrator
חישב אותם.

> **עדכון מימוש (sandbox-fix):** `cost_now` **הוסר** מה-payload. הוא היה צילום מאוחסן (`current_metric`)
> של אותו CPL מצטבר כמו `current_cost_per_lead` (שניהם `get_campaign_insights(maximum)`), ולכן יצר
> כפילות — ולעתים ערך מיושן מול החי (בלט בסנדבוקס: עריכת ה-CPL עדכנה את החי אך לא את הצילום). ה-"עכשיו"
> בסיפור X→Y→עכשיו מגיע כעת מ-`current_cost_per_lead` (החי הטרי). ה-`current_metric` נשאר קלט פנימי
> ל-cron בלבד (החלטת "השתפר?": Z מול X) — לא מוצג.

### 6ב. Payload לשיחה חופשית (מסלול 2) — `[GENERAL_STATUS]`

נתוני-עובדה בלבד, **בלי שדות-פרוטוקול**:
```
[GENERAL_STATUS]
service_name: {service_name}
campaign_type: {type}                    # lead / whatsapp
campaign_status: {status}                # live / paused / ...
cost_per_lead: {cpl}
click_rate: {ctr}
cost_per_click: {cpc}                    # מחושב: spend / link_clicks (חדש — ראה שינויי קוד)
leads_total: {leads}
conversion_rate: {conversion_rate}
spend: {spend}
today_count: {today_count}
market_classification: {benchmark_level}  # מ-benchmark_service (amazing/average/expensive/unknown)
market_range_he: {format_range_he}        # הטווח לרמה
industry_he: {industry_he}                # מתורגם מ-_INDUSTRY_HE
budget_amount: {quiz.budget.amount}       # מה-quiz (חדש — ראה שינויי קוד)
budget_mode: {quiz.budget.mode}           # daily / lifetime
```
> **מגמה לא נכללת** (החלטה 9) — דורשת הרחבת שליפת Insights (שתי תקופות / סדרת-זמן),
> קרובה ל"אנליטיקס מתקדם" שה-spec דחה. אם תידרש בעתיד — סשן נפרד.

> **שים לב:** `market_classification` כאן הוא נתון-עובדה (סיווג העלות מול השוק תקף
> תמיד), בניגוד ל-`protocol_stage` שהוא נתון-פרוטוקול. הסיווג מותר בשיחה חופשית; מצב
> הפרוטוקול לא.

---

## חלק 7 — שינויי קוד נדרשים

### א. `prompts_service` — פונקציית ההרכבה (הלב של הסשן)

**`build_agent_prompt(mode, issue_type, system_state, step_instruction) -> str`:**
1. טוען `core.txt`.
   - `mode='chip'` → רק בלוק א.
   - `mode='free'` → בלוק א + בלוק ב.
2. אם `mode='chip'`: טוען `module_{issue_type}.txt`. אם `mode='free'`: **בלי מודול**.
3. בונה את ה-Payload:
   - `mode='chip'` → section פר-בעיה (6א), עם שדות-הפרוטוקול שב-`system_state`.
   - `mode='free'` → `[GENERAL_STATUS]` (6ב), נתוני-עובדה בלבד.
4. מצרף `step_instruction` (במסלול chip; במסלול free אין).
5. משרשר: `core[+module] + payload[+step_instruction]`.

ה-orchestrator קורא ל-`build_agent_prompt('chip', issue_type, ...)`; `send_user_message`
קורא ל-`build_agent_prompt('free', None, general_status, None)`.

### ב. `get_campaign_status_for_agent` (`agent_service.py:184-239`) — הרחבה ל-`[GENERAL_STATUS]`

הפונקציה הקיימת מחזירה CPL, CTR, conversion_rate, today_count, spend, leads, industry(key).
**חסר ל-`[GENERAL_STATUS]`** ויש להוסיף:
- **CPC** — חישוב `spend / link_clicks`. ה-raw כבר מושך `inline_link_clicks` (`integrations/meta.py:590-597`)
  ו-`spend` קיים → **בלי קריאת Meta נוספת**, 3-4 שורות. (CPC מבוסס link-clicks, עקבי עם
  המכנה של conversion_rate.)
- **סיווג-שוק + טווח** — קריאה ל-`benchmark_service.classify_cpl(industry, cpl)` +
  `format_range_he(industry, level)`.
- **תקציב** — קריאה ל-`quiz_service.fetch_quiz_for_campaign` → `quiz_responses.answers.budget`
  (`BudgetSpec{amount, mode, days}`). **לא** מ-`campaigns` (אין שם budget).
- **status, type** — מ-`campaigns` (קיימים).
- **`industry_he`** — תרגום דרך המפה המשותפת (ראה ג).

> שיקול: עדיף wrapper חדש (`get_general_status_for_agent`) שעוטף את הקיים ומוסיף את
> השדות, מאשר לנפח את `get_campaign_status_for_agent` שמשמש את כרטיסי ה-UI. ה-UI לא
> צריך CPC/תקציב/סיווג. **להכריע ב-CC** לפי מבנה הקוד בפועל.

### ג. העברת מפות התרגום למודול משותף (החלטה 8)

`_INDUSTRY_HE` ו-`_LEVEL_HE` חיים כרגע ב-`agent_orchestrator.py:45-53`. שיחה חופשית
(מסלול 2, ב-`agent_service`) צריכה אותן ל-`industry_he`/`market_classification` בעברית.
**העבר אותן למודול משותף** (למשל `app/services/agent_i18n.py` או לתוך `prompts_service`),
שגם ה-orchestrator וגם `send_user_message` מייבאים. **לא לשכפל** — מקור אמת אחד.

### ד. env switch לכיבוי שיחה חופשית (החלטה 11)

`AGENT_FREE_CHAT_ENABLED` ב-`config.py` (pydantic-settings). כש-`false`:
- `send_user_message` (מסלול 2) מחזיר הודעה קבועה שמפנה את הלקוח לצ'יפים, או חוסם את
  הקלט החופשי ב-UI. **להכריע בנוסח/התנהגות עם גיא.**
- זרם הצ'יפ (מסלול 1) ממשיך לעבוד כרגיל — הוא לא תלוי בדגל.

> שיחה חופשית עשויה לרדת מהמוצר. הדגל מבטיח שכיבוי = שינוי במקום אחד, בלי לפרק קוד.

### ה. המרת קבצי-פר-בעיה (אם 7.5/7.6 כבר נכתבו)

אם CC כבר יצר `problem_1.txt`/`problem_2.txt`/`problem_3.txt` (מבנה 7.4א/7.5/7.6
המקורי) — להמיר ל-`core.txt` + 3 `module_*.txt` לפי חלקים 2-5. הלוגיקה (orchestrator,
מנוע שלבים) ללא שינוי — רק קבצי הטקסט ופונקציית ההרכבה.

---

## חלק 8 — בדיקות

**`build_agent_prompt`:**
1. `mode='chip'`, `issue_type='high_cpl'` → core(בלוק א) + module_high_cpl + section high_cpl. **בלי בלוק ב.**
2. `mode='free'` → core(בלוק א + ב) + `[GENERAL_STATUS]`. **בלי מודול, בלי step_instruction.**
3. `mode='chip'` → Payload **לא** מכיל את ה-sections של בעיות אחרות.
4. `mode='free'` → Payload **לא** מכיל שדות-פרוטוקול (`protocol_stage` וכו').

**`[GENERAL_STATUS]` (הרחבת הפונקציה):**
5. CPC מחושב נכון מ-`spend/link_clicks`; `link_clicks=0` → טיפול בחלוקה-באפס (null/0, לא קריסה).
6. סיווג-שוק + טווח מוזרקים (קריאה ל-`benchmark_service`).
7. תקציב מוזרק מה-quiz; אין quiz/budget → שדה ריק, לא קריסה.
8. `industry=NULL` → `market_classification=unknown`, `industry_he` ריק/"לא זוהה".

**מפות תרגום:**
9. `_INDUSTRY_HE`/`_LEVEL_HE` נגישות גם ל-orchestrator וגם ל-`send_user_message` מהמודול המשותף.

**env switch:**
10. `AGENT_FREE_CHAT_ENABLED=false` → `send_user_message` חוסם/מפנה; זרם הצ'יפ ממשיך לעבוד.
11. `AGENT_FREE_CHAT_ENABLED=true` → שיחה חופשית עובדת.

**מכסה (אימות מול 7.1/7.4א):**
12. הודעת לקוח חופשית (מסלול 2) → נספרת ב-`agent_chat_quota`.
13. לחיצת צ'יפ (מסלול 1) → **לא** נספרת.

---

## חלק 9 — לא בסשן

- **פרומפט היצירה** (מסלול 3) — הקריאה הנפרדת שמייצרת 3 קופי/כותרות/שאלת-סינון/תיקון.
  כל כללי-היצירה (זוויות, אורכי קופי, כללי-ניסוח שאלה, טון גיא) חיים שם. **מסמך נפרד.**
- **מגמה/השוואת-תקופות** ב-`[GENERAL_STATUS]` (החלטה 9) — דורש הרחבת שליפת Insights.
- **הכרעת שורה 4 בבעיה 1** ([מועמד למחיקה]) — אחרי ש-CC מריץ 7.4א ורואים אם המסלול נגיש.
- **גרסת A/B רזה (גנספארק)** — הנוסח הרזה של גנספארק כגרסה B בסנדבוקס. לא פרודקשן.
- **כל מה שב-7.4ב לא-בסשן** (ניטור יזום, וואטסאפ, בעיות 2-3 אוטונומיות — Phase 8).

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **זה ארגון פרומפט + שכבת הרכבה, לא שינוי קוד-סוכן.** ה-orchestrator/push/מנוע-השלבים
   מ-7.4ב — ללא שינוי. רק `build_agent_prompt` חדש, הרחבת הפונקציה ל-`[GENERAL_STATUS]`,
   העברת המפות, ו-env switch.

2. **חוק 7 הוא הקו האדום.** הפרומפט אף פעם לא מכיל `if/then` להחלטה — הקוד בחר את המצב
   והזריק הוראה אחת. אם מוצאים את עצמנו שולחים למודל "תחליט אם נעול/חורג/השתפר" — טעות.

3. **שני מסלולי קריאה, אל תבלבל.** צ'יפ (orchestrator + section + step_instruction +
   בלוק א) מול שיחה חופשית (`send_user_message` + `[GENERAL_STATUS]` + בלוק א+ב). מסלול
   שלישי (יצירה) נפרד לגמרי.

4. **עובדה מול פרוטוקול.** `[GENERAL_STATUS]` נושא נתוני-עובדה בלבד. `protocol_stage`
   ושות' לא נשלחים בשיחה חופשית — הם מצב-מכונה שתקף רק בזרם-הצ'יפ. שליחתם בחופשי = המודל
   "מדבר פרוטוקול" בלי שהופעל.

5. **ערכי enum באנגלית ב-Payload, המודל מתרגם.** `EXPENSIVE` נשאר באנגלית; המודל פולט
   "גבוה". חוק שפה 4 ב-core.

6. **שדה לא-רלוונטי → לא נשלח** (לא null). המבנה הקבוע הוא של ה-section הפעיל; שדות שלא
   רלוונטיים למצב פשוט לא מופיעים.

7. **שתי שורות template בלבד** (Lock + חוק-ברזל בבעיה 1; Lock בבעיה 2). השאר הוראות-LLM.
   ה-template מוחזר ישירות בלי קריאת מודל.

8. **מקור אמת אחד לתרגום ולחוקים.** המפות במודול משותף, ה-Core בקובץ אחד מחולק. לא לשכפל
   בין מסלולים.

9. **Sentry context.** exceptions ב-`build_agent_prompt` → `mode`, `issue_type`, השלב
   ברצף. חוסר שדה חובה = באג קוד (לא תלונת-מודל ללקוח, החלטה 13) → Sentry + הודעה גנרית.

10. **אין migration חדש** — משתמש בטבלאות קיימות (7.3.5). רק קוד, קבצי prompt, ו-env.

---

# Session 7.6.5 — בעיה 4: חוסר התאמה תקציבית (מעט לידים מול תקציב) ✅ בוצע (PR1-6)

> **✅ מומש (PR1-5, branch `claude/eloquent-bardeen-9kGfT`):** עמודת `monthly_lead_goal` + שאלת quiz + צ'יפ
> מותנה (PR1); `budget_feasibility_service` + `diagnose_problem_4` + `market_cpl_for_industry` (PR2); מסלול א'
> (approve) + טבלת `budget_agreements` + `meta.update_ad_set` (PR3); מסלול ב' (decline) + לולאה reactive (PR4);
> סנדבוקס + docs (PR5). **2 התאמות מול התכנון:** (א) התקציב ב-Meta ברמת **ad_set** (לא campaign); (ב)
> `monthly_lead_goal` = **עמודה** ב-campaigns (לא JSONB). `market_cpl` = `(amazing_max+expensive_above)/2`.
> **✅ חלק 7 (force-creative) — מומש ב-PR6:** מיני-session (`budget_mismatch` ב-`_STEP_PLANS`+migration 0104,
> `window_hours=None` ב-`push`) → `force_creative` (תיעוד `creative_against_advice`+`cpl_before` reserve-first →
> 3 וריאציות `solution_service` → `push` בלי מחזור מדידה → סגירת session). לולאת ה-reactive הועשרה (תאריך +
> `cpl_now` חי). endpoint `budget/force-creative` + סנדבוקס + frontend (gating ל-spend כמו `מאשר`). פירוט מלא:
> `docs/problem4-force-creative-plan.md`.
>
> בלוק להוספה ל-ROADMAP.md תחת `## Phase 7 · סוכן AI פנימי`. תכנון מלא להעברה ל-CC.
> **מיקום:** 7.7 — הצ'יפ הרביעי בפרוטוקול. **תלוי ב-7.4א** (Lock, orchestrator, סדרה), **7.3.5** (benchmark — עלות-ליד-שוק), **7.2** (CPL חי). **לא תלוי ב-7.4ב** (אין מנוע שלבים כאן).
> **חשוב:** זו **בעיה חדשה (רביעית)** — לא דורסת את 7.4/7.5/7.6. הצ'יפ "מעט פניות".

---

## מבוא ותיאור Session

שלוש הבעיות הקיימות מטפלות בקמפיין **לא בריא** (עלות גבוהה, פניות גרועות, מודעה נדחתה). בעיה 4 שונה במהותה: היא מטפלת במצב שבו **הקמפיין דווקא בריא לגמרי** — העלות לליד תקינה לפי השוק — אבל הלקוח מתלונן על מעט לידים, כי **התקציב פשוט קטן מדי כדי להגיע ליעד הכמותי שלו.**

זו לא בעיה שמתקנים בקמפיין. אין מה "לשפר" — המתמטיקה פשוט לא מסתדרת: `₪50 ביום × עלות-שוק תקינה = X לידים`, והלקוח רוצה `2X`. הסוכן לא מנסה לתקן קמפיין תקין (וכך להרוס אותו) — הוא **מציג את האמת המתמטית** ונותן ללקוח להחליט.

**ההבדל המרכזי מבעיה 1:** בעיה 1 שואלת "האם העלות גבוהה מהשוק?" ומשפרת. בעיה 4 שואלת "**האם היעד בכלל בר-השגה עם התקציב?**" — חישוב מתמטי חדש, לא סיווג benchmark.

**שני דברים חדשים לגמרי שלא קיימים באף מודול אחר:**
1. **אבחון מתמטי** — חישוב היתכנות: האם `(תקציב × ימים) / עלות-ליד-שוק ≥ יעד`.
2. **תיעוד משפטי** — שמירת הסכמות מתוארכות לשליפה עתידית ("כפי שאישרת ב-X..."), כהגנה מפני תלונות. אף מודול אחר לא שומר הסכמות.

**מה בסשן:**
- שאלה חדשה ב-quiz: יעד כמותי חודשי (`monthly_lead_goal`, nullable).
- צ'יפ "מעט פניות" — **מותנה**: מופיע רק אם יש יעד.
- `budget_feasibility_service` — האבחון המתמטי הדטרמיניסטי.
- שני צ'יפי בחירה (העלה תקציב / אל תעלה) + מסלול לכל אחד.
- העלאת תקציב ב-Meta (`update_campaign_budget`).
- טבלה חדשה `budget_agreements` — התיעוד המשפטי (3 סוגים).
- ניסיון קריאייטיב חד-פעמי במסלול הסירוב (לא מנוע בעיה 1).
- לולאה reactive: לקוח חוזר → שליפת תיעוד → אבחון מחדש.
- endpoints: `diagnose` (אבחון), `approve_budget` (העלאה), `decline_budget` (סירוב), `force_creative` (התעקשות).

**מה לא בסשן:**
- מנוע השלבים של בעיה 1 (אין שלבים 2-7 כאן).
- מעקב 120 שעות אקטיבי (הלולאה כאן reactive — לקוח חוזר, לא cron).
- ניטור יזום (Phase 8).
- שינוי היעד אחרי הקמת הקמפיין (היעד נקבע ב-quiz; עריכתו — הרחבה עתידית).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | שם הבעיה | `budget_mismatch` — בעיה רביעית. לא דורסת קיימות |
| 2 | מקור היעד הכמותי | **שאלה חדשה ב-quiz:** "האם יש לך יעד של כמות לידים לחודש? אם כן — ציין; אם לא — דלג." נשמר `monthly_lead_goal` (nullable) |
| 3 | הצ'יפ מותנה | **"מעט פניות" מופיע רק אם `monthly_lead_goal` קיים.** דילג ב-quiz → אין צ'יפ → אין בעיה. אין מצב-קצה "נכנסנו בלי יעד" |
| 4 | האבחון | **מתמטי דטרמיניסטי (חוק 7).** הקוד מחשב `(daily_budget × 30) / market_cpl ≥ goal`. ה-LLM רק מנסח. לא סיווג benchmark |
| 5 | היעד בר-השגה (אבל הלקוח מתלונן) | הסוכן מסביר "לפי התקציב והשוק אתה אמור להגיע ליעד — הבעיה כנראה לא בתקציב" ו**מפנה לבעיה 1** (אולי העלות גבוהה / הקמפיין לומד) |
| 6 | מי בוחר מסלול א'/ב' | **הלקוח, לא הקוד.** המערכת מציגה את הבעיה + שני צ'יפים. הבחירה של הלקוח קובעת. (כמו override בבעיה 1) |
| 7 | שינוי קריאייטיב במסלול הסירוב | **לא הפעלה מלאה של מנוע בעיה 1.** ניסיון חד-פעמי: 3 וריאציות, אישור, העלאה. בלי סדרה/שלבים/cron |
| 8 | נקודות התיעוד | **שלוש:** (א) הסכמה להעלאת תקציב, (ב) סירוב/בחירה להישאר, (ג) התעקשות על קריאייטיב נגד המלצה. כל אחת בטבלה עם סוג+תאריך+נתונים |
| 9 | הלולאה | **reactive, לא cron.** לקוח חוזר ומתלונן → שליפת התיעוד האחרון → הצגתו → חזרה לאבחון (שלב 1). אין מעקב אקטיבי |
| 10 | תיעוד הסירוב | **קריטי להגנה.** הסירוב ("בחרתי להישאר על ₪50") הוא התיעוד שמגן הכי הרבה מול "למה מעט לידים?!" |

---

## תלויות

1. **7.4א** — `agent_orchestrator`, `lock_service`, `optimization_sessions`, סדר החשיבה (Lock → אבחון → LLM). בעיה 4 משתמשת באותו orchestrator.
2. **7.3.5** — `benchmark_service` (עלות-ליד-שוק לענף — המספר שמזין את החישוב), `campaigns.industry`.
3. **7.2** — `get_campaign_status_for_agent` (תקציב יומי נוכחי, CPL חי).
4. **3.4** — קמפיין חי עם `meta_campaign_id` (להעלאת תקציב).
5. **quiz** (Phase קודם) — להוספת השאלה החדשה.

> **תלות הפוכה ל-7.6:** התיעוד והאבחון יכולים להשתמש ב-`send_agent_notification` (7.6) אם רוצים להודיע ללקוח. ב-MVP לא חובה — הלולאה reactive (הלקוח חוזר מעצמו).

---

## חלק 1 — שאלת ה-quiz החדשה

הוספת שאלה ל-quiz של יצירת הקמפיין.

**הניסוח (כפי שאמיר אישר):**
> "האם יש לך יעד של כמות לידים לחודש? אם כן — ציין מהו."
> "אם אין לך יעד כזה — לחץ 'דלג'."

**שמירה:** `campaigns.monthly_lead_goal integer` (**nullable** — null = דילג).

**השפעה:** השדה הזה הוא ה**שער** לכל הבעיה. בלעדיו (null), הצ'יפ "מעט פניות" לא מופיע (חלק 2), והבעיה לא נגישה. לכן כשנכנסים לבעיה — היעד **תמיד** קיים. אין null-handling בתוך הזרימה.

---

## חלק 2 — הצ'יפ המותנה

מבנה הצ'אט הראשי הופך **דינמי** — מספר הצ'יפים תלוי בלקוח.

```
"הקמפיין שלך — מה הבעיה שנטפל בה?"
→ צ'יפ: עלות לפנייה גבוהה        (תמיד)
→ צ'יפ: פניות לא איכותיות        (תמיד)
→ צ'יפ: מעט פניות                (רק אם monthly_lead_goal IS NOT NULL)
```

**מימוש:** כשהקוד בונה את רשימת הצ'יפים להצגה (ב-7.1/7.4א), הוא בודק `if campaign.monthly_lead_goal is not None` → מוסיף את הצ'יפ השלישי.

> "מעט פניות" הוא הצ'יפ היחיד מהשלושה שמותנה. השניים האחרים תמיד מוצגים.

---

## חלק 3 — האבחון המתמטי (`budget_feasibility_service`)

הלב הדטרמיניסטי. רץ כשהלקוח לוחץ "מעט פניות", **לפני** שה-LLM נכנס (חוק 7).

**הקלט:**
- `daily_budget` — התקציב היומי הנוכחי (מ-7.2 / Meta).
- `market_cpl` — עלות-ליד ממוצעת בשוק לענף (מ-`benchmark_service`, 7.3.5).
- `monthly_lead_goal` — היעד (מה-quiz).

**החישוב:**
```
expected_leads = (daily_budget × 30) / market_cpl
feasible = expected_leads >= monthly_lead_goal
required_budget (W) = (monthly_lead_goal × market_cpl) / 30   # התקציב היומי הדרוש ליעד
```

**שתי תוצאות:**

**א. `feasible = true`** (היעד בר-השגה, אבל הלקוח מתלונן — החלטה 5):
הבעיה כנראה לא בתקציב. הסוכן מנסח:
> "לפי התקציב שלך (₪Y ביום) ועלות הליד בשוק (₪X), אתה אמור להגיע לכ-[expected_leads] לידים בחודש — שזה מעל היעד שלך. אז הבעיה כנראה לא בתקציב. בוא נבדוק אם עלות הליד שלך גבוהה מהממוצע."
→ מפנה לבעיה 1 (הצ'יפ "עלות גבוהה" / מריץ את אבחון בעיה 1).

**ב. `feasible = false`** (היעד לא בר-השגה):
→ ממשיך לחלק 4 (הצגת הבעיה + הצ'יפים).

> **חוק 7:** כל החישוב הזה בקוד. ה-LLM לא "מעריך" אם היעד בר-השגה — הוא מקבל `feasible`, `expected_leads`, `required_budget` מוכנים, ומנסח. ה-benchmark (`market_cpl`) מגיע מ-7.3.5, לא מהמודל.

---

## חלק 4 — הצגת הבעיה + שני הצ'יפים

כש-`feasible = false`, הסוכן מציג את האמת המתמטית (LLM מנסח את המספרים שהקוד חישב):

> "בוא נסתכל על המספרים יחד. עלות ליד ממוצעת בשוק לענף שלך היא ₪[X]. כדי להגיע ליעד שלך של [Z] לידים בחודש, נדרש תקציב יומי של כ-₪[W]. התקציב הנוכחי שלך, ₪[Y] ביום, מאפשר מתמטית כ-[expected_leads] לידים — פחות מהיעד. זה לא עניין של איכות הקמפיין; הוא עובד תקין. זה פער מתמטי בין התקציב ליעד."

ואז **שני צ'יפי בחירה** (הלקוח בוחר, לא הקוד — החלטה 6):
```
→ צ'יפ: הגדל תקציב ל-₪[W]
→ צ'יפ: השאר את התקציב הקיים
```

---

## חלק 5 — מסלול א': העלאת תקציב

הלקוח לחץ "הגדל תקציב".

1. **אישור מפורש:** הסוכן מציג שוב את הפעולה המדויקת:
   > "אני מעלה את התקציב היומי מ-₪[Y] ל-₪[W]. שים לב: התשלום למטא ייגבה ישירות מהאשראי שלך לפי התקציב החדש. מאשר?"
   [מאשר / ביטול]
2. **תיעוד (לפני הביצוע):** כתיבה ל-`budget_agreements` — סוג `budget_increase_approved`, תאריך, `old_budget`, `new_budget`, הנתונים שהוצגו (X, Y, Z, W).
3. **ביצוע:** `meta_service.update_campaign_budget(meta_campaign_id, new_budget)`.
4. **אישור ללקוח:** "התקציב עודכן ל-₪[W] ביום. אתה אמור לראות עלייה בכמות הלידים בהתאם."

> **תיעוד לפני ביצוע** (reserve-first): רושמים את ההסכמה לפני הקריאה ל-Meta. אם Meta נכשל — יש rollback של התיעוד, אבל ההסכמה המתועדת לא הולכת לאיבוד אם הביצוע מצליח וה-confirmation נכשל.

---

## חלק 6 — מסלול ב': סירוב + ניסיון קריאייטיב אופציונלי

הלקוח לחץ "השאר את התקציב הקיים".

1. **תיעוד הסירוב (קריטי — החלטה 10):** כתיבה ל-`budget_agreements` — סוג `budget_increase_declined`, תאריך, הנתונים שהוצגו. **זה התיעוד שמגן הכי הרבה.**
2. **הסבר הסוכן:**
   > "הבנתי. היעד לכמות הלידים יישאר בהתאם לתקציב הנוכחי — כלומר כ-[expected_leads] לידים בחודש. אפשר טכנית לנסות שינויי קריאייטיב כדי לשפר מעט את העלות לליד, אבל אני חייב להיות כן: זה לא מומלץ כלל. הקמפיין שלך עובד תקין, ושינוי כרגע עלול דווקא לפגוע ביציבות שלו בלי סיבה מקצועית."
   [הבנתי, משאיר כמו שזה / בכל זאת נסה קריאייטיב]
3. **אם הלקוח בחר "משאיר כמו שזה":** סוף. הסירוב מתועד, הלולאה תיסגר reactive.
4. **אם הלקוח התעקש על קריאייטיב** → חלק 7.

---

## חלק 7 — התעקשות על קריאייטיב (ניסיון חד-פעמי)

הלקוח התעקש לנסות קריאייטיב נגד ההמלצה.

1. **תיעוד ההתעקשות (החלטה 8ג):** כתיבה ל-`budget_agreements` — סוג `creative_against_advice`, תאריך, `cpl_before` (העלות לפני, לשליפה עתידית). **זה המסלול היחיד שבו הסוכן עושה משהו שהוא עצמו אמר שמזיק — ולכן הכי צריך תיעוד.**
2. **יצירת 3 וריאציות:** קריאה ל-`solution_service` (מ-7.4ב — אותו service). 3 קופי/כותרות/זוויות.
3. **אישור + העלאה:** דרך `optimization_push_service` (מ-7.4ב — אותו דפוס אטומי). **אבל:** ללא פתיחת סדרה (`optimization_session`), ללא `window_ends_at`, ללא מנוע שלבים. ניסיון בודד.
4. **אישור ללקוח:** "העליתי את הווריאציות החדשות. כפי שהסברתי, אני לא צופה שיפור משמעותי — הקמפיין כבר היה תקין."

> **השוני המהותי מבעיה 1:** אותם services (`solution_service`, `optimization_push_service`), אבל **בלי המנגנון סביבם** — אין סדרה, אין 120 שעות, אין שלבים, אין לולאה. זה ניסיון חד-פעמי שהלקוח כפה. `cpl_before` נשמר לתיעוד, לא למעקב אקטיבי.

---

## חלק 8 — הלולאה ה-reactive

המנגנון שסוגר את הכל. **לא cron — תגובתי.**

כשהלקוח **חוזר ומתלונן שוב** ("מעט פניות") — הקוד, לפני האבחון:
1. **שולף את התיעוד האחרון** מ-`budget_agreements` לקמפיין הזה.
2. **מציג אותו ללקוח** לפי הסוג:
   - `budget_increase_declined` → "כפי שבחרת ב-[תאריך], התקציב נשאר על ₪[Y], ולכן כמות הלידים מוגבלת לכ-[expected_leads] בחודש. רוצה לשקול שוב להעלות תקציב?"
   - `creative_against_advice` → "ב-[תאריך] התעקשת לנסות שינוי קריאייטיב נגד המלצתי. כפי שהזהרתי, העלות לליד הייתה ₪[cpl_before] לפני, ועכשיו היא ₪[cpl_now]. הבעיה לא הייתה בקריאייטיב אלא בפער התקציב."
   - `budget_increase_approved` → "ב-[תאריך] העלינו את התקציב ל-₪[W]. אם עדיין מעט לידים, בוא נבדוק שוב את המספרים."
3. **חוזר לאבחון (חלק 3)** — מחשב מחדש את ההיתכנות עם הנתונים הנוכחיים (אולי התקציב השתנה, אולי עלות השוק השתנתה).

> **התיעוד הוא הזיכרון.** הלולאה לא צריכה cron או מעקב — בכל פעם שהלקוח חוזר, הקוד שולף את ההיסטוריה ומאבחן מחדש. זה reactive, פשוט, ועמיד.

---

## חלק 9 — Migration

קובץ חדש (מספר רץ הבא).

**א. `campaigns.monthly_lead_goal`:**
```sql
ALTER TABLE public.campaigns
  ADD COLUMN IF NOT EXISTS monthly_lead_goal integer;  -- nullable; null = דילג ב-quiz
```

**ב. טבלת התיעוד `budget_agreements`:**
```sql
CREATE TABLE public.budget_agreements (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id uuid NOT NULL REFERENCES public.campaigns(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  agreement_type text NOT NULL CHECK (agreement_type IN (
    'budget_increase_approved',   -- הסכים להעלות תקציב
    'budget_increase_declined',   -- בחר להישאר (קריטי להגנה)
    'creative_against_advice'     -- התעקש על קריאייטיב נגד המלצה
  )),

  -- הנתונים שהוצגו בזמן ההסכמה (לשליפה: "כפי שאישרת ב-X, עלות השוק הייתה Y")
  market_cpl numeric,
  daily_budget_at_time numeric,    -- Y
  monthly_lead_goal_at_time integer,  -- Z
  required_budget numeric,         -- W
  expected_leads integer,

  -- ספציפי לפי סוג
  new_budget numeric,              -- ל-approved: התקציב החדש
  cpl_before numeric,              -- ל-creative_against_advice: העלות לפני השינוי

  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_budget_agreements_campaign
  ON public.budget_agreements (campaign_id, created_at DESC);
```
- שורה פר-הסכמה. השליפה ב-reactive = האחרונה לפי `created_at DESC`.
- RLS: SELECT לבעלים, כתיבה דרך service_role (כמו שאר הטבלאות).

---

## חלק 10 — שינויי קוד נדרשים

**א.** quiz — שאלה חדשה + שמירת `monthly_lead_goal` (חלק 1).
**ב.** בניית הצ'יפים — תנאי על `monthly_lead_goal` (חלק 2).
**ג.** `budget_feasibility_service` חדש — האבחון המתמטי (חלק 3).
**ד.** `meta_service.update_campaign_budget` — העלאת תקציב ב-Meta (חלק 5). לבדוק אם כבר קיים מ-3.x.
**ה.** `budget_agreement_service` — כתיבה/שליפה של התיעוד (חלקים 5-8).
**ו.** endpoints ב-`routers/agent.py`:
   - `POST .../diagnose` — מורחב לטפל ב-`budget_mismatch` (או endpoint ייעודי).
   - `POST .../budget/approve` — מסלול א'.
   - `POST .../budget/decline` — מסלול ב'.
   - `POST .../budget/force-creative` — התעקשות (חלק 7).
**ז.** orchestrator — ניתוב הצ'יפ "מעט פניות" דרך `budget_feasibility_service` (חלק 3).
**ח.** migration (חלק 9).
**ט.** prompt — מודול חדש `module_budget_mismatch` (במבנה המודולרי, אם אומץ) או החלק המתאים בפרומפט.

---

## חלק 11 — בדיקות

**אבחון מתמטי:**
1. `feasible=false` (תקציב נמוך מהיעד) → מציג בעיה + 2 צ'יפים.
2. `feasible=true` (תקציב מספיק, לקוח מתלונן) → מפנה לבעיה 1.
3. החישוב נכון: `(budget×30)/cpl` מול goal.
4. `market_cpl` מגיע מ-benchmark (7.3.5), לא מהמודל.

**צ'יפ מותנה:**
5. `monthly_lead_goal=null` → הצ'יפ "מעט פניות" לא ברשימה.
6. `monthly_lead_goal=100` → הצ'יפ מופיע.

**מסלול א' (העלאה):**
7. אישור → תיעוד `budget_increase_approved` נכתב **לפני** קריאת Meta.
8. `update_campaign_budget` נקרא עם הסכום הנכון.
9. Meta נכשל → rollback של התיעוד.

**מסלול ב' (סירוב):**
10. סירוב → תיעוד `budget_increase_declined` נכתב.
11. "משאיר כמו שזה" → סוף, בלי פעולת Meta.

**התעקשות (חלק 7):**
12. התעקשות → תיעוד `creative_against_advice` עם `cpl_before`.
13. 3 וריאציות נוצרות, מועלות — **בלי** פתיחת `optimization_session`, בלי `window_ends_at`.

**לולאה reactive:**
14. לקוח חוזר → שליפת התיעוד האחרון → הצגה לפי סוג → אבחון מחדש.
15. תיעוד `declined` קודם → הצגת "כפי שבחרת ב-X...".
16. תיעוד `creative_against_advice` קודם → הצגת "התעקשת ב-X, העלות הייתה Y, עכשיו Z".

---

## חלק 12 — לא בסשן

- מנוע השלבים של בעיה 1 (אין שלבים כאן).
- מעקב 120 שעות אקטיבי (הלולאה reactive).
- עריכת היעד אחרי הקמת קמפיין (היעד מה-quiz; עריכה — הרחבה).
- ניטור יזום (Phase 8) — הסוכן שמזהה לבד שהיעד לא בר-השגה.
- התראות וואטסאפ (Phase 8).

---

## חלק 13 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **בעיה רביעית — לא דורסת קיימות.** `budget_mismatch` נפרד מ-7.4/7.5/7.6. הצ'יפ הרביעי.

2. **שני דברים חדשים שאין באף מודול:** אבחון מתמטי (חישוב היתכנות) ותיעוד משפטי (`budget_agreements`). שאר התשתית (Lock, orchestrator, solution_service, push) מ-7.4.

3. **חוק 7 נשמר.** הקוד מחשב `feasible`/`required_budget`/`expected_leads` מ-benchmark; ה-LLM מנסח את המספרים. המודל לא מעריך היתכנות.

4. **הצ'יפ מותנה = השער.** היעד תמיד קיים כשנכנסים לבעיה (כי בלי יעד אין צ'יפ). אין null-handling בזרימה.

5. **הלקוח בוחר מסלול, לא הקוד.** המערכת מציגה בעיה + 2 צ'יפים. הבחירה קובעת. כמו override בבעיה 1.

6. **שלוש נקודות תיעוד, והסירוב הכי חשוב.** הסכמה / סירוב / התעקשות. הסירוב ("בחרתי להישאר") מגן הכי הרבה מול "למה מעט לידים?!".

7. **תיעוד לפני ביצוע (reserve-first).** רושמים את ההסכמה לפני הפעולה ב-Meta. עקבי עם כל הדפוסים של 7.4ב.

8. **ניסיון הקריאייטיב הוא חד-פעמי, לא מנוע בעיה 1.** אותם services, בלי סדרה/שלבים/cron. `cpl_before` לתיעוד, לא למעקב.

9. **הלולאה reactive — אין cron.** התיעוד הוא הזיכרון. לקוח חוזר → שליפה → אבחון מחדש. פשוט ועמיד.

10. **`update_campaign_budget`** — לבדוק אם כבר קיים מ-3.x. אם כן, להשתמש; אם לא, להוסיף ל-`meta_service`.

11. **מספר migration + מספר Phase (7.6.5?)** — לתאם ב-ROADMAP מול הסשנים הקיימים.



---

# Session 7.7Q

## Creative VIP Studio (לשונית יצירת/שדרוג חומרים)

> **⚠️ עדכון gating (החלטת-מוצר, אמיר — יולי 2026): Creative זמין ל-*כל מנוי פעיל*, לא Premium-only.**
> ה-"Premium gating" שמתואר ב-7.7Q-א להלן היה הגבלה שגויה (מעולם לא הוחלט premium-only) → תוקן:
> ה-endpoints קוראים `require_active_subscription` (status∈{trial,active} + tier≠pending — כל חבילה
> basic/premium/whatsapp; pending=טרם-בחר-חבילה → 402 `NotActiveSubscriptionError`), והמכסה ניתנת לכל
> tier מוכר (`_ELIGIBLE_TIERS`; tier לא-מוכר/None → 0 fail-safe, המכסות עצמן זהות לכל חבילה). יומן/בוט/
> פגישות **נשארו** Premium-only (`require_premium_access`). ה-frontend כבר היה פתוח (nav לא-gated) —
> השינוי backend בלבד. *(pending לא נכלל: בלי קמפיין חי יצירה נכשלת 409 ממילא; ה-flow — בחר חבילה→Studio.)*
>
> **פוצל ל-3 תת-סשנים (אישור המשתמש): 7.7Q-א (תשתית) → 7.7Q-ב (Creator) → 7.7Q-ג (Studio).**
>
> **7.7Q-א ✅ Done (תשתית — creative_assets + 3 מכסות + גלריה + Premium gating):** migration 0063
> (`creative_assets`, `asset_type` generated/upgraded + `published_at` למכסת publish, RLS+GRANT
> select-own). `creative_service` — 3 מכסות חודשיות (יצירה 15 / פרסום 15 / שדרוג 30, ספירה-חי מ-
> `creative_assets` בחלון anniversary, `assert_quota` block-before-action), `list_gallery` (RLS).
> endpoints `GET /me/creative/quota` + `GET /me/creative/gallery` (Premium→402). helper משותף
> `subscription_service.get_billing_period`. **3 מונים** (לא 2) לפי תשובות אמיר.
> **[PR-C3b — per-campaign]** המכסות (generation/publish/upgrade/revise) עברו **per-campaign** (חלון
> `campaigns.cycle_*`, COUNT `WHERE campaign_id`; migration 0132 — 6 RPCs re-keyed `p_campaign_id`, advisory-locks
> per-campaign פרט ל-revise per-source). `get_billing_period` **הוסר** (הקורא-האחרון של `subscriptions.tier` ב-quota-
> path — מכשיר C4). `get_campaign_creative_quota_status` + `get_user_creative_quota_aggregate` (len==1 passthrough
> → zero frontend); endpoint `GET /campaigns/{id}/creative/quota`; campaign_id דרך `get_live_campaign` (interim; picker→C2).
>
> **7.7Q-ב ✅ Done (מסלול ב' The Creator — generate + publish):** `POST /me/creative/generate`
> (`ad_generation_service.generate_image_bytes_from_prompt` — prompt חופשי ישיר ל-gpt-image-2, reuse
> `_raw_image`+`_extract_image_bytes`; Storage). `image_url` שומר את ה-**storage path** (לא signed URL —
> פג אחרי 7 ימים); signed URL טרי מיוצר on-read (`_sign_asset`) בגלריה, בתגובות, ובהורדה ל-publish (Cursor).
> `POST /me/creative/{id}/publish`
> (Ad רביעי לקמפיין ה-live היחיד — `campaign_service.get_live_campaign`; reuse helpers ה-push מ-
> `optimization_push_service`: `_gate_token`/`_build_opt_creative_spec`/`_build_opt_ad_params`/
> `_download_image`/`_is_transient`; **RPC אטומי** `reserve_creative_publish` (CAS + אכיפת מכסת publish
> תחת advisory lock פר-user — מונע פרסום כפול **וגם** חריגת מכסה מקבילית, Cursor/כלל 2); rollback בכשל;
> token פג 190→`mark_token_expired`). **save-as-you-go (Cursor):** `_record_published_ad`
> (`creative_assets.published_ad_id`) **מיד אחרי `create_ad`**, ואז `_link_published_ad` (RPC
> `link_creative_vip_ad` — `campaigns.meta_ad_ids` atomic append + שורת `ads` `angle='creative_vip'`,
> migration 0067 הרחיב CHECK ו-UNIQUE→partial). idempotent-return: `published_ad_id` קיים → משלים linkage
> (resume אחרי crash). בלעדי הקישור ליד מה-Ad אבוד (find_by_ad_id), דחייתו (בעיה 3) orphan, Insights
> שבורים. generate משתמש ב-`insert_generated_creative_asset` (אותו דפוס אטומי). קופי = **2 שדות**
> (headline+body); checkbox אישור-אחריות חובה
> (`responsibility_accepted` Literal-True). מכסת publish נספרת דרך RPC `creative_quota_counts` על slot
> תפוס = finalized או reserved-recently בתוך TTL (10 דק'); reserve תקוע (crash בין reserve ל-create)
> משוחרר אוטומטית אחרי ה-TTL — re-claimable ולא נספר ב-cap (Cursor, TTL recovery כמו offer-lock 0062).
>
> **7.7Q-ג ✅ Done (מסלול א' The Studio — upgrade):** `POST /me/creative/upgrade` (multipart — ה-endpoint
> הראשון בפרויקט עם `UploadFile`; הוסף `python-multipart` ל-requirements + render). הלקוח מעלה JPG/PNG
> ≤10MB (ולידציה ב-router, גבול HTTP: 415 type / 413 size / 422 ריק) → `ad_generation_service.upgrade_image_bytes`
> (`gpt-image-2` ב-**edit mode**, `client.images.edit`, reuse `_with_retry`+`_extract_image_bytes`;
> `_raw_image_edit` בונה `BytesIO` **טרי בכל retry** — stream שנצרך לא נקרא שוב, Cursor) → `upgrade_creative`
> שומר את תמונת-המקור (`source_image_url`) ואת התוצר (`image_url`) ב-Storage + RPC אטומי
> `insert_upgraded_creative_asset` (migration 0070 — אכיפת מכסת שדרוג 30 תחת advisory lock פר-user, מירור
> `insert_generated_creative_asset`, כלל 2); cleanup של **שני** הקבצים על כל כשל (orphan storage). פרומפט
> קבוע `prompts/creative/image_upgrade.txt` דרך `prompts_service.build` (כלל 8 — phase='creative'). **לא
> מתפרסם** — מוצג בגלריה בלבד (החלטה 6); `_sign_asset` חותם גם את `source_image_url` on-read. **7.7Q הושלם.**

> בלוק להוספה ל-ROADMAP.md תחת `## Phase 7` (או Phase נפרד — לתאם מספור).
> תכנון מלא להעברה ל-CC.
> **פיצ'ר עצמאי** — לא חלק מפרוטוקול הסוכן (7.4–7.6). יצירת-תוכן ידנית ביוזמת הלקוח,
> לא אופטימיזציה אוטונומית. **לא תלוי בסוכן**, אבל ראה תלות בבעיה 3 (חלק "תלויות").
> **מיועד ל-Premium בלבד** (לשונית VIP).

---

## מבוא ותיאור Session

עד כה כל יצירת התוכן הייתה **חלק מאשף הקמפיין** (3.2/3.3 — קופי+תמונה אוטומטיים
באונבורדינג) או **חלק מהפרוטוקול** (7.4–7.6 — הסוכן מייצר תיקונים). ה-Creative VIP
הוא הראשון שבו **הלקוח יוזם יצירת תוכן עצמאית**, מתי שהוא רוצה, מנותק מהקמפיin
ומהפרוטוקול.

העמוד (`Creative VIP Studio`) כולל **שני מסלולים פעילים** + גלריה:
- **מסלול א' (The Studio) — שדרוג תמונה קיימת:** הלקוח מעלה תמונה מהמכשיר, ה-AI משפר
  אותה לרמה מסחרית. **(זה Phase 3.5 שנדחה — גיא החזיר ל-MVP.)**
- **מסלול ב' (The Creator) — יצירת תמונה מאפס:** הלקוח מתאר ל-AI תמונה רצויה, ה-AI
  יוצר אותה, הלקוח **כותב טקסט ידנית**, והמודעה מתפרסמת לקמפיין קיים.
- **גלריית מוצרים:** כל התוצרים (יצירות + שדרוגים) נשמרים ומוצגים בתחתית העמוד.

> **מסלול ג' (סרטוני אנימציה AI) — מבוטל.** מופיע בפרוטוטייפ אבל **לא בסקופ** — עקבי
> עם דחיית הווידאו ב-spec §3 ("יצירת וידאו — לא בסקופ ה-MVP"). לא לבנות.

המטרה: בסוף הסשן, לקוח Premium נכנס ל-Creative VIP →
- **מסלול ב':** מתאר "תמונה של מדריך שחייה עם ילדים בבריכה" → AI יוצר תמונה → הלקוח
  כותב קופי ידני → מאשר → מתפרסם **כמודעה רביעית** בקמפיין קיים.
- **מסלול א':** מעלה תמונת-מוצר מהטלפון → AI משפר (Ultra HD, תאורה, חדות) → התוצאה
  נשמרת בגלריה.
- שני המסלולים סופרים מהמכסה החודשית; מכסה מלאה → **חסימת יצירה**.

**מה בסשן:**
- **מסלול ב'** — יצירת תמונה מ-prompt (reuse `gpt-image-2` מ-3.3) + קלט-טקסט ידני +
  פרסום כ-Ad רביעי לקמפיin קיים.
- **מסלול א'** — image editing (יכולת חדשה, `gpt-image-2` edit mode) + העלאת תמונת-מקור.
- **גלריית מוצרים** — טבלה חדשה, מציגה את שני סוגי התוצרים.
- **שני מונה מכסה** — `creative_generation_quota` (מסלול ב') + `creative_upgrade_quota`
  (מסלול א'), עם אכיפת חסימה.

**מה לא בסשן:**
- **מסלול ג' (וידאו/אנימציה)** — מבוטל לחלוטין.
- ניהול-קמפיין כלשהו (זה לא משנה קמפיין — רק מוסיף מודעה / שומר תמונה).
- בדיקת-מדיניות על הטקסט הידני (החלטה 4 — אין בדיקה; בעיה 3 תופסת דחיות).

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מסלול ב' — איך המודעה מתפרסמת | **כ-Ad רביעי בקמפיין קיים** (מתווסף ל-Ad Set הקיים, לא קמפיין חדש). הלקוח בוחר לאיזה קמפיין |
| 2 | מסלול ב' — מי כותב את הטקסט | **הלקוח, ידנית.** ה-AI יוצר רק את התמונה. הלקוח כותב קופי בעצמו ואז מאשר העלאה (הבהרת גיא ב-WhatsApp) |
| 3 | מסלול ב' — קלט ליצירת התמונה | **הלקוח מתאר ל-AI** את התמונה הרצויה (prompt חופשי). reuse של `gpt-image-2` (3.3) |
| 4 | טקסט ידני — בדיקת מדיניות? | **אין בדיקה.** הטקסט עולה כמו שהוא. אם מטא דוחה — בעיה 3 (7.6) תופסת. (ראה תלות) |
| 5 | מסלול א' — מה הוא | **שדרוג תמונה שהלקוח מעלה** מהמכשיר. image editing (`gpt-image-2` edit mode), פרומפט שדרוג מלא (חלק 3). **זה Phase 3.5 שחוזר ל-MVP** |
| 6 | מסלול א' — מה קורה לתוצאה | **נשמרת בגלריית המוצרים.** לא מתפרסמת אוטומטית (בניגוד למסלול ב') |
| 7 | גלריית מוצרים | **טבלה אחת לשני הסוגים** — `creative_assets` עם `asset_type` (`generated`/`upgraded`). מציגה את שניהם |
| 8 | מכסות | **שני מונים נפרדים:** יצירה (מסלול ב') + שדרוג (מסלול א'). ספירה-בחלון anniversary (כמו spec §6). מספרים — ראה "פתוח" |
| 9 | אכיפת מכסה | **חסימת יצירה** כשהמכסה מלאה (לא "מפסיק להגיב" כמו הבוט — כאן חוסם ממש לפני הפעולה) |
| 10 | מסלול ג' (אנימציה) | **מבוטל.** וידאו מחוץ לסקופ (spec §3). לא לבנות |
| 11 | אישור-אחריות | **checkbox חובה** לפני פרסום (מסלול ב') — "אני מאשר שבדקתי את תוצרי ה-AI ואת העלאת המודעה לחשבוני" |

---

## תלויות

1. **3.3 — יצירת תמונה (`gpt-image-2`).** מסלול ב' עושה reuse של מנגנון יצירת-התמונה.
   `integrations/openai.py` כבר עוטף את gpt-image-2.
2. **3.5 — עריכת/שיפור תמונות.** היה ⏸️ נדחה — **חוזר כאן** (מסלול א'). זו היכולת
   החדשה היחידה (image editing mode, להבדיל מ-generation).
3. **3.4 — דפוס ההעלאה ל-Meta.** מסלול ב' מוסיף Ad לקמפיין קיים — reuse חלקי
   (יצירת Ad בודד, לא Campaign+AdSet). ראה חלק 2.
4. **בעיה 3 / 7.6 — webhook דחיות. תלות קריטית (החלטה 4):** הטקסט הידני עולה בלי
   בדיקת-מדיניות. רשת-הביטחון היחידה היא שמטא תדחה ובעיה 3 תטפל. **אם 7.6 לא קיים
   כשמודעה כזו נדחית — הדחייה נופלת בשקט והלקוח לא יודע.** לכן Creative VIP מניח
   ש-7.6 קיים. לתעד; לשקול לחסום פרסום מסלול ב' עד ש-7.6 קיים (env flag).
5. **Supabase Storage** — bucket `campaign-images` (3.3) קיים. מסלול א' צריך גם
   **upload מהמכשיר** (קלט חדש — תמונת-מקור), לא רק שמירת פלט.
6. **5.x — Premium.** העמוד ל-Premium בלבד. gating קיים (כמו הבוט).
7. **subscriptions §6** — דפוס המונים (`agent_chat_quota` וכו'). שני המונים החדשים
   על אותו דפוס (ספירה-בחלון, בלי reset).

---

## חלק 1 — מסלול ב' (The Creator): יצירת תמונה מאפס + פרסום

הזרימה (4 צעדים, הלקוח מעורב בכל אחד):

1. **תיאור** — הלקוח כותב prompt חופשי בעברית ("תמונה של מדריך שחייה עם ילדים בבריכה,
   אווירה כיפית"). reuse של מנגנון התרגום-ל-prompt-תמונה מ-3.3 אם קיים, או ישירות.
2. **יצירה** — `gpt-image-2` יוצר תמונה (Medium 1:1, כמו 3.3). התמונה נשמרת ל-Storage
   + רשומה ב-`creative_assets` (`asset_type='generated'`).
3. **טקסט ידני** — הלקוח **כותב את הקופי בעצמו** בשדה טקסט. ה-AI **לא** מייצר קופי כאן
   (זה ההבדל המהותי מ-3.2). אין בדיקת-מדיניות (החלטה 4).
4. **אישור + פרסום** — checkbox אישור-אחריות (החלטה 11) → הלקוח בוחר לאיזה **קמפיין
   קיים** המודעה מצטרפת → המודעה מתפרסמת כ-**Ad רביעי** ב-Ad Set של אותו קמפיין.

> **ספירת מכסה:** צעד 2 (היצירה) סופר מ-`creative_generation_quota`. הבדיקה **לפני**
> היצירה — מכסה מלאה → חסימה (החלטה 9), לא מתחילים יצירה.

---

## חלק 2 — פרסום כ-Ad רביעי (פעולת Meta)

**זה לא דפוס-3.4 מלא.** 3.4 יוצר Campaign+AdSet+3 Ads מאפס. כאן הקמפיין וה-Ad Set
**כבר קיימים** — רק מוסיפים Ad רביעי.

**`add_ad_to_campaign(campaign_id, image_url, copy_text) -> AdResult`:**
1. שולף את ה-`meta_ad_set_id` של הקמפיj הקיים (מ-`campaigns`).
2. יוצר Ad creative חדש (התמונה + הטקסט הידני).
3. יוצר Ad חדש ב-Ad Set הקיים, ACTIVE.
4. שומר `meta_ad_id` ב-`ads` (שורה רביעית, `status='live'`).

**הבדלים מ-3.4:**
- **אין יצירת Campaign/AdSet** — קיימים.
- **אין 3 Ads** — Ad בודד.
- **rollback פשוט יותר** — אם יצירת ה-Ad נכשלת, אין מה לנקות חוץ מה-Ad עצמו (לא קמפיין
  שלם). אבל עדיין אטומי: או שה-Ad נוצר ונשמר, או כלום.
- **idempotency** — `meta_ad_id` נשמר; אם כבר נוצר, לא ליצור כפול.

> **reuse מ-3.4:** את ה-creative building, ה-gates הרלוונטיים (תמונה תקינה, טוקן תקף),
> ודפוס ה-rollback — מ-`integrations/meta.py` / `optimization_push_service`. לא לשכפל.

> **תלות בבעיה 3 (החלטה 4):** ה-Ad הזה נושא טקסט שהלקוח כתב בלי בדיקה. אם מטא דוחה
> אותו אחרי שעלה — ה-status עובר ל-`rejected` (enum קיים מ-spec §2א), ו-webhook הדחיות
> (7.6) אמור לתפוס. **בלי 7.6, דחייה כזו שקטה.**

---

## חלק 3 — מסלול א' (The Studio): שדרוג תמונה קיימת

**זו היכולת החדשה — image editing (Phase 3.5 שחוזר).**

הזרימה:
1. **העלאה** — הלקוח גורר/בוחר תמונה מהמכשיר (JPG/PNG, מקס' 10MB — מהפרוטוטייפ).
   העלאה ל-Storage (`campaign-images` או bucket ייעודי).
2. **שדרוג** — `gpt-image-2` ב-**edit mode** (לא generation) עם תמונת-המקור + פרומפט
   השדרוג (חלק 3א). מחזיר תמונה משופרת.
3. **שמירה** — התמונה המשופרת נשמרת ל-Storage + רשומה ב-`creative_assets`
   (`asset_type='upgraded'`). **מוצגת בגלריה** (החלטה 6). **לא מתפרסמת אוטומטית.**

> **ספירת מכסה:** צעד 2 סופר מ-`creative_upgrade_quota`. בדיקה לפני — מלא → חסימה.

> **הבדל טכני מהותי מ-3.3:** 3.3 הוא **generation** (טקסט→תמונה). זה **editing**
> (תמונה+הנחיה→תמונה משופרת). `integrations/openai.py` צריך פונקציה חדשה ל-edit mode
> של gpt-image-2. **לאמת שה-API תומך ב-image editing** (gpt-image-2 — לבדוק mode).

### 3א. פרומפט השדרוג (קובץ — חוק 8)

קובץ חדש `prompts/creative/image_upgrade.txt`. **פרומפט קבוע, בלי פרמטרים** (התמונה
היא הקלט, לא הטקסט). הטקסט המלא:

```text
המטרה הכללית

להפוך את התמונה שהועלתה לגרסה הטובה ביותר האפשרית ברמה מסחרית ומקצועית,
תוך שמירה על הנושא, המסר והמבנה המקורי של התמונה.

שיפור איכות התמונה
להעלות את איכות התמונה לרמה של פרסום מקצועי.
להגדיל רזולוציה לאיכות Ultra HD.
לשפר חדות ופרטים קטנים בלי ליצור עיוותים.
לשפר תאורה, טווח דינמי, צללים והארות.
לתקן בעיות חשיפה תוך שמירה על מראה טבעי.
להסיר רעשים, פיקסליזציה, טשטוש ופגמים.
לשפר צבעים, עומק צבע ואיזון גוונים.
לשפר קונטרסט בצורה טבעית.
ליצור תחושת עומק טובה יותר והפרדה בין האובייקט לרקע.
לשפר טקסטורות של עור, שיער, בגדים, מוצרים ורקעים.
לשפר ריאליזם וחדות כללית.
לשפר קווי מתאר ודיוק בפרטים.
לשפר את מוקדי המשיכה בתמונה.
להפוך את התמונה למרשימה ומקצועית יותר.

שמירה על המצולם / המוצר
לא לשנות את זהות האנשים.
לא לשנות מבנה פנים.
לא לשנות הבעות פנים.
לא לשנות פרופורציות גוף.
לא לשנות את המוצר או מאפייניו.
לא לשנות לוגואים ומיתוג.
לשמור על הקומפוזיציה המקורית.
לשמור על המסר השיווקי המקורי.
לשמור על אותנטיות ואמינות.

שמירה על טקסטים
לשמור על כל הטקסט הקיים.
לשפר קריאות של טקסט מטושטש.
לשמור על סגנון הפונט.
לשמור על היררכיה ועיצוב הטקסט.
לא למחוק טקסט חשוב.
לא לעוות טקסטים.

אופטימיזציה לפרסום
לגרום לתמונה להיראות כאילו צולמה בסטודיו מקצועי.
להגביר תחושת אמון וסמכות.
לשפר התאמה לפרסום ומיתוג.
להגדיל את היכולת לעצור גולשים בפיד.
להתאים למודעות פייסבוק.
להתאים למודעות אינסטגרם.
להתאים לדפי נחיתה.
להתאים לאתרי אינטרנט.
להתאים לחומרי פרסום מודפסים.
ליצור תחושת פרימיום ויוקרה כאשר מתאים.

רמת האיכות המבוקשת
צילום מסחרי ברמת פרימיום.
איכות פרסום יוקרתית.
Ultra HD.
פוטוריאליסטי.
תאורת סטודיו מקצועית.
עומק קולנועי.
חדות גבוהה במיוחד.
ריטוש מקצועי.
סטנדרט של מותגים גדולים.
איכות מגזין.
איכות של קמפיין פרסום זוכה פרסים.
מקצועיות מקסימלית.
קריאות מקסימלית.
פוטנציאל המרה מקסימלי.

בקרת איכות
לשמור על מראה טבעי.
לא ליצור מראה מלאכותי.
לא ליצור עור פלסטיק.
לא לעוות גוף או פנים.
לא להגזים ב-HDR.
לא להגזים בצבעים.
לא ליצור ארטיפקטים.
לא לבצע שינויים לא רצויים באובייקט המרכזי.

המטרה הסופית

ליצור את הגרסה המקצועית, החדה, האמינה והמשכנעת ביותר של התמונה,
ברמה שמתאימה לקמפיינים מסחריים ולפרסום בתשלום.
```

---

## חלק 4 — גלריית מוצרים + טבלה

**`creative_assets` (טבלה חדשה):**
```sql
CREATE TABLE public.creative_assets (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  campaign_id uuid REFERENCES public.campaigns(id) ON DELETE SET NULL,  -- nullable: שדרוג לא חייב קמפיין

  asset_type text NOT NULL CHECK (asset_type IN ('generated','upgraded')),
  image_url text NOT NULL,            -- ב-Storage
  source_image_url text,              -- מסלול א' בלבד: תמונת-המקור שהועלתה
  prompt_text text,                   -- מסלול ב': ה-prompt; מסלול א': ריק/הפרומפט הקבוע
  copy_text text,                     -- מסלול ב': הטקסט הידני (אם פורסם)
  published_ad_id text,               -- מסלול ב': meta_ad_id אם פורסם; אחרת NULL
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_creative_assets_user
  ON public.creative_assets (user_id, created_at DESC);
```
- **טבלה אחת לשני הסוגים** (החלטה 7) — `asset_type` מבדיל.
- הגלריה: `SELECT ... WHERE user_id = ? ORDER BY created_at DESC` — מציגה את שניהם.
- **GRANT + RLS** (spec §2א) — `authenticated`, `auth.uid() = user_id`.

> **ספירת מכסה מהטבלה הזו:** המונים נספרים מ-`creative_assets` בחלון anniversary:
> - `creative_generation_quota` → `COUNT WHERE asset_type='generated' AND created_at ∈ [period_start, +1mo)`.
> - `creative_upgrade_quota` → `COUNT WHERE asset_type='upgraded' AND ...`.
> ספירה-חיה (כמו 4.2 ללידים), לא counter. בלי reset.

---

## חלק 5 — מכסות (שני מונים)

**ב-`subscriptions`** (או נגזר מ-`creative_assets` COUNT — ראה למטה):
- `creative_generation_quota` — מסלול ב'. `// מספר מגיא — ראה פתוח`
- `creative_upgrade_quota` — מסלול א'. `// מספר מגיא`

> **שאלת מימוש:** האם המכסות הן עמודות-תקרה ב-`subscriptions` (כמו `lead_quota`),
> או קבועים פר-tier בקוד? נטייה: כמו `lead_quota` — עמודה, נגזרת מ-tier בהרשמה
> (Premium → 15/30). **להכריע ב-CC** לפי דפוס המכסות הקיים.

**אכיפה (החלטה 9 — חסימה):**
- לפני יצירה/שדרוג → בדיקת מונה מול תקרה.
- מלא → **חסימת הפעולה** + הודעה ("הגעת למכסת היצירות החודשית. המכסה תתחדש ב-{תאריך}").
- **לא** "מפסיק להגיב" כמו הבוט — כאן הפעולה עצמה נחסמת לפני שמתחילה.
- התקרה מתחדשת ב-anniversary (`current_period_start + 1mo`), כמו כל המכסות.

---

## חלק 6 — Endpoints

**א. `POST /me/creative/generate`** (מסלול ב', צעד 2) — מקבל prompt → בודק מכסה →
`gpt-image-2` יוצר → שומר `creative_assets` (`generated`) → מחזיר תמונה.

**ב. `POST /me/creative/publish`** (מסלול ב', צעד 4) — מקבל `asset_id` + `copy_text` +
`campaign_id` → `add_ad_to_campaign` (חלק 2) → מעדכן `creative_assets.copy_text`/
`published_ad_id`. דורש את ה-checkbox (החלטה 11) מאומת ב-payload.

**ג. `POST /me/creative/upgrade`** (מסלול א') — מקבל תמונה (multipart upload) → בודק
מכסה → Storage → `gpt-image-2` edit → שומר `creative_assets` (`upgraded`) → מחזיר.

**ד. `GET /me/creative/gallery`** — מחזיר את `creative_assets` של המשתמש (שני הסוגים).

**ה. `GET /me/creative/quota`** — מחזיר את שני המונים + תקרות + תאריך חידוש.

כל ה-endpoints — Premium gating (כמו הבוט).

---

## חלק 7 — שינויים נדרשים בקוד

**א.** `integrations/openai.py` — פונקציה חדשה ל-**image editing** (gpt-image-2 edit
mode), בנוסף ל-generation הקיים. **לאמת שה-API תומך.** (חלק 3)
**ב.** `integrations/meta.py` — `add_ad_to_campaign` (Ad בודד ל-Ad Set קיים). reuse
creative-building מ-3.4. (חלק 2)
**ג.** `services/creative_service.py` — **חדש.** הלוגיקה של שני המסלולים + מכסות.
**ד.** `routers/creative.py` — **חדש.** 5 ה-endpoints (חלק 6), Premium gating.
**ה.** migration — `creative_assets` (חלק 4) + שני מונים ב-`subscriptions` (אם עמודות).
**ו.** `subscription_service.py` — הקצאת מכסות Premium בהרשמה (אם עמודות).
**ז.** `models/` — `CreativeAsset`, `CreativeGenerate`, `CreativePublish`.
**ח.** Storage — לוודא bucket + RLS prefix לתמונות-מקור (מסלול א').

---

## חלק 8 — בדיקות

**מסלול ב' (יצירה + פרסום):**
1. prompt → `gpt-image-2` יוצר → `creative_assets` (`generated`), מכסה נספרת.
2. מכסה מלאה → **חסימה** לפני יצירה, הודעת מכסה.
3. טקסט ידני + checkbox + בחירת קמפיין → Ad רביעי נוצר ב-Ad Set הקיים.
4. checkbox לא מאומת → פרסום נדחה (אישור-אחריות חובה).
5. כשל ביצירת Ad → rollback (אין Ad חצי-בנוי), `creative_assets` נשאר בלי `published_ad_id`.
6. idempotency — פרסום כפול של אותו asset → לא יוצר Ad כפול.

**מסלול א' (שדרוג):**
7. העלאת תמונה → Storage → `gpt-image-2` edit → `creative_assets` (`upgraded`).
8. מכסה מלאה → חסימה לפני שדרוג.
9. התוצאה **לא** מתפרסמת אוטומטית (רק נשמרת בגלריה).
10. קובץ > 10MB / לא JPG/PNG → נדחה לפני העלאה.

**גלריה + מכסות:**
11. גלריה מציגה את שני הסוגים (`generated` + `upgraded`).
12. מונה יצירה סופר רק `generated`; מונה שדרוג רק `upgraded`.
13. מכסות נספרות בחלון anniversary; אחרי חידוש period → אפס.

**Premium gating:**
14. לקוח לא-Premium → אין גישה ל-Creative VIP (403/הסתרה).

**מסלול ג':**
15. אין endpoint/UI לאנימציה/וידאו (מבוטל).

---

## חלק 9 — לא בסשן

- **מסלול ג' (סרטוני אנימציה/וידאו)** — מבוטל. spec §3.
- **בדיקת-מדיניות על הטקסט הידני** — אין (החלטה 4). בעיה 3 (7.6) תופסת דחיות.
- **עריכה מתקדמת** (crop, פילטרים ידניים) — לא. רק שדרוג-AI אוטומטי.
- **שיתוף/הורדה מהגלריה** — אם נדרש, הרחבה עתידית. ב-MVP: תצוגה + פרסום (מסלול ב').

---

## חלק 10 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **שני מסלולים, מטרות שונות.** ב' (יצירה) → מתפרסם לקמפיין. א' (שדרוג) → נשמר בגלריה.
   אל תאחד את ה-flow — הם נפרדים פרט לגלריה המשותפת.

2. **מסלול ב' = generation (reuse 3.3); מסלול א' = editing (חדש, 3.5).** זה ההבדל
   הטכני המהותי. `gpt-image-2` generation כבר קיים; edit mode צריך הוספה + אימות API.

3. **פרסום = Ad רביעי, לא קמפיין חדש.** reuse חלקי מ-3.4 (Ad בודד ל-AdSet קיים). לא
   דפוס-3.4 מלא — אין Campaign/AdSet/3-Ads.

4. **אין בדיקת-מדיניות על הטקסט הידני (החלטה 4) — תלות קריטית בבעיה 3.** הטקסט עולה
   כמו שהוא; אם נדחה, בעיה 3 (7.6) תופסת. **בלי 7.6 — דחייה שקטה.** לשקול env flag
   שחוסם פרסום מסלול ב' עד ש-7.6 קיים.

5. **טבלה אחת `creative_assets` לשני הסוגים** (`asset_type`). גם הגלריה וגם שתי
   ספירות-המכסה מהטבלה הזו. לא טבלאות נפרדות.

6. **חסימה, לא האטה (החלטה 9).** מכסה מלאה → הפעולה נחסמת לפני שמתחילה, עם הודעת
   חידוש. שונה מהבוט (§5) שממשיך לקלוט ורק מפסיק להגיב.

7. **המכסות בפרוטוטייפ לא מעודכנות.** המספרים הסופיים מגיא (יצירה/שדרוג) — ראה פתוח.
   לא לקחת מהצילום (30/50 שם ≠ הסופי).

8. **Premium בלבד.** gating כמו הבוט. לקוח Basic לא רואה את הלשונית.

9. **הפרומפט קובץ, לא קוד (חוק 8).** `prompts/creative/image_upgrade.txt`. פרומפט
   קבוע בלי פרמטרים (התמונה היא הקלט).

10. **Storage — קלט חדש.** מסלול א' מעלה תמונת-מקור מהמכשיר (לא רק שומר פלט כמו 3.3).
    upload endpoint + RLS prefix.

11. **migration** — `creative_assets` + (אולי) שני מונים ב-`subscriptions`. לתאם מספר.

---

## פתוח / ממתין לגיא

1. **מספרי המכסה הסופיים** — יצירה (מסלול ב') ושדרוג (מסלול א') לחודש. (אמיר ציין
   שהמספרים רשומים אצלו — להזין; הפרוטוטייפ מראה 30/50 אבל לא מעודכן.)
2. **בחירת קמפיין לפרסום (מסלול ב')** — אם ללקוח כמה קמפיינים פעילים, הוא בוחר לאיזה
   המודעה מצטרפת. אם קמפיין אחד — אוטומטי. לוודא UX.
3. **env flag לתלות בבעיה 3** — האם לחסום פרסום מסלול ב' עד ש-7.6 קיים? (החלטה 4 הופכת את 7.6 לתנאי-בטיחות.) — החלטת אמיר/גיא.


---

# תשובות מאמיר:
- **מכסות:**
יצירת פוסטים מאפס - 15 בחודש
פרסום לקמפיין - 15;בחודש
שדרוג חומרי לקוח - 30 בחודש

- **בחירת קמפיין לפרסום:**
בינתיים יהיה אפשר לפתוח רק קמפיין אחד.

- **env flag לתלות בבעיה 3:**
לא צריך

---

Session 7.7W

# Session — הגדרות התראות + ערוץ וואטסאפ לבעל-העסק

> בלוק להוספה ל-ROADMAP.md תחת `## Phase 7` או Phase נפרד — לתאם מספור.
> תכנון מלא להעברה ל-CC.
> **שני רכיבים כרוכים:** (1) עמוד הגדרות שבו בעל-העסק בוחר לאיזה ערוץ כל סוג-התראה
> נשלח; (2) ערוץ וואטסאפ-לבעלים חדש, בנוסף למייל הקיים.
> **נשען על תשתית קיימת:** `send_agent_notification` (7.6, ערוץ פרמטרי) + WABA שנסגר
> ממילא לבוט (Phase 5).

---

## מבוא ותיאור Session

ב-7.6 בנינו את `send_agent_notification` עם **ערוץ פרמטרי** (`channel='email'` ב-MVP,
מבני לתמוך ב-`whatsapp`). הסשן הזה **ממש את ה-`whatsapp` branch** + בונה את **עמוד
ההגדרות** שנותן לבעל-העסק שליטה: לאיזה ערוץ כל סוג-התראה הולך.

**ההבחנה שמעצבת את הסיווג** (אושרה מול spec §9): יש שני סוגי התראות-סוכן —
- **התראות שדורשות פעולה + לינק לדשבורד** (מודעות חדשות לאישור, תיקון דחייה) →
  **מתאים למייל** (הלינק פותח את העמוד שבו מאשרים).
- **התראות-תובנה בלי פעולה** (העלות השתפרה, הבעיה נפתרה) → **מתאים לוואטסאפ**
  (קצר, מיידי, אין מה ללחוץ).

הרעיון: **ברירות-מחדל לפי הסיווג הזה, אבל המשתמש שולט** — בעמוד ההגדרות הוא יכול
לשנות לכל סוג לאן יישלח (מייל / וואטסאפ / שניהם).

**הבחנה ארכיטקטונית קריטית — ערוץ הוואטסאפ-לבעלים ≠ קו הבוט:**
- **קו הבוט (`whatsapp_lines`, 5.0):** מספר **פר-לקוח-Premium**, דו-כיווני, מדבר עם
  הלידים בתוך חלון-24h (session messages, בלי template).
- **ערוץ-לבעלים (כאן):** מספר **מרכזי אחד של Campaign AI**, חד-כיווני, שולח התראות
  לכל בעלי-העסקים (Basic + Premium). הודעה **יזומה** (מחוץ לחלון) → **דורשת template
  מאושר**. **לא משתמש ב-`whatsapp_lines` בכלל.**

המטרה: בסוף הסשן, בעל-עסק נכנס ל"הגדרות" → רואה את סוגי-ההתראות עם ברירות-מחדל
(פעולה→מייל, תובנה→וואטסאפ) → משנה לטעמו. וכש-`send_agent_notification` רץ, הוא
שולח לערוץ/ים שהמשתמש בחר לאותו סוג. ערוץ הוואטסאפ נשען על המספר המרכזי + template
התראות, soft-disabled עד אישור מטא.

**מה בסשן:**
- **עמוד הגדרות-התראות** — מטריצת סוג×ערוץ, ברירות-מחדל לפי הסיווג, שמירת העדפות.
- **טבלת העדפות** — `notification_preferences` פר-user.
- **`whatsapp` branch ב-`send_agent_notification`** — שליחה מהמספר המרכזי + template.
- **קריאת ההעדפות** — `send_agent_notification` בודק את העדפת המשתמש לסוג ובוחר ערוץ/ים.
- **soft-disable** — `WHATSAPP_PRODUCTION_READY` חוסם וואטסאפ עד WABA+template מאושרים.

**מה לא בסשן:**
- **WABA + Business Verification** — תשתית מטא, נסגרת ממילא בשביל הבוט (Phase 5). לא
  משוכפלת כאן. ערוץ-לבעלים משתמש באותו WABA.
- **`whatsapp_lines`** — לא רלוונטי (זה קו-פר-ליד של הבוט, לא ערוץ-לבעלים).
- **הגשת ה-template למטא** — פעולה ידנית (admin), כמו templates של 5.4.
- **התראות-מערכת** (קמפיין עלה, חיוב) — אלה ערוץ נפרד (4.6, מייל), לא התראות-סוכן.
  הסשן עוסק בהתראות-הסוכן (`send_agent_notification`), לא בהתראות-המערכת.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מבנה עמוד ההגדרות | **מטריצת סוג-התראה × ערוץ.** לכל סוג, בחירת מייל / וואטסאפ / שניהם |
| 2 | ברירות מחדל | **לפי הסיווג:** פעולה+לינק (אישור מודעות/דחייה) → מייל ✅. תובנה בלי פעולה (שיפור/פתרון) → וואטסאפ ✅. (המשתמש יכול לשנות) |
| 3 | ערוץ וואטסאפ-לבעלים | **מספר מרכזי אחד** של Campaign AI, לא פר-לקוח. שולח לכל בעלי-העסקים (Basic+Premium). **לא `whatsapp_lines`** |
| 4 | תלות ב-WABA | **משותף עם הבוט** (Phase 5). נסגר ממילא. ערוץ-לבעלים משתמש באותו WABA, בלי תשתית נפרדת |
| 5 | template | **template אחד גנרי, UTILITY, נפרד מ-templates הבוט.** הודעה יזומה לבעלים דורשת template מאושר (חוק מטא). פשוט: `"עדכון על הקמפיין שלך: {{1}}"` |
| 6 | על איזו תשתית | **`send_agent_notification` (7.6) — ערוץ פרמטרי.** מוסיפים `whatsapp` branch. לא בונים מנגנון-שליחה שני |
| 7 | soft-disable | **`WHATSAPP_PRODUCTION_READY`** (כמו 5.4). וואטסאפ מושבת עד WABA+template מאושרים. עד אז הכל יוצא במייל. בעמוד ההגדרות — אופציית וואטסאפ אפורה/"בקרוב" |
| 8 | מי מקבל וואטסאפ | **כל בעל-עסק שבוחר זאת** (לא תלוי בחבילה). ערוץ-מערכת גלובלי, לא פיצ'ר-Premium |
| 9 | קריאת ההעדפות | `send_agent_notification` בודק `notification_preferences` לסוג הנוכחי → שולח לערוץ/ים שנבחרו. אין העדפה → ברירת-מחדל (החלטה 2) |
| 10 | שניהם בו-זמנית | **מותר.** משתמש יכול לבחור גם מייל וגם וואטסאפ לאותו סוג → שתי הודעות |

---

## תלויות

1. **7.6 — `send_agent_notification` + 4 הסוגים.** הליבה. הסשן מרחיב אותה ב-`whatsapp`
   branch + קריאת העדפות. **תלות ישירה — 7.6 חייב להיות קיים.**
2. **Phase 5 (הבוט) — WABA + `integrations/meta_whatsapp.py`.** ערוץ-לבעלים משתמש
   באותו WABA ובאותו wrapper-שליחה. **הקוד קיים** (אמיר אישר שכל קוד הבוט מומש);
   החסר הוא תשתית-מטא (WABA פעיל + template), שנסגרת בשביל הבוט ממילא.
3. **5.4 — דפוס templates + `WHATSAPP_PRODUCTION_READY`.** ה-soft-disable וה-template
   על אותו דפוס. template ההתראות הוא נוסף על שלושת ה-templates של הבוט.
4. **4.6 — `send_notification` + Resend.** ערוץ המייל הקיים. נשאר ללא שינוי; הסשן
   מוסיף ערוץ לצידו, לא מחליף.
5. **subscriptions / users** — `notification_preferences` תלוי ב-user. צריך גם את
   **מספר הטלפון של בעל-העסק** (לאן שולחים וואטסאפ) — לאמת שקיים, אחרת לאסוף.

> **תלות קריטית — מספר הטלפון של בעל-העסק.** כדי לשלוח לו וואטסאפ צריך את הטלפון שלו.
> **לאמת מול CC:** האם הטלפון של בעל-העסק נאסף איפשהו (בהרשמה / `billing_profiles`
> /quiz)? אם לא — צריך לאסוף אותו (שדה בהגדרות / אונבורדינג). בלי טלפון, אין וואטסאפ.

---

## חלק 1 — סוגי ההתראות + ברירות המחדל

ארבעת הסוגים מ-7.6 (`send_agent_notification`), עם ברירת-המחדל לפי הסיווג (החלטה 2):

| `notification_type` | תוכן | פעולה? | ברירת מחדל |
|---|---|---|---|
| `ad_rejected` | מודעה נדחתה, הכנתי תיקון לאישור | ✅ לינק לאישור | **מייל** |
| `step_advanced` | הפתרון הקודם לא הספיק, הצעה חדשה לאישור | ✅ לינק לאישור | **מייל** |
| `series_resolved` | הבעיה נפתרה, העלות התייצבה על {X} | ❌ תובנה | **וואטסאפ** |
| `new_lead` | ליד חדש: {שם}, {טלפון} | ❌ עדכון | **וואטסאפ** |

> **ההיגיון:** התראה שדורשת **לחיצה על לינק לדשבורד** (אישור מודעה) → מייל, כי הלינק
> חי שם והעמוד נפתח בנוחות. התראה שהיא **רק ידיעה** (השתפר / ליד חדש) → וואטסאפ, כי
> היא קצרה ומיידית ואין מה ללחוץ. **המשתמש יכול לשנות כל אחד.**

> ב-MVP (לפני אישור template) **כל הסוגים יוצאים במייל** בפועל, גם אם ברירת-המחדל
> וואטסאפ — כי `WHATSAPP_PRODUCTION_READY=false`. ההעדפה נשמרת, רק לא מתממשת עד שהערוץ
> נדלק. (ראה חלק 4.)

---

## חלק 2 — טבלת העדפות

**`notification_preferences` (טבלה חדשה):**
```sql
CREATE TABLE public.notification_preferences (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  notification_type text NOT NULL,   -- ad_rejected / step_advanced / series_resolved / new_lead
  email_enabled boolean NOT NULL DEFAULT true,
  whatsapp_enabled boolean NOT NULL DEFAULT false,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  UNIQUE (user_id, notification_type)
);
```
- שורה פר (user, notification_type). UNIQUE מונע כפילות.
- **שני בוליאנים** (לא ערוץ-יחיד) — כי משתמש יכול לבחור **שניהם** (החלטה 10).
- **GRANT + RLS** (spec §2א): `auth.uid() = user_id`, CRUD למשתמש על שלו.

> **ברירות מחדל — איפה נקבעות?** לא בעמודות ה-DB (שם `email=true, whatsapp=false` גנרי),
> אלא **בלוגיקה**: כשמשתמש לא הגדיר העדפה לסוג, הקוד נופל לברירת-המחדל של החלטה 2
> (לפי טבלת חלק 1). אפשרות מימוש: seed שורות בברירות-המחדל הנכונות בהרשמה, **או**
> פונקציית `get_preference(user, type)` שמחזירה את העדפת-המשתמש או את ברירת-המחדל
> אם אין. **נטייה: פונקציה עם fallback** — לא צריך seed, וברירות-המחדל חיות במקום אחד
> בקוד. להכריע ב-CC.

---

## חלק 3 — `whatsapp` branch ב-`send_agent_notification`

הרחבת `send_agent_notification` (7.6) — מימוש הענף שהוכן מראש.

**הזרימה המעודכנת:**
```
send_agent_notification(user_id, notification_type, context):
  1. שלוף העדפת המשתמש לסוג (get_preference → email_enabled, whatsapp_enabled)
     # אין העדפה → ברירת-מחדל לפי החלטה 2
  2. אם email_enabled:
       send via Resend (4.6) — הקוד הקיים, ללא שינוי
  3. אם whatsapp_enabled AND WHATSAPP_PRODUCTION_READY:
       send via WhatsApp (חדש — מספר מרכזי + template)
  4. idempotency פר-ערוץ: sent_notifications עם (type, target_id, channel)
     # אותו אירוע יכול לצאת בשני ערוצים → שתי רשומות idempotency, אחת לכל ערוץ
```

**שליחת הוואטסאפ (החדש):**
- משתמש ב-`integrations/meta_whatsapp.py` (קיים מהבוט) — **אבל מהמספר המרכזי**, לא
  מ-`whatsapp_lines` של הלקוח.
- שולח **template** (`agent_notification_v1`, UTILITY) עם משתנה אחד = תוכן ההתראה
  בעברית.
- יעד: **הטלפון של בעל-העסק** (ראה תלות 5 — לאמת מאיפה).
- ה-deep-link (לסוגים שדורשים פעולה) — בוואטסאפ נכנס כחלק מהטקסט/משתנה ה-template.

> **למה template ולא טקסט חופשי:** התראה לבעל-העסק היא הודעה **יזומה** (הוא לא שלח
> לנו) → מחוץ לחלון 24h → מטא דורשת template מאושר. זה שונה מהבוט שמגיב ללידים בתוך
> החלון (session message, בלי template). לכן template נפרד.

> **`new_lead` ובחלון 24h:** אם בעל-העסק שלח הודעה למספר המרכזי לאחרונה (פתח חלון),
> אפשר טקסט חופשי. אבל אי אפשר להסתמך על זה — בעל-עסק לא מתכתב עם המספר המרכזי. אז
> **תמיד template** להתראות-לבעלים. פשוט ובטוח.

---

## חלק 4 — soft-disable (`WHATSAPP_PRODUCTION_READY`)

אותו דפוס כמו 5.4 (אם כבר קיים שם — reuse אותו flag, לא חדש).

**ב-`app_settings` (או env):** `WHATSAPP_PRODUCTION_READY` (boolean, default false).

**ההתנהגות:**
- **`false`** (MVP, עד אישור מטא): גם אם `whatsapp_enabled=true` בהעדפות — הוואטסאפ
  **לא נשלח**. ההתראה יוצאת במייל בלבד (אם `email_enabled`), או לא יוצאת אם המשתמש
  כיבה מייל (edge case — ראה למטה). בעמוד ההגדרות, אופציית הוואטסאפ **אפורה/"בקרוב"**.
- **`true`** (אחרי WABA+template מאושרים): הוואטסאפ נשלח לפי ההעדפות. אופציית הוואטסאפ
  בעמוד פעילה.

> **edge case — משתמש שכיבה מייל ובחר רק וואטסאפ, כשהוואטסאפ מושבת:** ההתראה לא תצא
> בכלל. **פתרון:** כשהוואטסאפ מושבת, עמוד-ההגדרות לא מאפשר לכבות מייל אם וואטסאפ הוא
> הערוץ היחיד. או: fallback — אם שום ערוץ פעיל בפועל, שלח מייל בכל זאת (מייל = רשת
> ביטחון). **נטייה: fallback למייל** — עדיף התראה שמגיעה בערוץ לא-מועדף מאשר התראה
> שנעלמת. להכריע.

> **כשהבוט נדלק (5.4 `WHATSAPP_PRODUCTION_READY=true`) — הערוץ-לבעלים נדלק יחד.** אם
> זה אותו flag, אז ברגע שמטא מאשרת את ה-WABA וה-templates (גם של הבוט וגם
> `agent_notification_v1`), שניהם פעילים. אם רוצים הפרדה (הבוט פעיל אבל ערוץ-לבעלים
> עוד לא) — flag נפרד `AGENT_WHATSAPP_READY`. **נטייה: flag אחד** (פשוט), אלא אם יש
> סיבה להפריד. להכריע.

---

## חלק 5 — עמוד הגדרות-התראות (UI)

עמוד "הגדרות" (לשונית חדשה / חלק מלשונית קיימת).

**התצוגה — מטריצה:**
לכל סוג-התראה (4 שורות), שני checkboxes: מייל | וואטסאפ.
```
                                          מייל    וואטסאפ
מודעות חדשות לאישור (ad_rejected)          ✅       ⬜ (אפור/בקרוב)
הצעה חדשה לאישור (step_advanced)           ✅       ⬜
הבעיה נפתרה (series_resolved)              ⬜       ✅ (אפור/בקרוב)
ליד חדש (new_lead)                        ⬜       ✅ (אפור/בקרוב)
```
- ✅/⬜ מאותחלים לברירות-המחדל (החלטה 2).
- **כל שינוי נשמר** (PATCH ל-`notification_preferences`).
- כשהוואטסאפ מושבת (`WHATSAPP_PRODUCTION_READY=false`) — עמודת הוואטסאפ **אפורה**
  עם tooltip "בקרוב" (ההעדפה נשמרת אבל לא פעילה).
- **שדה טלפון** (אם לא נאסף במקום אחר — תלות 5): "מספר וואטסאפ לקבלת התראות".

**Endpoints:**
- `GET /me/notification-preferences` — מחזיר את ההעדפות (או ברירות-מחדל אם אין).
- `PATCH /me/notification-preferences` — מעדכן סוג מסוים (`email_enabled`/`whatsapp_enabled`).

> ה-UI לא נבנה במלואו כאן (כמו שאר הסשנים) — endpoints + לוגיקה. מטריצה פשוטה
> מספיקה.

---

## חלק 6 — שינויים נדרשים בקוד

**א.** `notification_service.py` (7.6) — מימוש `whatsapp` branch ב-`send_agent_notification`
+ קריאת העדפות (`get_preference` עם fallback) + idempotency פר-ערוץ (חלק 3).
**ב.** `integrations/meta_whatsapp.py` (קיים) — פונקציית שליחת-template מהמספר המרכזי
(אם לא קיימת; ייתכן שיש מהבוט — reuse).
**ג.** migration — `notification_preferences` (חלק 2).
**ד.** `routers/` — `GET`/`PATCH /me/notification-preferences` (חלק 5). אם אין טלפון
בעל-עסק — גם שדה לאיסוף.
**ה.** `app_settings`/config — `WHATSAPP_PRODUCTION_READY` (אם לא קיים מ-5.4) +
אופציונלית `AGENT_WHATSAPP_READY` (חלק 4).
**ו.** template `agent_notification_v1` — להגיש למטא (ידני, admin). UTILITY, משתנה אחד.
**ז.** `models/` — `NotificationPreference`.

> **השינוי האמיתי קטן:** רוב התשתית קיימת (`send_agent_notification` מ-7.6,
> `meta_whatsapp` מהבוט, ערוץ מייל מ-4.6). הסשן בעיקר **מחבר** — מימוש branch שהוכן,
> טבלת העדפות, ועמוד שליטה. לא בונה מערכת-התראות חדשה.

---

## חלק 7 — בדיקות

**העדפות:**
1. משתמש בלי העדפה → ברירת-מחדל (ad_rejected→מייל, series_resolved→וואטסאפ).
2. PATCH משנה ad_rejected לוואטסאפ → נשמר, נקרא בהתראה הבאה.
3. משתמש בוחר גם מייל וגם וואטסאפ לסוג → שתי הודעות (החלטה 10).

**`send_agent_notification` עם ערוצים:**
4. `email_enabled=true` → מייל נשלח (Resend, כמו 7.6).
5. `whatsapp_enabled=true` + `WHATSAPP_PRODUCTION_READY=true` → template נשלח מהמספר
   המרכזי.
6. `whatsapp_enabled=true` + `WHATSAPP_PRODUCTION_READY=false` → וואטסאפ **לא** נשלח,
   מייל כן (אם enabled).
7. idempotency פר-ערוץ → אותו אירוע, שני ערוצים → שתי רשומות `sent_notifications`,
   כל ערוץ פעם אחת.

**soft-disable:**
8. `WHATSAPP_PRODUCTION_READY=false` → עמודת וואטסאפ אפורה בעמוד.
9. edge case: רק וואטסאפ enabled + מושבת → fallback למייל (לא נעלם).

**וואטסאפ-לבעלים ≠ קו-בוט:**
10. התראה לבעלים יוצאת מהמספר המרכזי, **לא** מ-`whatsapp_lines` של הלקוח (בדיקת
    ארכיטקטורה).
11. בעל-עסק **Basic** (בלי בוט) → כן מקבל וואטסאפ-התראות (ערוץ גלובלי, החלטה 8).

**טלפון:**
12. אין טלפון לבעל-עסק → וואטסאפ לא נשלח + הנחיה לעדכן (לא קריסה).

---

## חלק 8 — לא בסשן

- **WABA + Business Verification** — נסגר בשביל הבוט (Phase 5). לא כאן.
- **`whatsapp_lines`** — קו-בוט פר-ליד. לא רלוונטי לערוץ-לבעלים.
- **התראות-מערכת** (קמפיין עלה / חיוב / מכסה) — ערוץ 4.6, מייל. לא התראות-סוכן.
  (אם רוצים גם אותן בוואטסאפ — הרחבה עתידית נפרדת.)
- **`agent_alerts_quota`** — אם ההתראות-לבעלים נספרות במכסה (spec §6 מזכיר). **לבדוק
  מול גיא** אם יש מכסת-התראות, או שהן ללא הגבלה. (ב-spec יש `agent_alerts_quota`
  "ממתין למספרים מגיא".)
- **בחירת ערוץ פר-קמפיין** (לא רק פר-סוג) — לא ב-MVP. ההעדפה גלובלית פר-user.

---

## חלק 9 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **ערוץ-לבעלים ≠ קו-בוט.** מספר מרכזי אחד (לא `whatsapp_lines`), חד-כיווני, לכל
   הלקוחות. אם מוצאים את עצמנו מקצים קו פר-לקוח להתראות — טעות. זה ערוץ-מערכת גלובלי.

2. **template חובה (חוק מטא).** התראה לבעלים = הודעה יזומה = template מאושר. לא טקסט
   חופשי. template נפרד מ-templates הבוט (`agent_notification_v1`, UTILITY).

3. **בנוי על 7.6, לא חדש.** `send_agent_notification` כבר פרמטרי. הסשן מממש את
   ה-`whatsapp` branch שהוכן + מוסיף שכבת-העדפות. לא מנגנון-שליחה שני.

4. **ברירות-מחדל בלוגיקה, לא ב-DB.** פעולה→מייל, תובנה→וואטסאפ (החלטה 2). fallback
   כשאין העדפת-משתמש. לא seed קשיח.

5. **soft-disable כמו 5.4.** וואטסאפ מושבת עד WABA+template. ההעדפה נשמרת, לא מתממשת.
   עמודת וואטסאפ אפורה. כנראה אותו flag כמו הבוט (נדלקים יחד).

6. **fallback למייל ב-edge case.** משתמש שבחר רק וואטסאפ כשהוא מושבת → מייל בכל זאת.
   עדיף התראה לא-מועדפת מהתראה שנעלמת.

7. **idempotency פר-ערוץ.** אותו אירוע בשני ערוצים = שתי רשומות `sent_notifications`
   (type, target, channel). לא לבלבל עם idempotency פר-אירוע של 7.6.

8. **טלפון בעל-העסק — תלות.** לאמת מאיפה (הרשמה/quiz/billing_profiles). אם אין —
   לאסוף. בלי טלפון אין וואטסאפ.

9. **Basic מקבל וואטסאפ-התראות.** הערוץ גלובלי, לא Premium. לא לבלבל עם הבוט
   (שהוא Premium).

10. **Sentry context** — בשליחת וואטסאפ: `notification_type`, `user_id`, ערוץ, האם
    template נשלח. כשל template → Sentry (לא להפיל את המייל המקביל).

---

::: question
## פתוח / ממתין

> **✅ נסגר ומומש (ערוץ עדכוני-הסוכן בוואטסאפ — מוקדם, לפני שאר Phase 8):** migration `0100`
> (`agent_whatsapp_enabled` + RPC אטומי `create_agent_notification`), handler `_send_agent_whatsapp`
> (סימטרי ל-VIP, gate משותף flag-נפרד), endpoints `PATCH/GET /me/subscription/agent-whatsapp[/quota]`,
> toggle ב"החבילה שלי". 4 השאלות הוכרעו:

1. **טלפון בעל-העסק** — ✅ **משותף עם VIP** (`vip_alert_phone`); הבעלים מזין מספר אחד ב-toggle (אם VIP פעיל עם טלפון — לא מבקש שוב, `phone=null` והשרת לא דורס).
2. **`agent_alerts_quota`** — ✅ **כן, 30/חודש** (החלטת בעל-המוצר). ספירה אטומית ב-RPC (`FOR UPDATE` על שורת המנוי, `status in (pending,sent)`, חלון anniversary, trial→כל-ההיסטוריה); מעבר למכסה — **מייל בלבד** (רצפת-ביטחון).
3. **flag אחד או שניים** — ✅ **gate משותף, alert-flag נפרד**: `whatsapp_production_ready` משותף עם הבוט+VIP; `agent_notify_not_configured_alerted` נפרד מ-VIP כדי שה-edge-trigger לא יידרס.
4. **template — נוסח סופי** — ✅ **`agent_update_owner`** (UTILITY, 2 משתנים): "עדכון מהסוכן בקמפיין שלך / {{1}} / לצפייה: {{2}}"; `{{1}}` תקציר פר-type (4 סוגים), `{{2}}` deep-link (parity עם המייל). אושר מוצרית.

:::

---

## Phase 8 · ניטור רציף + אופטימיזציה אוטונומית

> **החלק הכי מסוכן בפרויקט.** הסוכן מבצע שינויים על קמפיין חי עם כסף אמיתי. בא אחרי שהליבה (3.4), הלידים (4), Insights (4.5) והסוכן (7) מוכחים. תלוי בהגנות מ-spec 2א חוק 6.

### Session 8.1 — cron ניטור יזום (high_cpl) ✅ (בוצע — high_cpl בלבד)
> **מומש (Session 8.1):** טריגר-רקע יומי (`daily_high_cpl_tick` ב-`worker/runner.py`) שסורק את כל הקמפיינים
> החיים (`campaign_service.fetch_active_campaigns`, live בלבד) ומריץ את **אותו גרעין-אבחון** של הצ'אט
> (`agent_orchestrator.assess_high_cpl` — חילוץ משותף, מונע drift בין הצ'אט לרקע). קמפיין `expensive` →
> פותח סדרה (אידמפוטנטי) + מכין 3 קופי + 3 תמונות (`generate_solution`), **עוצר ב-`push_status=NULL`**
> (מוכן-לאישור), ושולח התראת `PROPOSAL_READY` (מייל תמיד + וואטסאפ מותנה, כמו שאר עדכוני-הסוכן).
> **אפס פעולה על Meta בלי אישור אנושי** — מכאן המסלול (approve→push→Meta→ניטור-120ש') זהה לחלוטין לצ'אט.
> כל השערים (Lock=חלון-5-ימים, benchmark=חוק-הברזל, reserve-first) חלים על הרקע בלי שינוי; הרקע נוגע **רק**
> ב-`expensive`. פרטים: `docs/proactive-high-cpl-monitor-plan.md` + `docs/high-cpl-flow.md §14`.

- [x] cron יומי (`daily_high_cpl_tick`) שסורק כל קמפיין `live` ומריץ `run_high_cpl_scan` פר-קמפיין (בידוד-כשל)
- [x] זיהוי CPL גבוה דרך הגרעין המשותף (`assess_high_cpl`); רק `expensive` → פעולה (amazing/average/unknown/locked/no_data → skip שקט)
- [x] הכנת פתרון (3 קופי + תמונות) ל-**אישור** — עוצר ב-`push_status=NULL`, מתריע `PROPOSAL_READY` (לא עולה לבד)
- [x] הגנות: peek לפני יצירה (offer_change/מוצו-שלבים → skip), pending-guard (בלי regen יומי), idempotency ברמת-action
- [ ] **מחוץ לסקופ הסריקה (מכוון):** מודעה שנדחתה (`rejected`) = webhook (בעיה 3, לא cron); ירידת-לידים = בעיה 4 (מבוססת-תלונה). רק high_cpl מדיד-מספרית לזיהוי-עצמאי — ראה `proactive-high-cpl-monitor-plan.md §6`

### Session 8.2 — מנוע הפרוטוקול (דטרמיניסטי)
- [ ] לוגיקה שקובעת איזה שלב אופטימיזציה הבא לפי `optimization_actions`
- [ ] נעילת 4 ימים (`window_ends_at`)

### Session 8.3 — ביצוע אוטונומי + הגנות
- [ ] ביצוע פעולות הקטגוריה האוטונומית מול Meta API
- [ ] תקרת שינוי + לוג מלא ב-`optimization_actions`
- [ ] התראה ללקוח בדשבורד על כל פעולה (סיבה + תוצאה)
- [ ] הסוכן שולח התראת תובנה לוואטסאפ של הבעלים על כל פעולה אוטונומית (`agent_alerts_quota`) — **ה-ערוץ מומש** (opt-in `agent_whatsapp_enabled`, 30/חו׳, רדום עד WABA חי); 4 העדכונים הקיימים (7.6) כבר נשלחים גם לוואטסאפ. נשאר רק ה-trigger על הפעולות האוטונומיות של 8.3

**Done:** הסוכן מנטר רציף, מזהה בעיות, ומבצע אופטימיזציות אוטונומיות בטוחות עם לוג מלא.
**זהירות (CLAUDE.md):** autonomous-action patterns — באג שורף כסף לקוח. תקרות + לוג מלא הם חובה, לא רשות.

---

## תשתיות רוחביות (מלוות את כל ה-Phases)

### סביבת סימולציה לבדיקות
- [ ] עסקים/קמפיינים מומצאים + תרחישים, להרצת ה-flow בלי פייסבוק אמיתי
- [ ] בדיקת החלטות הסוכן על תרחישים בנויים (לפני חיבור ל-Insights אמיתי)
- [ ] נדרשת במיוחד לפני Phase 8 (שחרור האוטונומיה)

> מה היא בודקת: את כל ה-flow מקצה לקצה בלי פייסבוק אמיתי + את השכבה
> הדטרמיניסטית (פרוטוקול, נעילת 4 ימים, ספירת מכסה, תקרת שינוי) + תגובת
> הסוכן לתרחישים בנויים. מה היא **לא** בודקת: איכות השיפוט האוטונומי על
> דאטה אמיתית — את זה רק קמפיין חי נותן (Phase 8).

### Sandbox פנימי לבחינת הסוכן (כלי-פיתוח, לא ממשק לקוח)
- [ ] יצירת קמפיין-דמה + הזנה ידנית של מספרים (CPL, לידים, CTR)
- [ ] הסוכן האמיתי רץ מולו: ניתוח, צ'אט, פעולות אוטונומיות
- [ ] פעולות נכתבות ל-`optimization_actions` (לא ל-Meta) → הפעולה הבאה מתבססת עליהן
- [ ] נקודת-החלפה אחת ב-`integrations/meta.py` (מקור נתונים + יעד פעולות), לא קוד-סוכן נפרד

**Done:** אתה וגיא יוצרים קמפיין-דמה, מקלידים מספרים, והסוכן האמיתי מגיב ומבצע — הכל נשמר ב-DB, בלי לגעת במטא.

---

## אבני דרך מרכזיות

| אבן דרך | מתי | למה |
|---------|-----|-----|
| הצינור עובד | אחרי 0.2 | repo→Render→Supabase חי |
| MVP מינימלי מוכח | אחרי 3.4 | מודעות עולות לאוויר — הימור הליבה |
| לולאת לידים סגורה | אחרי 4.2 | ליד נכנס ונספר במכסה |
| בוט עובד | אחרי 5.3 | מודל ה-Premium מוכח |
| סוכן צ'אט עובד | אחרי 7.1 | "מנהל, לא מחולל" — ייעוץ וניתוח |
| אופטימיזציה אוטונומית | אחרי 8.3 | הסוכן מנהל קמפיין חי לבד |

---

# Sandbox — כלי פיתוח לבחינת הסוכן (תשתית רוחבית)

> בלוק להוספה ל-ROADMAP.md תחת `## תשתיות רוחביות`. תכנון מלא להעברה ל-CC.
> **לא מספר Phase** — כלי-פיתוח שחוצה את כל ה-Phases. נדרש **לפני Phase 8** (שחרור האוטונומיה): אי אפשר לשחרר אוטונומיה בלי לראות קודם שהסוכן מחליט נכון על תרחישים מבוקרים.
> **כלי פנימי — לא ממשק לקוח.** מאחורי הרשאת אדמין, רק לאמיר וגיא.

> **סטטוס מימוש (מתעדכן):**
> - ✅ **PR-S1** — תשתית seam: `MetaClient` (Protocol, 7 פעולות) + `RealMetaClient` (adapter דק) + factory (`app/services/meta_client.py` + `meta_client_factory.py`). אפס שינוי התנהגות.
> - ✅ **PR-S2** — `SandboxMetaClient` (`app/services/sandbox_meta_client.py`: insights מ-`sandbox_inputs`, push no-op שמחזיר `sandbox_*`) + ענף `is_sandbox` ב-factory (`campaign_service.fetch_is_sandbox`; שורה חסרה→False, DB-error→propagate) + migration `0101` (`campaigns.is_sandbox` + טבלת `sandbox_inputs` single-row, admin-only RLS) + guard `_gate_not_sandbox` לנתיבי בעיה-2/3 שעדיין עוקפים את ה-seam. טסטים: `tests/test_sandbox.py` + fixture `_default_not_sandbox` ב-conftest. **תומך בבעיה-1 (אופטימיזציית CPL) מקצה-לקצה; הסוכן עיוור — אפס `if is_sandbox` בקוד-הסוכן.**
> - ✅ **PR-S3a** — הרחבת ה-seam ל-lead_form/rejection (בעיה-2/3): `create_lead_form`/`subscribe_page_to_leadgen` ל-Protocol + המרת `push_screening`/`push_rejection_fix` ל-client מוזרק → הסרת ה-guards. **כל 3 נתיבי-הסוכן על ה-seam.**
> - ✅ **PR-S3b-1** — backend אדמין: `sandbox_service` (create_sandbox_campaign + run_diagnose/propose/approve + advance_time[→run_tick האמיתי] + update_inputs, `_assert_sandbox_campaign` בראש כל wrapper) + `routers/admin/sandbox.py` (מאחורי `require_admin_token`) + `models/sandbox.py`. ה-endpoints קוראים לזרימה האמיתית; ה-seam מזריק SandboxMetaClient לפי is_sandbox (אפס `if is_sandbox`). טסטים: `tests/test_sandbox_admin.py` (27).
> - ✅ **PR-S3b-2** — עמוד HTML/UI (`templates/admin/sandbox.html` + `GET /admin/sandbox` ב-`page_router` לא-מוגן): טופס יצירת-קמפיין + הזנת-מספרים + כפתורי diagnose/propose/approve/advance-time מחווטים ל-6 ה-endpoints, token ב-localStorage, כל פלט דרך `textContent` (XSS-safe, כלל קריטי 4). **מסלול הסנדבוקס הושלם.** נותר ידני בלבד: אימות e2e חי (admin token + Supabase) + לוודא ש-`_SANDBOX_IMAGE_URL` נגיש ל-`_download_image` בהרצה חיה (שקול placeholder same-origin).

---

::: info

# 2 בדיקות מקדימות למימוש הסשן

**1. ריכוז קריאות למטא**
האם כל קריאות ה-Meta בקוד עוברות דרך integrations/meta.py, או שיש קריאות ישירות ב-services?

**2. ריכוז קריאות ה-LLM** 
"בדיוק כמו Meta: הסנדבוקס עובד רק אם כל גישת LLM דרך שכבה אחת. קריאות ישירות מפוזרות = התוספת דולפת. לרכז קודם.

:::

---

## מבוא ותיאור Session

בנינו את כל הפרוטוקול של הסוכן (7.4–7.6) — חוק הברזל, Lock, מנוע השלבים, מחזור 120 השעות, יצירת קופי, פעולות Meta. אבל **אין שום דרך לראות אם זה עובד** בלי להעלות קמפיין אמיתי עם כסף אמיתי ולחכות ימים. הסנדבוקס פותר את זה: כלי שבו אמיר וגיא יוצרים קמפיין-דמה, מקלידים מספרים ביד (CPL, לידים, אחוז לחיצה), ורואים את **הסוכן האמיתי** מגיב בזמן אמת — מאבחן, עוצר/ממשיך, יוצר קופי, "מבצע" — והכול נשמר ב-DB בלי לגעת ב-Meta.

**העיקרון הארכיטקטוני המרכזי (וזה כל הסשן):**
הפיתוי הוא לבנות "גרסת בדיקה" של הסוכן, או לפזר `if sandbox_mode:` ברחבי הקוד. **שתי הגישות האלה הן טלאי** — כי אז הקוד שרץ בסנדבוקס שונה מהפרודקשן, ואתה בודק משהו אחר ממה שיעבוד באמת. הפתרון השורשי: **נקודת החלפה אחת**, דרך dependency injection. `integrations/meta.py` הופך לממשק (interface) עם שני מימושים — אמיתי (Meta API) וסנדבוקס (DB בלבד). הסוכן מקבל "client" ולא יודע איזה מהם הוזרק. **אפס `if` בקוד-הסוכן.**

המשמעות: ה-orchestrator, חוק הברזל, ה-Lock, מנוע השלבים, יצירת הקופי (LLM **אמיתי**!), כתיבת `optimization_actions` — כולם רצים בדיוק כמו בפרודקשן. רק שכבת ה-Meta מוחלפת: מקור הנתונים (במקום Insights → המספרים שהוקלדו) ויעד הפעולות (במקום Meta API → no-op שכותב ל-DB).

המטרה: בסוף הסשן, אמיר/גיא נכנסים לעמוד אדמין, יוצרים קמפיין-דמה, מקלידים "CPL ₪65, ענף קוסמטיקה", לוחצים "הרץ אבחון" → הסוכן האמיתי מחזיר את האבחון של בעיה 1 → מקלידים "דמה שעברו 120 שעות" → ה-cron האמיתי מודד ומחליט — הכול בלי Meta, הכול נשמר ב-DB.

**מה בסשן:**
- ממשק `MetaClient` (abstract) + שני מימושים: `RealMetaClient`, `SandboxMetaClient`.
- dependency injection — הזרקת ה-client הנכון לפי הקשר (קמפיין-דמה → סנדבוקס).
- עמוד אדמין: יצירת קמפיין-דמה, טופס הזנת מספרים, כפתור הקפצת-זמן.
- `SandboxMetaClient`: קלט (מספרים מוקלדים) + פלט (no-op + כתיבה ל-DB).
- כפתור "דמה 120 שעות" — מריץ את `monitor_optimization_windows` ידנית על הקמפיין-דמה.

**מה לא בסשן:**
- בדיקת **איכות** השיפוט האוטונומי על דאטה אמיתית — את זה רק קמפיין חי נותן (Phase 8). הסנדבוקס בודק שהפרוטוקול **רץ נכון**, לא שההחלטות **נכונות עסקית** על נתוני אמת.
- סימולציית CI אוטומטית על תרחישים בנויים (זה פריט נפרד ב"סביבת סימולציה"). הסנדבוקס הוא אינטראקטיבי-ידני.
- חיבור Meta אמיתי כלשהו.
- ממשק ללקוחות — אדמין בלבד.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | איך מחליפים את Meta | **ממשק + DI, לא `if sandbox`.** `MetaClient` abstract, שני מימושים. הסוכן מקבל client מוזרק, לא יודע איזה. **אפס `if` בקוד-הסוכן** — שורש, לא טלאי |
| 2 | מה רץ אמיתי | **הכול פרט ל-Meta.** orchestrator, חוק ברזל, Lock, מנוע שלבים, **יצירת קופי (LLM אמיתי)**, כתיבת `optimization_actions`. רק קלט/פלט Meta מוחלף |
| 3 | מה מזינים ביד | **טופס גמיש אחד** לכל 3 הבעיות: CPL, לידים, אחוז לחיצה, מחיר לקליק (בעיה 1); + לידים עם `screening_answers` ותיוג (בעיה 2); + סטטוס מודעה + סיבת דחייה (בעיה 3) |
| 4 | איך מקפיצים זמן | **כפתור "דמה 120 שעות"** שמריץ את `monitor_optimization_windows` ידנית על הקמפיין-דמה (או מזיז `window_ends_at` לעבר). אין המתנה אמיתית |
| 5 | נגישות | **אדמין בלבד.** flag/הרשאה. לא ללקוחות. קמפיין-דמה מסומן `is_sandbox=true` |
| 6 | פעולות הסוכן | **נכתבות ל-DB (`optimization_actions`), לא ל-Meta.** הפעולה הבאה מתבססת עליהן — בדיוק כמו בפרודקשן. רק ההעלאה ל-Meta היא no-op |

---

## תלויות

1. **7.4–7.6** — כל הסוכן. הסנדבוקס מריץ אותו כמות שהוא.
2. **`integrations/meta.py`** — קיים (כל קריאות Meta עוברות דרכו). הסנדבוקס הופך אותו לממשק. **זו נקודת ההחלפה היחידה.**
3. **7.3.5** — `optimization_sessions`/`optimization_actions` (הפעולות נכתבות לשם).
4. **0.5** — auth + הרשאות (לגישת אדמין).

> **תלות קריטית בארכיטקטורה הקיימת:** הסנדבוקס עובד **רק אם** כל קריאות Meta כבר עוברות דרך `integrations/meta.py` ולא מפוזרות בקוד. אם יש קריאות Meta ישירות ב-services (לא דרך השכבה) — צריך לרכז אותן קודם. **לאמת מול CC שכל גישת Meta מרוכזת.**

---

## חלק 1 — ממשק `MetaClient` (הלב הארכיטקטוני)

הפיכת `integrations/meta.py` לממשק עם שני מימושים.

**`MetaClient` (abstract base / Protocol):**
מגדיר את כל הפעולות שהסוכן עושה מול Meta. לפחות:
- `get_campaign_insights(campaign_id) -> Insights` — קלט (CPL, CTR, לידים...).
- `get_ad_status(ad_id) -> AdStatus` — סטטוס מודעה (לבעיה 3).
- `update_ad_creative(ad_id, creative) -> Result` — עדכון creative (7.4ב).
- `create_lead_form(...) -> FormResult` — טופס ליד (7.5).
- `relink_ads_to_form(...)`, `subscribe_webhook(...)` — (7.5).
- כל פעולת Meta אחרת שהסוכן קורא לה.

**`RealMetaClient(MetaClient):`**
המימוש הקיים — קורא ל-Meta Graph/Marketing API. (הקוד הנוכחי של `integrations/meta.py` הופך לזה.)

**`SandboxMetaClient(MetaClient):`**
מימוש חדש:
- **קלט** (`get_*`): מחזיר את המספרים מהקמפיין-דמה (מטבלת `sandbox_inputs`, חלק 3) במקום מ-Meta.
- **פלט** (`update_*`, `create_*`): **no-op** מול Meta — מחזיר "הצלחה" מזויפת עם `meta_*_id` דמה, ו**כותב ל-DB** את מה שהיה אמור לקרות (לתצוגה). לא נוגע ב-Meta.

> **זה כל הקסם.** הסוכן קורא `client.update_ad_creative(...)`. בפרודקשן זה מעלה ל-Meta; בסנדבוקס זה no-op שמסמן הצלחה. הסוכן זהה לחלוטין בשני המקרים.

---

## חלק 2 — Dependency Injection

איך ה-client הנכון מגיע לסוכן בלי `if` בקוד-הסוכן.

**נקודת ההזרקה:** במקום שבו services יוצרים/מקבלים את ה-Meta client (כנראה factory או dependency ב-FastAPI).

**הלוגיקה:**
- אם הקמפיין מסומן `is_sandbox=true` → הזרק `SandboxMetaClient`.
- אחרת → `RealMetaClient`.

**מימוש מומלץ (FastAPI dependency):**
```python
def get_meta_client(campaign_id: UUID) -> MetaClient:
    campaign = get_campaign(campaign_id)
    if campaign.is_sandbox:
        return SandboxMetaClient(campaign_id)
    return RealMetaClient(campaign_id)
```
ה-services וה-orchestrator מקבלים `MetaClient` כפרמטר/dependency — **לא** קוראים ל-factory בעצמם, ולא בודקים `is_sandbox`.

> **ה-`if` היחיד בכל המערכת** הוא ב-`get_meta_client` הזה. הוא לא בקוד-הסוכן — הוא בשכבת ההזרקה. קוד-הסוכן מקבל client ומשתמש בו עיוור. זה ההבדל בין שורש (החלפה אחת) לטלאי (`if sandbox` מפוזר).

---

## חלק 3 — Migration: סימון סנדבוקס + קלט

קובץ חדש (מספר רץ הבא).

**א. `campaigns.is_sandbox`:**
```sql
ALTER TABLE public.campaigns
  ADD COLUMN IF NOT EXISTS is_sandbox boolean NOT NULL DEFAULT false;
```
- קמפיין-דמה מסומן `true`. כל השאר `false` (ברירת מחדל — בטוח).

**ב. `sandbox_inputs` — המספרים המוקלדים:**
```sql
CREATE TABLE public.sandbox_inputs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id uuid NOT NULL REFERENCES public.campaigns(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  -- מטריקות בעיה 1
  cpl numeric,
  leads_count integer,
  click_rate numeric,        -- אחוז לחיצה
  cost_per_click numeric,

  -- בעיה 3
  ad_status text,            -- 'ACTIVE' / 'DISAPPROVED'
  rejection_reason text,

  -- timestamp לגרסת הקלט (אפשר לעדכן מספרים ולהריץ שוב)
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_sandbox_inputs_campaign
  ON public.sandbox_inputs (campaign_id, created_at DESC);
```
- `SandboxMetaClient.get_campaign_insights` שולף את השורה האחרונה לקמפיין.
- לידים עם `screening_answers` ותיוג (בעיה 2) — נכתבים ישירות לטבלת `leads` הקיימת (עם `is_sandbox` על הקמפיין), כך שעמוד הלידים ומדידת האיכות עובדים כרגיל. אין צורך בשדה לידים נפרד כאן.

**ג. RLS:** אדמין בלבד. `sandbox_inputs` — policy שבודקת הרשאת אדמין (או פשוט: רק דרך endpoints מאומתי-אדמין, בלי policy ללקוח רגיל).

> **בטיחות:** `is_sandbox DEFAULT false` — קמפיין רגיל לעולם לא יהפוך בטעות לסנדבוקס. וקמפיין סנדבוקס לעולם לא ייגע ב-Meta (כי הוא מקבל `SandboxMetaClient`). שתי הגנות.

---

## חלק 4 — עמוד אדמין

עמוד פיתוח (לא ממשק לקוח). מאחורי הרשאת אדמין.

**יכולות:**
1. **יצירת קמפיין-דמה** — טופס מינימלי (שם שירות, ענף, אזור, מחיר) שיוצר `campaigns` עם `is_sandbox=true` + `quiz_responses` דמה (ל-`business_description`). דילוג על כל ה-flow של יצירת קמפיין אמיתי (3.x).
2. **טופס הזנת מספרים** (גמיש, החלטה 3) — שדות ל-CPL, לידים, אחוז לחיצה, מחיר לקליק, סטטוס מודעה, סיבת דחייה. כתיבה ל-`sandbox_inputs`. אפשר לעדכן ולהריץ שוב.
3. **הזנת לידים-דמה** (לבעיה 2) — טופס שמוסיף שורות ל-`leads` עם `screening_answers` ותיוג, על הקמפיין-דמה.
4. **כפתורי הפעלת הסוכן:**
   - "הרץ אבחון בעיה 1" → קורא ל-`POST /me/agent/.../diagnose` עם `high_cpl`.
   - "הרץ אבחון בעיה 2" → עם `low_quality_leads`.
   - "סמן מודעה כנדחתה" → מדמה webhook דחייה (בעיה 3).
   - "הצג פתרון" / "אשר פתרון" → propose/approve.
5. **כפתור "דמה שעברו 120 שעות"** (החלטה 4) — מריץ את `monitor_optimization_windows` ידנית על הקמפיין-דמה. מאפשר לראות את המדידה וההחלטה בלי להמתין.
6. **תצוגת תוצאה** — מה הסוכן החזיר (האבחון, הקופי שנוצר), ומה נכתב ל-`optimization_actions`/`optimization_sessions`.

> ה-UI לא נבנה כאן במלואו (כמו שאר הסשנים) — אבל ה-endpoints והלוגיקה כן. עמוד אדמין מינימלי מספיק.

---

## חלק 5 — כפתור הקפצת-זמן

הפרוטוקול מבוסס על חלון 120 שעות. בסנדבוקס אי אפשר לחכות.

**`POST /admin/sandbox/{campaign_id}/advance-time`:**
- אופציה א' (פשוטה): מזיז את `window_ends_at` של הפעולה הפעילה ל-`now() - 1 minute`, ואז מריץ את `monitor_optimization_windows` ידנית.
- אופציה ב' (נקייה): מריץ את `monitor_optimization_windows` עם פרמטר `force_campaign_id` שמתעלם מבדיקת הזמן לקמפיין הזה בלבד.
- **המלצה: א'** — לא משנה את ה-cron עצמו (שורש: ה-cron זהה בפרודקשן ובסנדבוקס; רק הזזנו את ה-timestamp). מזיז זמן, מריץ cron אמיתי.

> **חשוב:** הכפתור מריץ את **ה-cron האמיתי** (`monitor_optimization_windows` מ-7.4ב), לא העתק. כך מה שבודקים הוא בדיוק מה שירוץ בפרודקשן. רק ה-timestamp הוזז.

---

## חלק 6 — שינויים נדרשים בקוד הקיים

**א.** `integrations/meta.py` — פיצול ל-`MetaClient` (abstract) + `RealMetaClient` (הקוד הקיים) (חלק 1).
**ב.** `integrations/sandbox_meta.py` — `SandboxMetaClient` חדש (חלק 1).
**ג.** נקודת ה-DI (`get_meta_client` או דומה) — הזרקה לפי `is_sandbox` (חלק 2). **לוודא שכל ה-services מקבלים את ה-client בהזרקה, לא יוצרים בעצמם.**
**ד.** migration — `is_sandbox` + `sandbox_inputs` (חלק 3).
**ה.** `routers/admin.py` — endpoints לקמפיין-דמה, הזנת מספרים, הפעלת סוכן, הקפצת-זמן (חלק 4-5).
**ו.** הרשאת אדמין — middleware/dependency שמגן על `/admin/*`.
**ז.** `models/` — `SandboxInput`, `AdminCampaignCreate`.

> **השינוי הגדול הוא ד' (refactor של `meta.py`).** אם הקוד הקיים קורא ל-Meta ישירות ב-services (לא דרך שכבה אחת) — צריך לרכז קודם. זה החלק שהופך את כל השאר לאפשרי. **לבדוק את זה ראשון.**

---

## חלק 7 — בדיקות

**ממשק + DI:**
1. קמפיין `is_sandbox=true` → `get_meta_client` מחזיר `SandboxMetaClient`.
2. קמפיין רגיל → `RealMetaClient`.
3. ה-orchestrator/services מקבלים client בהזרקה — אין `is_sandbox` בקוד-הסוכן (בדיקת קוד/ארכיטקטורה).

**`SandboxMetaClient`:**
4. `get_campaign_insights` → מחזיר את `sandbox_inputs` האחרון, לא קורא ל-Meta.
5. `update_ad_creative` → no-op, מחזיר `meta_id` דמה, כותב ל-DB, **לא** קורא ל-Meta.
6. `create_lead_form` → no-op + DB.

**flow מלא בסנדבוקס (בעיה 1):**
7. קמפיין-דמה, CPL ₪22 קוסמטיקה → "הרץ אבחון" → תרחיש א' (עצירה), בלי סדרה.
8. CPL ₪65 → אבחון בעיה 1 → סדרה נפתחת (`optimization_sessions`), בלי Meta.
9. "הצג פתרון" → LLM **אמיתי** יוצר 3 קופי.
10. "אשר" → `optimization_actions` נכתב, `window_ends_at` נקבע, **בלי** העלאה אמיתית.
11. "דמה 120 שעות" → `monitor_optimization_windows` רץ → מודד את `sandbox_inputs` → מחליט.
12. עדכון `sandbox_inputs` ל-CPL נמוך יותר → הקפצת-זמן → "שיפור" → `success_monitoring`.
13. עדכון ל-CPL זהה → "אין שיפור" → מעבר שלב.

**flow בעיה 2:**
14. הזנת לידים-דמה עם תיוג `irrelevant` → "הרץ אבחון בעיה 2" → 4 קטגוריות.
15. בחירה → קופי מסנן (LLM אמיתי) → אישור → no-op Meta.
16. שלב 7: טופס + כותרת → no-op, נכתב ל-DB.

**flow בעיה 3:**
17. "סמן מודעה כנדחתה" + סיבה → סדרת `meta_rejection` + תיקון קופי (LLM אמיתי).

**בטיחות:**
18. קמפיין רגיל → לעולם לא מקבל `SandboxMetaClient`.
19. קמפיין-דמה → לעולם לא קורא ל-Meta API (mock מוודא 0 קריאות).

---

## חלק 8 — לא בסנדבוקס

- **איכות השיפוט על דאטה אמיתית** — Phase 8 (קמפיין חי). הסנדבוקס בודק שהפרוטוקול **רץ**, לא שההחלטה **נכונה עסקית** על נתוני אמת.
- **סימולציית CI אוטומטית** — פריט נפרד ("סביבת סימולציה"). זה אינטראקטיבי-ידני.
- **ממשק לקוח** — אדמין בלבד.
- **חיבור Meta כלשהו** — אפס.

---

## חלק 9 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **ממשק + DI, לא `if sandbox`.** זה כל הסשן. אם מוצאים את עצמנו כותבים `if campaign.is_sandbox:` בתוך orchestrator/service — **טעות**. ה-`if` היחיד הוא ב-`get_meta_client`. קוד-הסוכן עיוור.

2. **כל מה שלא-Meta רץ אמיתי — כולל LLM.** יצירת הקופי בסנדבוקס היא קריאה אמיתית ל-gpt-5.2. זה הערך: רואים מה הסוכן **באמת** יוצר, לא mock.

3. **לרכז את גישת Meta קודם.** הסנדבוקס עובד רק אם כל קריאות Meta דרך `integrations/meta.py`. אם יש קריאות ישירות ב-services — לרכז לפני. **לבדוק ראשון.**

4. **הקפצת-זמן מריצה את ה-cron האמיתי.** רק מזיזה `window_ends_at`. ה-cron זהה לפרודקשן. לא העתק.

5. **`is_sandbox DEFAULT false` — בטיחות.** קמפיין רגיל לא יהפוך בטעות. קמפיין-דמה לא ייגע ב-Meta. שתי הגנות בלתי-תלויות.

6. **לידים-דמה בטבלת `leads` הרגילה** (עם `is_sandbox` על הקמפיין), כדי שעמוד הלידים ומדידת האיכות עובדים כרגיל. לא טבלה נפרדת.

7. **נדרש לפני Phase 8.** זה הכלי שמאפשר לראות את הסוכן מחליט לפני שמשחררים אוטונומיה. לתעד את התלות ב-ROADMAP.

8. **Sentry context** — בסנדבוקס, לסמן את ה-exceptions כ-`sandbox=true` כדי לא לבלבל עם בעיות פרודקשן.

9. **מספר migration** — תלוי בסדר. לתאם ב-ROADMAP.

---

# Sandbox — תוספת: השוואת Multi-LLM פר-תפקיד

> תוספת לסשן `session-sandbox-agent-testing.md`. נכתבת כהמשך — לא מחליפה אותו.
> **מטרה:** בסנדבוקס, לראות איך GPT / Gemini / Claude מתמודדים עם **אותו** פלט סוכן, פר-תפקיד — כדי לבחור את המודל הטוב ביותר לכל תפקיד.
> **אותו עיקרון שורשי כמו ה-`MetaClient`:** נקודת החלפה אחת (`LLMClient` interface), ריבוי מימושים, אפס `if model ==` בקוד-הסוכן.

---

## מבוא ותיאור

הסנדבוקס מריץ את כל קריאות ה-LLM של הסוכן — אבל על **מודל אחד**. התוספת הזו מאפשרת להריץ אותן דרך **שלושה ספקים במקביל** ולהשוות.

**התובנה שמעצבת את הכל: לסוכן יש כמה תפקידים, ואין מנצח אחד.** מודל שונה עשוי להיות הכי טוב לכל תפקיד:
- **מסווג** (`classifier`) — ממפה סיבת דחייה לקטגוריה, מזהה ענף. משימה צרה עם תשובה "נכונה". רץ על מודל **מיני זול**.
- **מאבחן/יועץ** (`advisor`) — מנסח אבחון, מסביר חוק ברזל, מציג ללקוח. ה"מוח". משימה כבדה וסובייקטיבית.
- **קופירייטר** (`copywriter`) — כותב 3 קופי + כותרות. משימה יצירתית.

ההשוואה "מי הכי טוב" חסרת משמעות בלי לדעת **לאיזה תפקיד** — כי המנצח הוא פר-תפקיד.

**מה התוספת עושה:**
- מרכזת את כל קריאות ה-LLM לשכבה אחת (`LLMClient`) — **תנאי מקדים**.
- כל קריאה מקבלת **תווית תפקיד** (`classifier`/`advisor`/`copywriter`).
- 3 מימושים: OpenAI / Gemini / Anthropic. מודל פר-(תפקיד × ספק) מ-env. flag הפעלה פר-ספק מ-env.
- **בסנדבוקס:** `advisor` ו-`copywriter` רצים על כל הספקים הפעילים במקביל → 3 עמודות. `classifier` רץ על המיני שלו ברקע. הכול נשמר ל-DB.
- **בפרודקשן:** כל תפקיד רץ על ספק אחד (ברירת מחדל פר-תפקיד מ-env). אין ריבוי קריאות.

**מה לא:**
- ריבוי-LLM בפרודקשן — בסנדבוקס בלבד. פרודקשן = ספק אחד פר-תפקיד.
- בחירה אוטומטית של "המנצח" — אתה משווה בעין ובוחר ידנית (דרך env).
- השוואה ויזואלית של ה-`classifier` — רץ ונשמר, אבל לא מוצג ב-3 עמודות (יש לו תשובה נכונה, פחות מעניין ויזואלית). מוצגים רק `advisor` ו-`copywriter`.

---

## סיכום החלטות שננעלו

| # | שאלה | הכרעה |
|---|---|---|
| 1 | מבנה ההחלפה | **`LLMClient` interface + ריבוי מימושים** (כמו `MetaClient`). אפס `if model` בקוד-הסוכן |
| 2 | ריכוז קריאות | **תנאי מקדים.** קריאות LLM כיום מפוזרות → לרכז לשכבה אחת לפני הכול |
| 3 | תפקידים | **3:** `classifier` (מיני), `advisor` (חזק), `copywriter` (חזק). כל קריאה מתויגת |
| 4 | בהשוואה הוויזואלית | **`advisor` + `copywriter` בלבד** (3 עמודות). `classifier` רץ ברקע על המיני, לא מוצג |
| 5 | מודל פר-תפקיד-וספק | **env פר-(תפקיד × ספק).** מאפשר gpt-5.2→5.5, flash→pro בלי קוד |
| 6 | הפעלה/כיבוי ספק | **flag env פר-ספק.** ספק כבוי = עמודה שלא מופיעה (פותר גם "כשל מודל בודד") |
| 7 | קריאות בסנדבוקס | **כל הספקים הפעילים במקביל** (פי-3 עלות/זמן — מקובל לכלי פנימי) |
| 8 | כשל מודל בודד | **לא מפיל את ההשוואה.** מציגים את שהצליחו + שגיאה בעמודה הכושלת |
| 9 | שמירה | **נשמר ל-DB** (`sandbox_llm_comparisons`). הופך את הסנדבוקס למעבדה — השוואה לאורך זמן ובין גרסאות |
| 10 | פרודקשן | **ספק אחד פר-תפקיד**, ברירת מחדל מ-env פר-תפקיד. בלי ריבוי קריאות |
| 11 | ברירת מחדל פרודקשן | **פר-תפקיד, לא גלובלי.** אולי Claude לקופי, GPT לאבחון. `LLM_{ROLE}_PROD_PROVIDER` |

---

## חלק 1 — ריכוז קריאות ה-LLM (תנאי מקדים)

**זה הצעד הראשון — בלעדיו שאר התוספת לא אפשרית.**

כיום קריאות ה-LLM מפוזרות ב-services (קריאות ישירות ל-OpenAI ב-`solution_service`, `agent_orchestrator`, `rejection_service`, `ad_generation_service`...). לרכז את כולן לשכבה אחת.

> **בדיוק כמו Meta:** הסנדבוקס עובד רק אם כל גישת LLM דרך שכבה אחת. קריאות ישירות מפוזרות = התוספת דולפת. **לרכז קודם.** זה גם חוב טכני בכל מקרה.

**`LLMClient` (abstract / Protocol):** מגדיר את קריאת ה-LLM הגנרית, עם **תווית תפקיד**:
```python
async def complete(
    role: Literal["classifier", "advisor", "copywriter"],
    messages: list[Message],
    response_format: ... = None,
) -> LLMResponse
```
כל קריאה בסוכן עוברת דרך זה, עם ה-`role` המתאים. ה-`role` קובע איזה מודל (פר-תפקיד-וספק) וגם מאפשר לסנן את התצוגה/השמירה.

---

## חלק 2 — שלושה מימושים + env

**`OpenAILLMClient` / `GeminiLLMClient` / `AnthropicLLMClient`** — כל אחד מממש את `LLMClient` מול ה-API שלו.

**מבנה ה-env — מודל פר-(תפקיד × ספק):**
```
# מסווג — מיני, זול
LLM_CLASSIFIER_OPENAI_MODEL=gpt-5-mini
LLM_CLASSIFIER_GEMINI_MODEL=gemini-flash
LLM_CLASSIFIER_ANTHROPIC_MODEL=claude-haiku-...

# מאבחן/יועץ — חזק
LLM_ADVISOR_OPENAI_MODEL=gpt-5.2
LLM_ADVISOR_GEMINI_MODEL=gemini-pro
LLM_ADVISOR_ANTHROPIC_MODEL=claude-...

# קופירייטר — חזק
LLM_COPYWRITER_OPENAI_MODEL=gpt-5.2
LLM_COPYWRITER_GEMINI_MODEL=gemini-pro
LLM_COPYWRITER_ANTHROPIC_MODEL=claude-...

# הפעלה/כיבוי פר-ספק
LLM_OPENAI_ENABLED=true
LLM_GEMINI_ENABLED=true
LLM_ANTHROPIC_ENABLED=true

# ברירת מחדל בפרודקשן — פר-תפקיד (לא גלובלי)
LLM_CLASSIFIER_PROD_PROVIDER=openai
LLM_ADVISOR_PROD_PROVIDER=openai
LLM_COPYWRITER_PROD_PROVIDER=anthropic
```

- **שינוי גרסה** = שינוי env (5.2→5.5, flash→pro). בלי קוד.
- **כיבוי ספק** = `ENABLED=false` → עמודה לא מופיעה בסנדבוקס, ולא נקראת. פותר גם "מה אם אין מפתח Claude".
- **ברירת מחדל פר-תפקיד** = אתה בוחר את המנצח של כל תפקיד בנפרד (Claude לקופי, GPT לאבחון).

---

## חלק 3 — DI: מתי מודל אחד, מתי שלושה

**הרחבת ה-factory** (כמו `get_meta_client` בסנדבוקס המקורי):

- **פרודקשן** (קמפיין רגיל): `get_llm_client(role)` מחזיר client **אחד** — הספק מ-`LLM_{ROLE}_PROD_PROVIDER`. קריאה אחת.
- **סנדבוקס** (`is_sandbox`): לא client אחד — **מנהל השוואה** (`SandboxLLMComparator`) שקורא לכל הספקים הפעילים **במקביל** ל-`advisor`/`copywriter`, ולמיני בלבד ל-`classifier`.

> **`if` יחיד, בשכבת ההזרקה.** קוד-הסוכן קורא `client.complete(role=...)` ולא יודע אם מאחוריו ספק אחד או שלושה. אפס `if sandbox`/`if model` בסוכן.

**`SandboxLLMComparator.complete(role, messages, ...):`**
1. אם `role == 'classifier'` → רץ על המיני של ברירת המחדל בלבד (לא משווה). מחזיר תוצאה, שומר.
2. אם `role in ('advisor','copywriter')` → קורא לכל הספקים ה-`ENABLED` **במקביל** (asyncio.gather).
3. כל ספק שנכשל (שגיאה/JSON לא תקין) → נתפס, נשמר כ-error, **לא מפיל** את השאר (החלטה 8).
4. שומר שורה פר-ספק ל-`sandbox_llm_comparisons`.
5. מחזיר את כל התוצאות (לתצוגת 3 עמודות).

> **מה הסוכן ממשיך איתו?** בסנדבוקס, אחרי השוואה, צריך לבחור תוצאה אחת כדי להמשיך את ה-flow (הקופי שנכתב ל-`optimization_actions`). **המלצה:** ממשיכים עם ספק ברירת המחדל של אותו תפקיד (ה-`PROD_PROVIDER`), והשניים האחרים הם תצוגה/שמירה בלבד. כך ה-flow דטרמיניסטי, וההשוואה לא משנה את מה שנכתב ל-DB.

---

## חלק 4 — Migration: `sandbox_llm_comparisons`

קובץ חדש (מספר רץ הבא).
```sql
CREATE TABLE public.sandbox_llm_comparisons (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id uuid NOT NULL REFERENCES public.campaigns(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  role text NOT NULL CHECK (role IN ('classifier','advisor','copywriter')),
  scenario_label text,              -- איזה תרחיש/קריאה (אבחון בעיה 1, קופי שלב 2...)
  provider text NOT NULL CHECK (provider IN ('openai','gemini','anthropic')),
  model text NOT NULL,              -- הגרסה שרצה בפועל (gpt-5.2...) — לתיעוד בין גרסאות

  output jsonb,                     -- הפלט (קופי/אבחון/סיווג)
  error text,                       -- אם הספק נכשל
  latency_ms integer,               -- כמה זמן לקח (להשוואת מהירות)

  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_sandbox_llm_role
  ON public.sandbox_llm_comparisons (campaign_id, role, created_at DESC);
```
- שורה **פר-ספק פר-קריאה**. כך אפשר לשלוף "כל הפעמים ש-Gemini היה ב-`copywriter`", או להשוות `model='gpt-5.2'` מול `model='gpt-5.5'` על אותו תפקיד.
- `model` נשמר בפועל → השוואה **בין גרסאות** לאורך זמן (לא רק בין ספקים).
- `latency_ms` → גם השוואת מהירות, לא רק איכות.
- RLS: אדמין בלבד (כמו `sandbox_inputs`).

---

## חלק 5 — תצוגה (עמוד אדמין)

הרחבת עמוד האדמין מהסנדבוקס המקורי.

- כשרצים תרחיש שמפעיל `advisor` או `copywriter` → התצוגה מראה **3 עמודות** (OpenAI / Gemini / Anthropic), זו לצד זו, עם הפלט של כל אחד.
- כל עמודה: שם הספק + הגרסה (מ-env), הפלט, וזמן התגובה. עמודה כושלת → הודעת שגיאה במקום פלט.
- ספק כבוי (`ENABLED=false`) → העמודה פשוט לא מופיעה (2 עמודות, או 1).
- `classifier` → לא מוצג ב-3 עמודות. אפשר להציג את התוצאה היחידה שלו בשורה רגילה (רץ ברקע).
- היסטוריה: אפשר לשלוף השוואות קודמות מ-`sandbox_llm_comparisons` (כלי-מעבדה — להשוות לאורך זמן). תצוגה בסיסית מספיקה ל-MVP.

> ה-UI מכוער ומינימלי (כלי-פיתוח). 3 עמודות טקסט + כותרות. לא design system, לא ללקוחות.

---

## חלק 6 — שינויים נדרשים בקוד

**א.** ריכוז כל קריאות ה-LLM ל-`LLMClient` (חלק 1) — **תנאי מקדים, ראשון**.
**ב.** `integrations/llm/` — `OpenAILLMClient`, `GeminiLLMClient`, `AnthropicLLMClient` (חלק 2).
**ג.** קריאת env פר-(תפקיד × ספק) + flags (חלק 2).
**ד.** `SandboxLLMComparator` (חלק 3).
**ה.** הרחבת ה-factory — מודל אחד (פרודקשן) מול comparator (סנדבוקס) (חלק 3).
**ו.** migration `sandbox_llm_comparisons` (חלק 4).
**ז.** עמוד אדמין — תצוגת 3 עמודות (חלק 5).
**ח.** תיוג `role` בכל קריאת LLM קיימת בסוכן (classifier/advisor/copywriter).

---

## חלק 7 — בדיקות

**ריכוז + interface:**
1. כל קריאות הסוכן עוברות דרך `LLMClient` (בדיקת קוד — אין קריאת OpenAI ישירה מחוץ לשכבה).
2. כל קריאה מתויגת `role` נכון.

**מודל פר-תפקיד-וספק:**
3. `advisor` קורא לגרסה מ-`LLM_ADVISOR_OPENAI_MODEL`, לא מ-classifier.
4. שינוי env (5.2→5.5) → הקריאה משתמשת בחדש בלי קוד.

**פרודקשן (ספק אחד):**
5. קמפיין רגיל, `advisor` → קריאה אחת לספק מ-`LLM_ADVISOR_PROD_PROVIDER`.
6. `copywriter` → ספק מ-`LLM_COPYWRITER_PROD_PROVIDER` (יכול להיות שונה מ-advisor).
7. אין ריבוי קריאות בפרודקשן.

**סנדבוקס (השוואה):**
8. `is_sandbox`, `copywriter` → קריאה לכל הספקים ה-ENABLED במקביל.
9. 3 ספקים enabled → 3 שורות ב-`sandbox_llm_comparisons`.
10. ספק אחד `ENABLED=false` → לא נקרא, לא נשמר, עמודה לא מופיעה.
11. `classifier` בסנדבוקס → רץ על המיני בלבד, לא משווה, לא ב-3 עמודות.
12. ספק נכשל (JSON לא תקין) → נשמר `error`, השאר ממשיכים, ההשוואה לא נופלת.
13. ה-flow ממשיך עם ספק ברירת המחדל (PROD_PROVIDER) — מה שנכתב ל-`optimization_actions` דטרמיניסטי.
14. `latency_ms` נרשם פר-ספק.

**שמירה/היסטוריה:**
15. השוואה נשמרת עם `model` בפועל → אפשר להשוות 5.2 מול 5.5 לאורך זמן.

---

## חלק 8 — הערות לפיתוח (לסקירה לפני העברה ל-CC)

1. **ריכוז ה-LLM ראשון — תנאי מקדים.** בלי שכבה אחת, ההשוואה דולפת. בדיוק כמו Meta. זה גם חוב טכני טוב לסגור.

2. **`if` יחיד בשכבת ההזרקה.** קוד-הסוכן קורא `complete(role=...)` ולא יודע כמה ספקים מאחוריו. אפס `if model`/`if sandbox` בסוכן.

3. **תפקיד קובע מודל.** מסווג→מיני, אבחון/קופי→חזק. ה-env פר-(תפקיד × ספק), לא גלובלי.

4. **ברירת מחדל פרודקשן פר-תפקיד.** אולי Claude לקופי, GPT לאבחון. `LLM_{ROLE}_PROD_PROVIDER` נפרד לכל תפקיד — זו כל הפואנטה של ההשוואה.

5. **ה-flow ממשיך עם ברירת המחדל.** בסנדבוקס, ההשוואה היא תצוגה/שמירה; מה שנכתב ל-DB הוא תוצאת ה-PROD_PROVIDER. כך ה-flow דטרמיניסטי וההשוואה לא משנה תוצאות.

6. **classifier רץ אבל לא מוצג.** יש לו תשובה נכונה — פחות מעניין ויזואלית. נשמר לתיעוד, מוצג בשורה לא ב-3 עמודות.

7. **`model` נשמר בפועל.** מאפשר השוואה בין גרסאות (5.2 מול 5.5), לא רק בין ספקים. הסנדבוקס הופך למעבדה.

8. **כיבוי ספק = flag env.** עמודה נעלמת, לא שגיאה. פותר "אין מפתח Claude" ו"כשל מודל בודד" באותו מנגנון.

9. **פי-3 עלות/זמן בסנדבוקס** — מקובל לכלי פנימי. בפרודקשן קריאה אחת, אפס תוספת.

10. **מספר migration** — תלוי בסדר. לתאם ב-ROADMAP.
