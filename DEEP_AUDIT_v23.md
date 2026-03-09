# DEEP AUDIT: Economics Classroom v23 UNIFIED
## 23,984 Lines — Full Code Review

---

## 🔴 CRITICAL BUGS (Broken Features)

### 1. Anthropic/Claude Provider BROKEN in Lesson Generation

**Location:** `requestLessonFromAI()` — Lines 12416–12486

The main lesson generator treats ALL providers identically, but Anthropic requires completely different handling:

**Bug A — Wrong Auth Header:**
```
// Line 12424 — WRONG for Anthropic
headers['Authorization'] = `Bearer ${config.apiKey}`;

// Anthropic requires:
headers['x-api-key'] = apiKey;
headers['anthropic-version'] = '2023-06-01';
```

The HAL Assistant (line 9775) correctly uses `x-api-key`, but the *main lesson generator* does not. This means **every lesson generation attempt with Claude will return 401 Unauthorized**.

**Bug B — Wrong Response Parsing:**
```
// Line 12483 — WRONG for Anthropic
const content = data.choices?.[0]?.message?.content;

// Anthropic returns:
const content = data.content?.[0]?.text;
```

The HAL assistant (line 9791) parses this correctly, but the lesson generator doesn't. Even if auth worked, the response would be silently treated as empty.

**Bug C — Missing CORS Header:**
Anthropic's API requires `anthropic-dangerous-direct-browser-access: true` for direct browser calls. Without it, the browser blocks the request entirely with a CORS error. Neither the HAL assistant nor the lesson generator includes this header.

**Fix:** The `requestLessonFromAI` function needs an Anthropic-specific code path identical to what `halAskAI` already does, plus the CORS header.

---

### 2. Undefined Variable in `buildConsistentImagePrompt`

**Location:** Line 13133

```javascript
const prompt = buildImagePrompt(cleanAction, currentImageStyle, hasSecondary, ...);
//                                                                ^^^^^^^^^^^^
// `hasSecondary` is NOT declared in this function scope
// Should be `addSecondary` (declared on line 13121)
```

This causes a silent ReferenceError. In non-strict mode it evaluates to `undefined` (falsy), so secondary characters never appear when this code path is used.

---

### 3. Frame Blank Regex Fragility

**Location:** `renderCurrentDefinition()` — Lines 15620–15637

