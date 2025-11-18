# CHIM Cached Connector - File Interactions & Overwrite Analysis
## Version 1.3.3 - Comprehensive Dependency Map

**Purpose:** Analyze all file interactions to identify opportunities for minimizing overwrites
**Created:** 2025-11-16
**Scope:** All files currently included in CHIM Cached Connector v1.3.3 package

---

## Executive Summary

**Current State:** Package overwrites **11 files** across 5 directories
**Analysis Goal:** Understand dependencies to reduce overwrites in future versions
**Key Finding:** Most overwrites are due to tightly coupled architecture in CHIM core

### Files Overwritten by Package:
1. `connector/openrouterjsoncached.php` ← Core connector (MUST overwrite)
2. `connector/openrouterjsoncached_helpers.php` ← Connector helpers (MUST overwrite)
3. `connector/openrouterjsoncached_verbose.php` ← Debug version (COULD avoid)
4. `connector/OPENROUTERJSONCACHED_README.md` ← Documentation (COULD avoid)
5. `lib/core/llm_connector.class.php` ← Core LLM class (MUST overwrite)
6. `lib/chat_helper_functions.php` ← Chat processing (MUST overwrite)
7. `lib/data_functions.php` ← Main processing loop (MUST overwrite - v1.3.3)
8. `ui/core/llm_connectors.php` ← Configuration UI (MUST overwrite)
9. `ui/events-memories.php` ← Context UI (COULD avoid)
10. `prompts/dialogue_prompt.php` ← Prompt templates (COULD avoid)
11. `conf/conf_schema.json` ← Configuration schema (MUST overwrite)

---

## Part 1: Connector Core Files

### 1.1 connector/openrouterjsoncached.php
**Role:** Main connector implementation
**Size:** ~73 KB
**Version-Specific Code:** v1.3.3

**What It Does:**
- Handles API communication with OpenRouter/OpenAI/Anthropic/Gemini
- Implements streaming response processing
- Manages prompt caching (Anthropic/OpenAI/Gemini)
- Parses JSON and Simple format responses
- Handles thinking/reasoning tokens
- Splits simple format into sentences for streaming (v1.3.3)
- Returns complete sentences via handlesSentenceSplitting() (v1.3.3)

**Dependencies (Files It Requires):**
```
REQUIRES:
├── connector/openrouterjsoncached_helpers.php
│   └── buildSimpleFormatInstruction()
│   └── extractJson()
│   └── stripReasoningTokens()
│   └── extractReasoningFreeContent()
│   └── Various helper functions
│
├── connector/__jpd.php (CHIM core)
│   └── JSON parsing utilities
│
└── GLOBALS from main.php (CHIM core)
    ├── $GLOBALS["CONNECTOR"][$name] - connector configuration
    ├── $GLOBALS["HERIKA_NAME"] - character name
    ├── $GLOBALS["responseTemplate"] - JSON template
    ├── $GLOBALS["COMMAND_PROMPT"] - action prompts
    ├── $GLOBALS["COMMAND_PROMPT_ENFORCE_ACTIONS"] - action enforcement
    └── $GLOBALS["SCRIPTLINE_*"] - response metadata
```

**Depended On By:**
```
CALLED BY:
├── lib/data_functions.php
│   └── $connectionHandler->open() - Initialize connection
│   └── $connectionHandler->process() - Stream processing loop (v1.3.3 checks handlesSentenceSplitting())
│   └── $connectionHandler->processActions() - Action extraction
│   └── $connectionHandler->isDone() - Check completion
│   └── $connectionHandler->close() - Cleanup
│
├── lib/dynamic_update_util.php
│   └── $connectionHandler->process() - Dynamic updates
│
└── lib/rolemaster_helpers.php
    └── $connectionHandler->process() - Role processing
```

