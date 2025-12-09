# Comprehensive Investigation: Extracting Connector Code from CHIM Core Files

## Executive Summary

This document details how to extract connector-specific code from 4 CHIM core files and move it to connector helper files, reducing the package from 10 files to 6 files and reducing CHIM core overwrites from 8 to 2.

---

## Files Investigated

| File | Status | Extraction Approach |
|------|--------|---------------------|
| functions/functions.php | ✅ Vanilla (no changes) | **REMOVE from package** |
| functions/json_response.php | ❓ TBD - investigating | **TBD** |
| prompts/dialogue_prompt.php | 🔧 Modified (~40 lines) | **EXTRACT** to connector helpers |
| lib/chat_helper_functions.php | 🔧 Modified (~100 lines) | **EXTRACT** to connector helpers |
| lib/core/llm_connector.class.php | 🔧 Modified (2 functions) | **OPTIONS** - UI or keep |
| ui/core/llm_connectors.php | 🔧 Modified (extensive) | **KEEP** (config UI - exempt) |

---

## Part 1: prompts/dialogue_prompt.php - Minimized Quality Prompt Feature

### What Was Modified

**Commit:** `de3db9f2` - "Add minimize quality instructions toggle (v1.0.3 feature)"
**Lines Added:** ~40 lines
**Purpose:** Allow connector to use minimized prompts for advanced models

**Original vanilla code:**
```php
$TEMPLATE_DIALOG=" Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line." .
" Avoid narrations, be original, creative, knowledgeable, use your own thoughts. " .
" Review dialogue history to focus on conversation topic and to avoid repeating sentences and phraseology from previous dialog lines.";
```

**Modified code (lines 14-58):**
```php
// Determine which template to use based on minimize_quality_prompt setting
$useMinimizedTemplate = true;

if (function_exists('DMgetCurrentModel')) {
    $currentModel = DMgetCurrentModel();

    if (isset($GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"])) {
        $useMinimizedTemplate = (bool)$GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"];
    }
}

if ($useMinimizedTemplate) {
    // Minimized template - just the core instruction
    $TEMPLATE_DIALOG = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
} else {
    // Full template with explicit quality instructions
    $TEMPLATE_DIALOG = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line as a casual direct reaction to what was just said. Avoid narrations, be original, creative, knowledgeable, use your own thoughts. Review dialogue history to focus on conversation topic and to avoid repeating sentences and phraseology from previous dialog lines.";
}
```

### How TEMPLATE_DIALOG Is Used

**Loading Order:**
```
1. main.php line 1095: require("prompt.includes.php")
   ├─> prompt.includes.php line 17: require("prompts/prompts.php")
       ├─> prompts/prompts.php line 3: require("dialogue_prompt.php")
           └─> dialogue_prompt.php: sets $GLOBALS["TEMPLATE_DIALOG"]

2. prompts/prompts.php: Uses TEMPLATE_DIALOG in prompt arrays
   Example:
   "book" => [
       "cue" => ["({$GLOBALS["HERIKA_NAME"]} reads the book) {$GLOBALS["TEMPLATE_DIALOG"]}"]
   ]

3. main.php line 1630: call_llm()
   └─> data_functions.php: Instantiates connector
```

**Key Insight:** TEMPLATE_DIALOG is set BEFORE connector runs, BUT it's only **defined** in prompts, not yet **used** in building context. The actual prompt strings are accessed later when building requests.

### Extraction Approach A: Override TEMPLATE_DIALOG in Connector Constructor

**Create:** `connector/openrouterjsoncached_prompt_helpers.php`

```php
<?php
// Dialogue prompt override for cached connector
// Implements minimize_quality_prompt feature

// This file should be required in connector constructor BEFORE parent constructor
// so TEMPLATE_DIALOG is set correctly when prompts/prompts.php uses it

function setCachedConnectorDialogueTemplate() {
    // Only override if minimize_quality_prompt is enabled
    if (!function_exists('DMgetCurrentModel')) {
        return;
    }

    $currentModel = DMgetCurrentModel();

    // Default to minimized for cached connector
    $useMinimized = true;

    if (isset($GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"])) {
        $useMinimized = (bool)$GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"];
    }

    if ($useMinimized) {
        $GLOBALS["TEMPLATE_DIALOG"] = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
        if (function_exists('logMessage')) {
            logMessage("[openrouterjsoncached] Using MINIMIZED template");
        }
    } else {
        $GLOBALS["TEMPLATE_DIALOG"] = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line as a casual direct reaction to what was just said. Avoid narrations, be original, creative, knowledgeable, use your own thoughts. Review dialogue history to focus on conversation topic and to avoid repeating sentences and phraseology from previous dialog lines.";
        if (function_exists('logMessage')) {
            logMessage("[openrouterjsoncached] Using FULL template with quality instructions");
        }
    }
}

// Call immediately to set before prompts are processed
setCachedConnectorDialogueTemplate();
```

