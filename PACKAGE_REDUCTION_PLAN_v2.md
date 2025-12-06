# Package Reduction Plan v2.0
## Moving Connector-Specific Code to Helpers

### Discovery

All "overwritten" files in v1.4.0 package are **IDENTICAL to vanilla CHIM**:
- ✅ lib/chat_helper_functions.php - IDENTICAL
- ✅ prompts/dialogue_prompt.php - IDENTICAL
- ✅ functions/functions.php - IDENTICAL
- ✅ functions/json_response.php - IDENTICAL
- ✅ lib/core/llm_connector.class.php - IDENTICAL
- ✅ ui/core/llm_connectors.php - IDENTICAL

**What this means:** These files were modified during connector development and later merged into vanilla CHIM (via PR #19 on Nov 15). The package is now unnecessarily overwriting files with identical content.

---

## Analysis: What Was Added For The Connector?

### 1. lib/chat_helper_functions.php
**Added in commit a20b0fec (Nov 6):** Reasoning token filtering functions

```php
// Lines 166-279 (~120 lines added)
function stripReasoningTokens($text) { ... }
function hasUnclosedReasoningMarker($text) { ... }
function extractReasoningFreeContent($text) { ... }
```

**Called by:** CHIM core's `lib/data_functions.php` at lines 2978, 3036

**Purpose:** Strip <think>, <reasoning>, <thought> tokens during streaming

---

### 2. prompts/dialogue_prompt.php
**Added:** minimize_quality_prompt feature (lines 14-58)

```php
// Conditional template based on setting
if ($useMinimizedTemplate) {
    $TEMPLATE_DIALOG = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
} else {
    $TEMPLATE_DIALOG = " Write ... (long version with quality instructions)";
}
```

**Called by:** CHIM core's `prompts/prompts.php`

**Purpose:** Allow advanced models to skip verbose quality instructions

---

### 3. functions/functions.php
**Changes:** NONE - vanilla CHIM file

**Assessment:** No connector-specific modifications

---

### 4. functions/json_response.php
**Changes:** NONE - vanilla CHIM file

**Assessment:** No connector-specific modifications

---

### 5. lib/core/llm_connector.class.php
**Added in commit 35291284 (Nov 5):** Metadata JSON encoding

```php
// In create() and update() methods
if (isset($data['metadata']) && is_array($data['metadata'])) {
    $data['metadata'] = json_encode($data['metadata']);
}
```

**Called by:** CHIM core connector management system

**Purpose:** Encode connector metadata arrays as JSON for database storage

---

### 6. ui/core/llm_connectors.php
**Added:** Multiple connector-specific features:
- Thinking toggle UI and metadata editor (Nov 5)
- Thinking toggle fix - name attributes (Nov 15)
- Verbose connector removal (Dec 5)

**Called by:** CHIM web UI

**Purpose:** Configuration interface for cached connector

---

## Proposed Architecture

### Option A: Minimal Package (Recommended)

**Remove all files identical to vanilla CHIM:**

```
PACKAGE CONTENTS (4 files):
✓ connector/openrouterjsoncached.php
✓ connector/openrouterjsoncached_helpers.php
✓ INSTALLATION_INSTRUCTIONS.txt
✓ CHANGELOG.txt
```

**Rationale:**
- Vanilla CHIM already has all required features
- No need to overwrite identical files
- Package size: ~40 KB (was 88 KB)
- File count: 4 files (was 10 files)

**Trade-off:**
- Requires vanilla CHIM to have the features (already does)
- If user has older CHIM, features won't work

---

### Option B: Extract To Helpers (More Complex)

**Move connector-added code to separate helper files:**

#### B.1. Create `connector/cached_connector_reasoning_helpers.php`

```php
<?php
// Reasoning token filtering functions for cached connector
// Loaded globally so available to data_functions.php

function stripReasoningTokens($text) {
    if (empty($text)) {
        return $text;
    }

    // Common reasoning markers (case-insensitive)
    $patterns = [
        '/<think>.*?<\/think>/is',
        '/<thinking>.*?<\/thinking>/is',
        '/<reasoning>.*?<\/reasoning>/is',
        '/<thought>.*?<\/thought>/is',
        '/<reflection>.*?<\/reflection>/is',
        '/<cot>.*?<\/cot>/is',
        '/<scratchpad>.*?<\/scratchpad>/is',
        '/\[THINK\].*?\[\/THINK\]/is',
        '/\[THINKING\].*?\[\/THINKING\]/is',
    ];

    $cleaned = $text;
    foreach ($patterns as $pattern) {
        $cleaned = preg_replace($pattern, '', $cleaned);
    }

    $cleaned = preg_replace('/\s+/', ' ', $cleaned);
    return trim($cleaned);
}

function hasUnclosedReasoningMarker($text) {
    if (empty($text)) {
        return false;
    }

    // Check for opening tags without closing
    $markers = [
        '<think>' => '</think>',
        '<thinking>' => '</thinking>',
        '<reasoning>' => '</reasoning>',
        '<thought>' => '</thought>',
        '<reflection>' => '</reflection>',
        '<cot>' => '</cot>',
        '<scratchpad>' => '</scratchpad>',
        '[THINK]' => '[/THINK]',
        '[THINKING]' => '[/THINKING]',
    ];

    $lowerText = strtolower($text);

    foreach ($markers as $open => $close) {
        $openPos = stripos($lowerText, strtolower($open));
        if ($openPos !== false) {
            $closePos = stripos($lowerText, strtolower($close), $openPos);
            if ($closePos === false) {
                return true;
            }
        }
    }

    return false;
}

function extractReasoningFreeContent($text) {
    if (empty($text)) {
        return $text;
    }

    if (hasUnclosedReasoningMarker($text)) {
        return false;
    }

    $cleaned = stripReasoningTokens($text);

    if (hasUnclosedReasoningMarker($cleaned)) {
        return false;
    }

    return $cleaned;
}
```

**Loading:** Add to connector constructor
```php
// In openrouterjsoncached.php constructor
require_once(__DIR__."/cached_connector_reasoning_helpers.php");
```

**Problem:** CHIM core's `data_functions.php` calls these functions **before** connector runs
**Solution:** Functions defined in connector helpers are loaded into global scope, available when data_functions.php runs

**BUT:** If vanilla CHIM's `lib/chat_helper_functions.php` ALSO defines these functions, we get "function already defined" errors!

**Resolution:** Use `function_exists()` checks:
```php
if (!function_exists('stripReasoningTokens')) {
    function stripReasoningTokens($text) { ... }
}
```

This way:
- If vanilla CHIM has the functions: use those
- If vanilla CHIM doesn't: connector provides them

---

#### B.2. Create `connector/cached_connector_prompt_helpers.php`

```php
<?php
// Prompt template helpers for minimize_quality_prompt feature

// Override dialogue template based on connector setting
function setMinimizedDialoguePrompt() {
    if (!function_exists('DMgetCurrentModel')) {
        return;
    }

    $currentModel = DMgetCurrentModel();
    $useMinimizedTemplate = true;

    if (isset($GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"])) {
        $useMinimizedTemplate = (bool)$GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"];
    }

    if ($useMinimizedTemplate) {
        $GLOBALS["TEMPLATE_DIALOG"] = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
    } else {
        $GLOBALS["TEMPLATE_DIALOG"] = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line as a casual direct reaction to what was just said. Avoid narrations, be original, creative, knowledgeable, use your own thoughts. Review dialogue history to focus on conversation topic and to avoid repeating sentences and phraseology from previous dialog lines.";
    }
}

// Call this after prompts/dialogue_prompt.php loads
if (isset($GLOBALS["TEMPLATE_DIALOG"])) {
    setMinimizedDialoguePrompt();
}
```

**Problem:** `prompts/prompts.php` loads `dialogue_prompt.php` which sets the template **before** connector runs

**Solution:** Hook into prompt loading or override after it's set

**BUT:** No clean hook system in CHIM

**Alternative:** Accept that vanilla CHIM needs this feature

---

#### B.3. Handle `lib/core/llm_connector.class.php`

**Problem:** Metadata JSON encoding is in the class definition

**Can't override:** This is a class file loaded by CHIM core

**Solutions:**
1. **Accept vanilla CHIM has this** (simplest)
2. **Create wrapper class** (complex, doesn't help with overwrite issue)
3. **Have UI JSON encode before submitting** (changes UI behavior)

**Recommendation:** Leave in vanilla CHIM - it's a general improvement

---

#### B.4. Handle `ui/core/llm_connectors.php`

**Problem:** Configuration UI with connector-specific sections

**User exception:** "Configuration files or files on which the configuration files depend" can stay

**Decision:** **Keep as overwrite** (per user's exception)

---

## Recommendation Matrix

### Simple Approach (RECOMMENDED)

**Remove all identical files from package:**

| File | Action | Reason |
|------|--------|--------|
| connector/openrouterjsoncached.php | **KEEP** | Connector core |
| connector/openrouterjsoncached_helpers.php | **KEEP** | Connector helpers |
| lib/chat_helper_functions.php | **REMOVE** | Identical to vanilla |
| prompts/dialogue_prompt.php | **REMOVE** | Identical to vanilla |
| functions/functions.php | **REMOVE** | Identical to vanilla |
| functions/json_response.php | **REMOVE** | Identical to vanilla |
| lib/core/llm_connector.class.php | **REMOVE** | Identical to vanilla |
| ui/core/llm_connectors.php | **REMOVE** | Identical to vanilla |
| INSTALLATION_INSTRUCTIONS.txt | **KEEP** | Documentation |
| CHANGELOG.txt | **KEEP** | Documentation |

**Result:**
- Package: 4 files (was 10)
- Size: ~40 KB (was 88 KB)
- Reduction: 60% fewer files, 55% smaller

**Requirements:**
- User must have vanilla CHIM with all features (they do, it's in current repo)

**Installation:**
```
1. Copy connector/openrouterjsoncached.php to connector/
2. Copy connector/openrouterjsoncached_helpers.php to connector/
3. Done!
```

---

### Complex Approach (If Features Must Be Extracted)

**Create helper files with function_exists() guards:**

1. **connector/cached_connector_reasoning_helpers.php**
   - Reasoning token filtering (if not in vanilla)

2. **connector/cached_connector_prompt_helpers.php**
   - Dialogue template override (if not in vanilla)

3. **ui/core/llm_connectors.php**
   - Keep as overwrite (per user exception)

**Problems:**
- Duplicate function definitions if vanilla has them
- No clean way to override prompts after they're loaded
- Metadata encoding can't be easily extracted
- More complex, more points of failure

**Result:**
- Package: 5 files (connector core + helpers + UI)
- Doesn't actually avoid overwrites (UI still needs overwriting)
- More maintenance burden

---

## Final Recommendation

### Go with Simple Approach

**Package v1.5.0 should contain only:**

```
CHIM_Cached_Connector_v1.5.0/
├── connector/
│   ├── openrouterjsoncached.php
│   └── openrouterjsoncached_helpers.php
├── INSTALLATION_INSTRUCTIONS.txt
└── CHANGELOG.txt
```

**Why:**
- All features already in vanilla CHIM
- No need for complex helper extraction
- Cleaner, simpler, less maintenance
- 60% reduction in package size
- Zero overwrites of vanilla files

**Installation becomes:**
1. Copy 2 connector files
2. Done!

**Requirements note in docs:**
- Requires CHIM version with reasoning support (current version)
- Requires CHIM version with minimize_quality_prompt (current version)
- Requires CHIM version with metadata JSON encoding (current version)

**All requirements met by current vanilla CHIM in the repo.**

---

## Question For User

Do you want:

**A) Simple approach** (recommended):
- Remove all files identical to vanilla
- Package only connector files + docs
- 4 files total
- Requires current vanilla CHIM

**B) Complex approach:**
- Extract features to connector helpers
- Keep UI overwrite
- Add function_exists() guards
- 5+ files with helper architecture
- More complex, same overwrites for UI

**C) Something else?**

Please clarify your preference and I'll implement accordingly.
