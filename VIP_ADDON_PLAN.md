# תוכנית מימוש: תוספת VIP (התראות לבעלים) — חיוב מלא

> מסמך תכנון. אין מימוש לפני אישור (CLAUDE.md: "קודם מתכננים, אחר כך מממשים").
> בסיס הקוד הנמדד: ענף `claude/eloquent-bardeen-9kGfT` (HEAD `9240b42`, 19/06/2026).
>
> **⚠️ עודכן — חיוב יחסי מיידי (גובר על "חיוב מאוחד, בלי proration"):** הדלקת VIP על מנוי **active**
> עברה לחיוב **יחסי מיידי** (proration על הימים שנותרו במחזור), payment-first (דגל רק אחרי חיוב מוצלח).
> מומש ב-PR1 (`derive_vip_proration_agorot`) + PR2 (`_charge_vip_addon_now`, migration 0097 +
> `finalize_vip_addon_charge`, `GET /me/subscription/vip/preview`, `confirm_charge_agorot`, חשבונית יחסית).
> נתיב-החיוב provider-agnostic (דרך `billing.charge_saved_card`) — יעבור את הגירת Pelecard→Grow אוטומטית.
> הדלקה ב-**trial** נשארת דגל-בלי-חיוב. **PR3 (כיבוי=cancel-at-period-end, migration 0098) + PR4
> (frontend: preview→אישור→חיוב, מצב "פעיל עד X לא יתחדש", re-enable) — מומשו.**

## סטטוס מימוש
- ✅ **PR-A + PR-B** מומשו (מודל תמחור דו-צירי + migration 0089 + חיבור ל-2 קוראי החיוב + תיאור חשבונית VIP).
- 🟡 **PR-C חלקי:** backend **מלא** (`update_tier`+VIP, `set_vip_addon` + `PATCH /me/subscription/vip`,
  חשיפת `vip_owner_alerts` ב-GET) + **frontend toggle** ב"החבילה שלי" (מסלול 2, end-to-end) + תיקון 3
  אי-התאמות מחיר (VIP 250/350, whatsapp 797, vip-pill) + תיקון **bugbot** (whatsapp+VIP → KeyError לא-תפוס).
  **מסלול-1 frontend ✅ מחווט** (Option A): ב-`ob-finish` ה-frontend קורא `PATCH /me/subscription`
  (`api.choosePackage`) עם `tier`+`volume`+`tos_version`+`vip_owner_alerts`+`vip_alert_phone`; הטלפון נאסף
  כ**שאלה מותנית בשאלון** (נוסח §D.5, מתגלה כשמסומן ה-checkbox, ולידציית E.164). **נדחה:** שלב הכרטיס
  (`start-billing` Pelecard iframe + billing-profile) נשאר mock; gating אמיתי על `has_paid_access` נדחה עמו
  (raw-trial מהרשמה ב-`trial_ends_at=NULL` → אין גישה עד start-billing).
- ✅ **PR-E + PR-E2:** עמוד ליד בודד (`view-lead`: email + שאלות סינון ותשובות) + `GET /me/leads/{id}` +
  deep-link `/leads/{id}` (PR-E); **רשימת לידים** (`view-leads`: פילטר סטטוס + pagination + קליק→ליד) +
  **עריכת סטטוס** בעמוד הליד דרך `PATCH /me/leads/{id}` (PR-E2, CRM-lite פעיל). כולו frontend — ה-backend היה קיים.
  **סטייה ממסמך התכנון:** `screening_answers` מוצג **ישירות** (self-describing `{שאלה: תשובה}`) — **בלי** join
  ל-`screening_questions` של ה-lead-form (אין מפתח-join אמין; ה-dict כבר קריא — ראה §PR-E מעודכן למטה).
- ✅ **PR-D מומש (רדום מאחורי gate):** התראת WhatsApp לבעלים על ליד חדש — migration `0090` (channel
  `whatsapp` + status `skipped` + `vip_lead_alert` + טריגר VIP **בתוך** `insert_lead_and_event`), ענף
  `channel=whatsapp` ב-`handle_send_notification` + gate ייעודי (`_vip_notify_gate_status`) + קו-שולח
  `META_NOTIFY_PHONE_NUMBER_ID`. **ההפעלה בפועל ממתינה ל-3 תנאים:** אישור template ב-Meta +
  `META_NOTIFY_PHONE_NUMBER_ID`/`FRONTEND_URL` + flag `whatsapp_production_ready`. עד אז — skip שקט.
  **2 סטיות ממסמך התכנון (אומתו בקוד + PostgreSQL):** (א) **טריגר ב-RPC**, לא בפייתון — `insert_lead_and_event`
  לא מחזיר lead_id וההתראה `new_lead` כבר נוצרת שם, אז הוספנו ענף VIP מותנה בתוך אותו RPC (אטומי, anchor
  `lead:{uuid}`). (ב) **קו-שולח של הפלטפורמה** — `bot_service._send_template` דורש conversation+קו-עסק שאין
  ל-lead-tier; ה-handler קורא `meta_whatsapp.send_template_message` ישירות עם `META_NOTIFY_PHONE_NUMBER_ID`.

---

## 1. מטרה ותכולה

הוספת **add-on בתשלום** לחבילות Basic/Premium: "התראות VIP לבעלים". הלקוח שרוכש
את התוספת משלם תוספת חודשית קבועה (מעבר למחיר החבילה), והמערכת שולחת לבעל-העסק
התראות על לידים. החיוב **מאוחד** לחיוב החודשי הקיים (לא חיוב נפרד).