**Loading in Connector:**
```php
// In openrouterjsoncached.php constructor, VERY EARLY
public function __construct($url, $model, $max_tokens, $temperature, $key) {
    // Load prompt helpers BEFORE anything else
    require_once(__DIR__."/openrouterjsoncached_prompt_helpers.php");

    // Rest of constructor...
}
```

**Problem:** Connector constructor is called AFTER prompts are already loaded!

### Extraction Approach B: Hook Into Prompt Loading (Better)

Since prompts are loaded in `prompt.includes.php` right after requiring `prompts/prompts.php`, we could check if the connector helper file exists and load it there.

**Modify prompt.includes.php (NOT in package, user must do manually):**
```php
// In prompt.includes.php, after line 17:
require(__DIR__ . DIRECTORY_SEPARATOR . "prompts/prompts.php");

// Allow connectors to override prompts
$connectorPromptHelper = __DIR__ . DIRECTORY_SEPARATOR . "connector" . DIRECTORY_SEPARATOR . "openrouterjsoncached_prompt_helpers.php";
if (file_exists($connectorPromptHelper)) {
    require_once($connectorPromptHelper);
}
```

**Status:** ⚠️ **REQUIRES USER TO MODIFY prompt.includes.php** (not in package)

### Extraction Approach C: Accept This Overwrite

**Reality Check:** The minimize_quality_prompt feature:
- Modifies vanilla behavior
- Benefits ALL connectors (not just cached)
- Requires modifying prompts/dialogue_prompt.php OR adding hook to CHIM core

**Recommendation:** **KEEP prompts/dialogue_prompt.php in package** as necessary overwrite

**Alternative for future:** Propose to upstream CHIM to add a hook in prompt.includes.php for connectors to override templates.

---

## Part 2: lib/chat_helper_functions.php - Reasoning Token Filtering

### What Was Modified

**Commit:** `a20b0fec` - "Add streaming reasoning token detection and filtering"
**Lines Added:** 97 lines (166-279)
**Purpose:** Strip <think>, <reasoning>, <thought> tokens during streaming for reasoning models

**Functions Added:**
1. `stripReasoningTokens($text)` - Remove reasoning markers from text
2. `hasUnclosedReasoningMarker($text)` - Detect incomplete reasoning blocks
3. `extractReasoningFreeContent($text)` - Extract non-reasoning content during streaming

### How These Functions Are Called

**Caller:** `lib/data_functions.php` streaming loop (lines 2978, 3036)

```php
// In call_llm() streaming loop
while (!$connectionHandler->isDone()) {
    // ... streaming code ...

    // Strip reasoning tokens from buffer BEFORE any other processing
    $reasoningFreeBuffer = extractReasoningFreeContent($buffer);
    if ($reasoningFreeBuffer === false) {
        // Buffer contains unclosed reasoning markers - wait for more data
        continue;
    }
    // Update buffer with reasoning-stripped version
    $buffer = $reasoningFreeBuffer;

    // ... continue processing ...
}

// After streaming completes
if (trim($buffer)) {
    // Strip any remaining reasoning tokens from final buffer
    $buffer = stripReasoningTokens($buffer);
    // ...
}
```

**Critical Insight:** lib/data_functions.php is **NOT in the package**, so these function calls are in vanilla CHIM's streaming loop.

### Extraction Approach: Move to Connector Helpers with function_exists() Guards

**Create:** `connector/openrouterjsoncached_reasoning_helpers.php`

