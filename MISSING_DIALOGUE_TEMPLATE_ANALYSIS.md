# Missing Dialogue Template Prompt Analysis

## User's Observation

There's a prompt roughly "Respond as HERIKA_NAME" which:
- Is NOT the format instructions (actions/JSON)
- Should be inserted in the **uncached section**
- Is controlled by `minimize_quality_prompt` setting
- May be missing from openrouterjsoncached connector

## The Missing Prompt: TEMPLATE_DIALOG

### Where It's Defined

**File**: `prompts/dialogue_prompt.php` lines 29-48

**Content**:
```php
if ($useMinimizedPrompt) {
    // Minimized (recommended for advanced models)
    $TEMPLATE_DIALOG = " Write {HERIKA_NAME}'s next dialogue line.";
} else {
    // Verbose (for older/smaller models)
    $TEMPLATE_DIALOG = " Write {HERIKA_NAME}'s next dialogue line." .
        " Avoid narrations, be original, creative, knowledgeable, use your own thoughts. " .
        " Review dialogue history to focus on conversation topic and to avoid " .
        "repeating sentences and phraseology from previous dialog lines.";
}
```

**Configuration**:
```php
$useMinimizedPrompt = $GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"] ?? true;
```

### Where It SHOULD Be Used

Based on CHIM architecture and user's description:
- Should be in **uncached section** (changes per turn based on context)
- Should tell LLM how to roleplay/respond
- Should NOT be in system prompt (that's cached)
- Should NOT be format instructions (that's about JSON/simple format)

### Where It's Currently Used

**Search Results**:
```bash
grep -rn "TEMPLATE_DIALOG" --include="*.php" /home/user/HerikaServer/connector/
# NO RESULTS in connector directory!
```

**Conclusion**: `TEMPLATE_DIALOG` is **NOT USED** by openrouterjsoncached connector at all!

### Where Other Connectors Use It

Looking at other files that use TEMPLATE_DIALOG:
- `prompts/prompts.php` - Event cues (book reading, combat, etc.)
- Extensions use it for specific events

**Pattern**: Always appended to the last instruction/cue for the LLM

## Analysis: Why It's Missing

### Connector Variables Available

**openrouterjsoncached.php** has two instruction variables:

1. **customInstruction** (line 290):
   ```php
   $customInstruction = $GLOBALS["CONNECTOR"][$this->name]["custom_system_instruction"] ?? '';
   ```
   - Used in: Line 399, system prompt (CACHED)
   - Purpose: User-configured instruction for system

2. **lastCustomInstruction** (line 291):
   ```php
   $lastCustomInstruction = $GLOBALS["CONNECTOR"][$this->name]["custom_last_instruction"] ?? '';
   ```
   - Used in: Lines 504-506, dialogue section (UNCACHED)
   - Purpose: User-configured instruction added near end

**Neither of these is TEMPLATE_DIALOG!**

### What Should Happen

TEMPLATE_DIALOG should be:
1. Read from `$GLOBALS["TEMPLATE_DIALOG"]` (set by dialogue_prompt.php)
2. Inserted in uncached section
3. Either:
   - Appended to last user message, OR
   - Added as separate message before last message, OR
   - Added after custom_last_instruction

## Proposed Location for TEMPLATE_DIALOG

### Option A: Append to Last User Message

```php
// Line 509: Instead of just adding instruction
$instruction = array_pop($contentTextToSend);

// NEW: Append TEMPLATE_DIALOG if available
if (isset($GLOBALS["TEMPLATE_DIALOG"]) && !empty($GLOBALS["TEMPLATE_DIALOG"])) {
    $instruction['text'] .= $GLOBALS["TEMPLATE_DIALOG"];
}

$completeEventList[] = $instruction;
```

**Pros**: Simple, keeps it with user message
**Cons**: Modifies user message content

### Option B: Add as Separate Message Before custom_last_instruction

```php
// Line 506-509
if (!empty($lastCustomInstruction)) {
    $addToIndex = 1;
    $completeEventList[] = ['type' => 'text', 'text' => $lastCustomInstruction];
}

// NEW: Add TEMPLATE_DIALOG as separate message
if (isset($GLOBALS["TEMPLATE_DIALOG"]) && !empty($GLOBALS["TEMPLATE_DIALOG"])) {
    $addToIndex++;
    $completeEventList[] = ['type' => 'text', 'text' => $GLOBALS["TEMPLATE_DIALOG"]];
}

$completeEventList[] = $instruction;
```

**Pros**: Keeps it separate, clear structure
**Cons**: Increases message count, affects cache calculation

### Option C: Add After Last User Message (Most Logical)

```php
// Lines 506-509
if (!empty($lastCustomInstruction)) {
    $addToIndex = 1;
    $completeEventList[] = ['type' => 'text', 'text' => $lastCustomInstruction];
}

$completeEventList[] = $instruction;

// NEW: Add TEMPLATE_DIALOG after instruction
if (isset($GLOBALS["TEMPLATE_DIALOG"]) && !empty($GLOBALS["TEMPLATE_DIALOG"])) {
    $addToIndex++;
    $completeEventList[] = ['type' => 'text', 'text' => $GLOBALS["TEMPLATE_DIALOG"]];
}
```

**Pros**: Follows CHIM pattern (instruction → cue), uncached
**Cons**: Changes final order

## Recommendation

**Option C** seems most aligned with CHIM architecture:
1. Dialogue history (cached)
2. Dynamic environment (uncached)
3. Custom last instruction (uncached, if set)
4. Last user message (uncached)
5. **TEMPLATE_DIALOG** (uncached) ← NEW

This puts the roleplay instruction at the very end, right before the LLM generates its response.

## Impact of minimize_quality_prompt Setting

With this implementation:
- **Enabled** (minimized): Adds short instruction "Write {NAME}'s next dialogue line."
- **Disabled** (verbose): Adds longer instruction with quality guidelines

This allows users to:
- Use minimized for advanced models (Claude, GPT-4) that don't need explicit guidance
- Use verbose for smaller models that benefit from detailed instructions

## Requires dialogue_prompt.php to be Loaded

**CRITICAL**: For this to work, `prompts/dialogue_prompt.php` must be loaded before connector runs, which should set `$GLOBALS["TEMPLATE_DIALOG"]`.

**Check**: Where is dialogue_prompt.php loaded?

Let me verify this is happening...
