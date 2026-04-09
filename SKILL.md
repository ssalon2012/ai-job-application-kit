---
name: job-application-kit
description: >
  Generates a complete, tailored job application kit from a resume and job description.
  Produces four ready-to-use Word documents: a tailored resume, a personalized cover letter,
  a hiring manager email, and fully written answers to every application question.

  Use this skill whenever a user wants to apply for a job, mentions a job posting they're
  interested in, uploads a resume and asks for help applying, or says things like "help me
  apply to this role", "write my cover letter for X", "tailor my resume for this job",
  "help me with this job application", or "how do I apply to [company]". Trigger even when
  the user only pastes a job description without saying "skill" — if they have a resume and
  a job, this skill is the right tool. Also trigger when the user provides application
  questions and wants answers written for them.
---

# Job Application Kit

This skill turns a resume and a job description into a complete, polished application package — four Word (.docx) files the user can submit immediately.

## What you'll produce

1. **Tailored Resume** — Same structure as the original, but rewritten bullets and summary sharpened for the specific role. No fabrication; sharpen what's already there.
2. **Cover Letter** — Personalized to the company with zero placeholder text. Research the company before writing.
3. **Hiring Manager Email** — Short, direct outreach email with a strong subject line, meant to accompany the application or be sent directly.
4. **Application Answers** — Every open-ended question on the application answered in full, copy-paste ready.

---

## Step 1: Gather inputs

Before doing any work, you need:
- **Resume file** — The user should upload it. Extract its content immediately using `pandoc --track-changes=all resume.docx -o resume.md`.
- **Job description** — Pasted text, a URL, or described. If a URL is given, fetch it.
- **Application questions** — Any open-ended fields in the application form (the user can paste them in).

If anything is missing, ask for it before proceeding.

---

## Step 2: Ask two quick clarifying questions

Use `AskUserQuestion` with two questions at once:

1. **"What U.S. state will you be working from?"** — needed for the application's payroll tax field. If you already have this from context, skip it.
2. **"Do you know the hiring manager's name?"** — offer options: Yes (they'll type it), No (use "Hiring Team").

Do not proceed past this point until you have both answers.

---

## Step 3: Research the company

Before writing anything, spend 1–2 searches learning about the company:
- What does their product/service actually do?
- Who are their customers?
- What's their mission, growth stage, or recent milestones?
- What makes this company genuinely interesting to a job seeker?

This research powers the personalization paragraph in the cover letter and the "Why this company" application answer. Generic cover letters get ignored — real specificity doesn't.

---

## Step 4: Build all four documents

Use `python-docx` (available via `from docx import Document`). Build all four in a single Python script. Save each to `/sessions/.../mnt/outputs/`.

### Document style guide

Use a consistent, professional style across all docs:
- **Font**: Arial throughout
- **Accent color** for headings: `RGBColor(0x1C, 0x2B, 0x6E)` (dark navy) — or match the user's existing resume color if they have one
- **Body text**: 10–11pt, black
- **Margins**: 0.75–1 inch
- **Spacing**: Tight but breathable — `space_after = Pt(6–12)` between paragraphs
- **Page size**: US Letter (set explicitly — python-docx defaults to A4)

### Helper pattern (use this for every doc)

```python
from docx import Document
from docx.shared import Pt, RGBColor, Inches, Twips
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml.ns import qn
from docx.oxml import OxmlElement

def set_spacing(para, before=0, after=0):
    para.paragraph_format.space_before = Pt(before)
    para.paragraph_format.space_after = Pt(after)

def add_run(para, text, bold=False, italic=False, size=11, color=None):
    r = para.add_run(text)
    r.bold = bold
    r.italic = italic
    r.font.name = "Arial"
    r.font.size = Pt(size)
    r.font.color.rgb = color if color else RGBColor(0,0,0)
    return r

def bottom_border(para, color="1C2B6E", size=8):
    pPr = para._p.get_or_add_pPr()
    pBdr = OxmlElement('w:pBdr')
    b = OxmlElement('w:bottom')
    b.set(qn('w:val'), 'single')
    b.set(qn('w:sz'), str(size))
    b.set(qn('w:space'), '1')
    b.set(qn('w:color'), color)
    pBdr.append(b)
    pPr.append(pBdr)

def new_doc():
    doc = Document()
    s = doc.sections[0]
    s.page_width = Twips(12240)   # US Letter
    s.page_height = Twips(15840)
    s.top_margin = Inches(0.75)
    s.bottom_margin = Inches(0.75)
    s.left_margin = Inches(0.75)
    s.right_margin = Inches(0.75)
    return doc
```

### Resume rules
- Keep the user's original structure and section order
- Rewrite the professional summary to explicitly call out the skills this role cares about
- Tighten bullets: lead with a strong action verb, include a metric where one exists, cut filler words
- Add any role-relevant keywords from the job description that honestly apply to the candidate's experience
- Do NOT invent experience, skills, tools, or metrics that aren't in the original resume
- Use tab stops to right-align dates: `tabStops=[{type: TabStopType.RIGHT, position: TabStopPosition.MAX}]`

### Cover letter rules
- Length: 300–400 words, 4–5 paragraphs
- No placeholder text of any kind — every bracket filled in
- Structure:
  1. Opening: who you are, what you're applying for, the hook
  2. Experience: the strongest 1–2 selling points, with specifics and numbers
  3. Company fit: 2–4 sentences about *why this specific company*, drawing from your research
  4. Closing: confidence, call to action, contact info
- Sign with the candidate's actual name

### Hiring manager email rules
- Subject line formula: `[Role] Application — [Name] | [1 punchy differentiator]`
- Body: 4–5 short paragraphs, no longer than 150 words total
- Tone: confident, direct, not groveling
- End with contact info in the signature

### Application answer rules
- Answer every question in full — no "see resume" cop-outs
- Length: calibrate to the question. Factual questions (CRM experience) = 75–150 words. Story questions (complicated situation) = 150–200 words using a mini narrative arc: situation → action → result.
- For "Why this company" questions: use your company research to write something genuine and specific
- For the "complicated situation" question: construct a plausible, specific story from the user's actual work history. Pick one that shows listening past the surface-level objection and finding a resolution
- Yes/No questions: answer directly, then stop
- Preferred start date: use whatever the user provided

---

## Step 5: Verify and deliver

After building all files:

1. Run a quick sanity check — open each file and confirm no `[placeholder]` text remains and word counts are reasonable (resume: 300–500 words, cover letter: 300–400 words)
2. Save all files to `/sessions/.../mnt/outputs/` with clear names:
   - `[Name]_Resume_[Company].docx`
   - `[Name]_Cover_Letter_[Company].docx`
   - `[Name]_Hiring_Manager_Email_[Company].docx`
   - `[Name]_Application_Answers_[Company].docx`
3. Share all four with `computer://` links
4. Add a brief note reminding the user to: swap in the hiring manager's name if they find it, and double-check any numbers or dates against their memory

---

## Tips for quality

- **Read the job description twice.** The first time for the obvious requirements; the second time for the underlying signal — what kind of person succeeds here, what problems they're solving, what vocabulary they use. Mirror that vocabulary in the resume and cover letter.
- **Don't over-format the resume.** Clean, spacious, and readable beats crammed and clever.
- **The cover letter personalization paragraph is the hardest and most important sentence.** Spend real time on it. A single specific observation about the company (a product detail, a mission statement, a recent funding round) beats three generic sentences about "being passionate about growth."
- **Application question answers are often the actual filter.** Write them like short essays, not bullet points.