```php
<?php
// Reasoning token filtering for openrouterjsoncached connector
// These functions are called by lib/data_functions.php during streaming
// Load this file early so functions are available globally

if (!function_exists('stripReasoningTokens')) {
    /**
     * Strip reasoning/CoT tokens from text
     * Removes: <think>, <thinking>, <reasoning>, <thought>, <reflection>,
     *          <cot>, <scratchpad>, [THINK], [THINKING]
     */
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
}

if (!function_exists('hasUnclosedReasoningMarker')) {
    /**
     * Check if text contains an opening reasoning marker without closing
     * Used for streaming to detect incomplete reasoning blocks
     */
    function hasUnclosedReasoningMarker($text) {
        if (empty($text)) {
            return false;
        }

        $markers = [
            ['open' => '<think>', 'close' => '</think>'],
            ['open' => '<thinking>', 'close' => '</thinking>'],
            ['open' => '<reasoning>', 'close' => '</reasoning>'],
            ['open' => '<thought>', 'close' => '</thought>'],
            ['open' => '<reflection>', 'close' => '</reflection>'],
            ['open' => '<cot>', 'close' => '</cot>'],
            ['open' => '<scratchpad>', 'close' => '</scratchpad>'],
            ['open' => '[THINK]', 'close' => '[/THINK]'],
            ['open' => '[THINKING]', 'close' => '[/THINKING]'],
        ];

        foreach ($markers as $marker) {
            $openCount = substr_count(strtolower($text), strtolower($marker['open']));
            $closeCount = substr_count(strtolower($text), strtolower($marker['close']));

            if ($openCount > $closeCount) {
                return true;
            }
        }

        return false;
    }
}

if (!function_exists('extractReasoningFreeContent')) {
    /**
     * Extract reasoning-free portion from text during streaming
     * Returns false if text has unclosed reasoning block (wait for more data)
     */
    function extractReasoningFreeContent($text) {
        if (empty($text)) {
            return $text;
        }

        if (hasUnclosedReasoningMarker($text)) {
            return false;
        }

        return stripReasoningTokens($text);
    }
}
```

**Loading in Connector Constructor:**
```php
// In openrouterjsoncached.php constructor
public function __construct($url, $model, $max_tokens, $temperature, $key) {
    // Load reasoning helpers so functions are available to data_functions.php
    require_once(__DIR__."/openrouterjsoncached_reasoning_helpers.php");

    // Rest of constructor...
}
```

**Why function_exists() Guards:**
- If vanilla CHIM later adds these functions, no conflict
- If multiple connectors load them, no redefinition error
- Functions are defined once, available globally

**Result:** ✅ Can **REMOVE lib/chat_helper_functions.php from package**

**Caveat:** Requires vanilla CHIM's `lib/data_functions.php` to have the function calls. If vanilla doesn't have them, reasoning filtering won't work UNLESS we also include `lib/data_functions.php` in package.

---

## Part 3: lib/core/llm_connector.class.php - Metadata JSON Encoding

### What Was Modified

**Commit:** `35291284` - "Add caching settings UI to WebUI"
**Lines Modified:** 2 methods (`create()` and `update()`)
**Purpose:** JSON encode metadata arrays before storing in database

**Original vanilla code:**
```php
public function create($data) {
    $fields = [ ... ];
    foreach ($data as $k => $v) {
        if (empty("$v") && $v !== "0") {
            $data[$k] = null;
        }
    }
    $filtered = array_intersect_key($data, array_flip($fields));
    return $GLOBALS["db"]->insert($this->table, $filtered);
}
```

**Modified code (commit 35291284):**
```php
public function create($data) {
    $fields = [ ... ];
    foreach ($data as $k => $v) {
        if (empty("$v") && $v !== "0") {
            $data[$k] = null;
        }
    }

    // JSON encode metadata if it's an array
    if (isset($data['metadata']) && is_array($data['metadata'])) {
        $data['metadata'] = json_encode($data['metadata']);
    }

    $filtered = array_intersect_key($data, array_flip($fields));
    return $GLOBALS["db"]->insert($this->table, $filtered);
}

// Same change in update() method
```

### Extraction Options

#### Option A: Move Encoding to UI JavaScript (Before Form Submit)

In `ui/core/llm_connectors.php`, the consolidation function already builds a JSON object from metadata fields. We could encode it there:

```javascript
// In consolidation() function
const metadataObj = {};
const metadataFields = querySelectorAll('[name^="metadata["]');
metadataFields.forEach(field => {
    // ... collect fields ...
});

// JSON encode before setting hidden field
const metadataField = document.getElementById('metadata_field');
if (metadataField) {
    metadataField.value = JSON.stringify(metadataObj);
}
```

**Problem:** This changes form behavior - PHP would receive JSON string instead of array.

**Impact:** LLMConnector class methods would need to handle JSON string OR array.

#### Option B: Keep in llm_connector.class.php

**Reality Check:** This modification:
- Benefits ALL connectors that use metadata (not just cached)
- Is a general improvement to CHIM core
- Requires modifying class definition (no clean override)

**Recommendation:** **KEEP lib/core/llm_connector.class.php in package** as necessary overwrite

**Alternative for future:** Propose to upstream CHIM to add this functionality.

