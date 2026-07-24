# תוכנית: חיוב VIP מיידי בהדלקה (יחסי) + ביטול בסוף מחזור

> **סטטוס:** תכנון מאושר — טרם מימוש. דורש אישור סופי לפני קוד (CLAUDE.md: "קודם מתכננים").
> **הקשר:** באג מוצרי — לקוח שמדליק VIP באמצע מחזור מקבל אותו **חינם** עד החיוב הבא. צריך חיוב מיידי.
> **מסמך רקע:** `docs/VIP_ADDON_PLAN.md` (העיצוב המקורי — חיוב מאוחד, בלי proration).

---

## 1. מטרה

תוספת ה-VIP (התראות WhatsApp לבעל-העסק על לידים) עולה ₪250–350 לחודש. כיום, מי שמדליק
אותה באמצע מחזור החיוב נהנה ממנה **חינם** עד מועד החיוב הבא. המטרה: **חיוב מיידי ויחסי**
בהדלקה, ו**ביטול בסוף מחזור** — סימטרי לאופן שבו ביטול מנוי עובד היום.

---

## 2. מה הקוד עושה היום (מאומת)

### 2.1 הדלקת VIP
`set_vip_addon` (`subscription_service.py:1682`) מבצע CAS אטומי שמדליק את `vip_owner_alerts=true`
**מיד, בלי חיוב**. החיוב מתמזג לחיוב החודשי הבא: `_charge_subscription` קורא את הדגל בזמן
החיוב ומחשב `derive_monthly_price_agorot(tier, quota, vip)` — סכום **שטוח**, חבילה + VIP מלא
**קדימה**. **אין** שום לוגיקת proration / ימים-שנשארו / חיוב-למפרע בקוד (אומת ב-grep).

**מסקנה:** התקופה החלקית (מהדלקה עד מועד החיוב) באמת חינם. זה giveaway, לא חיוב-בדיעבד.

### 2.2 ביטול מנוי (הדפוס שנשקף)
`cancel_subscription` (`subscription_service.py:1611`) משנה `status='canceled'` **מיד**, אך **לא
שולל גישה מיד**. `has_paid_access` (`models/subscription.py:118`) הוא **שדה מחושב**: למנוי
`canceled` מחזיר `true` עד `current_period_start + חודש`. כלומר **הסטטוס מתחלף מיד, הגישה
נשמרת עד סוף המחזור ששולם** — בלי cron ייעודי, מחושב בקריאה.

---

## 3. ההחלטות (ננעלו)

| # | החלטה |
|---|--------|
| 1 | **הדלקה (active, אמצע מחזור):** חיוב **יחסי מיידי** על הימים שנשארו; הדגל נדלק **רק אחרי חיוב מוצלח** (payment-first). |
| 2 | **טריאל:** דגל בלבד, **בלי** חיוב; VIP מתמזג לחיוב הראשון (יום 8). שומר על "אין חיוב בטריאל". |
| 3 | **בלי סף מינימלי** — גובים את הסכום היחסי גם אם זעיר. |
| 4 | **כיבוי (active):** **cancel-at-period-end** — ההתראות נמשכות עד מועד החיוב הבא, **בלי החזר**. |
| 5 | **כיבוי בטריאל:** כיבוי מיידי של הדגל (לא שולם כלום → אין תקופה לשמר). |

---

## 4. נוסחת ה-proration

מחזור anniversary: `[current_period_start, current_period_start + חודש)`.
```
ימים_שנשארו = (מועד_חיוב_הבא − עכשיו)
ימי_המחזור  = (מועד_חיוב_הבא − current_period_start)
חיוב_יחסי   = round(ימים_שנשארו / ימי_המחזור × מחיר_VIP_חודשי_אגורות)
```
- החיוב המיידי מכסה `[עכשיו, מועד_חיוב_הבא)`; חיוב ה-anniversary הבא מכסה
  `[מועד_חיוב_הבא, +חודש)` — **בלי חפיפה, בלי double-charge**.
- חישוב לפי ימים בפועל (חודשי anniversary משתנים 28–31 יום). עיגול לאגורה (`round`).
- קצוות: הדלקה ביום הראשון של המחזור → ≈ מחיר מלא; ביום האחרון → סכום זעיר (גובים, אין סף).