**Why We Must Overwrite:**
1. **v1.3.3 Feature:** Added `handlesSentenceSplitting()` method (lines 134-138)
2. **v1.3.2 Feature:** Modified `$customInstruction` handling (lines 427, 431, 482-484)
3. **v1.3.1 Feature:** Added conditional "Provide variety" filtering (lines 383-387)
4. **v1.2.x Features:** Caching, thinking toggle, simple format parsing
5. **Core Architecture:** This file IS the connector - cannot be separated

**Opportunities to Avoid Overwrite:**
- **NONE** - This is the core connector file and must be overwritten for any connector updates

---

### 1.2 connector/openrouterjsoncached_helpers.php
**Role:** Helper functions for connector
**Size:** ~20 KB
**Version-Specific Code:** v1.1.20 (no changes in v1.3.x)

**What It Does:**
- `buildSimpleFormatInstruction()` - Constructs simple format prompt
- `extractJson()` - Extracts JSON from mixed content
- `stripReasoningTokens()` - Removes thinking tags from text
- `extractReasoningFreeContent()` - Handles unclosed reasoning markers
- Various parsing and utility functions

**Dependencies (Files It Requires):**
```
REQUIRES:
├── GLOBALS from main.php
│   └── $GLOBALS["HERIKA_NAME"] - for format instructions
│
└── No direct file dependencies
```

**Depended On By:**
```
CALLED BY:
└── connector/openrouterjsoncached.php
    ├── buildSimpleFormatInstruction() - Called at line 432
    ├── extractJson() - Called at line 1306
    ├── stripReasoningTokens() - Called at line 1323
    └── extractReasoningFreeContent() - Used in buffer processing
```

**Why We Must Overwrite:**
1. **Simple Format Support:** buildSimpleFormatInstruction() required for simple format
2. **Reasoning Support:** stripReasoningTokens() required for thinking models
3. **Tight Coupling:** Functions are called directly by connector

**Opportunities to Avoid Overwrite:**
- **MEDIUM** - Could potentially be dynamically loaded/required only when needed
- **CHALLENGE:** Connector code directly calls these functions, would need refactoring

---

### 1.3 connector/openrouterjsoncached_verbose.php
**Role:** Verbose/debug version of connector
**Size:** ~94 KB
**Version-Specific Code:** v1.0.12 (unchanged since initial version)

**What It Does:**
- Same as openrouterjsoncached.php but with extensive logging
- Used for debugging and troubleshooting
- NOT the primary connector in normal use

**Dependencies (Files It Requires):**
```
REQUIRES:
└── Same as openrouterjsoncached.php
```

**Depended On By:**
```
CALLED BY:
└── Only if user explicitly selects "openrouterjsoncached_verbose" as connector
```

**Why We Currently Overwrite:**
1. **Completeness:** Included for debugging purposes
2. **Consistency:** Keep verbose version in sync with main version

**Opportunities to Avoid Overwrite:**
- **HIGH** - This file is OPTIONAL and rarely used
- **RECOMMENDATION:** Exclude from package, provide separately as debug tool
- **IMPACT:** Would reduce package size by ~94 KB and one file overwrite

---

### 1.4 connector/OPENROUTERJSONCACHED_README.md
**Role:** Connector documentation
**Size:** ~9 KB
**Version-Specific Code:** Documentation only

**What It Does:**
- Documents connector features and usage
- Provides technical reference
- No executable code

**Dependencies (Files It Requires):**
```
REQUIRES:
└── None (documentation only)
```

**Depended On By:**
```
CALLED BY:
└── None (documentation only)
```

**Why We Currently Overwrite:**
1. **Documentation:** Keeps docs in sync with connector version
2. **Convention:** Standard practice to include README

**Opportunities to Avoid Overwrite:**
- **HIGH** - This file is DOCUMENTATION and has no code dependencies
- **RECOMMENDATION:** Host documentation externally (GitHub wiki, separate docs repo)
- **IMPACT:** Would reduce one file overwrite with zero code impact

---

## Part 2: Core Library Files

### 2.1 lib/core/llm_connector.class.php
**Role:** Base class for all LLM connectors
**Size:** ~18 KB
**Version-Specific Code:** v1.1.20+ (metadata handling)

