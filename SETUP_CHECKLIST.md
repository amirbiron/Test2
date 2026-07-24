# Setup Checklist — Campaign AI

מרכז את כל מה שצריך להגדיר **ידנית** לפני בטא/פרודקשן: env vars + הגדרות בשירותים חיצוניים.
לא קוד — ה-runbook התפעולי. כשמגיעים להקמה, עוברים פה שורה-שורה.

> **תחזוקה (CLAUDE.md):** כל הוספת env var ב-`config.py`, או תלות חדשה בהגדרה חיצונית
> (Meta/Supabase/ספק תשלום/וכו') — **חובה לעדכן את הקובץ הזה באותו PR**. אחרת נשכח להגדיר בהקמה.

מקרא: `🔑` סוד (לעולם לא ב-git/לוגים — כלל 7) · `⚠️` gotcha שקל לפספס · `[ ]` ממתין · `[x]` הוגדר

---

## 1. Environment Variables

נטענים ב-`app/config.py`. **Core חובה** = ה-boot נכשל בלעדיהם. **אופציונלי** = הפיצ'ר יורד בחן
(degradation → 503/שגיאת דומיין), ה-boot לא קורס (כלל 10). מוגדרים ב-Render Environment Group.

### Core — חובה (boot נכשל)
- [ ] `SUPABASE_URL`
- [ ] `SUPABASE_ANON_KEY` 🔑
- [ ] `SUPABASE_SERVICE_ROLE_KEY` 🔑 — service_role, עוקף RLS
- [ ] `SUPABASE_DB_URL` 🔑 — connection string **session-mode** (IPv4/pooler), ל-migrations בלבד (`preDeployCommand`). בלעדיו ה-app עולה אבל migrations נכשלות.

### Auth / Cookies
- [ ] `COOKIE_SIGNING_SECRET` 🔑 — חתימת state של OAuth. בלעדיו חיבור Meta **וגם** login עם פייסבוק (1.3) מושבתים (503 ב-`/auth/oauth/facebook/*`).
- [ ] `COOKIE_SECURE` (prod: `true`) · `COOKIE_SAMESITE` (`lax`) · `COOKIE_DOMAIN`
  - **dev מקומי על http:** הגדר `COOKIE_SECURE=false` — אחרת הדפדפן מפיל בשקט את עוגיית `refresh_token` (Secure לא נשלחת/נשמרת על http) וה-refresh/login נשברים. בפרודקשן (https) → `true`. (`tests/conftest.py` כבר עושה זאת לטסטים.)
- [ ] `OAUTH_REDIRECT_URI` — callback של Facebook login (`{backend}/auth/oauth/facebook/callback`). חייב רשום ב-Supabase → Auth → URL Configuration.

### Frontend (same-origin)
- [ ] **הפרונטד מוגש ע"י FastAPI עצמו** מ-`app/web/` (mount של StaticFiles על `/`, `html=True`): `/` → `app/web/index.html`, `/api.js` → `app/web/api.js`. אין שרת-פרונט נפרד ואין CORS (same-origin). `docs/prototype/index.html` עבר ל-`app/web/` (היסטוריה ב-git); `docs/prototype/` נשאר עם מסמכי העיצוב בלבד.

### Meta — OAuth + Marketing + Leadgen (Phase 1-4)
- [ ] `META_APP_ID` · `META_APP_SECRET` 🔑
- [ ] `META_REDIRECT_URI` — דיאלוג ה-OAuth (חיבור חשבון מודעות)
- [ ] `META_WEBHOOK_VERIFY_TOKEN` 🔑 — handshake של webhook ה-**leadgen** (4.1)
- [ ] `META_API_VERSION` — default `v25.0` (⚠️ Meta מעדכנת גרסאות כל ~3ח' — לבדוק changelog)
- [ ] `PRIVACY_POLICY_URL` — ל-Lead Form (אם חסר → `{frontend}/privacy`; אם גם הוא חסר → push של lead חסום)
- [ ] **Meta App Dashboard — Subscribe ל-`ad_review` field** (7.6ג, בעיה 3): ה-webhook הקיים של leadgen מקבל גם דחיות מודעה אם ה-app subscribed ל-field זה. בלי subscribe, הסוכן לא יזהה דחיות אוטומטית. **ידני, פר-app**.

### Meta — WhatsApp Cloud API (Phase 5.0)
- [ ] `META_WABA_ID` — ה-WhatsApp Business Account (N קווים תחתיו)
- [ ] `META_ACCESS_TOKEN` 🔑 — ⚠️ **System User Token** (60d+ renewal), לא User Access Token (פג ב-60 יום בשקט)
- [ ] `META_WHATSAPP_VERIFY_TOKEN` 🔑 — handshake של webhook ה-WhatsApp (**נפרד** מ-leadgen)
- [ ] `META_GRAPH_BASE_URL` — default `https://graph.facebook.com` (override רק לטסטים)
- [ ] `META_NOTIFY_PHONE_NUMBER_ID` 🔑 — (PR-D) קו-שולח של הפלטפורמה להתראות VIP לבעלים. **שונה** מקווי-הלקוחות (`whatsapp_lines`): `phone_number_id` של קו ייעודי תחת ה-WABA שלנו. בלעדיו התראות ה-VIP מדולגות (skip). נדרש **בנוסף** ל-`whatsapp_production_ready`

### Meta — WhatsApp Templates (Phase 5.4)
- [ ] `META_TEMPLATE_BOT_OPENING_NAME` — default `bot_opening_v1`. ⚠️ לעדכן לשם שמטא אישרה (אחרת כל שליחה נכשלת ב-132xxx)
- [ ] `META_TEMPLATE_BOT_FOLLOWUP_NAME` — default `bot_followup_v1`
- [ ] `META_TEMPLATE_BOT_HANDOFF_NAME` — default `bot_handoff_to_owner_v1`
- [ ] `META_TEMPLATE_BOT_PENDING_APPOINTMENT_NAME` — default `bot_pending_appointment_v1` (5.5d — נדרש רק לתיאום תורים; התראת תור לבעל העסק)
- [ ] `META_TEMPLATE_BOT_APPOINTMENT_CONFIRMATION_NAME` — default `bot_appointment_confirmation_v1` (5.5e — לליד אחרי אישור)
- [ ] `META_TEMPLATE_BOT_APPOINTMENT_REJECTED_NAME` — default `bot_appointment_rejected_v1` (5.5e — לליד אחרי דחייה/ביטול)
- [ ] `META_TEMPLATE_BOT_APPOINTMENT_CANCELLED_OWNER_NAME` — default `bot_appointment_cancelled_owner_v1` (5.5e-2 — לבעל העסק אחרי ביטול ע"י הליד)
- [ ] `META_TEMPLATE_NEW_LEAD_OWNER_VIP_NAME` — default `bot_new_lead_owner_vip_v1` (PR-D — התראת VIP לבעל-העסק על ליד חדש; UTILITY, 4 משתנים). ⚠️ נדרש רק לתוספת VIP
- [ ] `META_TEMPLATE_AGENT_UPDATE_OWNER_NAME` — default `agent_update_owner_v1` (ערוץ עדכוני-הסוכן החינמי לבעלים; UTILITY, 2 משתנים: תקציר פר-type + deep-link). ⚠️ נדרש להפעלת ערוץ עדכוני-הסוכן בוואטסאפ

### Google Calendar — תיאום תורים (Phase 5.5)
- [ ] `GOOGLE_CLIENT_ID` — OAuth 2.0 Client ID מ-Google Cloud Console (`*.apps.googleusercontent.com`)
- [ ] `GOOGLE_CLIENT_SECRET` 🔑
- [ ] `GOOGLE_REDIRECT_URI` — `{backend}/auth/google/callback`. חייב רשום ב-Google Cloud Console → Authorized redirect URIs
- [ ] ℹ️ נוסף ל-web בלבד ב-5.5b; ה-worker יקבל `GOOGLE_CLIENT_ID`/`SECRET` ב-5.5c (רענון tokens)
- [ ] ℹ️ אין מפתח הצפנה נפרד — ה-tokens של היומן מוצפנים ב-**Supabase Vault** (כמו Meta), לא Fernet
- [ ] ⚠️ **הדגל `appointment_scheduling_enabled` דלוק** (migration `0112`) — קביעת-פגישה ישירה ביומן היא כעת
  פעולת-הסיום היחידה של הבוט ב-UI (Calendly הוסר מ-UI; נשאר תקף ב-runtime לקונפיגים legacy). דורש:
  `GOOGLE_CLIENT_ID/SECRET/REDIRECT_URI` לעיל (חיבור-יומן דרך "הגדרות הבוט" → "חבר יומן Google") **וגם**
  את ה-templates של תיאום-התורים לעיל (5.5d-e) — שרגל-הוואטסאפ שלהם כבויה-רכה עד `whatsapp_production_ready`
  + WABA. בעל-העסק מאשר תורים ב-frontend `/my-appointments` (deep-link לפאנל "התורים שלי", Premium; נתיב-SPA נפרד מ-API `/appointments`). כיבוי: `set_value('appointment_scheduling_enabled', false)`.

### Pelecard — תשלומים (Phase 2.6)
- [ ] `PELECARD_TERMINAL` 🔑 · `PELECARD_USER` 🔑 · `PELECARD_PASSWORD` 🔑
- [ ] `PELECARD_HOST` — `gateway20.../sandbox` בפיתוח, `gateway21...` בפרודקשן

### Green Invoice / Morning — חשבוניות (Phase 2.6.2)
- [ ] `GREEN_INVOICE_KEY_ID` 🔑 · `GREEN_INVOICE_KEY_SECRET` 🔑 (דורש מסלול Best+)
- [ ] `GREEN_INVOICE_ENV` — `production` / `sandbox`

### Resend — התראות email (Phase 4.6)
- [ ] `RESEND_API_KEY` 🔑 · `RESEND_FROM_EMAIL` (⚠️ בלי דומיין מאומת → sandbox `onboarding@resend.dev`) · `RESEND_REPLY_TO`

### Telegram — ערוץ-התראות לצוות התמיכה (SUPPORT_ESCALATION_PLAN, סבב-1)
- [ ] `TELEGRAM_BOT_TOKEN` 🔑 — token של ה-Bot מ-BotFather. בלעדיו ערוץ-הטלגרם מושבת (ההתראה skip; ה-thread נשמר וגלוי ב-`/admin/support` ממילא) — degradation (כלל 10), ה-boot לא קורס
- [ ] `TELEGRAM_ALERT_CHAT_ID` — מזהה ה-chat/קבוצה של הצוות (היעד הגלובלי, **לא** של הלקוח). התראה על כל פנייה/הודעה חדשה לתמיכה נשלחת לכאן
- [ ] `TELEGRAM_API_BASE_URL` (`https://api.telegram.org`) — default OK (override רק לטסטים)

### OpenAI + Admin (Phase 3.1.5)
- [ ] `OPENAI_API_KEY` 🔑 — יצירת קופי+תמונות
- [ ] `ADMIN_TOKEN` 🔑 — X-Admin-Token ל-admin endpoints (prompt-tester, whatsapp lines, billing resolve-charge-unknown, **sandbox** `/admin/sandbox` — כלי-פיתוח לבחינת הסוכן ללא Meta). בלעדיו → 503. (הסנדבוקס לא דורש env נוסף; propose מריץ LLM אמיתי → `OPENAI_API_KEY` לעיל.)
- [ ] `OPENAI_TEXT_MODEL` (`gpt-5.2`) · `OPENAI_IMAGE_MODEL` (`gpt-image-2`) · `OPENAI_AGENT_MODEL` (`gpt-5.2`, 7.1 — צ'אט הסוכן) — defaults OK
- [ ] `AGENT_FREE_CHAT_ENABLED` (default `true`, 7.4.X) — מתג השיחה החופשית של הסוכן. `false` → הסוכן מפנה את הלקוח לכפתורי הפעולה (בלי LLM/מכסה). זרם הצ'יפ (אבחון) לא מושפע. default OK
- [ ] ⚠️ `DEV_BYPASS_EMAILS` (**dev-only**, comma-separated emails) — עוקף את שער-התשלום (`require_paid_access` + `has_paid_access`) עבור המיילים הרשומים, לדיבאג ה-flow שאחרי התשלום כשאין סליקה. **בנוסף (QA)**: מאפשר לאדמין לדלג על מסך בחירת מספר-WhatsApp ב-onboarding (כפתור "⏭️ דלג" ב-frontend + עקיפת בדיקת-הבעלות ב-`campaign_service` לקמפיין whatsapp, כדי לבחון את שאר המסכים לפני App Review). הדגל נחשף ל-frontend דרך `GET /me` (`is_dev_bypass`, מחושב מהמייל המאומת בלבד — לא מגוף-הבקשה). **לעולם לא בפרודקשן** (כל מייל ברשימה מקבל גישה חינם + עקיפת-בעלות). ריק=כבוי (default). מוגדר → אזהרת `logger.warning` ב-startup

### Sentry + שונות
- [ ] `SENTRY_DSN` 🔑 — ניטור (אופציונלי; בלעדיו `capture_alert` → logger בלבד)
- [ ] `FRONTEND_URL` · `BACKEND_URL` — לבניית redirect/callback URLs
- [ ] `STORAGE_BUCKET_CAMPAIGN_IMAGES` (`campaign-images`) · `STORAGE_SIGNED_URL_EXPIRES_SECONDS` (`604800`) — defaults OK

---

## 2. הגדרות בשירותים חיצוניים (ידני — לא בקוד)

### Supabase
- [ ] migrations — **אוטומטי** ב-deploy (`render.yaml: preDeployCommand: python -m app.db.migrate`)
- [ ] Vault — מאחסן את ה-Meta token (fb_connections) וה-billing token מוצפנים (RPC service_role-only)
- [ ] RLS + GRANT — אוטומטי ב-migrations (⚠️ כל טבלה חדשה צריכה GRANT מפורש — CLAUDE.md Postgres 11)
- [ ] Facebook OAuth provider — Auth → Providers → Facebook = **ON** + הזן **App ID / App Secret** (מ-Meta App). ל-login עם פייסבוק (1.3). הכפתור "חיבור מהיר באמצעות פייסבוק" במסך הכניסה (`app/web/index.html`) **חי** — מפנה ל-`/auth/oauth/facebook/start`.
  - **אבטחה (auto-link מקובל):** Supabase ממזג זהות FB לחשבון-סיסמה קיים אוטומטית על אימייל **מאומת-ע"י-הספק**, ואין toggle שמכבה זאת. **מקובל** כי פייסבוק מחזיר אימייל רק אם אומת מול התיבה — תוקף בלי גישה לתיבת-הקורבן לא יגיע למיזוג (וגישה לתיבה = takeover ממילא דרך "שכחתי סיסמה"). ה-409 (`classify_oauth_callback_error`) נשאר כהגנת-עומק (יורה רק אם Supabase יחזיר conflict במקום למזג — edge של אימייל לא-מאומת). **Allow manual linking** — לא נדרש (זה ה-`linkIdentity` API, לא ה-auto-link). הרצת אימות לפני go-live: `docs/deployment/facebook-login-verification.md`.
  - **Confirm-email פטור ל-OAuth:** משתמש FB חדש מקבל session **מיד** (זהויות OAuth עם אימייל מאומת-ספק פטורות מ-Confirm-email; ה-gate חל על signup עם סיסמה בלבד).
- [ ] **Auth → URL Configuration → Site URL** = `FRONTEND_URL` (**לא** ברירת-המחדל `localhost:3000`!) — יעד קישור אישור-המייל אחרי signup. בלעדיו הקישור מצביע ל-localhost ושובר את האישור בפרוד (ERR_CONNECTION_REFUSED, 0.5.1)
- [ ] **Auth → URL Configuration → Redirect URLs** — כולל `{FRONTEND_URL}/**` (ה-`email_redirect_to` שה-signup שולח) + `OAUTH_REDIRECT_URI` (Facebook)
- [ ] **Auth → Providers → Email → Confirm email** — דלוק; signup לא מחזיר JWT עד אישור-מייל (0.5.1 / §7א)
- [ ] **Auth → Email Templates → "Confirm signup"** — הדבק את ה-HTML הממותג מ-`docs/deployment/supabase-email-templates/confirm-signup.html` (Subject: "אישור הרשמה ל-Campaign AI"). אחרת המייל יוצא עם template default של Supabase (נראה זר ללקוח, בלי מיתוג)
- [ ] **Auth → Email Templates → "Magic Link"** — הדבק את ה-HTML הממותג מ-`docs/deployment/supabase-email-templates/magic-link.html` (Subject: "קישור הכניסה שלך ל-Campaign AI"). הכפתור בו כבר מצביע ל-URL של זרימת **token_hash server-side** (חובה): `{{ .SiteURL }}/auth/magic-link/verify?token_hash={{ .TokenHash }}&type=email`. ⚠️ זו תבנית **נפרדת** מ-"Confirm signup" — **אל** תדביק כאן את ה-HTML של אישור-ההרשמה ו**אל** תשתמש ב-`{{ .ConfirmationURL }}`. הסיבה: ה-default (`{{ .ConfirmationURL }}`) מחזיר session ב-URL-hash → ה-backend **לא יכול** לשתול את ה-refresh ב-httpOnly cookie (§7א) ו-magic-link לא יעבוד. עם ה-URL הזה, הלינק פוגע ב-endpoint שלנו → `verify_otp` server-side → cookie → redirect ל-`/login/success`. (`{{ .SiteURL }}`=`FRONTEND_URL`; same-origin → כבר מכוסה ב-Redirect URLs `/**`.) כניסה-**לרשומים-בלבד** נאכפת ב-backend (`should_create_user=false` + pre-check `email_exists`, migration 0099) — לא צריך הגדרה ב-dashboard
- [ ] **Custom SMTP** (Auth → SMTP Settings) — **נדרש ל-production**: ה-default SMTP של Supabase מוגבל (~3-4 מיילים/שעה). הגדר דרך Resend (`smtp.resend.com:587`, user `resend`, pass = `RESEND_API_KEY`, from = `noreply@{verified-domain}`) — דורש דומיין מאומת ב-Resend (SPF/DKIM). עד אז: template ממותג + SMTP של Supabase (beta בלבד)

### Meta — Developer Console + Business Manager
- [ ] App ב-Meta Developer Console + products: Facebook Login, Marketing API, WhatsApp
- [ ] OAuth redirect URIs (`META_REDIRECT_URI`, `OAUTH_REDIRECT_URI`)
- [ ] **Leadgen webhook** — App → Webhooks → `{backend}/webhooks/meta-leads`, verify token, field `leadgen` (4.1)
- [ ] **WhatsApp WABA** — WhatsApp Business Account יחיד · 📖 מדריך מלא צעד-אחר-צעד: `docs/deployment/waba-setup.md`
- [ ] **System User** — ב-Business Manager תחת ה-WABA, token עם life ארוך (`META_ACCESS_TOKEN`)
- [ ] **Phone number + Display Name** — ⚠️ provisioning ידני (Display Name 1-3 ימי עסקים). ה-`phone_number_id` נכנס ל-`PATCH /admin/whatsapp/lines/{id}`
- [ ] **WhatsApp webhook** — App → Webhooks → `{backend}/webhooks/whatsapp`, `META_WHATSAPP_VERIFY_TOKEN`, subscribe field `messages` (handshake מ-5.0; POST קליטת הודעות פעיל מ-5.2 — בלי ה-subscribe הבוט לא יקבל תשובות לידים)
- [ ] **App Review** — permissions ל-production (`ads_management`, `leads_retrieval`, `whatsapp_business_messaging`, `business_management`, `whatsapp_business_management`...) ⚠️ נדרש לבטא אמיתי. `whatsapp_business_management` (2.1.5) — לשליפת מספר ה-WhatsApp Business של המשתמש ב-onboarding (`/me/whatsapp-business-numbers`); **Dev mode:** Admin/Developer/Tester מקבל מיד; **Production:** דורש App Review. עד אישור — השליפה נכשלת ב-permission → degrade-to-empty → המשתמש רואה מדריך הקמה (לא שגיאה). ה-degrade מלוגג את סטטוס ה-scopes בפועל (`/me/permissions`) לאבחון. **לזרימה המלאה של קמפיין WhatsApp (CTWA — יצירה+דחיפה+תנאי-מקדים חיבור-מספר-לעמוד):** `docs/deployment/whatsapp-campaign-setup.md`
- [ ] **בחירת עמוד עסקי (1.1)** — ה-OAuth מבקש `business_management` (בנוסף ל-`pages_show_list`) כי בלעדיו `/me/accounts` מחזיר רשימה **ריקה** לדפים שמנוהלים תחת Business Portfolio (תיק עסקי) → המשתמש רואה "לא נמצא עמוד עסקי" למרות שאישר את הדף. `list_pages` עושה גם fallback ל-`/me/businesses`→owned/client pages (דורש את אותו scope). **Dev mode:** המשתמש המתחבר חייב להיות Admin/Developer/Tester על ה-App (אז מקבל את ההרשאות בלי App Review). **Production (משתמשים אמיתיים):** `business_management` ב-Advanced Access דורש App Review
- [ ] Test phone number — חינמי, 5 נמענים מוגדרים מראש; מספיק ל-5.0-5.3
- [ ] **3 Message Templates** (5.4) — WhatsApp Manager → Message Templates → Create. UTILITY, Hebrew. ה-bodies ב-`app/prompts/phase5/templates/`; הליך ב-`docs/deployment/meta-templates-submission.md`. אישור 1-3 ימי עסקים. אחרי אישור: עדכן `META_TEMPLATE_*_NAME` + הפעל את ה-flag (למטה)
- [ ] **soft-disable flag** (5.4) — עד אישור 3 ה-templates, הבוט **לא** שולח (skip). אחרי אישור: `update public.app_settings set value='true'::jsonb, updated_by='admin' where key='whatsapp_production_ready';`
- [ ] **התראת VIP לבעלים (PR-D)** — להפעלת התוספת נדרשים **יחד**: (1) template `bot_new_lead_owner_vip` מאושר (UTILITY, 4 משתנים — ראה `meta-templates-submission.md`), (2) `META_NOTIFY_PHONE_NUMBER_ID` (קו-שולח ייעודי תחת ה-WABA), (3) `FRONTEND_URL` (ל-deep-link `{{4}}`), (4) ה-flag `whatsapp_production_ready=true`. עד אז התראות ה-VIP מדולגות (skip שקט; הליד עצמו + מייל `new_lead` תקינים)
- [ ] **ערוץ עדכוני-הסוכן בוואטסאפ (חינמי)** — להפעלה נדרשים **יחד**: (1) template `agent_update_owner` מאושר (UTILITY, 2 משתנים — ראה `meta-templates-submission.md`), (2) `META_NOTIFY_PHONE_NUMBER_ID`, (3) `FRONTEND_URL` (deep-link), (4) ה-flag `whatsapp_production_ready=true` (gate משותף עם VIP, flag-alert נפרד). עד אז העדכונים מדולגים בוואטסאפ (skip שקט; **מייל נשלח תמיד**). הבעלים מפעיל opt-in ב"החבילה שלי" (`agent_whatsapp_enabled`, מכסה 30/חודש)

### Google Cloud Console — תיאום תורים (Phase 5.5)
פירוט מלא: **`docs/deployment/google-calendar-setup.md`** (runbook צעד-אחר-צעד מ-5.5b).
- [ ] Project + הפעלת **Google Calendar API**
- [ ] **OAuth consent screen** — External; scope יחיד: `https://www.googleapis.com/auth/calendar` (ה-email+timezone נשלפים מה-primary calendar — אין צורך ב-`userinfo.email`/`openid`). ל-MVP: **Testing mode** + test users (אין צורך ב-verification עד 100 users). ⚠️ ל-production מעבר ל-100 users — verification (privacy policy + ToS, 4-6 שבועות)
- [ ] **OAuth 2.0 Client ID** (Web application) — Authorized redirect URI = `GOOGLE_REDIRECT_URI` (`{backend}/auth/google/callback`)
- [ ] **soft-disable flag** (5.5) — `appointment_scheduling_enabled` כבוי כברירת מחדל; bot_config חוסם `bot_schedule_appointment`. ⚠️ **להפעיל רק אחרי שכל מחזור החיים נפרס** (5.5b חיבור יומן ✅ + 5.5c-d runtime ✅ + 5.5e-1 owner panel ✅ + 5.5e-2 lead cancellation ✅) **ו-4 templates התורים אושרו במטא** (`bot_pending_appointment` + `bot_appointment_confirmation` + `bot_appointment_rejected` + `bot_appointment_cancelled_owner`; ראה Templates 4-7 ב-`meta-templates-submission.md`). הפעלה: `update public.app_settings set value='true'::jsonb, updated_by='admin' where key='appointment_scheduling_enabled';`

### תמונות חדשות ברענון-קריאייטיב (A3) — מתג-חירום גלובלי
- [ ] **אין env חדש** — יצירת-התמונות משתמשת ב-`OPENAI_API_KEY` + Storage הקיימים (כמו ה-wizard). הדגל פר-action (`include_images`) דולק אוטומטית בפרודקשן ל-3 סוגי-הרענון (`creative_refresh`/`angle_change`/`offer_change`).
- [ ] **מתג-חירום (kill-switch)** — `optimization_images_forced_off` ב-`app_settings`, כבוי כברירת מחדל (=תמונות מותרות). לכפיית **recycle** מיידי בכל המערכת בלי deploy (גיבוי לדגל הפר-action): `update public.app_settings set value='true'::jsonb, updated_by='admin' where key='optimization_images_forced_off';`. לביטול: `value='false'` (או מחיקת השורה). גובר אבסולוטית על הדגל הפר-action (push קורא אותו חי בכל approve/resume).

### Resend
- [ ] domain verification (SPF/DKIM/DMARC) — אחרת מיילים נופלים ל-spam / sandbox
- [ ] API key (`RESEND_API_KEY`)

### Telegram — התראות-תמיכה לצוות (SUPPORT_ESCALATION_PLAN, סבב-1)
- [ ] **צור Bot** — שלח `/newbot` ל-[@BotFather](https://t.me/BotFather), קבל את ה-token → `TELEGRAM_BOT_TOKEN`
- [ ] **chat_id של הצוות** — צור קבוצה (או ערוץ), הוסף את ה-Bot, ואז השג את ה-`chat_id` (למשל דרך `https://api.telegram.org/bot<token>/getUpdates` אחרי הודעה בקבוצה, או בוט-עזר כמו `@getidsbot`) → `TELEGRAM_ALERT_CHAT_ID` (לקבוצות: מספר **שלילי**). בלי שני אלה — ההתראות מדולגות (skip שקט); ה-thread עדיין נגיש לצוות ב-`/admin/support`

### Green Invoice / Morning
- [ ] API keys (מסלול **Best+**)
- [ ] ⚠️ **אישור רשות המסים** (gov.il) ל-מספר הקצאה — **TTL 3 חודשים, חידוש ידני!** בלעדיו חשבוניות B2B מעל הסף נדחות ע"י רו"ח הלקוח (HTTP 200 מטעה). ראה `docs/integrations/green-invoice/SKILL.md` שלב 2
- [ ] webhook (אם בשימוש — מסלול Extra)

### Pelecard
- [ ] terminal credentials מהספק
- [ ] IPN URL — `{backend}/webhooks/billing/pelecard` (⚠️ פלאקארד לא חותם HMAC — אימות דרך ConfirmationKey + GetTransaction)

### Render
- [ ] **API service** (web) — `python -m app.main` / uvicorn. מגיש גם את הפרונטד ב-`/` מ-`app/web/` (same-origin, בלי CORS)
- [ ] **Worker service** (background) — `python -m app.worker.runner` (jobs queue + crons: cleanup קמפיינים 5דק' · בוט שעתי · רענון טוקן Meta יומי [6.1] · ניטור-אופטימיזציה שעתי [7.4ב] · סריקת high_cpl יזומה יומית [8.1] — מכינה קופי לאישור לקמפיינים חיים עם CPL גבוה, מתריעה `PROPOSAL_READY`). monotonic in-memory, ללא הגדרה חיצונית (וואטסאפ ל-`PROPOSAL_READY` כפוף לאותו gate WABA כמו שאר עדכוני-הסוכן — קו 126; המייל עובד מיד)
- [ ] Environment Group משותף לשני השירותים
- [ ] `preDeployCommand: python -m app.db.migrate`

---

## 3. לפני בטא (gate סופי)
- [ ] Email deliverability — לשלוח 3 ההתראות ל-Gmail/Outlook/Walla, לוודא שלא נופל ל-spam
- [ ] Meta App Review מאושר (permissions production)
- [ ] Green Invoice — אישור רשות המסים פעיל (לא פג)
- [ ] WhatsApp production phone number (אחרי App Review; test number לא מספיק לבטא)
- [ ] ADMIN_TOKEN חזק + רשימת אדמינים מעודכנת
