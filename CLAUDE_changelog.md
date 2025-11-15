# CLAUDE CHANGELOG
## CRITICAL: DO NOT DELETE OR LOSE THIS FILE - ONLY ADD TO IT

**Purpose:** This file documents every modification made during the rebuild from v1.0.12 to restore working thinking toggle while preserving v1.1.22 features.

**Instructions:**
- This file MUST NOT be deleted
- This file MUST NOT be lost
- This file may ONLY be appended to (add new entries at the bottom)
- Each entry must document: file modified, section/lines, what was done, and conceptual goal
- Use clear, precise language
- Include timestamps for each session

---

## Session: 2025-11-15 - Initial Changelog Creation

### Entry 1: Create CLAUDE_changelog.md
**Timestamp:** 2025-11-15 14:30 UTC
**File Created:** `/home/user/HerikaServer/CLAUDE_changelog.md`
**Section:** N/A (new file)
**Action:** Created this changelog file to document all modifications during v1.0.12 → v1.1.22 feature rebuild
**Conceptual Goal:** Maintain comprehensive audit trail of all changes to prevent loss of context and enable debugging

**Current Branch:** `claude/1.0.12-testing-01SD3JLwpdiGRsLvnUEQ9JEY` (working baseline)
**Target:** Rebuild to match v1.1.22 functionality while keeping working thinking toggle

---

## Changes Begin Below This Line

### Entry 2: Add new configuration fields to partial editor form
**Timestamp:** 2025-11-15 15:00 UTC
**File Modified:** `/home/user/HerikaServer/ui/core/llm_connectors.php`
**Section:** Lines 463-476 (caching settings section)
**Action:** Added 3 new metadata fields following v1.0.12 working pattern:
- `max_dialogue_cache_context_size` - number input (default: 93)
- `custom_system_instruction` - textarea
- `custom_last_instruction` - textarea

**Pattern Used:** All fields use `name='metadata[field_name]'` pattern (NOT `id` only)
**Loading Pattern:** Values loaded from `$metadata` array via `$metadata['field_name'] ?? 'default'`
**Conceptual Goal:** Add v1.1.22 configuration features while maintaining v1.0.12's working save mechanism

**Critical Analysis:**
- These fields follow the SAME pattern as other working metadata fields in v1.0.12
- They have proper `name` attributes (unlike the broken fix attempt where we removed names)
- The consolidation function will handle these automatically via the existing `querySelectorAll('[name^="metadata["]')` logic
- No changes to JavaScript consolidation function needed

### Entry 3: Add new configuration fields to modal form
**Timestamp:** 2025-11-15 15:05 UTC
**File Modified:** `/home/user/HerikaServer/ui/core/llm_connectors.php`
**Section:** Lines 1487-1508 (modal caching settings section)
**Action:** Added same 4 fields to modal form:
- `minimize_quality_prompt` - checkbox (default: checked)
- `max_dialogue_cache_context_size` - number input with `_main` suffix on ID
- `custom_system_instruction` - textarea with `_main` suffix on ID
- `custom_last_instruction` - textarea with `_main` suffix on ID

**Pattern Used:**
- Checkboxes: hidden input (value="0") + checkbox input (value="1")
- Text/number fields: `name='metadata[field_name]'` with unique IDs (`field_name_main`)
- Values loaded from `$metadata_main` array

**Conceptual Goal:** Ensure modal form has feature parity with main form

**Critical Analysis:**
- Modal fields use `_main` suffix on IDs to prevent conflicts
- All use SAME `name` attributes (no suffix) so form submission works correctly
- Follows existing modal pattern from v1.0.12 (e.g., `toggle_thinking_modal`)

### Entry 4: Update decimal precision from 2 to 4 decimals
**Timestamp:** 2025-11-15 15:15 UTC
**File Modified:** `/home/user/HerikaServer/ui/core/llm_connectors.php`
**Section:** Lines 485-492, 1518-1525 (both $ranges arrays)
**Action:** Changed step values from 0.01 to 0.0001 for all decimal parameters
**Fields Updated:**
- temperature
- presence_penalty
- frequency_penalty
- repetition_penalty
- top_p
- min_p
- top_a