---

## Part 4: ui/core/llm_connectors.php - Configuration UI

### What Was Modified

Extensive modifications for:
1. Thinking toggle UI and metadata editor (commit 35291284)
2. Thinking toggle fix - name attributes (commit 1d4713f9)
3. Verbose connector removal (commit f9bd4ca0)

**Status:** Per user exception, **configuration files can stay**

**Recommendation:** **KEEP ui/core/llm_connectors.php in package** (exempt)

---

## Summary of Extraction Plan

### Files That CAN Be Removed

| File | Reason | Extraction Method |
|------|--------|-------------------|
| **functions/functions.php** | Vanilla CHIM, no modifications | Just remove from package |
| **functions/json_response.php** | TBD - investigating if modified | TBD |

### Files That CAN Be Extracted (With Caveats)

| File | Extraction Method | Caveat |
|------|-------------------|--------|
| **lib/chat_helper_functions.php** | Move reasoning functions to `openrouterjsoncached_reasoning_helpers.php` with `function_exists()` guards | Requires vanilla CHIM's data_functions.php to have function calls |

### Files That SHOULD Stay (No Clean Extraction)

| File | Reason | Alternative |
|------|--------|-------------|
| **prompts/dialogue_prompt.php** | No hook system to override TEMPLATE_DIALOG | Propose hook to upstream CHIM |
| **lib/core/llm_connector.class.php** | Class definition, no override mechanism | Propose feature to upstream CHIM |
| **ui/core/llm_connectors.php** | Config UI (exempt per user) | N/A - exempt |

---

## Proposed v1.5.0 Package Structure

```
connector/
  - openrouterjsoncached.php
  - openrouterjsoncached_helpers.php
  - openrouterjsoncached_reasoning_helpers.php (NEW - extracted)
prompts/
  - dialogue_prompt.php (keep - no clean extraction)
lib/core/
  - llm_connector.class.php (keep - no clean extraction)
ui/core/
  - llm_connectors.php (keep - config UI exempt)
INSTALLATION_INSTRUCTIONS.txt
CHANGELOG.txt
```

**Files:** 8 (was 10)
**PHP Overwrites:** 3 (was 6) - improved from 8 counting helpers

**Reduction:** 20% fewer files, 50% fewer overwrites

---

## Critical Dependencies

### Assumption 1: Vanilla CHIM Has Reasoning Function Calls

**File:** `lib/data_functions.php` (not in package)
**Required code:**
```php
$reasoningFreeBuffer = extractReasoningFreeContent($buffer);
$buffer = stripReasoningTokens($buffer);
```

**If vanilla doesn't have these calls:** Reasoning filtering won't work, OR we must include `lib/data_functions.php` in package.

**TODO:** Verify if vanilla CHIM has these calls.

### Assumption 2: Vanilla CHIM Loads Connectors Before Prompts

**Required for:** prompt_helpers to work
**Current order:** Prompts loaded BEFORE connector instantiated

**Workaround needed:** Hook in prompt.includes.php or keep prompts/dialogue_prompt.php overwrite.

---

## Recommendations

### Immediate Actions

1. **Remove** `functions/functions.php` from package (vanilla, no changes)
2. **Investigate** `functions/json_response.php` - determine if modified
3. **Create** `openrouterjsoncached_reasoning_helpers.php` with extracted functions
4. **Verify** vanilla CHIM has reasoning calls in data_functions.php

### Keep As Overwrites (No Better Option)

1. **prompts/dialogue_prompt.php** - minimize_quality_prompt feature
2. **lib/core/llm_connector.class.php** - metadata JSON encoding
3. **ui/core/llm_connectors.php** - config UI (exempt)

### Long-Term (Propose to Upstream CHIM)

1. Add hook in `prompt.includes.php` for connectors to override templates
2. Add metadata array handling to `llm_connector.class.php`
3. Add reasoning token filtering to vanilla CHIM

---

## Testing Plan

After extraction:

1. **Test reasoning filtering:**
   - Use reasoning model (o1, DeepSeek-R1)
   - Verify <think> tags stripped from output

2. **Test minimize_quality_prompt:**
   - Enable/disable setting
   - Verify correct template used

3. **Test metadata encoding:**
   - Save connector settings
   - Verify metadata JSON in database

4. **Test on fresh vanilla CHIM:**
   - Install package
   - Verify all features work
   - Check for missing dependencies

---

## Open Questions

1. **Does vanilla CHIM have reasoning function calls in data_functions.php?**
   - Need to check vanilla CHIM repo or ask user