**What It Does:**
- Defines base LLMConnector class
- Handles connector instantiation
- Manages metadata persistence (save/load configuration)
- Provides common methods for all connectors

**Dependencies (Files It Requires):**
```
REQUIRES:
├── Database connection ($GLOBALS["db"])
│   └── For saving/loading connector metadata
│
└── GLOBALS from main.php
    └── Connector configuration arrays
```

**Depended On By:**
```
CALLED BY:
├── ui/core/llm_connectors.php
│   └── For saving connector configurations
│   └── For loading connector metadata
│
└── All connector implementations (inherit from this class)
```

**Why We Must Overwrite:**
1. **Metadata Fields:** Handles new configuration fields (minimize_quality_prompt, etc.)
2. **Save/Load Logic:** Persists connector settings to database
3. **Base Class:** All connectors depend on this for metadata management

**Opportunities to Avoid Overwrite:**
- **LOW** - Core infrastructure file that manages metadata
- **CHALLENGE:** Would require plugin/hook system for metadata extension
- **POTENTIAL:** Could add metadata extension hooks to CHIM core

---

### 2.2 lib/chat_helper_functions.php
**Role:** Chat processing and output functions
**Size:** ~70 KB
**Version-Specific Code:** Various CHIM versions

**What It Does:**
- `returnLines()` - Sends sentences to game/TTS
- Mood processing and extraction
- Translation integration
- TTS generation calls
- Response formatting and cleanup
- Listener/target processing

**Dependencies (Files It Requires):**
```
REQUIRES:
├── Database ($GLOBALS["db"])
│   └── For logging and context
│
├── TTS modules (various tts/*.php files)
│   └── For text-to-speech generation
│
├── Translation class
│   └── For multilingual support
│
└── GLOBALS from main.php
    ├── $GLOBALS["HERIKA_NAME"]
    ├── $GLOBALS["PLAYER_NAME"]
    ├── $GLOBALS["SCRIPTLINE_*"]
    └── Various configuration globals
```

**Depended On By:**
```
CALLED BY:
├── lib/data_functions.php
│   └── returnLines($sentences) - Called at lines 3008, 3053
│   └── Main processing loop for sending responses
│
└── Various other CHIM core functions
```

**Why We Must Overwrite:**
1. **Connector Integration:** returnLines() processes connector output
2. **Response Flow:** Critical path for all LLM responses
3. **TTS Integration:** Handles text-to-speech generation
4. **CHIM Core:** This is a core CHIM file, not connector-specific

**Opportunities to Avoid Overwrite:**
- **VERY LOW** - This is a CHIM core file
- **NOTE:** We may not actually need to overwrite this for connector features
- **ACTION NEEDED:** Audit whether connector package truly requires this file

---

### 2.3 lib/data_functions.php
**Role:** Main LLM processing loop and data handling
**Size:** ~182 KB
**Version-Specific Code:** v1.3.3 (streaming logic added)

**What It Does:**
- Main while() loop that processes LLM streaming responses
- Calls `$connectionHandler->process()` repeatedly
- Manages sentence extraction and buffering
- **v1.3.3 NEW:** Conditional MINIMUM_SENTENCE_SIZE logic (lines 2988-3019)
- Handles reasoning token stripping
- Calls returnLines() to send sentences
- Manages user interrupt detection

**Dependencies (Files It Requires):**
```
REQUIRES:
├── Connector object ($connectionHandler)
│   └── process() - called at line 2961
│   └── handlesSentenceSplitting() - checked at line 2990 (v1.3.3)
│   └── isDone() - checked at line 2972
│   └── processActions() - called at line 3062
│
├── lib/chat_helper_functions.php
│   └── returnLines($sentences) - called at lines 3008, 3053
│
├── Translation class
│   └── For translation support
│
└── Database ($GLOBALS["db"])
    └── For user interrupt detection
```

**Depended On By:**
```
CALLED BY:
├── main.php (CHIM core)
│   └── Main request processing
│
└── Various CHIM endpoints
```

