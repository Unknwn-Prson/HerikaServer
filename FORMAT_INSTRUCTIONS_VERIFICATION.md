# Format Instructions Adaptation Verification

**Date**: 2025-11-12
**Version**: v1.1.2
**Status**: ✅ VERIFIED

---

## Verification Summary

Format instructions properly adapt to both response format settings (JSON/Simple) and all mood/action/listener/target toggle combinations.

---

## JSON Format Adaptation

**Location**: `openrouterjsoncached.php` lines 376-392

### Logic Flow
```php
1. Start with $GLOBALS["responseTemplate"] (full template)
2. Check each toggle:
   - if (!$this->_includeMood) → unset($template['mood'])
   - if (!$this->_includeActions) → unset($template['action'])
   - if (!$this->_includeTarget) → unset($template['target'])
   - if (!$this->_includeListener) → unset($template['listener'])
3. Build instruction: "Use ONLY this JSON object: " + json_encode($template)
```

### Test Cases

**All Enabled** (default):
```json
Template: {"character":"NPC","listener":"...","message":"...","mood":"...","action":"...","target":"..."}
```

**Only Mood + Message**:
```json
Template: {"character":"NPC","message":"...","mood":"..."}
(listener, action, target removed)
```

**Only Message** (all toggles off):
```json
Template: {"character":"NPC","message":"..."}
(mood, listener, action, target removed)
```

**✅ Verified**: JSON template correctly adapts to all toggle combinations

---

## Simple Format Adaptation

**Location**: `openrouterjsoncached_helpers.php` lines 475-517

### Logic Flow
```php
1. Build $parts array based on toggles:
   - if ($includeMood) $parts[] = 'mood'
   - if ($includeListener) $parts[] = 'listener'
   - if ($includeActions) $parts[] = 'action'
   - if ($includeTarget) $parts[] = 'target'
2. If empty($parts) → return "Respond naturally with your dialogue."
3. Build format: '(' + implode(')(', $parts) + ')'
4. Build instruction with descriptions
5. Build example with sample values
```

### Test Cases

**All Enabled**:
```
Format: (mood)(listener)(action)(target)
Instruction: "Begin your response by noting your emotional state, who you're speaking to,
              intended action, action target in parentheses like this: (mood)(listener)(action)(target)"
Example: (neutral)(Player)(Talk)(Lydia) I'm worried about that cave we passed.
```

**Mood + Listener Only**:
```
Format: (mood)(listener)
Instruction: "Begin your response by noting your emotional state, who you're speaking to
              in parentheses like this: (mood)(listener)"
Example: (neutral)(Player) I'm worried about that cave we passed.
```

**Actions + Target Only**:
```
Format: (action)(target)
Instruction: "Begin your response by noting your intended action, action target
              in parentheses like this: (action)(target)"
Example: (Talk)(Lydia) I'm worried about that cave we passed.
```

**Nothing Enabled**:
```
Instruction: "Respond naturally with your dialogue."
(No format markers required)
```

**✅ Verified**: Simple format correctly adapts to all toggle combinations

---

## Dependency Enforcement

**Location**: `openrouterjsoncached.php` lines 331-333

```php
// Enforce dependency: target required if actions enabled
if ($this->_includeActions) {
    $this->_includeTarget = true;
}
```

**Test Cases**:
- Actions enabled, Target disabled → **Target automatically enabled**
- Actions disabled, Target disabled → Both stay disabled
- Actions enabled, Target enabled → Both stay enabled

**✅ Verified**: Target automatically enabled when actions are enabled

---

## Configuration Loading

**Location**: `openrouterjsoncached.php` lines 309-328

```php
$this->_responseFormat = isset($GLOBALS["CONNECTOR"][$this->name]["response_format"])
    && in_array($GLOBALS["CONNECTOR"][$this->name]["response_format"], ['json', 'simple'])
    ? $GLOBALS["CONNECTOR"][$this->name]["response_format"]
    : 'json';

$this->_includeActions = isset($GLOBALS["CONNECTOR"][$this->name]["include_actions_list"])
    ? (bool)$GLOBALS["CONNECTOR"][$this->name]["include_actions_list"]
    : true;

$this->_includeMood = isset($GLOBALS["CONNECTOR"][$this->name]["include_mood_requirement"])
    ? (bool)$GLOBALS["CONNECTOR"][$this->name]["include_mood_requirement"]
    : true;

$this->_includeTarget = isset($GLOBALS["CONNECTOR"][$this->name]["include_target_requirement"])
    ? (bool)$GLOBALS["CONNECTOR"][$this->name]["include_target_requirement"]
    : true;

$this->_includeListener = isset($GLOBALS["CONNECTOR"][$this->name]["include_listener_requirement"])
    ? (bool)$GLOBALS["CONNECTOR"][$this->name]["include_listener_requirement"]
    : true;
```

**Defaults**:
- Response format: `json`
- Include actions: `true`
- Include mood: `true`
- Include target: `true`
- Include listener: `true`

**✅ Verified**: All settings properly loaded from configuration

---

## Logging Verification

**Location**: `openrouterjsoncached.php` line 335

```php
logMessage("Response Format Config: format={$this->_responseFormat}, actions={$this->_includeActions},
           mood={$this->_includeMood}, target={$this->_includeTarget}, listener={$this->_includeListener},
           uncached={$dialogue_cache_uncached_count}");
```

**Log Output Example**:
```
Response Format Config: format=simple, actions=1, mood=1, target=1, listener=0, uncached=4
```

**✅ Verified**: Configuration logged for debugging

---

## Edge Cases Handled

### Edge Case 1: All Toggles Disabled (Simple Format)
**Behavior**: Returns "Respond naturally with your dialogue." (line 484)
**Result**: ✅ Gracefully handles empty format

### Edge Case 2: Actions Without Target (JSON)
**Behavior**: Target automatically enabled (line 331-333)
**Result**: ✅ Prevents invalid configuration

### Edge Case 3: Actions Without Target (Simple)
**Behavior**: Target automatically enabled before format building
**Result**: ✅ Consistent across both formats

### Edge Case 4: Invalid Response Format
**Behavior**: Defaults to 'json' (line 311)
**Result**: ✅ Safe fallback

---

## Integration with UI

**Location**: `ui/core/llm_connectors.php`

**Response Format Section** (lines 453-471 Profiles, 1491-1509 Main):
- Format dropdown (json/simple)
- 4 toggle checkboxes:
  - Include Mood
  - Include Listener
  - Include Actions
  - Include Target

**Collapsible State**: Open by default (`open` attribute)

**✅ Verified**: UI provides all necessary controls

---

## Conclusion

**Status**: ✅ ALL CHECKS PASSED

The format instructions correctly adapt to:
1. ✅ Response format setting (json vs simple)
2. ✅ Mood toggle (on/off)
3. ✅ Listener toggle (on/off)
4. ✅ Actions toggle (on/off)
5. ✅ Target toggle (on/off)
6. ✅ Dependency enforcement (actions requires target)
7. ✅ Edge cases (all disabled, invalid values)
8. ✅ Configuration loading
9. ✅ Logging for debugging

**Ready for production use.**