---

## 5. מכונת המצבים — 3 מצבי VIP על מנוי active

```
        ┌─────────┐  הדלקה: חיוב יחסי מיידי → success   ┌──────────────────┐
        │ 1. כבוי  │ ──────────────────────────────────► │ 2. פעיל, יתחדש   │
        │ off      │ ◄──────────────────────────────────│ vip_owner_alerts │
        └─────────┘   anniversary + cancel-pending       └────────┬─────────┘
             ▲         (cron: כיבוי + חבילה-בלבד)                  │ כיבוי
             │                                                     ▼
             │                                          ┌──────────────────────┐
             └──────────────────────────────────────── │ 3. פעיל, מבוטל-בסוף   │
                  anniversary (cron: off + חבילה-בלבד)  │ vip_cancel_at_period │
                                                        │ _end=true            │
                     הדלקה-מחדש (ניקוי הדגל, בלי חיוב)  └──────────┬───────────┘
                  ◄────────────────────────────────────────────────┘ (3→2)
```

| מצב | `vip_owner_alerts` | `vip_cancel_at_period_end` | התראות נשלחות? |
|-----|--------------------|----------------------------|----------------|
| 1. כבוי | false | false | לא |
| 2. פעיל, יתחדש | true | false | כן |
| 3. פעיל, מבוטל-בסוף | true | true | **כן** (עד ה-anniversary) |

**המעברים:**
- **1→2 (הדלקה):** חיוב יחסי מיידי → על הצלחה: `vip_owner_alerts=true`.
- **2→3 (כיבוי):** `vip_cancel_at_period_end=true`, **משאירים `vip_owner_alerts=true`**. בלי חיוב/החזר.
- **3→2 (הדלקה-מחדש לפני המועד):** ניקוי `vip_cancel_at_period_end` בלבד. **בלי חיוב** — כבר שולם על המחזור.
- **3→1 (anniversary, cron):** `_charge_subscription` רואה את הדגל → `vip_owner_alerts=false` + ניקוי + **חיוב חבילה-בלבד**.
- **2→2 (anniversary, cron):** רגיל — חבילה + VIP מלא, נשאר on.

**סימטריה:** ה-proration שילם על `[הדלקה, מועד-חיוב)`; הכיבוי שומר VIP בדיוק עד אותו מועד.

---

## 6. נתיב החיוב המיידי (הדלקה על active)

reuse של התשתית הקיימת (reserve-first + idempotency + finalize, כמו `_charge_subscription`):

1. **בדיקת מצב 3 קיים** — אם כבר יש חיוב VIP מוצלח למחזור הזה (re-enable): רק לנקות `vip_cancel_at_period_end`, בלי חיוב. (מעבר 3→2.)
2. **reserve-first:** INSERT `billing_charge_attempts` עם `attempt_type='vip_addon'` + idempotency key `{sub_id}:vip_addon:{period_anchor}`. UNIQUE → double-click חוסם חיוב כפול.
3. **חישוב** `derive_vip_proration_agorot(...)` (נוסחה §4).
4. **חיוב:** `billing.charge_saved_card(amount=proration, reference=attempt_id)`.
5. **תוצאה:**
   - **success** → RPC אטומי `finalize_vip_addon_charge`: `vip_owner_alerts=true` + `vip_alert_phone` + attempt→succeeded. **לא נוגע** ב-`current_period_start`/status. → חשבונית (`issue_invoice_for_charge`, תיאור "VIP — חלק יחסי, X ימים").
   - **declined (permanent)** → הדגל **לא** נדלק; מחיקת/סימון ה-attempt; שגיאה ברורה בעברית (402).
   - **מעורפל (transient/timeout)** → הדגל לא נדלק; שגיאת "נסה שוב"; ה-idempotency key מגן מ-double-charge ב-retry (אם החיוב כן עבר — ה-retry מזהה כפילות ומדליק). **נקודת הכסף הרגישה — סקירה קפדנית.**

---

## 7. שינוי הסכמה

```sql
-- migration חדש:
alter table public.subscriptions
  add column if not exists vip_cancel_at_period_end boolean not null default false;
```
- אין צורך ב-`vip_started_at` — המודל המיידי גובה עכשיו, בלי "זיכרון מתי הודלק".
- `attempt_type='vip_addon'` — ערך טקסט חדש ב-`billing_charge_attempts`, בלי שינוי סכמה.