**Why We Must Overwrite:**
1. **v1.3.3 CRITICAL:** Added handlesSentenceSplitting() check (lines 2988-2991)
2. **v1.3.3 CRITICAL:** Conditional buffer size logic (lines 2993-2998)
3. **v1.3.3 CRITICAL:** Conditional position check (lines 3010-3017)
4. **Streaming Performance:** Enables immediate sentence sending for simple format
5. **CHIM Core:** This is a core CHIM file, not connector-specific

**Opportunities to Avoid Overwrite:**
- **LOW** - Contains critical v1.3.3 streaming logic
- **POTENTIAL:** Could add hooks/callbacks to CHIM core for custom buffering logic
- **CHALLENGE:** Would require significant CHIM core refactoring

**Why This Is Problematic:**
- This is a HUGE CHIM core file (182 KB)
- We only modify ~30 lines (2988-3019)
- Overwrites 182 KB to change 30 lines
- High risk of conflicts with future CHIM updates

---

## Part 3: User Interface Files

### 3.1 ui/core/llm_connectors.php
**Role:** Connector configuration interface
**Size:** ~150 KB
**Version-Specific Code:** v1.3.3 (version display)

**What It Does:**
- Renders connector configuration forms
- Handles connector creation/editing/deletion
- Manages metadata fields (minimize_quality_prompt, custom instructions, etc.)
- JavaScript for form consolidation and submission
- Displays connector version information

**Dependencies (Files It Requires):**
```
REQUIRES:
├── lib/core/llm_connector.class.php
│   └── LLMConnector::save() - for persisting configurations
│   └── LLMConnector::load() - for loading configurations
│
├── Database ($GLOBALS["db"])
│   └── For connector CRUD operations
│
└── conf/conf_schema.json
    └── For configuration field definitions
```

**Depended On By:**
```
CALLED BY:
└── CHIM web interface
    └── Settings → LLM Connectors page
```

**Why We Must Overwrite:**
1. **New Fields:** Adds configuration fields (minimize_quality_prompt, caching settings, etc.)
2. **Version Display:** Shows connector version (v1.3.3 at lines 410, 1443)
3. **UI Integration:** Forms must match connector capabilities
4. **Field Mapping:** JavaScript consolidation must handle new metadata fields

**Opportunities to Avoid Overwrite:**
- **MEDIUM** - Could potentially use plugin/template system
- **CHALLENGE:** Would require CHIM core to support UI extensions
- **POTENTIAL:** Dynamic field injection via JavaScript/CSS

**Why This Is Problematic:**
- This is a HUGE CHIM core UI file (150 KB)
- We add relatively few fields (maybe 20-30 lines total)
- Overwrites 150 KB to add a few configuration fields
- High risk of conflicts with future CHIM UI updates

---

### 3.2 ui/events-memories.php
**Role:** Events and memories management interface
**Size:** ~59 KB
**Version-Specific Code:** Unknown (may not be connector-specific)

**What It Does:**
- Manages conversation context/memories
- Event log viewing and management
- Context history display

**Dependencies (Files It Requires):**
```
REQUIRES:
├── Database ($GLOBALS["db"])
│   └── For event/memory CRUD operations
│
└── CHIM core globals
```

**Depended On By:**
```
CALLED BY:
└── CHIM web interface
    └── Events/Memories page
```

**Why We Currently Overwrite:**
1. **Unclear** - This file may not actually be connector-specific
2. **Historical:** May have been included in earlier versions for completeness

**Opportunities to Avoid Overwrite:**
- **VERY HIGH** - This file appears to be CHIM core, not connector-specific
- **ACTION NEEDED:** Audit whether connector package truly requires this file
- **RECOMMENDATION:** Remove from package if not connector-specific

---

## Part 4: Configuration and Prompt Files

### 4.1 conf/conf_schema.json
**Role:** Configuration schema definitions
**Size:** ~87 KB
**Version-Specific Code:** Various

**What It Does:**
- Defines available configuration options for all of CHIM
- Specifies field types, defaults, validation
- Used by configuration UI to render forms

