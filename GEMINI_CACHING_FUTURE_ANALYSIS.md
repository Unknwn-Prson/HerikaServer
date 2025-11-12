# Gemini Caching - For Future Analysis

**Status**: NOT INVESTIGATED YET
**Priority**: LOW - Only analyze when Gemini caching is actually needed
**Last Updated**: 2025-11-12

---

## Background

The cache connector has a special caching formula for Gemini provider that differs from the standard Anthropic/OpenAI approach. The formula produces irregular cache intervals, and it's unclear whether this is intentional or a bug.

## The Formula

```php
// openrouterjsoncached.php lines 522-536
if ($this->_provider_caching == "Gemini") {
    logMessage("Using gemini caching (ignores dialogue_cache_uncached_count)");
    $offset = 10;
    $elements = count($completeEventList);
    $batchSize = $CONTEXTHISTORY - $offset;  // e.g., 50 - 10 = 40
    $batchNumber = floor($elements / $batchSize);

    $indexToCache = max(0, ($batchNumber * $CONTEXTHISTORY) - $offset);

    if ($indexToCache >= $elements) {
        $indexToCache = $elements - 1;
    }

    // v1.1.1 bounds check (prevents crashes)
    if ($indexToCache == 0) {
        if ($elements > 33) {
            $indexToCache = 33;
        } else {
            $indexToCache = -1;  // Skip caching
        }
    }
}
```

## Observed Behavior

With `CONTEXTHISTORY = 50` (typical):

| Elements | batchNumber | Formula | Cache Index | Interval from Previous |
|----------|-------------|---------|-------------|------------------------|
| 0-39     | 0           | (0×50)-10 | 0 → 33     | First cache            |
| 40-79    | 1           | (1×50)-10 | 40         | +40 from start         |
| 80-119   | 2           | (2×50)-10 | 90         | +50 from previous      |
| 120-159  | 3           | (3×50)-10 | 140        | +50 from previous      |

**Pattern**: First interval is special (33 or 40), then regular 50-message intervals.

## Questions for Future Investigation

### 1. How does Gemini's context caching API actually work?
   - Does it use `cache_control` markers like Anthropic?
   - What are the minimum/maximum cache size requirements?
   - Does it batch cache entries?
   - Does it have special refresh intervals?

### 2. Is the irregular interval pattern intentional?
   - **Hypothesis A**: Bug - formula mixes `batchSize` (40) and `CONTEXTHISTORY` (50)
   - **Hypothesis B**: Intentional - Gemini requires bootstrap cache, then regular intervals
   - **Hypothesis C**: Intentional - Gemini has specific chunk/batch requirements

### 3. What is `CONTEXTHISTORY`?
   - Default value: 50
   - Is this Gemini-specific?
   - Does it relate to Gemini's context window?
   - Where is it configured?

### 4. Where did this formula originate?
   - Original CHIM 1.3.5 connector?
   - Gemini API documentation?
   - Empirical testing?
   - User's implementation?

### 5. Why ignore `dialogue_cache_uncached_count`?
   - Standard caching uses this setting to keep last N messages uncached
   - Gemini ignores it completely
   - Is this because Gemini caching works fundamentally differently?

## Potential Issues Identified (Unverified)

### Issue A: Formula Inconsistency
**Observation**: Uses `batchSize` for calculating batch number, but `CONTEXTHISTORY` for cache placement
```php
$batchNumber = floor($elements / $batchSize);           // Uses 40
$indexToCache = ($batchNumber * $CONTEXTHISTORY) - $offset;  // Uses 50
```

**If this is a bug, possible fix**:
```php
// Option 1: Use batchSize consistently
$indexToCache = ($batchNumber * $batchSize) - 1;  // 0, 39, 79, 119...

// Option 2: Use CONTEXTHISTORY consistently
$batchNumber = floor($elements / $CONTEXTHISTORY);
$indexToCache = ($batchNumber * $CONTEXTHISTORY) - $offset;  // 0, 40, 90, 140...
```

**If this is intentional**: Document why Gemini needs this specific pattern.

### Issue B: addToIndex Not Accounted For
**Observation**: Standard caching adjusts for `custom_last_instruction`, Gemini doesn't
```php
// Standard path (line 516):
$lastIndex = $totalElements - $dialogue_cache_uncached_count - 1 - $addToIndex;

// Gemini path (line 531):
$indexToCache = max(0, ($batchNumber * $CONTEXTHISTORY) - $offset);
// No adjustment for addToIndex!
```

**Possible fix** (if needed):
```php
if ($addToIndex > 0) {
    $indexToCache = max(0, $indexToCache - $addToIndex);
}
```

### Issue C: Minimum Token Approximation
**Observation**: v1.1.1 fix assumes 33 entries ≈ 33 tokens

**Reality**: Token count varies widely based on message content
- Short messages: 1-5 tokens each
- Long messages: 50-100+ tokens each

**Current approach**: Conservative heuristic (good enough for safety)

**For accuracy**: Would need actual tokenizer for the specific model

## What's Not in Question

### v1.1.1 Bounds Check Fix (lines 541-550)
**This fix is CORRECT and should be kept**:
```php
if ($indexToCache == 0) {
    if ($elements > 33) {
        $indexToCache = 33;
    } else {
        $indexToCache = -1;  // Skip caching
    }
}
```

This prevents array bounds errors when:
- Early in conversation (few messages)
- Array doesn't have 34+ elements

**Benefit**: Prevents crashes, gracefully skips caching when insufficient messages

## Recommendation

**DO NOT change Gemini formula** until:
1. Gemini context caching API is documented/understood
2. User confirms Gemini caching is actually being used
3. Testing shows current formula doesn't work

**Priority**: LOW - Only investigate if:
- User reports Gemini caching issues
- User actually uses Gemini provider
- Gemini caching becomes critical feature

## References

- `CACHE_LOGIC_ANALYSIS.md` - Original analysis (Issue #3)
- `GEMINI_FORMULA_ANALYSIS.md` - Detailed formula trace
- `openrouterjsoncached.php` lines 522-558 - Implementation

---

**Note**: This document should be revisited only when Gemini caching is actively needed or issues are reported. Current implementation works (doesn't crash), and changing it without understanding Gemini's requirements could break working functionality.
