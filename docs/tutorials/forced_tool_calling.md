# Forced Tool Calling and Advanced Tool Calling Features

This document provides comprehensive documentation on forced tool calling and related advanced features in XGrammar, with code citations and implementation details.

## Table of Contents

1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Implementation Architecture](#implementation-architecture)
4. [Forced Tool Calling](#forced-tool-calling)
5. [Tool Choice Modes](#tool-choice-modes)
6. [Parallel Tool Calling Control](#parallel-tool-calling-control)
7. [Supported Tool Calling Formats](#supported-tool-calling-formats)
8. [Code Examples](#code-examples)
9. [Advanced Patterns](#advanced-patterns)
10. [API References](#api-references)

## Overview

XGrammar provides powerful structural tag APIs that enable precise control over LLM output formats, including advanced tool calling features. These features allow you to:

- **Force tool calls**: Require the LLM to call at least one or exactly one tool
- **Control parallel execution**: Allow or prevent multiple tool calls in a single response
- **Support diverse formats**: Handle tool calling formats from various LLM providers (Llama, Qwen, DeepSeek, Gemma, Phi-4, Hermes)
- **Dynamic constraints**: Mix free-form text with structured tool calls

The structural tag API is more flexible and dynamic than traditional OpenAI-style tool calling APIs, providing fine-grained control over output structure through JSON-based format specifications.

## Core Concepts

### Structural Tags

Structural tags provide a JSON-config-based way to precisely describe the output format of an LLM. As documented in [`docs/tutorials/structural_tag.md`](structural_tag.md), they can be used to implement various output patterns including tool calling.

**Key Components:**

1. **Format Types**: Define the structure of output chunks
   - `const_string`: Match exact strings
   - `json_schema`: Match JSON following a schema
   - `tag`: Match `begin content end` patterns
   - `triggered_tags`: Dispatch based on triggers
   - `tags_with_separator`: Multiple tags with separators
   - And more...

2. **Triggers**: Prefixes that activate specific structural patterns
3. **Tags**: Complete patterns with begin/content/end sections

### Python API Definition

The structural tag formats are defined in [`python/xgrammar/structural_tag.py`](../../python/xgrammar/structural_tag.py):

```python
class TriggeredTagsFormat(BaseModel):
    """A format that matches triggered tags."""
    
    type: Literal["triggered_tags"] = "triggered_tags"
    triggers: List[str]              # Trigger prefixes
    tags: List[TagFormat]            # Tag patterns
    at_least_one: bool = False       # Force at least one tag
    stop_after_first: bool = False   # Stop after first tag
```

**Source Reference**: Lines 129-178 in `python/xgrammar/structural_tag.py`

### C++ Implementation

The C++ implementation in [`cpp/structural_tag.h`](../../cpp/structural_tag.h) defines the core structure:

```cpp
struct TriggeredTagsFormat {
  static constexpr const char* type = "triggered_tags";
  std::vector<std::string> triggers;
  std::vector<TagFormat> tags;
  bool at_least_one = false;      // Force at least one tag
  bool stop_after_first = false;  // Control parallel calling
  
  // ... constructor and implementation
};
```

**Source Reference**: Lines 131-154 in `cpp/structural_tag.h`

## Implementation Architecture

### Grammar Construction Pipeline

The tool calling features are implemented through a multi-stage pipeline:

1. **Parsing** (`cpp/structural_tag.cc`): JSON structural tag → Internal representation
2. **Analysis** (`StructuralTagAnalyzer`): Detect patterns and constraints
3. **Conversion** (`StructuralTagGrammarConverter`): Generate EBNF grammar
4. **Compilation** (`GrammarCompiler`): Compile to executable grammar
5. **Matching** (`GrammarMatcher`): Apply during generation

**Source Reference**: `cpp/structural_tag.cc` contains the `StructuralTagParser` class (lines 28-61) that orchestrates this process.

### Format Variant System

The system uses C++ variants and Python discriminated unions for type-safe format handling:

**C++ Implementation** (`cpp/structural_tag.h`, lines 39-50):
```cpp
using Format = std::variant<
    ConstStringFormat,
    JSONSchemaFormat,
    QwenXmlParameterFormat,
    AnyTextFormat,
    GrammarFormat,
    RegexFormat,
    SequenceFormat,
    OrFormat,
    TagFormat,
    TriggeredTagsFormat,
    TagsWithSeparatorFormat>;
```

**Python Implementation** (`python/xgrammar/structural_tag.py`, lines 221-236):
```python
Format = Annotated[
    Union[
        AnyTextFormat,
        ConstStringFormat,
        JSONSchemaFormat,
        # ... all format types
        TriggeredTagsFormat,
        TagsWithSeparatorFormat,
    ],
    Field(discriminator="type"),
]
```

## Forced Tool Calling

### The `at_least_one` Parameter

The `at_least_one` parameter in `TriggeredTagsFormat` and `TagsWithSeparatorFormat` enables forced tool calling by requiring at least one tag to be generated.

**When `at_least_one=True`:**
- The LLM **must** generate at least one tool call
- The first tag is generated at the beginning of output
- Implements OpenAI's `tool_choice="required"` mode
- Prevents the model from responding without using tools

**When `at_least_one=False`:**
- Tool calls are optional
- The LLM can choose whether to call tools
- Implements OpenAI's `tool_choice="auto"` mode

### Implementation Details

**Python Definition** (`python/xgrammar/structural_tag.py`, line 174):
```python
at_least_one: bool = False
"""Whether at least one of the tags must be generated."""
```

**C++ Definition** (`cpp/structural_tag.h`, line 135):
```cpp
bool at_least_one = false;
```

### Example: Required Tool Calling

From [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 763-785:

```json
{
    "type": "structural_tag",
    "format": {
        "type": "triggered_tags",
        "triggers": ["<function="],
        "tags": [
            {
                "begin": "<function=func1>",
                "content": {"type": "json_schema", "json_schema": {...}},
                "end": "</function>"
            },
            {
                "begin": "<function=func2>",
                "content": {"type": "json_schema", "json_schema": {...}},
                "end": "</function>"
            }
        ],
        "at_least_one": true  // This forces tool calling!
    }
}
```

This configuration ensures the LLM **must** call at least one of the specified tools.

## Tool Choice Modes

XGrammar supports all three OpenAI tool choice modes through structural tag configurations.

### Mode 1: Auto (`tool_choice="auto"`)

Let the model decide whether to use tools:

```json
{
    "type": "triggered_tags",
    "triggers": ["<function="],
    "tags": [...],
    "at_least_one": false  // Optional tool calling
}
```

**Behavior**: The LLM can respond with or without tool calls.

### Mode 2: Required (`tool_choice="required"`)

Require at least one tool call:

```json
{
    "type": "triggered_tags",
    "triggers": ["<function="],
    "tags": [...],
    "at_least_one": true  // Must call at least one tool
}
```

**Behavior**: The LLM must call at least one tool from the provided set.

**Documentation Reference**: [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 754-785

### Mode 3: Forced Specific Function (`tool_choice={"type": "function", "function": {"name": "func1"}}`)

Force calling a specific function:

```json
{
    "type": "structural_tag",
    "format": {
        "type": "tag",
        "begin": "<function=func1>",
        "content": {"type": "json_schema", "json_schema": {...}},
        "end": "</function>"
    }
}
```

**Behavior**: The LLM must call exactly the specified function.

**Documentation Reference**: [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 787-799

### OpenAI API Compatibility

The structural tag can implement the OpenAI Tool Calling API with strict format constraints. From [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 740-752:

> In the OpenAI Tool Calling API, a set of tools is provided using JSON schema. There are also several features: tool choice (control at least one tool or exactly one tool is called), parallel tool calling (allow only one tool or multiple tools can be called in one round), etc.
>
> You can construct the structural tag according to the provided tools, and the LLM's specific tool calling format. The structural tag can be used in XGrammar's constrained decoding workflow to enable strict format constraints.

## Parallel Tool Calling Control

### The `stop_after_first` Parameter

The `stop_after_first` parameter controls whether multiple tool calls can be made in a single response, implementing OpenAI's `parallel_tool_calls` functionality.

**When `stop_after_first=True`:**
- Only one tool call is allowed per response
- Generation stops after the first tag completes
- Implements `parallel_tool_calls=false`
- Useful for sequential workflows

**When `stop_after_first=False`:**
- Multiple tool calls are allowed (default)
- The model can call several tools in one response
- Implements `parallel_tool_calls=true`
- Enables parallel execution patterns

### Implementation Details

**Python Definition** (`python/xgrammar/structural_tag.py`, lines 176-177):
```python
stop_after_first: bool = False
"""Whether to stop after the first tag is generated."""
```

**C++ Definition** (`cpp/structural_tag.h`, line 136):
```cpp
bool stop_after_first = false;
```

**Documentation Reference**: [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 241-243:
> `stop_after_first` will reach the end of the `triggered_tags` structure after the first tag is generated. If there are following tags, they will still be generated; otherwise, the generation will stop.

### Example: Disable Parallel Tool Calls

```json
{
    "type": "structural_tag",
    "format": {
        "type": "triggered_tags",
        "triggers": ["<function="],
        "tags": [
            {
                "begin": "<function=func1>",
                "content": {"type": "json_schema", "json_schema": {...}},
                "end": "</function>"
            },
            {
                "begin": "<function=func2>",
                "content": {"type": "json_schema", "json_schema": {...}},
                "end": "</function>"
            }
        ],
        "stop_after_first": true  // Only one tool call allowed!
    }
}
```

**Documentation Reference**: [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 808-832

### TagsWithSeparatorFormat

For formats like Phi-4 that use explicit separators between tool calls:

**Python Definition** (`python/xgrammar/structural_tag.py`, lines 180-216):
```python
class TagsWithSeparatorFormat(BaseModel):
    """Tags separated by a separator with parallel control."""
    
    type: Literal["tags_with_separator"] = "tags_with_separator"
    tags: List[TagFormat]
    separator: str                   # e.g., ", " or "\n"
    at_least_one: bool = False       # Force at least one tag
    stop_after_first: bool = False   # Control parallel calling
```

**C++ Definition** (`cpp/structural_tag.h`, lines 156-176):
```cpp
struct TagsWithSeparatorFormat {
  static constexpr const char* type = "tags_with_separator";
  std::vector<TagFormat> tags;
  std::string separator;
  bool at_least_one = false;
  bool stop_after_first = false;
  // ...
};
```

## Supported Tool Calling Formats

XGrammar's structural tags support tool calling formats from all major LLM providers. Each format is shown with implementation details from [`docs/tutorials/structural_tag.md`](structural_tag.md).

### 1. Llama JSON-based Tool Calling / Gemma

**Format**: `{"name": "function_name", "parameters": params}`

**Structural Tag** (lines 373-401):
```json
{
    "type": "triggered_tags",
    "triggers": ["{\"name\":"],
    "tags": [
        {
            "begin": "{\"name\": \"func1\", \"parameters\": ",
            "content": {"type": "json_schema", "json_schema": {...}},
            "end": "}"
        },
        {
            "begin": "{\"name\": \"func2\", \"parameters\": ",
            "content": {"type": "json_schema", "json_schema": {...}},
            "end": "}"
        }
    ]
}
```

### 2. Llama Custom Tool Calling

**Format**: `<function=function_name>params</function>`

**Structural Tag** (lines 403-431):
```json
{
    "type": "triggered_tags",
    "triggers": ["<function="],
    "tags": [
        {
            "begin": "<function=func1>",
            "content": {"type": "json_schema", "json_schema": {...}},
            "end": "</function>"
        },
        {
            "begin": "<function=func2>",
            "content": {"type": "json_schema", "json_schema": {...}},
            "end": "</function>"
        }
    ]
}
```

### 3. Qwen 2.5/3 and Hermes

**Format**:
```
<tool_call>
{"name": "get_current_temperature", "arguments": {"location": "San Francisco, CA, USA"}}
</tool_call>
```

**Structural Tag** (lines 433-463):
```json
{
    "type": "triggered_tags",
    "triggers": ["<tool_call>"],
    "tags": [
        {
            "begin": "<tool_call>\n{\"name\": \"func1\", \"arguments\": ",
            "content": {"type": "json_schema", "json_schema": {...}},
            "end": "}\n</tool_call>"
        },
        {
            "begin": "<tool_call>\n{\"name\": \"func2\", \"arguments\": ",
            "content": {"type": "json_schema", "json_schema": {...}},
            "end": "}\n</tool_call>"
        }
    ]
}
```

### 4. DeepSeek

**Format**:
````
<｜tool▁calls▁begin｜><｜tool▁call▁begin｜>function<｜tool▁sep｜>function_name_1
```jsonc
{params}
```<｜tool▁call▁end｜>
````

**Structural Tag** (lines 465-514):
```json
{
    "type": "triggered_tags",
    "triggers": ["<｜tool▁calls▁begin｜>"],
    "tags": [
        {
            "begin": "<｜tool▁calls▁begin｜>",
            "end": "<｜tool▁calls▁end｜>",
            "content": {
                "type": "tags_with_separator",
                "separator": "\n",
                "tags": [
                    {
                        "begin": "<｜tool▁call▁begin｜>function<｜tool▁sep｜>function_name_1\n```jsonc\n",
                        "content": {"type": "json_schema", "json_schema": {...}},
                        "end": "\n```<｜tool▁call▁end｜>"
                    },
                    {
                        "begin": "<｜tool▁call▁begin｜>function<｜tool▁sep｜>function_name_2\n```jsonc\n",
                        "content": {"type": "json_schema", "json_schema": {...}},
                        "end": "\n```<｜tool▁call▁end｜>"
                    }
                ]
            }
        }
    ],
    "stop_after_first": true
}
```

**Key Features**:
- Nested structure: `triggered_tags` containing `tags_with_separator`
- Multiple tool calls separated by newlines
- Special token delimiters
- `stop_after_first=true` to prevent generation after tool calls block

### 5. Phi-4-mini

**Format**: `<|tool_call|>[{"name": "function_name_1", "arguments": params}, ...]<|/tool_call|>`

**Structural Tag** (lines 516-557):
```json
{
    "type": "triggered_tags",
    "triggers": ["<|tool_call|>"],
    "tags": [
        {
            "begin": "<|tool_call|>[",
            "end": "]<|/tool_call|>",
            "content": {
                "type": "tags_with_separator",
                "separator": ", ",
                "tags": [
                    {
                        "begin": "{\"name\": \"function_name_1\", \"arguments\": ",
                        "content": {"type": "json_schema", "json_schema": {...}},
                        "end": "}"
                    },
                    {
                        "begin": "{\"name\": \"function_name_2\", \"arguments\": ",
                        "content": {"type": "json_schema", "json_schema": {...}},
                        "end": "}"
                    }
                ]
            }
        }
    ],
    "stop_after_first": true
}
```

**Key Features**:
- Tool calls organized in a JSON array
- Comma-space separator between calls
- Allows multiple parallel tool calls within the array

### 6. Qwen XML Parameter Format

For Qwen models that use XML-style parameters:

**Format**: `<parameter=name>value</parameter>`

**Python Definition** (`python/xgrammar/structural_tag.py`, lines 36-66):
```python
class QwenXMLParameterFormat(BaseModel):
    """A format that matches Qwen XML function calls."""
    
    type: Literal["qwen_xml_parameter"] = "qwen_xml_parameter"
    json_schema: Union[bool, Dict[str, Any]]
```

**Example** (lines 318-335 in `docs/tutorials/structural_tag.md`):
```json
{
    "type": "qwen_xml_parameter",
    "json_schema": {
        "type": "object",
        "properties": {"name": {"type": "string"}, "age": {"type": "integer"}},
        "required": ["name", "age"]
    }
}
```

**Accepts**:
```xml
<parameter=name>Bob</parameter><parameter=age>100</parameter>
<parameter=name>"Bob&lt;"</parameter><parameter=age>100</parameter>
```

**Implementation**: Uses `JSONFormat::kXML` mode in the JSON schema converter (`cpp/json_schema_converter.h`, lines 18-21, 44-46).

**Test Coverage**: Comprehensive tests in [`tests/python/test_function_calling_converter.py`](../../tests/python/test_function_calling_converter.py) validate various XML parameter scenarios.

## Code Examples

### Example 1: Basic Forced Tool Calling

**Scenario**: Force the model to call at least one weather-related function.

```python
import xgrammar as xgr
from xgrammar.structural_tag import StructuralTag, TriggeredTagsFormat, TagFormat, JSONSchemaFormat

# Define tool schemas
get_weather_schema = {
    "type": "object",
    "properties": {
        "location": {"type": "string"},
        "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["location"]
}

get_forecast_schema = {
    "type": "object",
    "properties": {
        "location": {"type": "string"},
        "days": {"type": "integer", "minimum": 1, "maximum": 7}
    },
    "required": ["location", "days"]
}

# Create structural tag with forced tool calling
structural_tag = StructuralTag(
    format=TriggeredTagsFormat(
        triggers=["<function="],
        tags=[
            TagFormat(
                begin="<function=get_weather>",
                content=JSONSchemaFormat(json_schema=get_weather_schema),
                end="</function>"
            ),
            TagFormat(
                begin="<function=get_forecast>",
                content=JSONSchemaFormat(json_schema=get_forecast_schema),
                end="</function>"
            )
        ],
        at_least_one=True,      # Force at least one tool call
        stop_after_first=False  # Allow multiple tool calls
    )
)

# Compile to grammar
grammar = xgr.Grammar.from_structural_tag(structural_tag)
```

**Reference**: Grammar creation API documented in [`python/xgrammar/grammar.py`](../../python/xgrammar/grammar.py), lines 285-350.

### Example 2: Single Tool Call Only

**Scenario**: Allow tool calls but only one at a time (no parallel execution).

```python
structural_tag = StructuralTag(
    format=TriggeredTagsFormat(
        triggers=["<function="],
        tags=[...],  # Same as above
        at_least_one=False,    # Optional tool calling
        stop_after_first=True  # Only one tool call allowed
    )
)
```

### Example 3: Force Specific Function

**Scenario**: Always call a specific function (e.g., search).

```python
from xgrammar.structural_tag import TagFormat

search_schema = {
    "type": "object",
    "properties": {
        "query": {"type": "string"},
        "max_results": {"type": "integer"}
    },
    "required": ["query"]
}

# Force this specific function
structural_tag = StructuralTag(
    format=TagFormat(
        begin="<function=search>",
        content=JSONSchemaFormat(json_schema=search_schema),
        end="</function>"
    )
)

grammar = xgr.Grammar.from_structural_tag(structural_tag)
```

**Reference**: This pattern is documented in [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 787-799.

### Example 4: Integration with LLM Engine

**Scenario**: Complete end-to-end usage with constrained decoding.

```python
import xgrammar as xgr
from transformers import AutoTokenizer, AutoConfig

# Setup tokenizer
model_id = "meta-llama/Llama-3.2-1B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)
config = AutoConfig.from_pretrained(model_id)

# Create tokenizer info and compiler
tokenizer_info = xgr.TokenizerInfo.from_huggingface(
    tokenizer, 
    vocab_size=config.vocab_size
)
compiler = xgr.GrammarCompiler(tokenizer_info, max_threads=8)

# Compile structural tag
compiled_grammar = compiler.compile_structural_tag(structural_tag)

# Create matcher for constrained generation
matcher = xgr.GrammarMatcher(compiled_grammar)

# During generation loop:
token_bitmask = xgr.allocate_token_bitmask(1, tokenizer_info.vocab_size)

# For each generation step:
can_reach_end = matcher.fill_next_token_bitmask(token_bitmask)
# Apply token_bitmask to logits before sampling
# ... sample next token ...
# matcher.accept_token(next_token)
```

**Reference**: This pattern follows the workflow documented in [`docs/tutorials/workflow_of_xgrammar.md`](workflow_of_xgrammar.md).

## Advanced Patterns

### Pattern 1: Forced Reasoning + Tool Calling

**Scenario**: Require reasoning before tool calls (common in chain-of-thought prompting).

From [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 604-649:

```python
from xgrammar.structural_tag import SequenceFormat, AnyTextFormat

# Force <think>...</think> followed by tool call
reasoning_and_tools = StructuralTag(
    format=SequenceFormat(
        elements=[
            # First: forced reasoning
            TagFormat(
                begin="<think>",
                content=AnyTextFormat(),
                end="</think>"
            ),
            # Then: forced tool call
            TriggeredTagsFormat(
                triggers=["<function="],
                tags=[
                    TagFormat(
                        begin="<function=func1>",
                        content=JSONSchemaFormat(json_schema={...}),
                        end="</function>"
                    ),
                    TagFormat(
                        begin="<function=func2>",
                        content=JSONSchemaFormat(json_schema={...}),
                        end="</function>"
                    )
                ],
                at_least_one=True,      # Must call a tool after thinking
                stop_after_first=True   # Only one tool call
            )
        ]
    )
)
```

**Key Features**:
- Uses `SequenceFormat` to chain multiple format requirements
- `AnyTextFormat()` allows free-form reasoning text
- Forces both reasoning and tool calling
- Implements Example 3 from the structural tag documentation

### Pattern 2: Optional Reasoning with Flexible Tool Calls

**Scenario**: Allow optional reasoning followed by optional multiple tool calls.

```python
reasoning_optional = StructuralTag(
    format=SequenceFormat(
        elements=[
            # Optional reasoning tag
            TagFormat(
                begin="<think>",
                content=AnyTextFormat(),
                end="</think>"
            ),
            # Optional multiple tool calls
            TriggeredTagsFormat(
                triggers=["<function="],
                tags=[...],
                at_least_one=False,     # Tool calls optional
                stop_after_first=False  # Multiple calls allowed
            )
        ]
    )
)
```

### Pattern 3: Multi-format Support (DeepSeek-style)

**Scenario**: Support complex nested structures with inner parallelism.

From [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 651-701:

```python
deepseek_pattern = StructuralTag(
    format=SequenceFormat(
        elements=[
            # Optional reasoning
            TagFormat(
                begin="<think>",
                content=AnyTextFormat(),
                end="</think>"
            ),
            # Complex tool calling block
            TriggeredTagsFormat(
                triggers=["<｜tool▁calls▁begin｜>"],
                tags=[
                    TagFormat(
                        begin="<｜tool▁calls▁begin｜>",
                        end="<｜tool▁calls▁end｜>",
                        content=TagsWithSeparatorFormat(
                            separator="\n",
                            tags=[...],  # Individual tool calls
                            at_least_one=True,      # Must have tools in block
                            stop_after_first=True   # Stop after block
                        )
                    )
                ],
                stop_after_first=True  # Stop after tool calls block
            )
        ]
    )
)
```

**Key Features**:
- Three-level nesting: `SequenceFormat` → `TriggeredTagsFormat` → `TagsWithSeparatorFormat`
- Inner `at_least_one=True` ensures at least one tool in the block
- Outer `stop_after_first=True` stops after tool block completes
- Multiple tools can be called within the block (inner parallel calling)

### Pattern 4: Conditional Tool Calling (OR Pattern)

**Scenario**: Choose between different response modes.

```python
from xgrammar.structural_tag import OrFormat

# Either respond with text OR make a tool call
conditional_response = StructuralTag(
    format=OrFormat(
        elements=[
            # Option 1: Simple text response
            TagFormat(
                begin="<response>",
                content=AnyTextFormat(),
                end="</response>"
            ),
            # Option 2: Tool call
            TriggeredTagsFormat(
                triggers=["<function="],
                tags=[...],
                at_least_one=True,
                stop_after_first=True
            )
        ]
    )
)
```

**Reference**: `OrFormat` is documented in [`python/xgrammar/structural_tag.py`](../../python/xgrammar/structural_tag.py), lines 107-114.

### Pattern 5: Qwen Hybrid Mode (Non-thinking)

**Scenario**: Force non-thinking mode with empty think tags.

From [`docs/tutorials/structural_tag.md`](structural_tag.md), lines 703-738:

```python
from xgrammar.structural_tag import ConstStringFormat

qwen_non_thinking = StructuralTag(
    format=SequenceFormat(
        elements=[
            # Force empty thinking tag
            ConstStringFormat(value="<think></think>"),
            # Then allow optional tool calls
            TriggeredTagsFormat(
                triggers=["<tool_call>"],
                tags=[...],
                at_least_one=False  # Optional tools
            )
        ]
    )
)
```

**Key Features**:
- Uses `ConstStringFormat` to force exact string match
- Implements Qwen's non-thinking mode constraint
- Combines fixed and flexible elements

## API References

### Grammar Creation

**Method**: `Grammar.from_structural_tag()`

**Location**: [`python/xgrammar/grammar.py`](../../python/xgrammar/grammar.py), lines 285-350

**Signatures**:
```python
# New API (recommended)
@staticmethod
def from_structural_tag(
    structural_tag: Union[StructuralTag, str, Dict[str, Any]]
) -> "Grammar":
    """Create a grammar from a structural tag."""

# Legacy API (deprecated)
@staticmethod
def from_structural_tag(
    tags: List[StructuralTagItem], 
    triggers: List[str]
) -> "Grammar":
    """Deprecated: Use StructuralTag class instead."""
```

**Parameters**:
- `structural_tag`: StructuralTag object, JSON string, or dictionary

**Returns**: `Grammar` object for use in constrained generation

**Raises**:
- `InvalidJSONError`: Invalid JSON format
- `InvalidStructuralTagError`: Invalid structural tag specification
- `TypeError`: Invalid arguments

### StructuralTag Class

**Location**: [`python/xgrammar/structural_tag.py`](../../python/xgrammar/structural_tag.py), lines 273-325

**Methods**:

```python
@staticmethod
def from_legacy_structural_tag(
    tags: List[StructuralTagItem], 
    triggers: List[str]
) -> "StructuralTag":
    """Convert legacy format to new StructuralTag."""

@staticmethod
def from_json(json_str: Union[str, Dict[str, Any]]) -> "StructuralTag":
    """Parse JSON to StructuralTag."""
```

### Core Format Classes

All format classes are defined in [`python/xgrammar/structural_tag.py`](../../python/xgrammar/structural_tag.py):

1. **ConstStringFormat** (lines 18-24): Match exact strings
2. **JSONSchemaFormat** (lines 27-33): Match JSON schema
3. **QwenXMLParameterFormat** (lines 36-66): Qwen XML format
4. **AnyTextFormat** (lines 68-72): Match any text
5. **GrammarFormat** (lines 75-82): Custom EBNF grammar
6. **RegexFormat** (lines 85-92): Regular expression pattern
7. **SequenceFormat** (lines 98-104): Sequential elements
8. **OrFormat** (lines 107-114): Alternative elements
9. **TagFormat** (lines 116-127): Begin-content-end pattern
10. **TriggeredTagsFormat** (lines 129-178): Trigger-based dispatching
11. **TagsWithSeparatorFormat** (lines 180-216): Separated tags

### C++ API

**Header**: [`cpp/structural_tag.h`](../../cpp/structural_tag.h)

**Main Function**:
```cpp
Result<Grammar, StructuralTagError> StructuralTagToGrammar(
    const std::string& structural_tag_json
);
```

**Location**: Line 194 in `cpp/structural_tag.h`

**Purpose**: Convert structural tag JSON to executable grammar

### Testing Utilities

**Location**: [`python/xgrammar/testing.py`](../../python/xgrammar/testing.py)

**Key Functions**:
- `_is_grammar_accept_string()`: Test if grammar accepts a string
- `_qwen_xml_tool_calling_to_ebnf()`: Convert Qwen XML to EBNF

**Test Examples**: [`tests/python/test_function_calling_converter.py`](../../tests/python/test_function_calling_converter.py) contains comprehensive test cases for tool calling formats.

## Best Practices

### 1. Choose Appropriate Control Parameters

- **Use `at_least_one=True`** when:
  - Tool calling is mandatory for the task
  - You want to prevent non-tool responses
  - Implementing `tool_choice="required"` behavior

- **Use `stop_after_first=True`** when:
  - Sequential processing is required
  - Multiple parallel calls would cause issues
  - Implementing `parallel_tool_calls=false` behavior
  - You want to maintain conversation flow after one tool call

### 2. Format-Specific Considerations

- **Llama models**: Support both JSON and custom `<function=>` formats
- **Qwen models**: Use XML parameters for Qwen 3, tool_call tags for Qwen 2.5
- **DeepSeek**: Requires nested structure with separator format inside triggered tags
- **Phi-4**: Uses JSON array format with explicit separators

### 3. Testing and Validation

Always test your structural tags with the testing utilities:

```python
from xgrammar.testing import _is_grammar_accept_string

# Test if your grammar accepts expected strings
assert _is_grammar_accept_string(grammar, expected_output)

# Test rejection of invalid strings
assert not _is_grammar_accept_string(grammar, invalid_output)
```

**Reference**: Testing examples in [`tests/python/test_structural_tag_converter.py`](../../tests/python/test_structural_tag_converter.py), lines 76-91.

### 4. Performance Optimization

- Enable grammar compiler caching for repeated patterns
- Use `max_threads` parameter in `GrammarCompiler` for parallel compilation
- Profile with the `Profiler` class from test files

**Reference**: [`tests/python/test_structural_tag_converter.py`](../../tests/python/test_structural_tag_converter.py), lines 13-49.

### 5. Error Handling

Handle structural tag errors appropriately:

```python
from xgrammar.exception import InvalidStructuralTagError, InvalidJSONError

try:
    grammar = xgr.Grammar.from_structural_tag(structural_tag)
except InvalidJSONError as e:
    # Handle JSON parsing errors
    print(f"Invalid JSON: {e}")
except InvalidStructuralTagError as e:
    # Handle structural tag validation errors
    print(f"Invalid structural tag: {e}")
```

## Conclusion

XGrammar's structural tag API provides powerful and flexible control over tool calling behavior through:

1. **Forced tool calling** via `at_least_one` parameter
2. **Parallel execution control** via `stop_after_first` parameter
3. **Multiple format support** for all major LLM providers
4. **Composable patterns** using sequence, or, and nested structures
5. **Type-safe implementation** in both Python and C++

These features enable implementation of the complete OpenAI Tool Calling API and support advanced patterns like reasoning-before-tool-calling, conditional responses, and complex multi-step workflows.

For more information:
- **Structural Tag Tutorial**: [`docs/tutorials/structural_tag.md`](structural_tag.md)
- **Advanced Topics**: [`docs/tutorials/advanced_structural_tag.md`](advanced_structural_tag.md)
- **API Reference**: [`docs/api/python/structural_tag.rst`](../api/python/structural_tag.rst)
- **Python Implementation**: [`python/xgrammar/structural_tag.py`](../../python/xgrammar/structural_tag.py)
- **C++ Implementation**: [`cpp/structural_tag.h`](../../cpp/structural_tag.h) and [`cpp/structural_tag.cc`](../../cpp/structural_tag.cc)

## Appendix: Complete Working Example

Here's a complete, runnable example demonstrating forced tool calling:

```python
import xgrammar as xgr
from xgrammar.structural_tag import (
    StructuralTag, 
    TriggeredTagsFormat, 
    TagFormat, 
    JSONSchemaFormat
)
from transformers import AutoTokenizer, AutoConfig

# Define tool schemas
weather_tools = {
    "get_weather": {
        "type": "object",
        "properties": {
            "location": {"type": "string"},
            "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
        },
        "required": ["location"]
    },
    "get_forecast": {
        "type": "object",
        "properties": {
            "location": {"type": "string"},
            "days": {"type": "integer", "minimum": 1, "maximum": 7}
        },
        "required": ["location", "days"]
    }
}

# Create structural tag with forced tool calling
structural_tag = StructuralTag(
    format=TriggeredTagsFormat(
        triggers=["<function="],
        tags=[
            TagFormat(
                begin=f"<function={name}>",
                content=JSONSchemaFormat(json_schema=schema),
                end="</function>"
            )
            for name, schema in weather_tools.items()
        ],
        at_least_one=True,      # Force at least one tool call
        stop_after_first=False  # Allow multiple tool calls
    )
)

# Setup for generation
model_id = "meta-llama/Llama-3.2-1B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)
config = AutoConfig.from_pretrained(model_id)

tokenizer_info = xgr.TokenizerInfo.from_huggingface(
    tokenizer, 
    vocab_size=config.vocab_size
)

# Compile grammar
compiler = xgr.GrammarCompiler(tokenizer_info, max_threads=8)
compiled_grammar = compiler.compile_structural_tag(structural_tag)

# Create matcher
matcher = xgr.GrammarMatcher(compiled_grammar)

# Test acceptance
from xgrammar.testing import _is_grammar_accept_string

grammar = xgr.Grammar.from_structural_tag(structural_tag)

# These should be accepted
valid_outputs = [
    '<function=get_weather>{"location": "Tokyo", "unit": "celsius"}</function>',
    '<function=get_forecast>{"location": "Paris", "days": 5}</function>',
    '<function=get_weather>{"location": "NYC"}</function><function=get_forecast>{"location": "LA", "days": 3}</function>'
]

for output in valid_outputs:
    assert _is_grammar_accept_string(grammar, output), f"Should accept: {output}"

# This should be rejected (no tool call)
invalid_output = "Just a regular text response"
assert not _is_grammar_accept_string(grammar, invalid_output), "Should reject non-tool output"

print("✓ All tests passed! Forced tool calling is working correctly.")
```

This example demonstrates:
- Tool schema definition
- Structural tag creation with forced calling
- Grammar compilation and matching
- Validation of correct behavior

The complete pattern can be adapted for any LLM and tool calling format supported by XGrammar.