**Dependencies (Files It Requires):**
```
REQUIRES:
└── None (data file)
```

**Depended On By:**
```
CALLED BY:
├── ui/core/llm_connectors.php
│   └── For rendering configuration forms
│
└── conf/conf_loader.php
    └── For loading configuration schema
```

**Why We Must Overwrite:**
1. **New Fields:** Defines new connector configuration options
2. **Schema Validation:** Ensures configuration integrity
3. **UI Generation:** Used to auto-generate configuration forms

**Opportunities to Avoid Overwrite:**
- **LOW** - Core configuration file
- **POTENTIAL:** Schema merging/extension system
- **CHALLENGE:** Would require CHIM core to support schema plugins

**Why This Is Problematic:**
- This is ENTIRE CHIM configuration schema (87 KB)
- We only add connector-specific fields (small portion)
- Overwrites all CHIM config schema to add connector fields
- High risk of conflicts with CHIM updates

---

### 4.2 prompts/dialogue_prompt.php
**Role:** Dialogue prompt templates
**Size:** ~5 KB
**Version-Specific Code:** v1.1.20+ (minimize_quality_prompt toggle)

**What It Does:**
- Defines $TEMPLATE_DIALOG prompt template
- Implements minimize_quality_prompt toggle
- Provides quality instruction variations

**Dependencies (Files It Requires):**
```
REQUIRES:
└── GLOBALS from main.php
    ├── $GLOBALS["HERIKA_NAME"]
    └── Connector configuration for minimize_quality_prompt
```

**Depended On By:**
```
CALLED BY:
├── CHIM core prompt system
│   └── When building dialogue prompts
│
└── connector/openrouterjsoncached_helpers.php
    └── Uses same quality instruction text in buildSimpleFormatInstruction()
```

**Why We Currently Overwrite:**
1. **minimize_quality_prompt:** Provides toggle between minimal/full quality instructions
2. **Consistency:** Simple format should match dialogue prompt behavior
3. **Quality Text:** Source of quality instruction text used by connector

**Opportunities to Avoid Overwrite:**
- **HIGH** - This is a CHIM core prompt file
- **ALTERNATIVE:** Connector could define its own quality text without requiring this file
- **RECOMMENDATION:** Move quality instruction text into connector helpers

---

## Part 5: Interaction Flow Diagrams

### 5.1 Request Processing Flow
```
User Input
    ↓
main.php (CHIM core)
    ↓
lib/data_functions.php::getResponse()
    ├→ $connectionHandler->open()
    │       ↓
    │   connector/openrouterjsoncached.php::open()
    │       ├→ Reads: conf/conf_schema.json (via GLOBALS)
    │       ├→ Reads: ui/core/llm_connectors.php (saved metadata)
    │       ├→ Reads: prompts/dialogue_prompt.php (quality instructions)
    │       └→ Builds request payload
    │
    ├→ while (!isDone())
    │   ├→ $connectionHandler->process()
    │   │       ↓
    │   │   connector/openrouterjsoncached.php::process()
    │   │       ├→ Calls: connector/openrouterjsoncached_helpers.php functions
    │   │       ├→ Returns: One sentence (simple) or chunk (JSON)
    │   │       └→ handlesSentenceSplitting() = true for simple format (v1.3.3)
    │   │
    │   ├→ Check: $connectionHandler->handlesSentenceSplitting() (v1.3.3)
    │   │       ↓
    │   ├→ Conditional: Bypass MINIMUM_SENTENCE_SIZE if simple format (v1.3.3)
    │   │
    │   └→ returnLines($sentences)
    │           ↓
    │       lib/chat_helper_functions.php::returnLines()
    │           ├→ TTS generation
    │           ├→ Translation (if enabled)
    │           └→ Output to game
    │
    └→ $connectionHandler->processActions()
            ↓
        connector/openrouterjsoncached.php::processActions()
```

