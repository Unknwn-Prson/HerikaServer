# Package Reduction Plan v2.1 - Extract Connector Code to Helpers
## Proper Assessment Based on Vanilla CHIM Baseline

### Current Situation

**Files Modified in This Repo for Connector:**
1. `lib/chat_helper_functions.php` - Added 3 reasoning functions (~100 lines)
2. `lib/data_functions.php` - Added calls to reasoning functions (NOT in package!)
3. `prompts/dialogue_prompt.php` - Added minimize_quality_prompt logic (~18 lines)
4. `lib/core/llm_connector.class.php` - Added metadata JSON encoding
5. `ui/core/llm_connectors.php` - Added thinking toggle UI, metadata editor, verbose cleanup
6. `functions/functions.php` - No changes (vanilla)
7. `functions/json_response.php` - No changes (vanilla)

**Critical Discovery:**
- `lib/data_functions.php` is modified in repo but **NOT in package**
- This means the package expects users to have modified `lib/data_functions.php`
- OR vanilla CHIM already has these modifications
- OR the reasoning functions are dead code

---

## Proposed Extraction Plan

### File 1: lib/chat_helper_functions.php

**What was added FOR connector:**
```php
// Lines 166-279 (~120 lines)
function stripReasoningTokens($text) { ... }
function hasUnclosedReasoningMarker($text) { ... }
function extractReasoningFreeContent($text) { ... }
```

**Extraction Plan:**
Move to `connector/openrouterjsoncached_reasoning_helpers.php`

```php
<?php
// Reasoning token filtering for cached connector
// Load this file early so functions are available to data_functions.php

if (!function_exists('stripReasoningTokens')) {
    function stripReasoningTokens($text) {
        if (empty($text)) {
            return $text;
        }

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

**Loading:**
- In connector `__construct()`: `require_once(__DIR__."/openrouterjsoncached_reasoning_helpers.php");`
- Uses `function_exists()` guards to avoid conflicts if vanilla CHIM adds these

**Result:**
- ✅ Remove `lib/chat_helper_functions.php` from package
- ✅ Functions still available to `lib/data_functions.php` (loaded globally)

**Caveat:**
- Requires `lib/data_functions.php` to actually call these functions
- If vanilla CHIM's `lib/data_functions.php` doesn't have the calls, functions are unused

---

### File 2: prompts/dialogue_prompt.php

**What was added FOR connector:**
```php
// Lines 14-44 (~30 lines)
$useMinimizedPrompt = true;

