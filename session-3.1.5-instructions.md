# Session 3.1.5 — הוראת ביצוע מלאה ל-Claude Code

> **קלט יחיד.** יישם בדיוק לפי המסמך. אין "אופטימיזציות," אין מעקפים, אין החלטות עצמאיות.
>
> **תלות:** דרוש שיסתיים 2.6.1 redesign (PR #3) לפני התחלה.
> **Branch חדש:** `feature/3.1.5-prompts-infrastructure`
> **env vars חובה:** `OPENAI_API_KEY`, `ADMIN_TOKEN`

---

## רקע — מה זה Session 3.1.5

תשתית גנרית לניהול פרומפטים בכל הפרויקט (Phase 3/5/7/8), + ממשק admin פנימי לבדיקה ידנית של pipeline שלם של יצירת מודעה.

**שינוי משמעותי מהתכנון המקורי של 3.1.5:**

בתכנון המקורי היו 3 קבצי פרומפט נפרדים (`copy_angle_1/2/3.txt`) שייקראו סדרתית. **גיא (השותף העסקי) קבע שזה ייעשה בקריאה אחת** המייצרת 3 וריאציות. זה משנה את המבנה:

- ❌ אין `copy_angle_1.txt`, `copy_angle_2.txt`, `copy_angle_3.txt`, `campaign_strategy.txt`.
- ✅ יש קובץ אחד `copy_generation.txt` שמייצר 3 וריאציות ב-call יחיד.
- ✅ יש `image_generation.txt` נפרד (call לתמונה פר variant, 3 פעמים מקבילית).
- ✅ יש **5 קבצי טון** (לא 4): `professional`, `friendly`, `luxury`, `direct`, `authoritative`.

---

## חלק 1 — מבנה התיקיות

צור את המבנה הבא:

```
app/
├── prompts/
│   ├── README.md
│   ├── phase3/
│   │   ├── copy_generation.txt
│   │   ├── image_generation.txt
│   │   └── tones/
│   │       ├── professional.txt
│   │       ├── friendly.txt
│   │       ├── luxury.txt
│   │       ├── direct.txt
│   │       └── authoritative.txt
│   └── (phase5/, phase7/, phase8/ יתווספו בעתיד — לא יוצר עכשיו)
├── services/
│   ├── prompts_service.py
│   └── ad_generation_service.py
└── routers/
    └── admin/
        ├── __init__.py
        └── prompt_tester.py
```

---

## חלק 2 — תוכן הקבצים

### 2.1 — `app/prompts/README.md`

```markdown
# Prompts Infrastructure

תיקייה זו מכילה את כל הפרומפטים של המערכת, מאורגנים לפי Phase.

## חוקים חשובים

1. **אסור לקרוא קובץ פרומפט ישירות מ-handler או service.** הגישה היחידה היא דרך `services/prompts_service.py`.
2. **כל פרומפט הוא קוד.** שינוי בפרומפט = שינוי בהתנהגות המוצר. דורש PR + סקירה.
3. **placeholder syntax:** `{field_name}` בלבד. לא Jinja2, לא `{{...}}`.

## מבנה ה-Placeholders

### `copy_generation.txt` מקבל:
- `{business_name}` — שם העסק (מהשאלון)
- `{business_description}` — תיאור העסק (מהשאלון)
- `{service_name}` — שם השירות הספציפי שעליו הקמפיין (מהשאלון)
- `{services_list}` — רשימת כל שירותי העסק (מהשאלון)
- `{offer}` — ההצעה השיווקית (מהשאלון, = "הצעת ערך")
- `{differentiation}` — היתרונות והבידול (מהשאלון, = "advantages")
- `{problem_solved}` — הבעיה שהעסק פותר (מהשאלון)
- `{campaign_goal}` — מטרת הקמפיין (`lead` / `whatsapp`, מ-`campaigns.type`)
- `{audience}` — תיאור קהל היעד (מהשאלון)
- `{gender}` — מגדר (`male`/`female`/`all`)
- `{age_range}` — טווח גיל (e.g. "25-45")
- `{location}` — אזור (e.g. "תל אביב + רדיוס 10 ק"מ")
- `{budget_info}` — מידע על תקציב (e.g. "תקציב יומי ₪150" או "תקציב כולל ₪3,000 ל-22 ימים")
- `{tone_instructions}` — מוזרק אוטומטית מקובץ הטון

### `image_generation.txt` מקבל:
- `{business_name}`, `{service_name}`, `{audience}`, `{tone_name}` (שם הטון, לא הוראות)
- `{copy_concept}` — קונספט התמונה שיצא מ-`copy_generation` (שדה ב-JSON)
- `{copy_text}` — הטקסט של המודעה לקונטקסט

## הוספת פרומפט חדש

1. צור קובץ `.txt` בתיקייה המתאימה (`phase3/`, `phase5/` וכו').
2. הוסף את ה-placeholders שלו ל-README הזה.
3. הוסף קריאה ל-`prompts_service.build('your_prompt_name', ...)` במקום שצריך.
4. **אל תקרא את הקובץ ישירות** — תמיד דרך השירות.
```

### 2.2 — `app/prompts/phase3/copy_generation.txt`

```
אתה מנהל קריאייטיב בכיר, קופירייטר ומנהל קמפיינים פייסבוק עם 15 שנות ניסיון בשוק הישראלי.

המשימה שלך: ליצור 3 גרסאות מודעה שונות לקמפיין בפייסבוק/אינסטגרם, לפי הזוויות הבאות:

1. **גישה רגשית** — מתחבר לרגש של הלקוח, סיפור, תחושת שייכות
2. **גישה של כאב ופתרון** — מציב את הבעיה ואז מציג את הפתרון
3. **גישה של תוצאה והצלחה** — מציג תוצאה קונקרטית שהלקוח יקבל

---

## נתוני העסק

שם העסק: {business_name}
תיאור העסק: {business_description}
שם השירות בקמפיין: {service_name}
כל שירותי העסק: {services_list}

הצעת הערך: {offer}
היתרונות והבידול: {differentiation}
הבעיה שהעסק פותר: {problem_solved}

## מטרת הקמפיין

{campaign_goal}

## קהל היעד

תיאור: {audience}
מגדר: {gender}
טווח גיל: {age_range}
אזור: {location}

## תקציב

{budget_info}

## הנחיות טון

{tone_instructions}

---

## דרישות פלט

לכל אחת מ-3 הגרסאות צור:

- **headline** — כותרת קצרה, עד 10 מילים. חייבת להיות חדה ומושכת תשומת לב.
- **body** — טקסט ראשי, 80-150 מילים. מעביר את המסר במלואו לפי הזווית הספציפית.
- **cta** — קריאה לפעולה, משפט קצר אחד בלבד (לדוגמה: "השאר פרטים עכשיו", "דבר איתנו בוואטסאפ").
- **image_concept** — תיאור קצר באנגלית של קונספט התמונה שמתאים לקופי הזה (חשוב: באנגלית, כי הוא יזין את מודל יצירת התמונות).

## חוקים נוקשים

1. **אין להמציא מידע שלא נמסר.** השתמש רק במה שיש בנתונים למעלה.
2. **כל גרסה חייבת להיות שונה מהותית** — לא ניסוח אחר של אותו רעיון, אלא זווית אחרת לחלוטין.
3. **התאמה לקהל** — שפה, גיל, מגדר וטון חייבים להתאים למי שפונים אליו.
4. **חשוב כמו מנהל קמפיינים מקצועי** — המטרה היא לידים איכותיים, לא רק טקסט יפה.
5. **כתוב בעברית** — כל הקופי בעברית. רק `image_concept` באנגלית.

## פורמט הפלט

החזר אך ורק JSON תקין במבנה הבא, ללא טקסט נוסף לפני או אחרי:

```json
{{
  "variants": [
    {{
      "angle": "emotional",
      "headline": "...",
      "body": "...",
      "cta": "...",
      "image_concept": "..."
    }},
    {{
      "angle": "pain_solution",
      "headline": "...",
      "body": "...",
      "cta": "...",
      "image_concept": "..."
    }},
    {{
      "angle": "result_success",
      "headline": "...",
      "body": "...",
      "cta": "...",
      "image_concept": "..."
    }}
  ]
}}
```
```

**שים לב:** ה-JSON בסוף משתמש ב-`{{` ו-`}}` (כפול) כי `str.format()` רואה `{` יחיד כ-placeholder. **זה חשוב — אם לא תכפיל את הסוגריים, ה-format ישבר.**

### 2.3 — `app/prompts/phase3/image_generation.txt`

```
You are an expert advertising photographer creating prompts for AI image generation.

Generate a detailed, professional image generation prompt for the following Facebook/Instagram ad:

## Business Context
- Business name: {business_name}
- Service in this campaign: {service_name}
- Target audience: {audience}
- Brand tone: {tone_name}

## Ad Copy Context
- Concept (provided by copywriter): {copy_concept}
- Ad text: {copy_text}

## Requirements (MUST include in the prompt)

- Advertising Photography style
- Ultra Realistic
- Professional Marketing Campaign quality
- High Conversion Creative
- Photorealistic rendering
- High Detail
- Natural Lighting
- NO TEXT in the image (text will be added separately as overlay)
- Social Media Advertisement composition
- Square 1:1 aspect ratio composition

## Rules

1. The image MUST match the copy concept and tone.
2. The image MUST be appropriate for the target audience demographics.
3. NO embedded text, watermarks, or logos in the image itself.
4. The composition should leave space for potential text overlay (rule of thirds).
5. Color palette should match the brand tone:
   - Professional/Authoritative → neutral, blue/gray tones
   - Friendly → warm, bright colors
   - Luxury → muted, sophisticated palette, gold/black accents
   - Direct → high contrast, bold colors

## Output Format

Return ONLY the image generation prompt as a single English paragraph.
No introduction, no explanation, no JSON. Just the prompt text.

The prompt should be 60-120 words long.
```

### 2.4 — `app/prompts/phase3/tones/professional.txt`

```
Professional Tone Instructions:

- Use clear, precise and trustworthy language.
- Emphasize expertise, professionalism and credibility.
- Maintain a respectful and business-oriented tone.
- Avoid slang and informal expressions.
- Use industry-appropriate terminology when relevant.
- Build trust through demonstrable competence and proven results.
- Stay focused on facts, outcomes, and measurable benefits.

Maintain this tone consistently across all generated copy.
Do not mix multiple tones.
Adapt the tone to the target audience and business type while maintaining professionalism.
```

### 2.5 — `app/prompts/phase3/tones/friendly.txt`

```
Friendly Tone Instructions:

- Write in a warm, approachable and conversational style.
- Create a feeling of personal connection.
- Use simple and easy-to-read language.
- Keep the tone positive and inviting.
- Use second person ("you", "your") to feel direct and personal.
- Include small touches of warmth (without being overly casual or unprofessional).
- Make the reader feel understood and welcomed.

Maintain this tone consistently across all generated copy.
Do not mix multiple tones.
Adapt the tone to the target audience and business type while maintaining warmth.
```

### 2.6 — `app/prompts/phase3/tones/luxury.txt`

```
Luxury Tone Instructions:

- Use elegant, refined and premium language.
- Emphasize quality, exclusivity and experience.
- Create a sense of prestige and sophistication.
- Avoid aggressive sales language.
- Use restrained, confident phrasing rather than loud promotion.
- Reference craftsmanship, heritage, or distinction when appropriate.
- Allow space for the reader to feel the value rather than being told.

Maintain this tone consistently across all generated copy.
Do not mix multiple tones.
Adapt the tone to the target audience and business type while maintaining sophistication.
```

### 2.7 — `app/prompts/phase3/tones/direct.txt`

```
Direct Tone Instructions:

- Be concise and action-oriented.
- Get to the point quickly.
- Focus on benefits and next steps.
- Use strong calls-to-action.
- Cut unnecessary words. Every sentence should advance the message.
- Use clear, decisive verbs.
- Create urgency where it's genuine, not manufactured.

Maintain this tone consistently across all generated copy.
Do not mix multiple tones.
Adapt the tone to the target audience and business type while maintaining directness.
```

### 2.8 — `app/prompts/phase3/tones/authoritative.txt`

```
Authoritative Tone Instructions:

- Demonstrate confidence and expertise.
- Position the business as a leading solution provider.
- Use decisive language that inspires trust.
- Reference experience, results, or unique capabilities.
- Speak from a position of knowledge — but never arrogance.
- Use clear declarative statements rather than hedged language.
- Build credibility through specificity (numbers, timeframes, concrete results).

Maintain this tone consistently across all generated copy.
Do not mix multiple tones.
Adapt the tone to the target audience and business type while maintaining authority.
```

---

## חלק 3 — `services/prompts_service.py`

מימוש מלא של ה-service:

**API ציבורי:**
- `PromptsService.build(prompt_name: str, tone: Optional[str] = None, phase: str = "phase3", **params) -> str`

**שגיאות מותאמות:**
- `PromptNotFoundError` — קובץ לא קיים, ריק, או מכיל רק "TODO"
- `PromptFormatError` — חסר placeholder

**זרימה פנימית:**
1. בניית path: `app/prompts/{phase}/{prompt_name}.txt`
2. אם הקובץ לא קיים → `PromptNotFoundError(f"Prompt {prompt_name} not found in {phase}")`
3. קריאת התוכן. אם ריק או מתחיל ב-"TODO" → `PromptNotFoundError(f"Prompt {prompt_name} not yet defined")`
4. אם `tone` סופק:
   - קריאת `app/prompts/{phase}/tones/{tone}.txt`
   - אותה בדיקת ריק/TODO
   - הזרקת התוכן ב-placeholder `{tone_instructions}` (אם קיים בפרומפט)
5. `str.format(**params)` על המחרוזת המאוחדת
6. אם חסר placeholder → catch `KeyError` והמרה ל-`PromptFormatError(f"Missing placeholder: {e}")`
7. החזרה

**אסור:** לקרוא קובץ פרומפט מחוץ ל-service הזה. ה-service הוא הגישה היחידה.

**טסטים נדרשים:**
- קובץ קיים, tone קיים, כל ה-params מסופקים → מחזיר string תקין
- קובץ לא קיים → `PromptNotFoundError`
- קובץ ריק → `PromptNotFoundError`
- קובץ "TODO" בלבד → `PromptNotFoundError`
- tone לא קיים → `PromptNotFoundError`
- חסר placeholder → `PromptFormatError` עם שם ה-placeholder החסר
- placeholder שלא מופיע בפרומפט נשלח → לא תופעל שגיאה (יתעלם בשקט)

---

## חלק 4 — `services/ad_generation_service.py`

**API ציבורי:**

```python
@dataclass
class CopyVariant:
    angle: Literal["emotional", "pain_solution", "result_success"]
    headline: str
    body: str
    cta: str
    image_concept: str

@dataclass
class GeneratedAds:
    variants: list[CopyVariant]
    images: list[str]  # URLs ל-3 תמונות, באותו סדר כמו variants

async def generate_copy_variants(
    business_name: str,
    business_description: str,
    service_name: str,
    services_list: str,
    offer: str,
    differentiation: str,
    problem_solved: str,
    campaign_goal: Literal["lead", "whatsapp"],
    audience: str,
    gender: Literal["male", "female", "all"],
    age_range: str,
    location: str,
    budget_info: str,
    tone: Literal["professional", "friendly", "luxury", "direct", "authoritative"],
) -> list[CopyVariant]:
    """
    Generates 3 ad copy variants in a SINGLE OpenAI call.
    Returns parsed CopyVariant list.
    """

async def generate_image_for_variant(
    variant: CopyVariant,
    business_name: str,
    service_name: str,
    audience: str,
    tone: str,  # שם הטון כמו "professional", לא ההוראות
) -> str:
    """
    Generates one image for one variant using gpt-image-2.
    Returns URL to generated image.
    """
```

**זרימה ב-`generate_copy_variants`:**

1. בניית הפרומפט דרך `prompts_service.build('copy_generation', tone=tone, **all_other_params)`
2. קריאה ל-OpenAI Chat Completions API (`gpt-4o-mini` או המודל שיוגדר ב-config):
   - `temperature=0.8` (גיוון בין הוריאציות)
   - `response_format={"type": "json_object"}`
   - `max_tokens=2000`
3. פענוח ה-JSON. אם נכשל → log + `raise OpenAIResponseError`
4. ולידציה: חייב להיות exactly 3 variants עם כל השדות. אחרת → `raise OpenAIResponseError`
5. ולידציה: angles חייבים להיות `emotional`, `pain_solution`, `result_success` (אחד מכל אחד). אחרת → `raise OpenAIResponseError`
6. ולידציה: `headline` עד 10 מילים, `body` בין 80-150 מילים. אם חורג → log warning **אבל ממשיכים** (לא לזרוק שגיאה — המודל עלול לחרוג מעט)
7. החזרת `list[CopyVariant]`

**זרימה ב-`generate_image_for_variant`:**

1. בניית הפרומפט דרך `prompts_service.build('image_generation', **params)` (בלי tone — הטון מועבר כפרמטר רגיל בשם `tone_name`)
2. קריאה ל-OpenAI Image API:
   - מודל: `gpt-image-2` (יש להוסיף ל-config; אם המודל הזה לא קיים עדיין ב-SDK, להשתמש ב-`dall-e-3` כ-fallback ולסמן ב-TODO)
   - גודל: `1024x1024` (1:1)
   - quality: `medium` (default)
3. החזרת ה-URL

**Retry logic:**
- אם OpenAI מחזיר שגיאת rate limit → exponential backoff עד 3 ניסיונות
- אם נכשל סופית → `raise OpenAIRateLimitError`
- שגיאות אחרות → `raise OpenAIError`

**אסור:** לכתוב ל-DB ב-Session הזה. הפונקציות pure — מקבלות פרמטרים, מחזירות פלט. שמירה ל-DB תיכנס ב-Session 3.2/3.3.

**טסטים נדרשים:**
- mock OpenAI response תקין → מחזיר 3 variants תקינים
- mock OpenAI response לא-תקין (חסר variant) → `OpenAIResponseError`
- mock OpenAI response עם angle שגוי → `OpenAIResponseError`
- mock רeite limit → retry 3 פעמים → `OpenAIRateLimitError`
- `generate_image_for_variant` עם mock OpenAI → מחזיר URL

---

## חלק 5 — `routers/admin/prompt_tester.py`

**Endpoints:**

```python
@router.get("/admin/prompt-tester", response_class=HTMLResponse)
async def prompt_tester_page(x_admin_token: str = Header(...)) -> HTMLResponse:
    """Returns the admin HTML form for testing prompts."""

@router.post("/admin/prompt-tester/generate")
async def prompt_tester_generate(
    request: GenerateRequest,
    x_admin_token: str = Header(...)
) -> GenerateResponse:
    """Runs the full ad generation pipeline and returns results."""
```

**אבטחה:**
- `ADMIN_TOKEN` env var — חובה. אם לא מוגדר, השרת לא עולה (raise on startup ב-`config.py`).
- בכל endpoint: בדיקת `x_admin_token == settings.ADMIN_TOKEN`. אחרת → `401`.
- **לא בקוקי, לא ב-query string.** רק ב-Header.

**`GenerateRequest` (Pydantic):**

```python
class GenerateRequest(BaseModel):
    business_name: str
    business_description: str
    service_name: str
    services_list: str
    offer: str
    differentiation: str
    problem_solved: str
    campaign_goal: Literal["lead", "whatsapp"]
    audience: str
    gender: Literal["male", "female", "all"]
    age_range: str
    location: str
    budget_info: str
    tone: Literal["professional", "friendly", "luxury", "direct", "authoritative"]
```

**`GenerateResponse`:**

```python
class GenerateResponse(BaseModel):
    copies: list[CopyVariantResponse]  # angle, headline, body, cta, image_concept
    images: list[str]  # URLs
    prompts_used: dict[str, str]  # debugging — הפרומפטים המלאים שנשלחו
```

**זרימה ב-`prompt_tester_generate`:**

1. אימות token
2. קריאה ל-`ad_generation_service.generate_copy_variants(...)` עם כל הפרמטרים מה-request
3. קריאה ל-`ad_generation_service.generate_image_for_variant(...)` **מקבילית** עבור 3 הוריאציות (`asyncio.gather`)
4. החזרת `GenerateResponse`
5. אם משהו נכשל באמצע → return 500 עם הודעת השגיאה. **אל תנסה להחביא חלקית** — ה-admin רוצה לראות כשלים.

**דף ה-HTML (Jinja2 template):**

צור `app/templates/admin/prompt_tester.html` עם:
- כל 13 השדות של `GenerateRequest` (text inputs לקצרים, textareas לארוכים)
- bullet radio buttons ל-5 הטונים
- כפתור Generate שעושה AJAX POST ל-`/admin/prompt-tester/generate`
- אזור תוצאות שמציג 3 כרטיסים — כל אחד עם copy + תמונה
- כפתור "Show prompts used" שמרחיב/מקפל את `prompts_used` (debugging)
- כפתור "Save fixture" → שומר ל-`localStorage` את הקלט הנוכחי
- כפתור "Load fixture" → טוען מ-`localStorage`

**טכנולוגיה:** vanilla HTML + CSS + JS. **לא React, לא build process.** Jinja2 לרינדור ה-template.

**Header bookkeeping:** הוסף את הטוקן כ-Header בכל קריאה: `X-Admin-Token: <token>`. ה-frontend יבקש את הטוקן בכניסה הראשונה ויאחסן ב-`localStorage` עד reload.

---

## חלק 6 — שינויים בקבצים קיימים

### 6.1 — `app/config.py`

הוסף את ה-env vars:

```python
OPENAI_API_KEY: str
ADMIN_TOKEN: str  # required, fail-on-startup if missing
OPENAI_TEXT_MODEL: str = "gpt-4o-mini"  # default
OPENAI_IMAGE_MODEL: str = "gpt-image-2"  # אם לא קיים ב-SDK, fallback ל-"dall-e-3" + TODO
```

### 6.2 — `app/main.py`

רישום ה-router של admin:

```python
from app.routers.admin import prompt_tester
app.include_router(prompt_tester.router)
```

### 6.3 — `requirements.txt`

ודא שיש:
- `openai>=1.0.0`
- `jinja2` (אם לא קיים)

### 6.4 — `CLAUDE.md`

הוסף סעיף חדש:

```markdown
## חוק: גישה לפרומפטים

אסור לקרוא ישירות קבצי `app/prompts/*.txt` מאף מקום בקוד. הגישה היחידה היא דרך `services/prompts_service.PromptsService.build(...)`.

הסיבה: ה-service מטפל ב-loading, format, tone injection, ושגיאות. הוא המקום היחיד שצריך לעדכן אם המבנה ישתנה.

**אסור:**
```python
with open("app/prompts/phase3/copy_generation.txt") as f:
    prompt = f.read()
```

**מותר:**
```python
prompt = prompts_service.build('copy_generation', tone='professional', **params)
```
```

---

## חלק 7 — סדר הביצוע

יישם בסדר הבא, **עצור אחרי כל שלב, הראה דיף, וחכה לאישור לפני שמתחיל את הבא:**

1. **שלב 1:** יצירת תיקיית `app/prompts/` עם כל 8 הקבצים (README + copy_generation + image_generation + 5 tones).
2. **שלב 2:** מימוש `services/prompts_service.py` + טסטים.
3. **שלב 3:** מימוש `services/ad_generation_service.py` + טסטים (עם mocks ל-OpenAI).
4. **שלב 4:** עדכון `config.py` + `main.py` + `requirements.txt` + `CLAUDE.md`.
5. **שלב 5:** מימוש `routers/admin/prompt_tester.py` + ה-HTML template.
6. **שלב 6:** הרצת כל הטסטים. ודא שלא נשברו טסטים קיימים.
7. **שלב 7:** בדיקה ידנית: הרץ את השרת מקומית, גלוש ל-`/admin/prompt-tester` עם הטוקן, מלא את הטופס, לחץ Generate. ודא שתוצאות חוזרות (קופי + 3 תמונות).

---

## חלק 8 — Done של 3.1.5

- `app/prompts/phase3/` מכילה את כל 8 הקבצים, כולם עם תוכן אמיתי (לא TODO).
- `prompts_service.build('copy_generation', tone='professional', business_name='Test', ...)` מחזיר string תקין.
- `ad_generation_service.generate_copy_variants(...)` רץ ומחזיר 3 וריאציות תקינות (עם mock).
- `ad_generation_service.generate_image_for_variant(...)` רץ ומחזיר URL (עם mock).
- `GET /admin/prompt-tester` (עם טוקן) → דף HTML עם form.
- `POST /admin/prompt-tester/generate` → 3 קופי + 3 תמונות (לפחות עם mocks).
- ניסיון ללא token → 401.
- כל הטסטים החדשים עוברים.
- אין רגרסיה בטסטים קיימים.

---

## חלק 9 — לא ב-3.1.5

- handlers ב-jobs queue — יתווספו ב-3.2/3.3.
- שמירה ל-DB — לא ב-3.1.5 (הפונקציות pure).
- versioning של פרומפטים — לא ב-MVP.
- A/B test בייצור — Post-MVP.
- DB-backed prompts — לא. בקוד ב-git.

---

## חלק 10 — כללי משחק

1. **עצור אחרי כל שלב.** הראה דיף, חכה לאישור.
2. **אם משהו לא ברור — שאל.** אל תחליט לבד.
3. **אם תרצה לחרוג מהמסמך — עצור, הסבר למה, חכה.** במיוחד בנוגע לפרומפטים — הם עברו אישור של גיא (השותף העסקי).
4. **טסטים הם חלק מההיקף.** לא דחויים.
5. **אל תיגע בפרומפטים עצמם.** הם תוכן עסקי, לא קוד טכני. אם נדרש שינוי בניסוח — תעצור ותגיד.

---

## חלק 11 — Sanity check לפני התחלה

לפני שמתחיל, ודא:

1. ✅ Branch 2.6.1 redesign (PR #3) הסתיים ונסגר/מומזג. אם לא — **עצור ואל תתחיל 3.1.5.**
2. ✅ `OPENAI_API_KEY` קיים ב-Render env vars.
3. ✅ `ADMIN_TOKEN` קיים ב-Render env vars (אקראי וארוך, נשמר במנהל סיסמאות).
4. ✅ Branch חדש: `feature/3.1.5-prompts-infrastructure`.

אם אחד מהארבעה לא מתקיים — עצור ובקש בירור.