### 5.2 Configuration Flow
```
User Opens Settings
    ↓
ui/core/llm_connectors.php
    ├→ Loads: conf/conf_schema.json (field definitions)
    ├→ Loads: lib/core/llm_connector.class.php (metadata)
    └→ Renders: Configuration form with new fields
        ├→ minimize_quality_prompt checkbox
        ├→ custom_system_instruction textarea
        ├→ max_dialogue_cache_context_size input
        └→ Caching settings

User Saves Configuration
    ↓
ui/core/llm_connectors.php::JavaScript consolidation
    └→ Collects all metadata fields

POST to server
    ↓
lib/core/llm_connector.class.php::save()
    └→ Persists to database

Next Request
    ↓
connector/openrouterjsoncached.php::open()
    ├→ Reads: $GLOBALS["CONNECTOR"][$name]["minimize_quality_prompt"]
    ├→ Reads: $GLOBALS["CONNECTOR"][$name]["custom_system_instruction"]
    └→ Uses configuration in request building
```

---

## Part 6: Critical Dependencies Summary

### 6.1 Files That MUST Be Overwritten (7 files)

1. **connector/openrouterjsoncached.php**
   - Reason: Core connector implementation
   - Can't avoid: This IS the connector
   - Dependencies: helpers, CHIM globals

2. **connector/openrouterjsoncached_helpers.php**
   - Reason: Required functions for connector
   - Can't avoid: Directly called by connector
   - Dependencies: None (standalone functions)

3. **lib/core/llm_connector.class.php**
   - Reason: Manages metadata persistence
   - Can't avoid: Base class for all connectors
   - Could improve: Add metadata extension hooks to CHIM

4. **lib/data_functions.php**
   - Reason: v1.3.3 streaming logic
   - Can't avoid: Contains critical performance fix
   - **PROBLEMATIC:** 182 KB file for 30 lines of changes
   - Could improve: Add processing hooks to CHIM core

5. **ui/core/llm_connectors.php**
   - Reason: Configuration UI for new fields
   - Can't avoid: Must render new metadata fields
   - **PROBLEMATIC:** 150 KB file for ~20 lines of new fields
   - Could improve: Dynamic field injection system in CHIM

6. **conf/conf_schema.json**
   - Reason: Defines new configuration fields
   - Can't avoid: Required for configuration system
   - **PROBLEMATIC:** 87 KB file to add a few connector fields
   - Could improve: Schema merging/plugin system in CHIM

7. **lib/chat_helper_functions.php**
   - Reason: **UNCLEAR** - May not actually be needed
   - **ACTION REQUIRED:** Audit if this is truly necessary
   - If not needed: Remove from package

---

### 6.2 Files That COULD Be Avoided (4 files)

1. **connector/openrouterjsoncached_verbose.php**
   - Current use: Debug/troubleshooting version
   - Why included: Completeness
   - **Recommendation:** Remove from package, provide separately
   - **Impact:** Save 94 KB, reduce 1 file overwrite

2. **connector/OPENROUTERJSONCACHED_README.md**
   - Current use: Documentation
   - Why included: Convention
   - **Recommendation:** Host externally (GitHub wiki/docs)
   - **Impact:** Save 9 KB, reduce 1 file overwrite

3. **ui/events-memories.php**
   - Current use: **UNCLEAR** - May not be connector-specific
   - Why included: Historical/unknown
   - **ACTION REQUIRED:** Audit if this is truly necessary
   - **Recommendation:** Remove if not connector-specific
   - **Impact:** Save 59 KB, reduce 1 file overwrite

4. **prompts/dialogue_prompt.php**
   - Current use: Quality instruction text source
   - Why included: minimize_quality_prompt toggle
   - **Alternative:** Define quality text in connector helpers
   - **Recommendation:** Move quality text to connector code
   - **Impact:** Save 5 KB, reduce 1 file overwrite

---

## Part 7: Overwrite Impact Analysis