### החלטות שננעלו מול בעל-המוצר
| נושא | החלטה |
|---|---|
| היקף החיוב | חיוב מלא דרך פלאקארד (J4), לא רק דגל |
| מבנה החיוב | **מאוחד** — VIP מתווסף לסכום החיוב החודשי הקיים, חשבונית אחת |
| ייצוג ב-DB | **עמודה בוליאנית** `vip_owner_alerts` ב-`subscriptions` (לא טבלה נפרדת) |
| מכסת לידים | VIP **לא** משנה מכסה (quota נשאר לפי tier) |
| תוקף הפעלה/ביטול | מהמחזור הבא — **בלי proration** (תואם החיוב החודשי המאוחד) |
| **ערוץ ההתראה** | **WhatsApp** (template מאושר ע"י Meta) — דרך תשתית WABA הקיימת |
| זכאות | basic/premium בלבד. `whatsapp` tier לא זכאי (אין מחיר במחירון; lead_quota=NULL) |
| **טריגר** | **כל ליד חדש** (בלי סינון "חם/קר") |
| **שעות שקט** | **אין** — שליחה מיידית בכל שעה |
| **לקוחות קיימים** | **אין** — אין צורך ב-reconciliation (שאלה 7.2 נסגרה) |
| **טלפון ההתראה** | **שדה ייעודי** שנאסף בשאלון כשנבחר VIP (לא `bot_config.fallback_value`) |

---

## 2. פער-שורש שהתגלה: מודל התמחור

המחירון הקיים בקוד (`app/integrations/billing.py:49`) ממפה מחיר לפי **tier בלבד**:

```python
_TIER_PRICES_AGOROT = {"basic": 39_700, "premium": 59_700, "whatsapp": 39_700}

def derive_monthly_price_agorot(tier: str) -> int:
    return _TIER_PRICES_AGOROT[tier]
```

אבל המחירון החדש מתמחר לפי **שני צירים** — `tier × lead_quota`:

| | 500 לידים | 1000 לידים |
|---|---|---|
| Basic | 397 | **497** ← לא קיים בקוד |
| Premium | 597 | **897** ← לא קיים בקוד |

הקוד הקיים **לא יודע** לתמחר מנוי של 1000 לידים. זהו באג קיים (לא רק חוסר ב-VIP),
וה-VIP נוגע בו ישירות כי **מחיר ה-VIP עצמו תלוי בכמות הלידים** (250 ל-500, 350 ל-1000).

לפי כלל ה-CLAUDE.md ("פתרון שורש, לא טלאי") — נתקן את מודל התמחור כך שיתבסס על
`(tier, lead_quota, vip)`, ולא נטליא רק את ה-VIP על מודל שגוי ממילא.

### 2.1 מטריצת המחירים המלאה (אגורות)

| מצב | 500 | 1000 |
|---|---|---|
| Basic | 39,700 | 49,700 |
| Basic + VIP | 64,700 | 84,700 |
| Premium | 59,700 | 89,700 |
| Premium + VIP | 84,700 | 124,700 |
| **VIP לבד (add-on)** | **25,000** | **35,000** |

אימות (base + addon = total): 39,700+25,000=64,700 ✓ · 49,700+35,000=84,700 ✓ ·
59,700+25,000=84,700 ✓ · 89,700+35,000=124,700 ✓.

`whatsapp` (79,700) נשאר כפי שהוא — לא חלק מהמחירון החדש, אין לו lead_quota (NULL),
ואינו זכאי ל-VIP.

---

## 3. חלוקה ל-PR-ים

חמישה PR-ים. הסדר המומלץ: **A → B → C** (מסחר+חיוב, מוגדר היטב), **E** (עמוד הליד —
תלות תוכן של ה-template), ואז **D** (ההתראה בפועל) לאחר אישור Meta + PR-E חי.

| PR | תוכן | תלויות |
|---|---|---|
| A | מודל נתונים (דגל + טלפון) + תמחור דו-צירי | — |
| B | חיבור התמחור לחיוב + חשבונית | A |
| C | רכישת VIP (checkbox ראשוני + toggle) + פרונטד | A, B |
| E | עמוד ליד (email + סינון) + `GET /me/leads/{id}` + deep-link | — (backend קיים) |
| D | התראת WhatsApp בפועל | אישור Meta, E |

---

### PR-A — מודל נתונים + תמחור (השורש)

**מטרה:** עמודת הדגל + מודל תמחור דו-צירי. אין נגיעה בזרימת החיוב עצמה (זה PR-B).

#### A.1 Migration — `supabase/migrations/0089_vip_owner_alerts.sql`
> **תוקן במימוש:** 0088 כבר תפוס → ה-migration בפועל הוא **0089**.
```sql
ALTER TABLE subscriptions
  ADD COLUMN vip_owner_alerts boolean NOT NULL DEFAULT false,
  ADD COLUMN vip_alert_phone  text;   -- טלפון להתראות VIP (נאסף בשאלון; ראה D.5)
```
- **שני השדות יחד** — `vip_owner_alerts` (הדגל) + `vip_alert_phone` (היעד). נכתבים
  אטומית באותו CAS (PR-C C.0). `vip_alert_phone` nullable (חובה רק כש-VIP=true,
  נאכף בשכבת ה-service, לא ב-DB constraint — כי VIP=false ⇒ NULL לגיטימי).
- **GRANT:** אין צורך בחדש. Postgres GRANT הוא ברמת טבלה — עמודה חדשה על טבלה
  שכבר granted מכוסה. כתיבת tier/VIP נעשית דרך admin client (`service_role`) שעוקף
  RLS+GRANT ממילא (CLAUDE.md דפוס Postgres 11).
- **RLS:** אין policy חדשה. הקריאה של המשתמש לשורת המנוי שלו כבר קיימת; העמודה
  תיחשף דרכה.
- **שיקוף במודל:** אם קיים ORM model ל-subscriptions (`app/models/`) — להוסיף את
  העמודה ל-`__table_args__`/המחלקה (CLAUDE.md דפוס Postgres 9: סטיית migration).
  לבדוק קיום בזמן המימוש.

#### A.2 מודל תמחור — `app/integrations/billing.py`
מחליפים את המפה החד-צירית במטריצה דו-צירית + פונקציית add-on. **נקודת שינוי אחת.**

```python
# מחיר חבילה בסיסי לפי (tier, lead_quota). אגורות.
_BASE_PRICES_AGOROT: dict[tuple[str, int | None], int] = {
    ("basic",   500):  39_700,
    ("basic",   1000): 49_700,
    ("premium", 500):  59_700,
    ("premium", 1000): 89_700,
    ("whatsapp", None): 39_700,   # ללא מכסת לידים
}

# תוספת VIP לפי כמות לידים. אגורות.
_VIP_ADDON_AGOROT: dict[int, int] = {500: 25_000, 1000: 35_000}


def vip_addon_price_agorot(lead_quota: int | None) -> int:
    """מחיר תוספת VIP באגורות לפי מכסת הלידים. KeyError אם המכסה לא נתמכת."""
    if lead_quota is None:
        raise KeyError("VIP add-on requires a lead_quota (basic/premium only)")
    return _VIP_ADDON_AGOROT[lead_quota]


def derive_monthly_price_agorot(
    tier: str, lead_quota: int | None, vip_owner_alerts: bool = False
) -> int:
    """מחיר חודשי כולל באגורות מ-(tier, lead_quota, vip). KeyError על שילוב לא מוכר."""
    total = _BASE_PRICES_AGOROT[(tier, lead_quota)]
    if vip_owner_alerts:
        total += vip_addon_price_agorot(lead_quota)
    return total
```

- **שינוי חתימה:** `derive_monthly_price_agorot(tier)` → `(tier, lead_quota, vip=False)`.
  כל הקוראים מתעדכנים ב-PR-B. **תוקן במימוש: יש 2 קוראים, לא 1** — `_charge_subscription`
  (~1206) **וגם** `resolve_charge_unknown` (~1471, רק ה-fallback כש-`amount_agorot is None`).
- **KeyError נשמר** כהתנהגות על שילוב לא מוכר (fail-loud; CLAUDE.md External SDK 10 —
  לא-מזוהה → שגיאה, לא ניחוש). הקורא ב-PR-B כבר מאמת `tier ∈ {...}` לפני הקריאה.

#### A.3 טסטים — `tests/test_billing_pricing.py`
- כל 4 שילובי base (basic/premium × 500/1000) + whatsapp.
- כל 4 שילובי VIP → סכום כולל תואם המטריצה.
- `vip_addon_price_agorot(500)==25_000`, `(1000)==35_000`.
- `vip_addon_price_agorot(None)` → KeyError; `derive_monthly_price_agorot("basic", 999)` → KeyError.
- VIP על whatsapp / lead_quota=None → KeyError (לא זכאי).

---

### PR-B — אינטגרציית החיוב (פלאקארד + חשבונית)

> **קריאת-חובה לפני נגיעה ב-`billing.py`/`pelecard.py`/`green_invoice.py`/billing_service**
> (CLAUDE.md): `docs/integrations/pelecard/SKILL.md` + `references/{api-endpoints,
> payment-parameters,error-codes}.md`; `docs/integrations/green-invoice/SKILL.md` +
> `references/{api-reference,document-workflows}.md`. גוצ'ות: J2 vs J4, אגורות vs ש"ח,
> dedupe על `PelecardTransactionId`/`provider_document_id`, JWT שפג בשקט.

#### B.1 חישוב הסכום — `app/services/subscription_service._charge_subscription`
המוקד: שורות ~1090–1156 ב-`subscription_service.py`.

1. **חילוץ שדות נוספים מה-CAS row** (כרגע מחולצים `id/tier/current_period_start/
   billing_card_last_four` ב-1092–1095). ה-`update().execute()` מחזיר representation
   מלא — `lead_quota` ו-`vip_owner_alerts` כבר בשורה, רק להוסיף `.get(...)`:
   ```python
   lead_quota = row.get("lead_quota")          # int | None
   vip = bool(row.get("vip_owner_alerts"))     # קריא רק אחרי PR-A migration
   ```
2. **ולידציה** (הרחבת הבלוק ב-1097–1104): אם `tier in ("basic","premium")` →
   `lead_quota` חייב להיות `int` ב-`{500, 1000}`, אחרת revert + `return False`
   (fail-loud, לא לחייב סכום שגוי). `whatsapp` → `lead_quota` הוא None (תקין).
   `bool` מוחרג מ-`int` (CLAUDE.md — `isinstance(True, int) is True`).
3. **חישוב הסכום** (שורה 1156):
   ```python
   amount = billing.derive_monthly_price_agorot(tier, lead_quota, vip)
   ```
   הסכום הכולל (כולל VIP) זורם ל-`billing.charge_saved_card(amount_agorot=amount, ...)`
   ל-`finalize` → ל-`issue_invoice_for_charge(amount_ils=...)`. **החיוב המאוחד "עובד מעצמו"**.

> **אטומיות (CLAUDE.md כלל 2 + State-machine 1):** אין שינוי בזרימת ה-CAS/attempt/
> finalize. הסכום נגזר מתוך אותה שורה שנתפסה ב-CAS — אין SELECT נוסף, אין race חדש.

#### B.2 חשבונית — `app/services/billing_service.issue_invoice_for_charge`
- **תיאור** (`billing_service.py:139-140`): כש-VIP פעיל, התיאור משקף זאת. נעביר את
  הדגל לפונקציה (פרמטר `vip_owner_alerts: bool = False`) ונרכיב:
  ```python
  label = _TIER_LABELS.get(tier, tier)
  if vip_owner_alerts:
      label = f"{label} + התראות VIP"
  description = f"מנוי Campaign AI — {label} — {period_start[:10]}"
  ```
- **הסכום** (`amount_ils`) כבר נכון — זורם מהחיוב בפועל. אין כפל-מקור-אמת לתמחור.
- **escape:** התיאור הוא טקסט פשוט שנשלח ל-Green Invoice JSON (לא HTML/mrkdwn) —
  אין סינטקס פעיל; אין צורך ב-escape ידני (CLAUDE.md כלל 6). לוודא בזמן המימוש
  שאין הזרקת user-data לא-נשלטת (כאן `label` הוא מחרוזת קבועה).
- **קורא:** `_charge_subscription` כבר מעביר `tier`; נוסיף `vip_owner_alerts=vip`.

#### B.3 טסטים — הרחבת `tests/test_billing*.py` / `test_subscription_service*.py`
- חיוב לקוח `basic/500/vip=true` → `charge_saved_card` נקרא עם `amount_agorot=64_700`.
- חיוב `premium/1000/vip=true` → 124,700.
- חיוב `premium/1000/vip=false` → 89,700 (מאמת שגם ה-base הדו-צירי תוקן).
- חשבונית של לקוח VIP → description כולל "+ התראות VIP".
- idempotency: שני workers מקבילים → רק חיוב אחד (CAS rowcount; קיים, לא נשבר).

---

### PR-C — מסחר: הפעלה/ביטול VIP + פרונטד

VIP נרכש בשני מסלולים, **שניהם** ב-PR-C:
- **מסלול 1 — בחירה ראשונית:** checkbox "תוספת VIP" ליד בחירת החבילה, כבר בהרשמה
  (יחד עם בחירת tier+volume). הדגל נכתב **אטומית** עם בחירת החבילה.
- **מסלול 2 — הוספה מאוחרת:** toggle במסך "החבילה שלי" ללקוח קיים.

> **סטטוס מימוש (2026-07):** אחרי שהחיוב עבר לפר-קמפיין (§8), שני המסלולים הותאמו למודל פר-קמפיין:
> - **מסלול 1 (VIP בהרשמה) — PR-A ✅:** `update_tier` כותב את בחירת-ה-VIP ל-`subscriptions.vip_owner_alerts`+
>   `vip_alert_phone` (כבר עבד). התיקון: `register_campaign_billing` (migration 0133) מעתיק את הבחירה לקמפיין
>   **הראשון בלבד** (`is_first`) ב-go-live → `campaigns.vip_owner_alerts`. `create_campaign` מעתיק tier/quota
>   אבל **לא** vip — כי vip הוא בחירת-הרשמה לקמפיין-ראשון, לא לכל draft (קמפיין-2 בוחר VIP בנפרד; register הוא
>   המקום היחיד שיודע `is_first`). החיוב (`charge_trial_end`→`charge_campaign`) גובה כולל VIP בסוף ה-trial —
>   אין VIP חינם. **הבאג שנסגר:** לפני PR-A הבחירה נבלעה (נרשמה ב-subscriptions ולא הגיעה לקמפיין).
> - **מסלול 2 (ניהול שוטף) — PR-B ✅:** פאנל VIP ב"החבילה שלי" (`renderVipPanelSection`/`renderVipPanel`) חוּוט
>   ל-`GET /campaigns/{id}/vip/preview` + `PATCH /campaigns/{id}/vip` (פר-קמפיין). קמפיין-יחיד→פאנל, כמה→בורר
>   (דפוס `startAgentChat`); off+trial→דגל-חינם, off+paid→preview→אישור→חיוב, on→cancel-at-period-end. `_vipPeriodEnd`
>   מ-`cycle_end_at`. הפאנל הישן פר-user + `api.js` setVipAddon/getVipPreview (endpoints 410) הוסרו.

#### C.0 מסלול 1 — VIP בבחירה הראשונית (`update_tier`)

הזרימה הקיימת: `PATCH /me/subscription` → `update_tier(user_id, tier, volume,
tos_version)` (`subscription_service.py:477`). ה-CAS הקיים מתנה ב-`tier='pending'`
(שורה ~487) ומעדכן `tier+lead_quota+tos` בפעולה אחת.

**הרחבה:**
- **Request model** (`UpdateTierRequest`): שדות חדשים `vip_owner_alerts: bool = False`
  ו-`vip_alert_phone: str | None = None`.
- **Router** (`subscription.py:131-142`): העברת שני השדות ל-service.
- **`update_tier`**: הוספת `vip_owner_alerts` + `vip_alert_phone` ל-`update_payload`
  (שורה ~496) — נכתבים **באותו CAS האטומי** עם tier/quota (CLAUDE.md State-machine 1
  — שדות מקושרים בטרנזקציה אחת). אין race חדש, אין כתיבה שנייה.
- **ולידציה (לפני ה-CAS):**
  - `vip_owner_alerts=True` + `tier ∉ {basic, premium}` → 422 (לא זכאי).
  - `vip_owner_alerts=True` + `vip_alert_phone` חסר/לא-ולידי → 422
    ("נדרש מספר טלפון תקין להתראות VIP"). ולידציה: `re.fullmatch(r'\+?\d{9,15}', ...)`
    אחרי `.strip()` (CLAUDE.md Postgres 8 — whitespace).
  - `vip_owner_alerts=False` → `vip_alert_phone` מתעלם/NULL.
  - fail-loud — לא כותבים דגל/טלפון לא-חוקי.
- **שאלון (frontend, C.2):** שאלת הטלפון מותנית — מופיעה רק אם ה-checkbox של VIP
  מסומן. הערך נשלח ב-`vip_alert_phone` יחד עם בחירת החבילה.
- **תמחור:** אין צורך בשינוי כאן — `_charge_subscription` (PR-B) קורא את הדגל בזמן
  החיוב. הבחירה הראשונית רק כותבת state; החיוב הראשון (trial-end/start-billing)
  כבר יחשב את הסכום הכולל הנכון.

#### C.1 מסלול 2 — Endpoint להוספה/ביטול מאוחר — `app/routers/subscription.py`
מסך ה-router הקיים: `PATCH /me/subscription` (`update_tier`), `POST /me/subscription/
cancel`. נוסיף toggle ל-VIP. שתי אפשרויות מימוש:

- **מועדף:** endpoint ייעודי `PATCH /me/subscription/vip` עם body `{enabled: bool}`.
  פשוט, לא מערבב עם בחירת tier.
- חלופה: הרחבת `update_tier` בשדה `vip_owner_alerts` אופציונלי.

**Service** (`subscription_service`): פונקציה חדשה `set_vip_addon(user_id, enabled)`:
- **CAS אטומי** (CLAUDE.md כלל 2): `UPDATE subscriptions SET vip_owner_alerts=:enabled
  WHERE user_id=:uid AND tier IN ('basic','premium') AND status IN ('active','trial')`
  + בדיקת `rowcount`. כך:
  - VIP מותר רק ל-basic/premium (לא whatsapp/pending) — נאכף ב-`WHERE`, לא בקוד.
  - server-authoritative: admin client, `user_id` מ-JWT (לא מ-body) — כמו `update_tier`.
- **תוקף:** הדגל משתנה מיד, אבל **החיוב משתנה רק במחזור הבא** (החיוב החודשי קורא
  את הדגל בזמן החיוב — `_charge_subscription`). אין proration, אין חיוב מיידי. זה
  עקבי עם החיוב המאוחד וה-cron החודשי (`current_period_start + month`).
- **rowcount=0** → שליפה לבירור הסיבה (404 אין מנוי / 409 tier לא-זכאי / כבר במצב
  המבוקש) — מיפוי ל-HTTPException בעברית גנרית (CLAUDE.md כלל 3, דפוס קריטי 6/7).

#### C.2 פרונטד — `app/web/index.html` + `app/web/api.js`
- **רשת המחירים** (`index.html:947-991`, כרגע basic=397 / premium=597 קשיחים):
  עדכון לכל 8 השורות החדשות + הצגת תוספת VIP (250/350 לפי מכסה). לוודא שהמחירים
  המוצגים נגזרים מאותו מקור-אמת לוגי (או לכל הפחות עקביים עם המטריצה ב-§2.1).
- **בחירה ראשונית — checkbox VIP (מסלול 1):** במסך בחירת החבילה, ליד בחירת
  tier+volume, checkbox "הוסף תוספת VIP" עם הצגת המחיר הדינמי (250 ל-500 לידים /
  350 ל-1000, מתעדכן לפי ה-volume שנבחר). מצב מסומן → נשלח `vip_owner_alerts: true`
  ב-body של `PATCH /me/subscription` יחד עם tier/volume/tos.
  - **gating UI:** ה-checkbox מוסתר/disabled כש-tier לא זכאי (אם יהיה בעתיד tier
    כזה); כרגע basic/premium בלבד נבחרים במסך זה.
  - **סכום מצטבר:** הצגת "סה"כ חודשי" שמתעדכן (חבילה + VIP) כדי שהמשתמש יראה את
    המחיר הסופי לפני אישור.
- **מסך "החבילה שלי" — toggle (מסלול 2):** (`view-myplan`, `index.html:~1389`):
  toggle "תוספת VIP" שקורא ל-`PATCH /me/subscription/vip`. הצגת מצב נוכחי
  (`vip_owner_alerts` מתוך `GET /me/subscription`).
- **XSS:** הזרקת ערכים ל-DOM דרך `textContent`, לא `innerHTML` (CLAUDE.md דפוס קריטי 4).
- `api.js`: wrapper ל-endpoint החדש (`PATCH /me/subscription/vip`) + הוספת
  `vip_owner_alerts` לקריאה הקיימת של בחירת החבילה.

#### C.3 טסטים
**מסלול 1 (בחירה ראשונית — `update_tier`):**
- `update_tier(basic, 500, vip=True)` → השורה נכתבת עם `tier=basic, lead_quota=500,
  vip_owner_alerts=true` **בפעולת CAS אחת** (לא שתי כתיבות).
- `update_tier(whatsapp, None, vip=True)` → 422 (VIP לא זכאי), השורה לא משתנה.
- `update_tier(premium, 1000, vip=False)` → `vip_owner_alerts=false` (default תקין).
- שתי בקשות מקבילות לבחירה → רק הראשונה תופסת (CAS `tier='pending'` קיים, לא נשבר).

**מסלול 2 (`set_vip_addon`):**
- על basic → true; rowcount=1.
- על whatsapp/pending → 409 (לא-זכאי), הדגל לא משתנה.
- על user לא קיים → 404.
- אידמפוטנטי: הפעלה כשכבר פעיל → לא שובר (rowcount=0 → 409 "כבר פעיל" או 200 no-op;
  להחליט בזמן המימוש, עדיף 200 idempotent).

**פרונטד:**
- checkbox מסומן → ה-body כולל `vip_owner_alerts:true`; הסכום המצטבר תואם §2.1.
- שינוי volume (500→1000) מעדכן את מחיר ה-VIP המוצג (250→350).

---

### PR-D — התראת VIP ב-WhatsApp לבעלים

> ✅ **מומש (רדום מאחורי gate)** — ראה §"סטטוס מימוש" למעלה. הפרטים למטה משקפים את **התכנון**; 3 ההבדלים
> בפועל: ה-migration הוא **`0090`** (לא `0089`/`0030` כפי שנכתב ב-D.2); ה**טריגר** ב-`insert_lead_and_event`
> (SQL — לא ענף פייתון כפי ש-D.4 הציע, כי ה-RPC לא מחזיר lead_id); ה-**handler** קורא
> `meta_whatsapp.send_template_message` עם `META_NOTIFY_PHONE_NUMBER_ID` (קו-שולח של הפלטפורמה — לא דרך
> `bot_service._send_template` שדורש conversation+קו-עסק שאין ל-lead-tier). gate ייעודי `_vip_notify_gate_status`
> (flag נפרד `vip_notify_not_configured_alerted`) — לא `_whatsapp_sends_enabled` (של ה-bot tick).

> **קריאת-חובה לפני נגיעה:** `docs/deployment/waba-setup.md` (487 שורות, runbook
> מלא), `app/integrations/meta_whatsapp.py` (function signatures, error classification),
> `app/services/whatsapp_template_registry.py` (מבנה רישום template).

**מטרה:** שליחת הודעת WhatsApp לבעל-העסק על ליד חדש, חסומה מאחורי `vip_owner_alerts`,
תוך כיבוד מנגנון ה-soft-disable הקיים.

#### D.1 הגדרת template חדש + הגשה ל-Meta (קריטי — lead time)

> ⚠️ **תלוי-זמן:** אישור template ב-Meta אורך **1–7 ימים**. **להגיש את ה-template
> מיד עם תחילת PR-A**, במקביל למימוש — אחרת PR-D ייתקע בהמתנה ל-Meta.

- **שם הגדרה ברישום** (`whatsapp_template_registry.py`): `bot_new_lead_owner_vip`.
- **קטגוריה:** **UTILITY** (לא MARKETING) — התראה תפעולית על אירוע ספציפי בחשבון הלקוח,
  עומד בהגדרות Meta ל-UTILITY.
- **שפה:** עברית (`he`).
- **פרמטרים (4) — סופי, מאושר מוצרית:**
  1. `{{1}}` — שם הקמפיין.
  2. `{{2}}` — שם הליד.
  3. `{{3}}` — טלפון הליד.
  4. `{{4}}` — URL לפאנל הליד (deep-link לעמוד הליד הספציפי; ראה PR-E).
- **גוף ההודעה (סופי להגשה):**
  ```
  התקבל ליד חדש בקמפיין {{1}}.

  שם: {{2}}
  טלפון: {{3}}

  לכל הפרטים (כולל תשובות הסינון): {{4}}
  ```
  > ⚠️ "כולל תשובות הסינון" ב-template **מחייב** שעמוד הליד בפאנל יציג את שאלות
  > הסינון ותשובותיהן (PR-E) — אחרת ה-template מבטיח משהו שלא קיים. PR-E הוא תלות
  > תוכן של ה-template.
- **הגשה:** לפי `docs/deployment/meta-templates-submission.md`. env חדש: `META_TPL_NEW_LEAD_OWNER_VIP`
  עם השם הסופי שאישרה Meta (ברירת-מחדל `bot_new_lead_owner_vip_v1`).
- **רישום:** הוספת entry ל-`TEMPLATES` dict ב-`whatsapp_template_registry.py:36-89`
  (פר הדוגמאות הקיימות `bot_opening`/`bot_followup`).

#### D.2 Migration — `0089_sent_notifications_whatsapp_channel.sql`

הטבלה הקיימת (migration `0030`) אוכפת `check (channel in ('email'))` (CLAUDE.md דפוס
Postgres 9: enum-as-check ב-migration). צריך להרחיב:

```sql
ALTER TABLE sent_notifications DROP CONSTRAINT sent_notifications_channel_check;
ALTER TABLE sent_notifications ADD CONSTRAINT sent_notifications_channel_check
  CHECK (channel IN ('email', 'whatsapp'));
```

- **שיקוף במודל ORM** אם יש (`app/models/notification*.py`).
- **VARCHAR length** (`whatsapp`=8 chars) — לבדוק שהעמודה רחבה מספיק (`email`=5; אם
  היא `VARCHAR(5)` זה באג קיים שצריך לתקן באותו migration). CLAUDE.md Postgres 2.

#### D.3 NotificationType + ערוץ ב-`notification_service.py`

- **enum חדש:** `NotificationType.VIP_LEAD_ALERT` (בנוסף ל-9 הקיימים).
- **send_agent_notification:** הוספת `channel: str = "email"` (כבר רמוז ב-roadmap
  הקיים, line 124-125). הצופן שמטפל ב-channel הוא ה-handler ב-worker — כאן רק להעביר.
- **handler חדש** ב-`app/worker/handlers.py` ל-channel=whatsapp:
  - חילוץ owner phone (D.5), template params מה-payload, קריאה ל-`bot_service._send_template`.
  - **idempotency (CLAUDE.md כלל 9):** ה-handler כבר נכנס דרך `sent_notifications`
    שיש לו UNIQUE על (user_id, type, anchor_id) — reserve-first קיים. לבדוק
    שקריאת ה-INSERT באמת קודמת ל-`await send_template_message`, ושעל IntegrityError
    (כפילות) ה-handler מחזיר success (idempotent, לא יורה שוב).

#### D.4 טריגר — ליד חדש

נקודת ה-trigger היחידה: ה-webhook של leadgen (Phase 7.2.5). יש להוסיף שם הסתעפות:

```python
# פסאודו-קוד (המיקום: handler ה-webhook אחרי persist של הליד)
sub = await get_subscription(user_id)
if sub.vip_owner_alerts:
    await notification_service.send_agent_notification(
        user_id=user_id,
        notification_type=NotificationType.VIP_LEAD_ALERT,
        anchor_id=f"lead:{lead.id}",
        channel="whatsapp",
        payload={
            "campaign_name": lead.campaign_name,  # {{1}}
            "lead_name": lead.name,               # {{2}}
            "lead_phone": lead.phone,             # {{3}}
            "panel_url": panel_url(lead),         # {{4}} — deep-link, ראה PR-E
        },
    )
```

- **גישה לפרומפטים — לא רלוונטי כאן** (CLAUDE.md כלל 8): גוף ההודעה מקודד ב-template
  של Meta, לא ב-`app/prompts/`. אין `prompts_service.build`.
- **TODO קיים** ב-`app/services/google_calendar_service.py:270` (`owner_alert_sent_at`)
  הוא מאותה משפחה — לפתור אותו בנפרד או במסגרת PR-D מורחב.

#### D.5 מקור מספר הטלפון של הבעלים — **שדה ייעודי**

**החלטה:** מספר הטלפון להתראות VIP הוא **שדה ייעודי** `vip_alert_phone`, **לא**
`bot_config.fallback_value` (זה משמש להעברת שיחות בבוט — כוונה אחרת, ערך אחר).
המספר נאסף **בשאלון** בעת בחירת VIP (שאלה מותנית: "מה מספר הטלפון שתרצה לקבל אליו
התראות על לידים?"). ראה C.0 — נאסף יחד עם בחירת החבילה.

- **מיקום העמודה:** `vip_alert_phone text` ב-`subscriptions`, נכתב **אטומית** עם
  `vip_owner_alerts` (אותו CAS — שדות מקושרים, CLAUDE.md State-machine 1). מתווסף
  ל-migration של PR-A (`0089`), לא migration נפרד. ראה §PR-A A.1.
- **ולידציה בכתיבה (C.0):** אם `vip_owner_alerts=True` → `vip_alert_phone` חובה
  ולידי (E.164: `re.fullmatch(r'\+?\d{9,15}', phone)`), אחרת 422. אם VIP=False →
  השדה אופציונלי/NULL.
- **נורמליזציה:** שמירה אחידה (E.164 מלא או stripped — לבחור פורמט אחד; ה-handler
  של WhatsApp מצפה ל-`.lstrip("+")` כמו `_notify_owner_cancellation`). לתעד את
  הפורמט הקנוני בעמודה.
- **edge — `vip_alert_phone` ריק בזמן שליחה** (לא אמור לקרות אם C.0 אכף, אבל הגנה):
  skip שקט + `capture_alert` edge-triggered (CLAUDE.md cron 11 + כלל 12ד); הליד עובר רגיל.
- **ביטול VIP (מסלול 2 / set_vip_addon):** האם לאפס `vip_alert_phone`? המלצה: **לא**
  לאפס — לשמור לנוחות הפעלה חוזרת. רק `vip_owner_alerts=false` חוסם שליחה.

#### D.6 אינטגרציית guards — `_whatsapp_sends_enabled`

ה-handler **חייב** לעבור דרך `bot_service._send_template` (שכבר מבצע את הבדיקה
ב-line 241), **או** לקרוא ל-`_whatsapp_sends_enabled()` ישירות לפני שליחה.

- **3 התרחישים** (`bot_cron_service.py:56-76`):
  1. **soft-disabled** (`whatsapp_production_ready=false`) → skip שקט, no-op. השארת
     הליד ללא alert זה תקין — production עדיין לא חי. **לא לסמן status=failed.**
  2. **production=true אבל env חסר** → alert edge-triggered (קיים, לא לכפול) + skip.
  3. **all OK** → שליחה רגילה.
- **התנהגות סימטרית** (CLAUDE.md כלל 12): אסור להחזיר success כשלא נשלח בפועל. ה-status
  ב-`sent_notifications` יהיה `pending` (cron retry) או `skipped` (terminal, no-retry) —
  לפי המתודולוגיה הקיימת ב-cron של notifications. עדיף **skipped** עם reason ברור, אחרת
  ה-cron יפצח אותו לנצח (CLAUDE.md cron 1 — terminal state explicit).

#### D.7 שגיאות Meta — מיפוי קיים

`meta_whatsapp.py:96-107` מסווג כבר:
- `MetaWhatsAppAuthError` (401 / code 190 — token פג) → permanent + `capture_alert`.
- `MetaWhatsAppTransientError` (429 / 5xx) → retry דרך ה-worker (`TransientError`
  mixin, CLAUDE.md כלל 11).
- `MetaWhatsAppPermanentError` (4xx אחרים) → terminal, סימון `send_failed` בליד הזה
  (לא במנוי הכולל — אסור לשרוף VIP בגלל ליד אחד).
- `MetaWhatsAppUnexpectedError` (status לא צפוי) → terminal + Sentry (כלל External
  SDK 10: unknown ≠ transient).

**אין שינוי במסווג** — VIP משתמש בקיים. רק לוודא שה-handler החדש מוגדר ב-runner
עם ה-mapping הנכון (`TransientError` נופל ל-retry).

#### D.8 escape של נתונים בתוך פרמטרי template (CLAUDE.md כלל 6)

ל-WhatsApp templates **אין סינטקס פעיל בפרמטרים** (לא HTML, לא markdown). הפרמטרים
מועברים כ-JSON ל-Graph API ומוטמעים מילולית. **אין צורך ב-escape טקסטואלי**.

**יש** מגבלות פורמט (Meta דוחה):
- אסור newline (`\n`) בפרמטר → ולידציה: `param.replace("\n", " ")`.
- אורך מקסימלי 1024 chars לפרמטר → truncate + `…` בסוף.
- אסור 4+ רווחים רצופים → normalize whitespace.

מומלץ helper קצר ב-`whatsapp_template_registry.py`:
```python
def sanitize_template_param(s: str, max_len: int = 1024) -> str:
    s = re.sub(r"\s+", " ", s).strip()
    return s[: max_len - 1] + "…" if len(s) > max_len else s
```

#### D.9 טסטים — `tests/test_vip_owner_alerts.py`
- ליד חדש + `vip_owner_alerts=true` + production-ready=true → קריאה אחת ל-`send_template_message`
  עם הפרמטרים הנכונים.
- VIP=false → אפס שליחות.
- production-ready=false → אפס שליחות, status=skipped (לא failed, לא pending).
- production-ready=true + env חסר → אפס שליחות, alert edge-triggered (mock `capture_alert`).
- `vip_alert_phone` ריק → skip + alert.
- `vip_alert_phone` לא תקין (לא ספרות) → skip + alert.
- transient של Meta (5xx) → retry (mixin), אחרי 3 ניסיונות → `sent_notifications.status=failed`.
- permanent של Meta (4xx ספציפי, לא 401) → status=failed מיד, ליד עובר רגיל.
- כפילות (אותו ליד נכנס פעמיים בגלל webhook retry) → התראה אחת בלבד
  (UNIQUE על anchor_id).
- ליד נכנס + הוא לא VIP אחרי-זה (toggle off באמצע) → התראה לא נשלחת (בדיקה
  בזמן ה-handler, לא בזמן יצירת ה-notification).

> **חסם:** PR-D דורש: (1) אישור Meta של ה-template (1–7 ימים — להתחיל מיד),
> (2) **PR-E** (עמוד הליד עם תשובות הסינון — היעד של `{{4}}` ומה שה-template מבטיח).
> טריגר/שעות-שקט כבר הוכרעו (כל ליד, בלי שקט). עד אז הדגל קיים בחיוב (A–C), ההתראה
> לא נשלחת בפועל.

---

### PR-E — עמוד ליד בפאנל (email + שאלות סינון ותשובות)

> **חדשות טובות:** ה-backend **כבר שומר** את כל הנתונים. הפער הוא כמעט כולו frontend
> + endpoint אחד חסר. זו תלות תוכן של ה-template ב-PR-D (`{{4}}` deep-link + "תשובות הסינון").

#### מה כבר קיים (לא לבנות מחדש)
- `leads.contact_email` — נשמר, מוחזר ב-API (`0026_leads.sql:23`).
- `leads.screening_answers JSONB` — נשמר מה-webhook (`lead_intake_service._parse_field_data`),
  מוחזר ב-`LeadResponse` (`app/models/lead.py:16-17`).
- הגדרות שאלות הסינון פר-קמפיין: `lead_form_fields.configuration.screening_questions`
  (עד 4 שאלות), נגיש דרך `GET /campaigns/{campaign_id}/lead-form`.

#### מה חסר (הפער)
> **✅ מומש (PR-E, מינימום).** עדכון מהמימוש: ה-**join** ל-`screening_questions` (להלן סעיף 2) **בוטל** —
> `screening_answers` כבר self-describing (`{question_text: answer}`, המפתח = ה-field name של Meta = טקסט
> השאלה; אומת ב-`tests/test_lead_intake.py`). אין מפתח-join אמין (`screening_questions` מכיל רק
> `question`+`answers`, בלי name/id). לכן הפרונטד **מציג את `screening_answers` ישירות** — robust, בלי
> קריאת `lead-form` נוספת. `get_lead` משכפל את `_LEAD_COLUMNS` הקיים. רשימת לידים מלאה = **PR-E2** (נדחה).

1. **Backend — `GET /me/leads/{lead_id}`** (`app/routers/leads.py`): כיום יש רק
   `GET /me/leads` (list) ו-`PATCH /me/leads/{id}` (status). חסר fetch של ליד בודד.
   - service: `leads_service.get_lead_by_id(user_id, lead_id)` — שליפה עם **בעלות**
     (WHERE user_id — לא לחשוף לידים של לקוח אחר; CLAUDE.md דפוס קריטי 6/7).
   - 404 אם לא קיים/לא שייך למשתמש (לא לחשוף קיום).
2. **Frontend — עמוד/מודאל ליד** (`app/web/index.html`): אין כיום `view-leads` כלל.
   - תצוגת ליד בודד: שם, טלפון, **email**, סטטוס, זמן.
   - **שאלות סינון + תשובות:** join בצד-לקוח — שאלות מ-`GET /campaigns/{id}/lead-form`
     (`configuration.screening_questions`) × תשובות מ-`screening_answers` (keyed by
     question name). render: לכל שאלה → התשובה של הליד.
   - **XSS (CLAUDE.md דפוס קריטי 4):** שאלות ותשובות מקורן ב-Meta/משתמש — הזרקה דרך
     `textContent`, לעולם לא `innerHTML`. זו נקודת-תורפה אמיתית (תוכן מבחוץ).
3. **Deep-link** ל-ליד בודד (היעד של `{{4}}`): דפוס URL (e.g., `/app#lead/{uuid}`),
   ניתוב ב-frontend שפותח את עמוד הליד ישירות. `panel_url(lead)` ב-PR-D מרכיב אותו.

#### היקף ההחלטה
ה-VIP template מבטיח "כולל תשובות הסינון" — לכן **המינימום ל-PR-E** הוא: endpoint
ליד בודד + deep-link + עמוד שמציג email + Q&A. תצוגת **רשימת לידים** מלאה (`view-leads`
עם טבלה/חיפוש) היא נחמדה-שיהיה אבל לא חוסמת את ה-VIP — אפשר לפצל ל-PR-E2 נפרד.

#### טסטים
- `GET /me/leads/{id}` של ליד שייך → 200 עם email + screening_answers.
- ליד של משתמש אחר → 404 (בעלות).
- ליד לא קיים → 404.
- frontend: ליד עם screening_answers ריק → תצוגה תקינה (אין שאלות), בלי קריסה.
- frontend: שאלה ללא תשובה (הליד דילג) → "—" / "לא נענה", לא undefined.
- frontend: תו `<` בתשובה → מוצג כטקסט (textContent), לא נפרש כ-HTML.

> **חסם PR-D↔PR-E:** ה-deep-link והעמוד צריכים להיות חיים **לפני** שה-VIP template
> מתחיל לשלוח (אחרת `{{4}}` מצביע לשום-מקום). סדר: PR-E לפני הפעלת PR-D.

---

## 4. עדכוני תיעוד (לפי CLAUDE.md — באותו PR)

| מסמך | מתי | מה |
|---|---|---|
| `docs/spec.md` | PR-A | מחירון דו-צירי + מודל ה-VIP add-on (מקור אמת) |
| `docs/ROADMAP.md` | סוף כל PR | ✅ ליחידת הסשן |
| `docs/SETUP_CHECKLIST.md` | PR-D | **env חדש:** `META_TPL_NEW_LEAD_OWNER_VIP` (שם ה-template שאישרה Meta) + תזכורת שהאישור חיצוני |
| `docs/frontend-integration.md` | PR-C, PR-E | מסך "החבילה שלי" → `PATCH /me/subscription/vip`; עמוד ליד → `GET /me/leads/{id}` |
| `docs/deployment/meta-templates-submission.md` | PR-D | תוספת template חדש לרשימה — `bot_new_lead_owner_vip` |
| `docs/deployment/waba-setup.md` | PR-D | אם נדרשת התאמה במדריך (בד"כ לא — התשתית קיימת) |
| `docs/backend-gaps.md` | לפי הצורך | אם יתגלה פער backend בחיווט הפרונטד |

---

## 5. מטריצת edge-cases (self-check לפני commit)

| # | Edge | טיפול | PR | כלל CLAUDE.md |
|---|---|---|---|---|
| 1 | שילוב tier/quota לא מוכר | KeyError (fail-loud), לא ניחוש מחיר | A | External SDK 10 |
| 2 | `lead_quota=None` + VIP | KeyError — whatsapp לא זכאי | A | External SDK 10 |
| 3 | `bool` כ-`int` ב-quota | החרגה מפורשת של bool | B | Postgres / numeric |
| 4 | חיוב כפול (2 workers) | CAS rowcount קיים — לא נשבר | B | כלל 2, Webhooks 1 |
| 5 | כשל חשבונית אחרי חיוב | reserve-first + cron retry קיים — לא נשבר | B | Webhooks 1 |
| 6 | toggle VIP race | CAS `UPDATE ... WHERE ... rowcount` | C | כלל 2 |
| 7 | VIP על tier לא-זכאי | נחסם ב-`WHERE`, 409 | C | State-machine 7 |
| 8 | תוקף הפעלה | מהמחזור הבא, בלי proration | C | החלטת מוצר |
| 9 | escape בתיאור חשבונית | טקסט פשוט, אין סינטקס פעיל | B | כלל 6 |
| 10 | פרמטרי template ל-Meta | sanitize: newline/whitespace/length | D | כלל 6 (variant) |
| 11 | idempotency התראת WhatsApp | reserve-first INSERT עם UNIQUE על `(user_id, type, anchor_id)` | D | כלל 9 |
| 12 | WABA soft-disabled | skip שקט, status=skipped (terminal, לא pending) | D | cron 1, cron 11 |
| 13 | WABA production-on + env חסר | skip + alert edge-triggered (קיים) | D | cron 11 |
| 14 | `fallback_value` ריק/לא תקין | skip + alert edge-triggered, ליד עובר רגיל | D | cron 11, כלל 12ד |
| 15 | Meta transient (5xx/429) | retry דרך `TransientError` mixin (קיים) | D | כלל 11 |
| 16 | Meta auth/permanent | terminal פר-התראה, **לא** פר-מנוי (לא לשרוף VIP בגלל ליד אחד) | D | External SDK 10 |
| 17 | toggle VIP off אחרי שה-notification נוצר | בדיקה גם בזמן ה-handler (לא רק בטריגר) — כלל 12ד | D | כלל 9, כלל 12 |
| 18 | webhook leadgen מגיע פעמיים | UNIQUE על anchor → INSERT שני נדחה (23505) → idempotent | D | Webhooks 1 |
| 19 | `vip_alert_phone` חובה כש-VIP=true | ולידציה 422 לפני CAS (E.164) | A/C | דפוס קריטי 7 |
| 20 | `GET /me/leads/{id}` של ליד זר | 404 (בעלות, WHERE user_id) | E | דפוס קריטי 6/7 |
| 21 | שאלת/תשובת סינון ב-DOM | `textContent` בלבד (תוכן מבחוץ) | E | דפוס קריטי 4 (XSS) |
| 22 | שאלה ללא תשובה (דילוג) | "—"/"לא נענה", לא undefined | E | ולידציית external input |
| 23 | `{{4}}` deep-link ללא עמוד | PR-E חי לפני הפעלת PR-D | D/E | חסם תוכן |

---

## 6. סדר ביצוע מוצע

> **מקבילות:** הגשת ה-template ל-Meta (D.1) מתחילה **ביום הראשון של PR-A**, כי
> האישור אורך 1–7 ימים ואחרת PR-D ייתקע. במקביל מתקדמים A→B→C על הקוד.

1. **יום 0 (במקביל ל-PR-A):** הגשת `bot_new_lead_owner_vip` ל-Meta לאישור.
2. **PR-A** — migration `0089` (דגל + `vip_alert_phone`) + מודל תמחור + טסטים. אין
   סיכון production: הדגל default=false, התמחור עוד לא בשימוש עד שמחברים ב-B.
3. **PR-B** — חיבור התמחור לזרימת החיוב + חשבונית. **אין reconciliation** — אין
   לקוחות קיימים (שאלה 7.2 נסגרה).
4. **PR-C** — מסחר: checkbox בבחירה הראשונית (+ שאלת טלפון) + toggle ב-My Plan +
   פרונטד. מכאן לקוחות יכולים לרכוש VIP. החיוב כבר עובד נכון (B); ההתראה עוד לא.
5. **PR-E** — עמוד הליד (email + שאלות סינון) + `GET /me/leads/{id}` + deep-link.
   **חייב להיות חי לפני PR-D** (היעד של `{{4}}`). ברובו frontend; backend מינימלי.
6. **PR-D** — התראת WhatsApp בפועל. **תלוי:** (א) Meta אישרה את ה-template (יום 0–7),
   (ב) PR-E חי. טריגר/שעות-שקט כבר הוכרעו. עד שלא הושלמו — PR-D ממתין.

---

## 7. שאלות פתוחות לפני התחלה

הוכרעו ✅: טריגר (כל ליד) · שעות-שקט (אין) · לקוחות קיימים (אין → אין reconciliation) ·
גוף ה-template (סופי, §D.1) · מקור הטלפון (שדה ייעודי `vip_alert_phone` נאסף בשאלון).

נותרו:
1. **ענף עבודה:** הענף `claude/admiring-darwin-spn1u6` מאחור 22 קומיטים אחרי
   `bardeen`. לפני PR-A צריך merge/rebase מ-bardeen (החלטה תפעולית, לא חלק מהתוכנית).
2. **היקף PR-E:** המינימום (עמוד ליד בודד + deep-link) חוסם את VIP. האם לבנות גם
   **רשימת לידים מלאה** (`view-leads` עם טבלה/סינון) באותו PR, או לפצל ל-PR-E2
   נפרד? המלצה: **לפצל** — המינימום מספיק ל-VIP, הרשימה היא feature עצמאי.
3. **טיפול ב-`vip_alert_phone` בביטול VIP:** לאפס או לשמור? המלצה: **לשמור** (נוחות
   הפעלה חוזרת); רק הדגל חוסם.
