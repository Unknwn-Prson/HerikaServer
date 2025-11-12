# Format Instruction Location - Complete Trace

## User Question
"The 'format instruction' is related to the 'Minimize quality prompt' setting. Please find where it is actually inserted."

## Answer: They Are DIFFERENT Things

### 1. Format Instruction (openrouterjsoncached.php)

**Location**: Lines 353-407 in `openrouterjsoncached.php`

**What it is**: The instruction telling the LLM how to format its response (JSON structure or simple format with parentheses)

**Built from**:
```php
// Line 353-358: Prefix (if PATCH_PROMPT_ENFORCE_ACTIONS enabled)
$prefix = $GLOBALS["COMMAND_PROMPT_ENFORCE_ACTIONS"] ?? "";

// Line 360-364: Speech style reinforcement
$speechReinforcement = "Use #SpeechStyle." (if HERIKA_SPEECHSTYLE set)

// Line 370-372: Available actions list
$availableActions = $GLOBALS["COMMAND_PROMPT"]  // Lists: Attack, Talk, SearchMemory, etc.

// Line 374-401: Format instruction based on response_format setting
if (response_format === 'json') {
    $formatInstruction = "Use ONLY this JSON object: {template}"
} else {
    $formatInstruction = buildSimpleFormatInstruction(...)  // "(mood)(listener)(action)(target)"
}

// Line 403-407: Combine into actionsText
$actionsText = $availableActions . "\n" . $formatInstruction
```

**Where it goes**:
- Line 439: Appended to system prompt: `$finalSend = $systemContentCurrent . "\n" . $actionsText`
- Line 441-445: Added to system entry with cache_control marker
- Line 449: Cached in `system_cache_{format}_{herika}.tmp`

**Result**: Format instruction is part of the SYSTEM prompt and is CACHED

---

### 2. Minimize Quality Prompt (dialogue_prompt.php)

**Location**: Lines 29-48 in `prompts/dialogue_prompt.php`

**What it is**: Controls whether CHIM's core dialogue template is verbose or minimized

**Configuration**:
```php
// Line 34-36: Check connector-specific setting
$useMinimizedPrompt = $GLOBALS["CONNECTOR"][$currentModel]["minimize_quality_prompt"] ?? true;
```

**Effect**:
```php
if ($useMinimizedPrompt) {
    // Minimized (recommended for advanced models)
    $TEMPLATE_DIALOG = " Write {HERIKA_NAME}'s next dialogue line.";
} else {
    // Verbose (for older/smaller models)
    $TEMPLATE_DIALOG = " Write {HERIKA_NAME}'s next dialogue line." .
        " Avoid narrations, be original, creative, knowledgeable, use your own thoughts. " .
        " Review dialogue history to focus on conversation topic and to avoid repeating " .
        "sentences and phraseology from previous dialog lines.";
}
```

**Where it goes**: This is set as `$GLOBALS["TEMPLATE_DIALOG"]` and used by CHIM's core prompt building system, NOT by openrouterjsoncached connector directly.

---

## Complete Message Sequence Sent to LLM

### Part 1: System Message (CACHED)

Built in `_openPart2()` lines 412-449:

```
1. Static system prompt (character personality, world info, etc.)
2. Available actions list ($GLOBALS["COMMAND_PROMPT"])
   <available_actions_list>
   AVAILABLE ACTION: Attack (description)
   AVAILABLE ACTION: Talk
   </available_actions_list>
3. Format instruction ($formatInstruction)
   - JSON: "Use ONLY this JSON object: {template}"
   - Simple: "Begin your response by noting (mood)(listener)(action)(target)"
```

**Cache marker**: Placed on this entire system message

---

### Part 2: User Message with Dialogue History (PARTIALLY CACHED)

Built in `_openPart3()` lines 464-604:

Current implementation inserts in this order:

```
1. Dialogue history (old messages) - CACHED
   [0] Player: Hello
   [1] NPC: Hi there
   [2] Player: How are you?
   ...
   [N-3] NPC: I'm doing well

2. Dynamic environment - INSERTED HERE (line 582, count-2)
   "ASSISTANT: Environmental Context: Weather rainy, time evening..."

3. Custom last instruction (if enabled) - UNCACHED
   "Remember to be concise"

4. Last user message (instruction) - UNCACHED
   "Player: What do you think about dragons?"
```

**Cache marker**: Placed at calculated index to keep last N messages uncached

---

## Key Findings

### Finding 1: Format Instruction Location

**WHERE**: System prompt (Part 1), at the end
**WHEN**: Built in `_openPart2()` lines 374-407
**CACHED**: Yes, part of system cache
**RELATED TO**: Response format setting (json/simple), NOT minimize_quality_prompt

### Finding 2: Minimize Quality Prompt

**WHERE**: CHIM core dialogue template (`dialogue_prompt.php`)
**EFFECT**: Controls verbosity of dialogue instruction
**NOT USED**: By openrouterjsoncached connector directly
**PURPOSE**: For connectors that use CHIM's template system (not this connector)

### Finding 3: Dynamic Environment Insertion

**WHERE**: Inserted into dialogue history (Part 2)
**POSITION**: At `count - 2` (line 582)
**RESULT**: Placed BEFORE last message and custom instruction
**CACHED**: No (intentionally in uncached zone)

---

## Answer to User's Question

The "format instruction" is **NOT** related to "minimize_quality_prompt". They are separate:

1. **Format instruction** = JSON template or simple format instruction
   - Located in system prompt (CACHED)
   - Built in openrouterjsoncached.php lines 374-407
   - Controlled by `response_format` setting (json/simple)

2. **Minimize quality prompt** = Verbosity of dialogue template
   - Located in dialogue_prompt.php
   - Used by CHIM core system
   - NOT used by openrouterjsoncached connector
   - Controlled by `minimize_quality_prompt` setting

The format instruction is inserted at the **end of the system prompt** (line 439), which is the first message sent to the LLM and is cached.

---

## Actual Payload Structure

```json
{
  "messages": [
    {
      "role": "system",
      "content": [
        {
          "type": "text",
          "text": "You are Lydia...\n\n<available_actions_list>\n...\n</available_actions_list>\n\nUse ONLY this JSON object: {...}",
          "cache_control": {"type": "ephemeral"}
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "Player: Hello"},
        {"type": "text", "text": "Lydia: Hi"},
        ...
        {"type": "text", "text": "Player: Question", "cache_control": {"type": "ephemeral"}},
        {"type": "text", "text": "ASSISTANT: Environmental Context: ..."},
        {"type": "text", "text": "Remember to be concise"},
        {"type": "text", "text": "Player: What do you think?"}
      ]
    }
  ]
}
```

The format instruction is the part at the end of the system message: "Use ONLY this JSON object: {...}"