### 7.1 Current Package Statistics
```
Total Files: 11
Total Size: ~812 KB

Breakdown:
- Connector files: 4 files, ~196 KB (MUST overwrite)
- Library files: 3 files, ~270 KB (2 MUST, 1 UNCLEAR)
- UI files: 2 files, ~209 KB (1 MUST, 1 UNCLEAR)
- Config files: 1 file, ~87 KB (MUST overwrite)
- Prompt files: 1 file, ~5 KB (COULD avoid)
```

### 7.2 Minimal Required Overwrites
If we audit and optimize, could potentially reduce to:
```
Required Files: 6-7 files (down from 11)
Required Size: ~640 KB (down from ~812 KB)

Remove:
- connector/openrouterjsoncached_verbose.php (-94 KB)
- connector/OPENROUTERJSONCACHED_README.md (-9 KB)
- ui/events-memories.php (-59 KB, if not needed)
- prompts/dialogue_prompt.php (-5 KB, move quality text to helpers)
- lib/chat_helper_functions.php (-70 KB, if not needed)

Keep:
- connector/openrouterjsoncached.php (CORE)
- connector/openrouterjsoncached_helpers.php (REQUIRED)
- lib/core/llm_connector.class.php (METADATA)
- lib/data_functions.php (v1.3.3 STREAMING)
- ui/core/llm_connectors.php (CONFIG UI)
- conf/conf_schema.json (SCHEMA)
```

### 7.3 Problematic Overwrites

**Most Problematic: lib/data_functions.php**
- Size: 182 KB
- Changes: ~30 lines (lines 2988-3019)
- Ratio: 0.016% of file modified
- Risk: High conflict potential with CHIM updates
- Recommendation: Request CHIM core add hooks for custom buffering logic

**Second Most Problematic: ui/core/llm_connectors.php**
- Size: 150 KB
- Changes: ~20-30 lines (new fields)
- Ratio: 0.02% of file modified
- Risk: High conflict potential with CHIM UI updates
- Recommendation: Request CHIM core add dynamic field injection

**Third Most Problematic: conf/conf_schema.json**
- Size: 87 KB
- Changes: Connector-specific fields only
- Ratio: Unknown (JSON structure)
- Risk: Medium conflict potential
- Recommendation: Request CHIM core support schema merging

---

## Part 8: Recommendations for Future Versions

### 8.1 Immediate Actions (Can Do Now)

1. **Audit lib/chat_helper_functions.php**
   - Determine if truly needed for connector
   - If not: Remove from package
   - Impact: -70 KB, -1 file overwrite

2. **Audit ui/events-memories.php**
   - Determine if connector-specific
   - If not: Remove from package
   - Impact: -59 KB, -1 file overwrite

3. **Remove connector/openrouterjsoncached_verbose.php**
   - Provide as separate debug tool
   - Not needed for normal operation
   - Impact: -94 KB, -1 file overwrite

4. **Remove connector/OPENROUTERJSONCACHED_README.md**
   - Host documentation externally
   - Link from package INSTALLATION_INSTRUCTIONS.txt
   - Impact: -9 KB, -1 file overwrite

5. **Move Quality Instruction Text**
   - From prompts/dialogue_prompt.php
   - Into connector/openrouterjsoncached_helpers.php
   - Remove prompts/dialogue_prompt.php from package
   - Impact: -5 KB, -1 file overwrite

**Total Potential Savings: -237 KB, -5 file overwrites**
**New Package: 6 files, ~575 KB (down from 11 files, 812 KB)**

---

### 8.2 Long-Term Solutions (Require CHIM Core Changes)

1. **Request Processing Hook System**
   - Ask CHIM dev to add hooks in lib/data_functions.php
   - Hook before MINIMUM_SENTENCE_SIZE check
   - Hook before position check
   - Allow connectors to register custom buffering logic
   - **Impact:** Eliminate need to overwrite 182 KB file

2. **Dynamic Field Injection System**
   - Ask CHIM dev to add UI extension system
   - Allow connectors to register custom fields via JSON/config
   - Render fields dynamically in configuration UI
   - **Impact:** Eliminate need to overwrite 150 KB UI file