if (function_exists('DMgetCurrentModel')) {
    $currentModel = DMgetCurrentModel();
    if (isset($GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"])) {
        $useMinimizedPrompt = (bool)$GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"];
    }
}

if ($useMinimizedPrompt) {
    $TEMPLATE_DIALOG = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
} else {
    $TEMPLATE_DIALOG = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line..." (long version);
}
```

**Problem:**
- `prompts/prompts.php` loads `dialogue_prompt.php` before connector runs
- Can't override template after it's already set

**Option A: Hook System (Complex)**
Create `connector/openrouterjsoncached_prompt_helpers.php`:
```php
<?php
// Override dialogue template after prompts load

// Register hook to override template (if CHIM has hook system)
if (isset($GLOBALS["HOOKS"])) {
    $GLOBALS["HOOKS"]["AFTER_PROMPTS_LOAD"][] = function() {
        if (!function_exists('DMgetCurrentModel')) {
            return;
        }

        $currentModel = DMgetCurrentModel();
        $useMinimized = isset($GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"])
            ? (bool)$GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"]
            : true;

        if ($useMinimized) {
            $GLOBALS["TEMPLATE_DIALOG"] = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
        }
    };
}
```

**Problem:** CHIM doesn't have a hook system

**Option B: Connector Override (Simple)**
In connector's `call()` method, override template before building prompt:
```php
public function call($buffer, $herikaid, $cmdid) {
    // Override dialogue template based on minimize_quality_prompt setting
    if (isset($this->metadata['minimize_quality_prompt']) && $this->metadata['minimize_quality_prompt']) {
        $GLOBALS["TEMPLATE_DIALOG"] = " Write {$GLOBALS["HERIKA_NAME"]}'s next dialogue line.";
    }

    // Continue with normal connector logic...
}
```

**Problem:** Template might already be used in prompt construction before connector's `call()` is invoked

**Option C: Accept File Overwrite**
- minimize_quality_prompt requires modifying `prompts/dialogue_prompt.php`
- No clean way to extract without CHIM architectural changes
- **Keep in package as necessary overwrite**

**Recommendation:** **Option C** - Keep prompts/dialogue_prompt.php in package

---

### File 3: lib/core/llm_connector.class.php

**What was added FOR connector:**
```php
// In create() and update() methods
if (isset($data['metadata']) && is_array($data['metadata'])) {
    $data['metadata'] = json_encode($data['metadata']);
}
```

**Problem:**
- This is a class definition loaded by CHIM core
- Can't extend/override without modifying the class file itself
- Metadata encoding is needed for ALL connectors that use metadata (general improvement)

**Options:**

**Option A: UI-Side Encoding**
Move JSON encoding to UI before form submission:
- In `ui/core/llm_connectors.php`, JSON encode metadata before sending to server
- Con: Changes UI behavior, adds complexity

**Option B: Connector-Side Encoding**
In connector, manually encode metadata before saving:
- Override connector's save method to encode metadata
- Con: Connector might not control when llm_connector.class methods are called

**Option C: Accept File Overwrite**
- Metadata JSON encoding is a general improvement
- Benefits any connector using metadata
- **Keep in package as necessary overwrite**

**Recommendation:** **Option C** - Keep lib/core/llm_connector.class.php in package

---

### File 4: ui/core/llm_connectors.php

**What was added FOR connector:**
1. Thinking toggle UI and metadata editor (commit 35291284)
2. Thinking toggle fix - name attributes (commit 1d4713f9)
3. Verbose connector removal (commit f9bd4ca0)

**Assessment:**
- Per user's exception: "Configuration files or files on which the configuration files depend" can stay
- This IS the configuration UI file

**Recommendation:** **Keep in package** (per user exception)

---

### Files 5-6: functions/functions.php, functions/json_response.php

**Changes:** NONE - completely vanilla CHIM files

**Recommendation:** **REMOVE from package** (no connector-specific code)

---

## Final Package Structure

### Minimal Package (Recommended)

```
CHIM_Cached_Connector_v1.5.0/
├── connector/
│   ├── openrouterjsoncached.php
│   ├── openrouterjsoncached_helpers.php
│   └── openrouterjsoncached_reasoning_helpers.php (NEW)
├── prompts/
│   └── dialogue_prompt.php (keep - no clean extraction method)
├── lib/
│   └── core/
│       └── llm_connector.class.php (keep - no clean extraction method)
├── ui/
│   └── core/
│       └── llm_connectors.php (keep - per user exception)
├── INSTALLATION_INSTRUCTIONS.txt
└── CHANGELOG.txt
```

**Files: 8 total (was 10)**
- 3 connector files (including new reasoning helpers)
- 3 CHIM overwrites (necessary - no clean extraction)
- 2 documentation files

**Removed from package:**
- ✅ functions/functions.php (vanilla CHIM, no changes)
- ✅ functions/json_response.php (vanilla CHIM, no changes)

**New file:**
- ✅ connector/openrouterjsoncached_reasoning_helpers.php (extracted from chat_helper_functions.php)

**Kept (necessary overwrites):**
- prompts/dialogue_prompt.php (no hook system to override)
- lib/core/llm_connector.class.php (class definition, no clean override)
- ui/core/llm_connectors.php (config UI - per user exception)

---

## Reduction Achieved

**v1.4.0:** 10 files, 88 KB
**v1.5.0:** 8 files, ~75 KB
**Reduction:** 20% fewer files, 15% smaller

**Overwrites reduced:**
- v1.4.0: 8 PHP files overwritten
- v1.5.0: 3 PHP files overwritten
- **Reduction: 62.5% fewer overwrites**

---

## Implementation Steps

1. **Create new helper file:**
   - `connector/openrouterjsoncached_reasoning_helpers.php`
   - Move reasoning functions from chat_helper_functions.php
   - Add `function_exists()` guards

2. **Update connector constructor:**
   ```php
   // In openrouterjsoncached.php __construct()
   require_once(__DIR__."/openrouterjsoncached_reasoning_helpers.php");
   ```

3. **Remove from package:**
   - Delete `functions/functions.php`
   - Delete `functions/json_response.php`
   - Delete `lib/chat_helper_functions.php`

4. **Update documentation:**
   - Update INSTALLATION_INSTRUCTIONS.txt
   - Update CHANGELOG.txt
   - Note requirements for lib/data_functions.php

5. **Test:**
   - Install on fresh vanilla CHIM
   - Verify reasoning functions work
   - Verify minimize_quality_prompt works
   - Verify metadata saves correctly

---

## Dependencies and Caveats

### Critical Dependency: lib/data_functions.php

**The connector's reasoning functions only work if `lib/data_functions.php` calls them.**

**Three scenarios:**

1. **Vanilla CHIM already has the calls:**
   - Functions work out of the box
   - No additional steps needed

2. **Vanilla CHIM doesn't have the calls:**
   - Functions are defined but never used
   - Users need to manually patch `lib/data_functions.php`
   - OR include `lib/data_functions.php` in package

3. **Make calls optional in data_functions.php:**
   - Modify data_functions.php to use `function_exists()`:
   ```php
   // In data_functions.php streaming loop
   if (function_exists('extractReasoningFreeContent')) {
       $reasoningFreeBuffer = extractReasoningFreeContent($buffer);
       if ($reasoningFreeBuffer === false) {
           continue;
       }
       $buffer = $reasoningFreeBuffer;
   }

   // Later...
   if (function_exists('stripReasoningTokens')) {
       $buffer = stripReasoningTokens($buffer);
   }
   ```
   - Makes reasoning support optional
   - Works with or without connector
   - Requires modifying `lib/data_functions.php` and including in package

---

## Recommendation

**Two paths forward:**

### Path A: Minimal Overwrites (8 files)
- Extract reasoning to helpers
- Remove vanilla files (functions.php, json_response.php)
- Keep 3 necessary overwrites (prompts, llm_connector.class, UI)
- **Requires vanilla CHIM to have data_functions.php modifications**

### Path B: Include data_functions.php (9 files)
- Same as Path A
- PLUS include `lib/data_functions.php` with optional reasoning calls
- Works on vanilla CHIM without modifications
- One additional overwrite

**User should choose:**
- Path A if vanilla CHIM already has reasoning support
- Path B if vanilla CHIM needs data_functions.php modifications

---

## Questions for User

1. **Does vanilla CHIM's `lib/data_functions.php` have reasoning function calls?**
   - If YES → Path A (8 files)
   - If NO → Path B (9 files) or require manual patching

2. **Is there a vanilla CHIM repository to compare against?**
   - Would help identify exact differences
   - Can verify what's truly vanilla vs connector-specific

3. **Acceptable overwrites:**
   - prompts/dialogue_prompt.php (minimize_quality_prompt)
   - lib/core/llm_connector.class.php (metadata encoding)
   - ui/core/llm_connectors.php (config UI)
   - Are these acceptable? Or must ALL overwrites be eliminated?

Please clarify and I'll implement accordingly.