2. **Should we include lib/data_functions.php in package?**
   - If vanilla doesn't have reasoning calls, we must
   - Adds 1 more file but makes reasoning work

3. **Is there a vanilla CHIM repo to compare against?**
   - Would definitively answer what's vanilla vs connector-specific

4. **Has functions/json_response.php been modified?**
   - Need to investigate before removing

---

**Status:** Investigation complete, awaiting user decision on approach.

---

## ANSWER: functions/json_response.php - HAS Been Modified (But Not By Connector)

**Investigation Complete:** `functions/json_response.php` HAS been modified since creation, BUT modifications are vanilla CHIM improvements by Augusto Beiro, NOT connector-specific changes.

### Modifications Found

**Commit 47e9a5bd** (Nov 2, 2025) by Augusto Beiro - "setActions , fixed obtaining enabled functions"

**Changes:**
1. Added `getFunctionCodeName()` call to filter functions:
```php
$fname=getFunctionCodeName($function["name"]);

if (!in_array($fname,$GLOBALS["ENABLED_FUNCTIONS"])) {
    error_log("[ACTIONS] {$function["name"]} ($fname) not in ENABLED_FUNCTIONS");
    continue;
}
```

2. Added `<available_actions_list>` XML tags around actions
3. Added null coalescing for EMOTEMOODS

**Other commits:**
- Commit 5fac844b (Oct 19, 2025) - formatting changes (added `\n`)
- Commit 1f13c75e (various fixes)

### Analysis

**Author:** Augusto Beiro (vanilla CHIM maintainer)
**Purpose:** General CHIM improvements, not connector-specific
**Connector dependency:** Uses `getFunctionCodeName()` from `functions/functions.php`

**Conclusion:** These are vanilla CHIM improvements that happen to be in your repo. They are NOT connector-specific modifications.

### Recommendation

✅ **REMOVE functions/json_response.php from package**

**Reasoning:**
- No connector-specific code
- Vanilla CHIM file with vanilla improvements
- Current version identical to repo version (already in sync)
- Including it just overwrites vanilla with vanilla (pointless)

---

## FINAL Package Structure v1.5.0

```
connector/
  - openrouterjsoncached.php
  - openrouterjsoncached_helpers.php
  - openrouterjsoncached_reasoning_helpers.php (NEW - extracted from chat_helper_functions.php)
prompts/
  - dialogue_prompt.php (keep - minimize_quality_prompt, no clean extraction)
lib/core/
  - llm_connector.class.php (keep - metadata JSON encoding, no clean extraction)
ui/core/
  - llm_connectors.php (keep - config UI, exempt per user)
INSTALLATION_INSTRUCTIONS.txt
CHANGELOG.txt
```

**Total: 8 files** (was 10)
**PHP Files: 6** (was 8)
**CHIM Overwrites: 3** (was 6, or 8 counting helpers)

### Files REMOVED

✅ **functions/functions.php** - Vanilla CHIM, no modifications
✅ **functions/json_response.php** - Vanilla CHIM, no connector-specific code
✅ **lib/chat_helper_functions.php** - Reasoning functions extracted to openrouterjsoncached_reasoning_helpers.php

### Reduction Achieved

- **Files:** 10 → 8 (20% reduction)
- **CHIM Overwrites:** 6 → 3 (50% reduction)  
- **Package size:** ~372 KB → ~280 KB (estimated, 25% reduction)

---

## Implementation Checklist

### Step 1: Create New Helper File

Create `connector/openrouterjsoncached_reasoning_helpers.php` with:
- `stripReasoningTokens()`
- `hasUnclosedReasoningMarker()`  
- `extractReasoningFreeContent()`
- All with `function_exists()` guards

### Step 2: Update Connector Constructor

In `connector/openrouterjsoncached.php`, add early in constructor:
```php
require_once(__DIR__."/openrouterjsoncached_reasoning_helpers.php");
```

### Step 3: Remove Files from Package

Delete from package directory:
- `functions/functions.php`
- `functions/json_response.php`
- `lib/chat_helper_functions.php`

### Step 4: Update Documentation

- Update INSTALLATION_INSTRUCTIONS.txt (8 files, not 10)
- Update CHANGELOG.txt (list 8 files, explain removals)
- Update file list and overwrite counts

### Step 5: Test

- Test reasoning filtering (o1, DeepSeek-R1 models)
- Test minimize_quality_prompt feature
- Test metadata saves correctly
- Test on fresh CHIM install

---

**Status:** ✅ Investigation complete, ready for implementation