3. **Schema Plugin System**
   - Ask CHIM dev to support conf_schema_connectors.json
   - Merge connector schemas at runtime
   - Keep connector config separate from core config
   - **Impact:** Eliminate need to overwrite 87 KB schema file

4. **Metadata Extension Hooks**
   - Ask CHIM dev to add metadata extension system
   - Allow connectors to register custom metadata fields
   - Handle persistence automatically
   - **Impact:** Simplify lib/core/llm_connector.class.php changes

**Total Potential Impact:**
- Could reduce to **2 files overwritten** (connector core only)
- Size: ~200 KB (connector files only)
- Zero CHIM core file conflicts

---

### 8.3 Plugin Architecture Vision

**Ideal State:**
```
CHIM Core (untouched)
    ├── Provides hook system
    ├── Provides extension points
    └── Loads connector plugins

Connector Plugin Package
    ├── connector/openrouterjsoncached.php (CORE)
    ├── connector/openrouterjsoncached_helpers.php (HELPERS)
    ├── connector_config.json (SCHEMA)
    └── connector_hooks.php (CUSTOM LOGIC)

Plugin registers with CHIM:
    ├── Configuration fields → Dynamic UI injection
    ├── Processing logic → Hook registration
    ├── Metadata handlers → Extension system
    └── Documentation → External links
```

**Benefits:**
- Zero CHIM core file conflicts
- Easy updates (update plugin only)
- Multiple connector versions can coexist
- Clean separation of concerns
- Reduced package size

---

## Part 9: Conclusion

### Current State Analysis

**Files We Must Overwrite (Can't Avoid):**
1. connector/openrouterjsoncached.php - Core connector
2. connector/openrouterjsoncached_helpers.php - Required functions
3. lib/core/llm_connector.class.php - Metadata management
4. lib/data_functions.php - v1.3.3 streaming logic
5. ui/core/llm_connectors.php - Configuration UI
6. conf/conf_schema.json - Configuration schema

**Files We Can Remove (Action Required):**
1. connector/openrouterjsoncached_verbose.php - Provide separately
2. connector/OPENROUTERJSONCACHED_README.md - Host externally
3. ui/events-memories.php - Audit if truly needed
4. prompts/dialogue_prompt.php - Move quality text to helpers
5. lib/chat_helper_functions.php - Audit if truly needed

### Immediate Optimization Potential

**Before Optimization:**
- 11 files overwritten
- 812 KB total size
- 5 CHIM core files modified (270 KB lib + 209 KB UI + 87 KB config + 5 KB prompts)

**After Optimization (Immediate):**
- 6 files overwritten (remove 5 non-essential)
- ~575 KB total size (save 237 KB)
- 3 CHIM core files modified (270 KB lib + 150 KB UI + 87 KB config)

**After Optimization (With CHIM Core Hooks):**
- 2 files overwritten (connector core only)
- ~200 KB total size (save 612 KB)
- 0 CHIM core files modified

### Key Findings

1. **Most Problematic:** lib/data_functions.php (182 KB for 30 lines)
2. **Second Problematic:** ui/core/llm_connectors.php (150 KB for ~20 lines)
3. **Third Problematic:** conf/conf_schema.json (87 KB for connector fields)

4. **Easy Wins:** Remove verbose connector, README, possibly 2 other files
5. **Medium-Term:** Move quality text to avoid prompts file overwrite
6. **Long-Term:** Request CHIM core add hook/plugin system

### Recommendations Priority

**Priority 1 (Do Now):**
- Audit lib/chat_helper_functions.php necessity
- Audit ui/events-memories.php necessity
- Remove verbose connector from package
- Remove README from package

**Priority 2 (Next Version):**
- Move quality instruction text to helpers
- Remove prompts/dialogue_prompt.php from package
- Document external dependencies clearly

**Priority 3 (Future):**
- Propose hook system to CHIM developer
- Propose UI extension system to CHIM developer
- Propose schema merging to CHIM developer

---

**Document Version:** 1.0
**Last Updated:** 2025-11-16
**Next Review:** When preparing v1.4.0 or when CHIM core updates