The regex matching for blank restoration uses:
```javascript
const regex = new RegExp(`<span class=['"]frame-blank['"]...${escapedBlank}...`, 'g');
```

If the AI generates gap_blanks containing regex special chars that `escapeRegExp` doesn't catch (parentheses in economic terms like "GDP (nominal)"), the regex breaks silently, blanks don't render, and the entire Logic Build section becomes non-functional for that definition.

---

## 🟠 AI GENERATION — Improvement Opportunities

### 4. System Prompt Is Over-Engineered

**Current:** ~200 lines with ASCII box-drawing characters (`═══`, `┌───┐`), redundant checklists, and repeated instructions.

**Problems:**
- LLMs process tokens, not visual formatting. The ASCII art wastes ~500 tokens per request while adding zero benefit.
- The same constraint (e.g., "EXACTLY 7 keywords") appears 6+ times, diluting focus.
- The JSON schema example is partially duplicated in both system and user prompts.

**Recommended Fix:**
- Strip all ASCII art, reduce to a focused ~80-line prompt.
- State each constraint ONCE with clear priority marking.
- Move the JSON schema into a single dedicated section.
- Use structured output (JSON mode) more aggressively — OpenAI's `response_format: { type: "json_object" }` is currently only enabled for Groq on retries, but works reliably on OpenAI too.

### 5. Temperature Too High for Structured Output

**Current:** `TEMPERATURE: 0.6`

For reliable JSON output with strict schema adherence, a temperature of **0.3–0.4** is much more appropriate. 0.6 introduces creative variation that causes:
- Extra/missing fields.
- Keyword count drift (AI generates 6 when asked for 7).
- Malformed gap_sentences.

The creativity should come from the *content* (story text, examples), not from structural compliance. Consider using 0.3 for the structure pass and letting the story text have more freedom via the prompt instructions.

### 6. Model Selection Is Outdated / Limited

| Provider | Current Model | Recommended | Why |
|----------|--------------|-------------|-----|
| OpenAI | `gpt-4o-mini` | Offer `gpt-4o` as premium option | 4o-mini often struggles with complex JSON |
| Groq | `llama-3.3-70b-versatile` | Good choice | ✅ |
| DeepSeek | `deepseek-chat` | `deepseek-chat` or `deepseek-reasoner` | Consider offering the reasoning model |
| Anthropic | `claude-sonnet-4-20250514` | Good choice (once CORS/auth fixed) | ✅ |
| **Missing** | — | **Google Gemini** | Free tier, excellent JSON, popular in China |

Adding Gemini (via `generativelanguage.googleapis.com`) would be valuable since it's free for moderate usage, works without VPN in China, and handles structured JSON well.

### 7. No Retry Backoff / No Streaming Feedback

- On rate-limit (429), the code immediately throws rather than waiting and retrying. Adding exponential backoff (wait 2s → 4s → 8s) before retrying would significantly improve reliability.
- There's no streaming support. The user stares at "This may take 30-60 seconds" with no feedback. Streaming the response would let you show partial progress and catch JSON errors earlier.

### 8. JSON Extraction Is Fragile

**Location:** `extractJsonFromResponse()` — Line 11834

The function tries to find JSON in the response via brace-matching. If the AI wraps its JSON in markdown fences with a preamble like "Here is the lesson:", the extraction works. But if the JSON itself contains strings with unbalanced braces (e.g., in gap_sentences like "The {market} responds..."), the extraction can silently truncate.

**Fix:** Use a proper JSON extraction: find the first `{`, then use a brace-depth counter that respects string escaping.

---

## 🟠 IMAGE GENERATION — Improvement Opportunities

### 9. Pollinations.ai Limitations

**Current:** Pollinations is the default free provider. Issues:
- **No actual seed consistency.** While the code sends `seed=42424242`, Pollinations doesn't guarantee deterministic output. Emma looks different in every panel.
- **Variable quality.** Results range from excellent to broken depending on server load.
- **No negative prompts.** Can't exclude text/watermarks reliably.
- **Resolution scaling** reduces already-mediocre quality further.

**Recommended Alternatives (by tier):**

| Tier | Provider | Cost | Key Benefit |
|------|----------|------|-------------|
| Free | **Stable Diffusion via fal.ai** | Free tier available | Actual seed consistency, better quality |
| Free | **Together.ai** | Free tier (50 req/day) | FLUX model, very high quality |
| Low | **Replicate (FLUX-schnell)** | ~$0.003/image | Best quality-to-cost, true seed support |
| Premium | **DALL-E 3** (already supported) | ~$0.04/image | Highest quality, no seed support |
| Premium | **Midjourney API** (beta) | Subscription | Best artistic quality |

### 10. Character Consistency Strategy

The "Emma Protocol" is a solid concept but the execution relies on prompt-only consistency, which is inherently unreliable across separate generation calls.

**Better approaches:**
- **IP-Adapter / Character Reference:** Replicate's FLUX models support image-to-image reference. Generate Emma ONCE, then use that image as a reference for all panels.
- **LoRA Fine-tuning:** For a production app, a custom LoRA trained on 10–15 images of "Emma" would guarantee consistency.
- **Fixed Pose Sheet:** Generate a character reference sheet first, then include it in subsequent calls.

### 11. Image Style Selector UX

The style selector (ghibli, pop art, cyberpunk, etc.) only takes effect during generation — there's no preview of what each style looks like. Adding small thumbnail previews per style would help teachers choose confidently.

---

## 🟡 SECTION-BY-SECTION FUNCTIONAL AUDIT

### Scene 0: Start Screen
**Status:** ✅ Working
- Boot animation, intro matrix rain, settings panel all functional.
- Minor: The "20 min" button is hardcoded as default (`btn20min.style.borderColor = '#ffff00'`) rather than reading the actual radio button state.

### Scene 1: Data Analysis (Story)
**Status:** ⚠️ Mostly Working
- Comic/Text toggle works.
- Fullscreen presentation mode works with keyboard navigation.
- **Issue:** Comic bubble text truncation (line 12777–12793) uses character counting on HTML-tagged text, which can cut mid-tag and produce broken HTML like `<span class="keyword" data-term="GDP...`.
- **Issue:** `wrapStoryKeywordsSafe` can produce nested `<span>` tags if a keyword appears inside another keyword's text.
- Image upload works but uses in-memory storage only (reset on refresh). Good choice for data safety but confusing when images disappear.

### Scene 2: Logic Construction (Frames)
**Status:** ⚠️ Has Issues
- Fill-in-the-blank mechanic works for simple cases.
- **Issue:** `initFrameBuilder()` (line 17138) resets `filledBlanksStorage` every time Scene 2 is entered (including when navigating back). Students lose their work.
- **Issue:** Dual click handling — each word card has both an inline `onclick` AND event delegation via `wordBankClickHandler`. This can cause double-fire on some browsers, placing the word and immediately removing it.
- **Issue:** When `frame.blanks` contains multi-word answers (e.g., "opportunity cost"), the matching logic does case-insensitive comparison but doesn't handle extra whitespace.

### Scene 3: Repository (Definitions)
**Status:** ✅ Working (but minimal)
- Renders definition cards correctly.
- No interactivity — just static text display.
- **Opportunity:** This scene could benefit from a flip-card or flashcard-style interaction, or a "match term to definition" mini-game.

### Scene 4: Data Entry (Writing)
**Status:** ⚠️ Has Issues
- Textarea prompts render correctly.
- Solutions toggle works.
- **Issue:** Textareas have **no auto-save**. Navigating away from Scene 4 and back wipes all student writing. This is the most complained-about UX issue in classroom apps.
- **Issue:** `showWriteSolutions()` (line 7811) hardcodes `color:#00ff00` instead of using `var(--primary-bright)`, breaking when any non-green theme is active.

### Scene 5: System Validation (MCQs)
**Status:** ✅ Working
- Question display, option selection, thinking timer, reveal animation all work.
- MCQ image upload per-question works.
- Score tracking works.
- Minor: The 10-second thinking timer countdown sounds `playBeep()` at 3, 2, 1 — could use distinct warning tones.

### Completion Scene
**Status:** ✅ Working
- Shows score summary, lesson plan generation, worksheet download.

### HAL Assistant (Sidebar)
**Status:** ⚠️ Partially Working
- Works with Groq, OpenAI, DeepSeek.
- **Broken with Anthropic** (CORS issue as described above).
- Lesson context injection is well-designed.
- Conversation history is properly managed (last 6 messages).

### Score Panel / Team System
**Status:** ✅ Working
- Team management, CSV import, random selector, scoring all functional.
- Winner celebration and MVP selection work.

### Settings Panel
**Status:** ✅ Working but Overwhelming
- Contains too many options in a single scrollable panel.
- The AI provider setup, image provider setup, theme selection, difficulty, and lesson management are all crammed together.

---

## 🟡 UI/UX DESIGN IMPROVEMENTS

### 12. Settings Panel Needs Restructuring

The settings modal is a massive single-scroll page with ~15 different config sections. This should be reorganized into **tabbed categories**:

- **AI Settings** — Provider, API key, model selection
- **Image Settings** — Image provider, style, quality
- **Display Settings** — Theme, text scale, calm mode
- **Lesson Management** — Save/load, templates, export
- **Teams** — (currently separate, which is good)

### 13. Hardcoded Colors Break Themes

Multiple locations use hardcoded `#00ff00` instead of CSS variables:
- `showWriteSolutions()` line 7811: `color:#00ff00`
- `renderWriteScene()` line 16937: `color:var(--primary-bright)` ✅ (correct here)
- Score panel inline styles in HTML use hardcoded green
- Several `showModal` calls embed green colors

All should use `var(--primary-color)` / `var(--primary-bright)` / `var(--text-color)`.

### 14. Mobile Responsiveness Gaps

- Comic panels use fixed-width calculations that overflow on phones.
- The frame builder's word bank cards don't wrap properly below 400px.
- The settings modal has `max-width: 650px` but no `width: 95vw` fallback.
- The HAL assistant sidebar width is fixed rather than responsive.

### 15. Accessibility Concerns

- Frame blanks have `role="button"` and `tabindex="0"` ✅ (good).
- But MCQ options lack `role="radio"` or `role="option"`.
- No `aria-live` region for the timer countdown (screen readers can't track it).
- Color-only feedback for correct/wrong answers (no icon/text alternative).

### 16. No Loading States for AI Operations

When generating a lesson or images, the only feedback is a modal with static text. There should be:
- An animated progress indicator (the CRT aesthetic suits a pulsing scanline or spinning HAL eye).
- Step-by-step progress that updates in real-time.
- A cancel button for long-running operations.

---

## 🟢 WHAT'S WORKING WELL

- **Theme system** — CSS variable architecture is clean and consistent.
- **Fluid typography** — The `--text-scale` + `clamp()` system is excellent for projection.
- **Z-index documentation** — The hierarchy comment at the top is a great maintenance aid.
- **Keyword-as-single-source-of-truth** — This architectural decision prevents data drift between sections.
- **Audio system** — Web Audio API synthesis is well-implemented with proper AudioContext init.
- **Comic fullscreen mode** — Keyboard nav, auto-play, dot indicators are polished.
- **Easter eggs** — Hacker mode, retro mode, disco mode add fun without interfering.

---

## PRIORITY FIX ORDER

| # | Fix | Impact | Effort |
|---|-----|--------|--------|
| 1 | Fix Anthropic provider (auth + parsing + CORS) | Critical — provider broken | Medium |
| 2 | Fix `buildConsistentImagePrompt` undefined variable | Bug — secondary chars broken | Trivial |
| 3 | Add textarea auto-save in Scene 4 | UX — students lose work | Low |
| 4 | Stop resetting filledBlanksStorage on Scene 2 re-entry | UX — students lose work | Low |
| 5 | Fix comic bubble HTML truncation | Visual — broken HTML | Medium |
| 6 | Replace hardcoded colors with CSS variables | Theme — broken non-green themes | Low |
| 7 | Reduce AI prompt verbosity + lower temperature | Quality — better JSON compliance | Low |
| 8 | Add Gemini as AI provider | Feature — free tier, no VPN in China | Medium |
| 9 | Add better image provider (fal.ai / Replicate) | Quality — character consistency | High |
| 10 | Restructure settings into tabs | UX — overwhelming settings | Medium |