---

## 8. קבצים שמושפעים

| קובץ | שינוי |
|------|--------|
| `subscription_service.py` `set_vip_addon` | הסתעפות: active+enable→נתיב חיוב; trial→דגל בלבד; active+disable→cancel-at-period-end; trial+disable→כיבוי מיידי; re-enable (מצב 3)→ניקוי דגל |
| `subscription_service.py` (חדש) | `_charge_vip_addon_now` — proration, reserve-first, charge, finalize |
| `subscription_service.py` `_charge_subscription` | אם `vip_cancel_at_period_end` → חבילה-בלבד + כיבוי `vip_owner_alerts` + ניקוי הדגל |
| `billing.py` (חדש) | `derive_vip_proration_agorot(...)` |
| migration חדש | עמודה `vip_cancel_at_period_end` + RPC `finalize_vip_addon_charge` |
| `routers/subscription.py` | מיפוי שגיאת decline → 402 |
| `billing_service.issue_invoice_for_charge` | reuse + תיאור "VIP — חלק יחסי (X ימים)" |
| `models/subscription.py` `SubscriptionResponse` | חשיפת `vip_cancel_at_period_end` (UI: "פעיל עד <תאריך>, לא יתחדש") |
| `app/web/index.html` | הצגת הסכום היחסי לפני אישור + טיפול ב-decline + תצוגת מצב 3 |

---

## 9. רישום סיכונים

| # | סיכון | חומרה | נטרול |
|---|--------|--------|--------|
| V1 | **double-charge בהדלקה** (double-click / retry) | 🔴 כסף | reserve-first + UNIQUE idempotency key `{sub}:vip_addon:{period}` |
| V2 | **תוצאה מעורפלת** — הדגל נדלק בלי חיוב ודאי, או חיוב בלי דגל | 🔴 כסף | payment-first (דגל רק ב-RPC אחרי success); retry idempotent |
| V3 | **עקביות "האם VIP פעיל?"** — 3 מצבים, כמה קוראים | 🟠 | קורא ההתראות נשען על `vip_owner_alerts` (נשאר נכון); UI/cron מכירים `vip_cancel_at_period_end` |
| V4 | **race עם ה-cron החודשי** בגבול ה-anniversary | 🟡 | period anchors שונים → idempotency keys שונים → בטוח |
| V5 | **UX decline בהדלקה** — נתיב כשל חדש | 🟡 | שגיאה ברורה בעברית; הדגל לא נדלק |

---

## 10. טסטים

- **proration formula:** קצוות (יום ראשון/אחרון של מחזור, חודש 28/31 יום), עיגול.
- **הדלקה:** success → דגל+חשבונית; declined → אין דגל+402; ambiguous → אין דגל+retry idempotent.
- **double-click:** חיוב אחד בלבד (UNIQUE).
- **כיבוי active:** מצב 3, התראות נמשכות, אין החזר; anniversary → off + חבילה-בלבד.
- **הדלקה-כיבוי-הדלקה אותו מחזור:** חיוב אחד, אין re-charge ב-re-enable.
- **טריאל:** הדלקה=דגל בלי חיוב; כיבוי=off מיידי; חיוב ראשון כולל/לא-כולל VIP בהתאם.

---

## 11. חלוקת PR

- **PR1 — `derive_vip_proration_agorot` + טסטי נוסחה** (כולל קצוות). `[S]`
- **PR2 — נתיב החיוב המיידי** + RPC `finalize_vip_addon_charge` + חשבונית + טיפול בכשל + idempotency + עמודה `vip_cancel_at_period_end`. טסטים. `[L]` — **לב העבודה**.
- **PR3 — ביטול cancel-at-period-end** ב-`set_vip_addon` + לוגיקת ה-cron החודשי. טסטים. `[M]`
- **PR4 — frontend** (סכום יחסי, decline, תצוגת מצב 3). `[M]`

> סדר: PR1 → PR2 → PR3 → PR4. PR2 ו-PR3 שניהם נוגעים ב-`set_vip_addon` — לתאם.
</content>