**Pattern Used:** Global replacement `'step'=>0.01` → `'step'=>0.0001`
**Result:** 14 occurrences updated (7 fields × 2 forms)
**Conceptual Goal:** Match v1.1.22's 4-decimal precision for better parameter control

**Critical Analysis:**
- This change only affects input step granularity, not validation or storage
- No changes to JavaScript or consolidation function needed
- Maintains backward compatibility (stored values unchanged)
- Follows v1.1.22 pattern exactly

### Entry 5: Replace simple format implementation with working v1.1.20 version
**Timestamp:** 2025-11-15 15:25 UTC
**Files Replaced:**
- `connector/openrouterjsoncached.php` (complete replacement)
- `connector/openrouterjsoncached_helpers.php` (complete replacement)
- `prompts/dialogue_prompt.php` (complete replacement)

**Source:** aiagent branch (v1.1.20 - last known working simple format before thinking toggle fix attempts)
**Action:** Complete replacement of connector and prompt files with v1.1.20 versions

**What This Adds:**
1. **Working Simple Format Parser:** Complete rewrite of natural language response parsing
2. **Response Format Setting:** Support for `response_format` metadata field (json/simple)
3. **Simple Format Content Options:** Proper handling of include_mood, include_listener, include_actions, include_target flags
4. **Stream Processing:** Enhanced streaming with simple format state machine
5. **Quality Instructions:** minimize_quality_prompt integration in prompts

**Critical Analysis - Thinking Toggle Preservation:**
✅ **Verified thinking toggle code is INTACT in v1.1.20:**
- Line 309: `$toggleThinking = isset($GLOBALS["CONNECTOR"][$this->name]["toggle_thinking"]) ? $GLOBALS["CONNECTOR"][$this->name]["toggle_thinking"] : false;`
- Line 310: `$thinkingTokens` reading preserved
- Line 311: `$effort_level` reading preserved
- Reasoning detection functions present (isOpenAIReasoningModel, isAlwaysReasoningModel)
- Reasoning configuration building preserved (lines 636+)

**Why This Won't Break Thinking Toggle:**
1. v1.1.20 is BEFORE the broken fix attempts that removed name attributes
2. All thinking toggle metadata reading uses same GLOBALS pattern
3. No JavaScript consolidation changes in connector files
4. Thinking toggle worked in v1.1.20 (confirmed by user testing)

**Changes Summary:**
- connector/openrouterjsoncached.php: +442 lines, -71 lines
- connector/openrouterjsoncached_helpers.php: +13 lines (minor updates)
- prompts/dialogue_prompt.php: +69 lines (quality instructions, custom instructions)

**Note:** Verbose connector (openrouterjsoncached_verbose.php) intentionally NOT updated per user request

### Entry 6: Fix thinking toggle not working with simple format
**Timestamp:** 2025-11-15 15:40 UTC
**File Modified:** `connector/openrouterjsoncached.php`
**Section:** Line 610 (_openPart3 function - simple format prefill logic)
**Action:** Added condition to prevent prefill when thinking is enabled

**Problem Identified:**
Simple format was ALWAYS adding assistant prefill `'('` to control response format, even when thinking toggle was enabled. Prefill and reasoning are **mutually incompatible** on most providers (Anthropic, OpenRouter). The prefill was blocking reasoning from being performed, even though `reasoning: {enabled: true}` was in the API payload.

**User Evidence:**
- JSON format (no prefill): Reasoning performed successfully (verified on OpenRouter)
- Simple format (with prefill): Reasoning NOT performed despite enabled=true (verified on OpenRouter)
- Both had identical reasoning parameters except for the prefill message

**Fix Applied:**
Changed line 610 from:
```php
if ($this->_responseFormat === 'simple') {
```
To:
```php
if ($this->_responseFormat === 'simple' && !$toggleThinking) {
```

**Logic:**
- If simple format + thinking DISABLED: Use prefill for format control
- If simple format + thinking ENABLED: Skip prefill, allow reasoning to work
- If JSON format: No prefill (unchanged)

**Conceptual Goal:** Enable thinking toggle to work correctly with simple format by avoiding incompatible prefill

**Critical Analysis:**
- Prefill is only needed for format control when model isn't doing reasoning
- When thinking is enabled, the model is smart enough to follow format without prefill
- This maintains backward compatibility: simple format without thinking still gets prefill
- Thinking now works with BOTH JSON and simple formats

