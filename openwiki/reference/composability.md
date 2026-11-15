---
type: API Reference
title: Runnable Composition Methods
description: Complete reference for composing Runnables using pipe, chain, branch, and parallel operators, including RunnableSequence, RunnableParallel, and related composition patterns.
tags: [runnable, composition, lcel, pipe, chain, branch, parallel, sequence, API-reference]
verified:
  - by: openwiki/0.5.0
    at: 2026-11-15T08:15:26.682Z
sources:
  - id: openwiki-source-a1981e868973f6fd7f71e12e
    resource: repo://libs/core/langchain_core/runnables/base.py
  - id: openwiki-source-48e94bbe49ab4f33ba87e9cb
    resource: repo://libs/core/langchain_core/runnables/branch.py
  - id: openwiki-source-f9f4c1dc4f9cdf80d824ce15
    resource: repo://libs/core/langchain_core/runnables/fallbacks.py
  - id: openwiki-source-7a970e5b49bde2fe4f9be3db
    resource: repo://libs/core/langchain_core/runnables/passthrough.py
generated: { by: "openwiki/0.5.0", at: "2026-11-15T08:15:26.682Z" }
---

## Overview

Composability is a core strength of the Runnable interface. LangChain provides multiple methods to combine Runnables into complex workflows: **sequential piping** via the `|` operator or `.pipe()`, **parallel execution** via dict literals and `RunnableParallel`, **conditional branching** via `RunnableBranch`, and **fallback handling** via `.with_fallbacks()`. All composition methods automatically inherit synchronous, asynchronous, batch, and streaming support from the components they combine.

## Sequential Composition: Pipe and Chain

### Pipe Operator `|`

The `|` operator is the primary syntax for composing Runnables sequentially. It chains the output of one Runnable as the input to the next.

**Basic Syntax:**

```python
result = runnable_a | runnable_b | runnable_c
```

The pipe operator creates a `RunnableSequence`, where `runnable_a` executes first, passes its output to `runnable_b`, which passes its output to `runnable_c`.

**Flexible Input Types:**

The `|` operator accepts Runnables and callable-like objects via implicit coercion:

- **Runnable objects**: Pass through directly
- **Dictionaries**: Coerced to `RunnableParallel` (see [Parallel Composition](#parallel-composition-dict-literals-and-runnableparallel) below)
- **Callables** (functions, lambdas): Wrapped in `RunnableLambda`
- **Generators**: Wrapped in `RunnableGenerator`

**Example:**

```python
from langchain_core.runnables import RunnableLambda

def add_one(x: int) -> int:
    return x + 1

def multiply_by_two(x: int) -> int:
    return x * 2

# Using pipe operator with callables (automatically wrapped)
chain = RunnableLambda(add_one) | multiply_by_two
result = chain.invoke(5)  # (5 + 1) * 2 = 12
```

**Overloads and Type Safety:**

The `__or__` method has multiple overloads to support:
- Runnable-to-Runnable composition: `Runnable[A, B] | Runnable[B, C] → Runnable[A, C]`
- Callable composition: `Runnable[A, B] | (B → C) → Runnable[A, C]`
- Stream transform composition: `Runnable[A, B] | (Iterator[B] → Iterator[C]) → Runnable[A, C]`
- Dynamic Runnable selection: `Runnable[A, B] | (B → Runnable[B, C]) → Runnable[A, C]`
- Parallel mapping: `Runnable[A, B] | {key: Runnable[B, C]} → Runnable[A, dict[str, C]]`

### `.pipe()` Method

The `.pipe()` method provides an alternative to the `|` operator. It accepts multiple Runnables and returns a new `RunnableSequence`:

```python
def pipe(
    self,
    *others: Runnable[Any, Other] | Callable[[Any], Other],
    name: str | None = None,
) -> RunnableSerializable[Input, Other]:
    """Compose this Runnable with others to create a RunnableSequence."""
```

**Parameters:**
- `*others`: Variable-length argument list of Runnables or callables
- `name`: Optional name for the resulting sequence

**Equivalent to:** `RunnableSequence(self, *others, name=name)` or `self | others[0] | ...`

**Example:**

```python
from langchain_core.runnables import RunnableLambda

chain = RunnableLambda(add_one).pipe(
    RunnableLambda(multiply_by_two),
    RunnableLambda(lambda x: x + 10)
)
# Equivalent to: RunnableLambda(add_one) | multiply_by_two | (lambda x: x + 10)
result = chain.invoke(5)  # ((5 + 1) * 2) + 10 = 22
```

### `@chain` Decorator

The `@chain` decorator provides a lightweight way to create a Runnable from a function. It automatically wraps the function in a `RunnableLambda` and sets the Runnable's name to the function name.

```python
from langchain_core.runnables import chain

@chain
def my_function(input_data: dict) -> dict:
    """Process input data."""
    # Function body can call other Runnables
    prompt = PromptTemplate.from_template("Hello, {name}!")
    model = ChatOpenAI()
    return {"result": model.invoke(prompt.format(**input_data))}
```

**Key Points:**
- Automatically traces dependencies on Runnables called within the function
- Supports sync, async, streaming, and batch execution modes
- Preserves the function's docstring and name for observability

## `RunnableSequence`: Sequential Execution

**Location:** `repo://libs/core/langchain_core/runnables/base.py#L3094-L3880`

`RunnableSequence` is the fundamental composition primitive for sequential execution. It chains multiple Runnables so the output of one becomes the input of the next.

### Structure and Initialization

```python
class RunnableSequence(RunnableSerializable[Input, Output]):
    first: Runnable[Input, Any]
    """The first Runnable in the sequence."""
    middle: list[Runnable[Any, Any]] = []
    """The middle Runnables in the sequence."""
    last: Runnable[Any, Output]
    """The last Runnable in the sequence."""
    
    def __init__(
        self,
        *steps: RunnableLike[Any, Any],
        name: str | None = None,
        first: Runnable[Any, Any] | None = None,
        middle: list[Runnable[Any, Any]] | None = None,
        last: Runnable[Any, Any] | None = None,
    ) -> None:
        """Create a RunnableSequence.
        
        Args:
            steps: The steps to include in the sequence (must have ≥2 steps).
            name: Optional name for the sequence.
            first: The first Runnable (alternative constructor).
            middle: The middle Runnables (alternative constructor).
            last: The last Runnable (alternative constructor).
        
        Raises:
            ValueError: If the sequence has less than 2 steps.
        """
```

**Key Behaviors:**

1. **Flattening**: When a `RunnableSequence` is passed as a step, it is automatically flattened into the sequence. This prevents unnecessary nesting.

   ```python
   seq1 = a | b
   seq2 = c | d
   seq3 = seq1 | seq2  # Flattened to: a | b | c | d
   ```

2. **Minimum Steps**: A `RunnableSequence` must have at least 2 steps. Single-step sequences are invalid.

3. **Type Properties**: The input type is determined by the first Runnable; the output type by the last.

### Execution Model

`RunnableSequence` executes steps in order. Each step receives the output of the previous step as input:

- **`invoke(input)`**: Executes each step synchronously in sequence
- **`ainvoke(input)`**: Executes each step asynchronously in sequence
- **`batch(inputs)`**: Runs `batch` on each step in order (more efficient than naive `invoke` calls)
- **`stream(input)`**: Preserves streaming properties of components if all implement `transform`

### Streaming Behavior

`RunnableSequence` automatically enables end-to-end streaming when components support it:

- If all steps implement the `transform` method, streaming flows through the entire chain without blocking
- If any step does not implement `transform`, streaming begins only after that step completes (blocking point)
- **Note:** `RunnableLambda` does not support `transform` by default; place them carefully to avoid blocking streams

**Example with Streaming:**

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

# All components support streaming
prompt = PromptTemplate.from_template("Tell me a joke about {topic}")
model = ChatOpenAI()
parser = StrOutputParser()

chain = prompt | model | parser

# Stream outputs token-by-token
for chunk in chain.stream({"topic": "dogs"}):
    print(chunk, end="", flush=True)
```

## Parallel Composition: Dict Literals and `RunnableParallel`

**Location:** `repo://libs/core/langchain_core/runnables/base.py#L3883-L4416`

### Dict Literal Syntax

The simplest way to run multiple Runnables in parallel is to embed a dictionary in a pipe expression. Each key-value pair becomes a parallel branch:

```python
from langchain_core.runnables import RunnableLambda

def add_one(x: int) -> int:
    return x + 1

def multiply_two(x: int) -> int:
    return x * 2

# Dict literal automatically becomes RunnableParallel
chain = RunnableLambda(add_one) | {
    "incremented": RunnableLambda(lambda x: x + 10),
    "doubled": RunnableLambda(multiply_two),
}

result = chain.invoke(5)
# Output: {"incremented": 16, "doubled": 12}
#         ((5+1)+10=16,  (5+1)*2=12)
```

### `RunnableParallel` Class

For more control, explicitly instantiate `RunnableParallel`:

```python
class RunnableParallel(RunnableSerializable[Input, dict[str, Any]]):
    """Runs a mapping of Runnables in parallel, returning a dict of outputs."""
    
    steps__: Mapping[str, Runnable[Input, Any]]
    """Dict mapping string keys to Runnables."""
    
    def __init__(
        self,
        steps__: Mapping[str, Runnable[Input, Any] | Callable[[Input], Any]] | None = None,
        **kwargs: Runnable[Input, Any] | Callable[[Input], Any],
    ) -> None:
        """Create a RunnableParallel.
        
        Args:
            steps__: Dict of steps.
            **kwargs: Keyword arguments mapping names to Runnables/callables.
        """
```

**Construction Methods:**

```python
from langchain_core.runnables import RunnableParallel, RunnableLambda

# Method 1: Dict argument
parallel = RunnableParallel(
    {"add_one": lambda x: x + 1, "multiply_two": lambda x: x * 2}
)

# Method 2: Keyword arguments
parallel = RunnableParallel(
    add_one=lambda x: x + 1,
    multiply_two=lambda x: x * 2,
)

# Method 3: Mixed
parallel = RunnableParallel(
    {"add_one": lambda x: x + 1},
    multiply_two=lambda x: x * 2,
)
```

### Input Type Handling

All branches receive the **same input**. If the input is a dictionary and all branches expect dict inputs, `RunnableParallel` automatically infers the merged input schema:

```python
from langchain_core.runnables import RunnableParallel

# If branches expect different keys, schema is merged
parallel = RunnableParallel(
    name_upper=lambda x: x["name"].upper(),
    name_lower=lambda x: x["name"].lower(),
    has_email=lambda x: "email" in x,
)

result = parallel.invoke({"name": "Alice", "email": "alice@example.com"})
# Output: {"name_upper": "ALICE", "name_lower": "alice", "has_email": True}
```

### Execution Guarantees

- **Concurrency**: All branches execute concurrently (via `asyncio.gather` for async, thread pool for sync)
- **Ordering**: Output dict keys are in consistent order; values correspond to input order
- **Error Handling**: If `return_exceptions=False` (default), first exception stops execution

### Real-World Example: Parallel LLM Chains

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.runnables import RunnableParallel

model = ChatOpenAI()

# Create two independent chains
joke_chain = (
    ChatPromptTemplate.from_template("Tell me a joke about {topic}") | model
)
poem_chain = (
    ChatPromptTemplate.from_template("Write a 2-line poem about {topic}") | model
)

# Run both in parallel
runnable = RunnableParallel(joke=joke_chain, poem=poem_chain)

result = runnable.invoke({"topic": "bears"})
# Output: {"joke": AIMessage(...), "poem": AIMessage(...)}
```

## Branching: Conditional Execution

**Location:** `repo://libs/core/langchain_core/runnables/branch.py#L43-L250`

`RunnableBranch` selects which branch to execute based on a condition. It evaluates conditions in order and runs the first matching branch; if no condition matches, runs a default branch.

### Structure and Initialization

```python
class RunnableBranch(RunnableSerializable[Input, Output]):
    """Runnable that selects a branch based on condition predicates."""
    
    branches: Sequence[tuple[
        Runnable[Input, bool],
        Runnable[Input, Output]
    ]]
    """List of (condition, branch) pairs."""
    default: Runnable[Input, Output]
    """Default branch if no condition matches."""
    
    def __init__(
        self,
        *branches: tuple[
            Runnable[Input, bool] | Callable[[Input], bool],
            RunnableLike[Input, Output],
        ] | RunnableLike[Input, Output],
    ) -> None:
        """Create a RunnableBranch.
        
        Args:
            *branches: Tuples of (condition, branch) plus a final default branch.
        
        Raises:
            ValueError: If fewer than 2 branches provided.
        """
```

### Usage Pattern

The last argument to `RunnableBranch` is the default branch; earlier arguments are `(condition, branch)` tuples:

```python
from langchain_core.runnables import RunnableBranch, RunnableLambda

# Define branches
is_string = lambda x: isinstance(x, str)
to_uppercase = RunnableLambda(lambda x: x.upper())

is_number = lambda x: isinstance(x, int)
add_ten = RunnableLambda(lambda x: x + 10)

# Default
unknown = RunnableLambda(lambda x: "Unknown type")

# Create branch
branch = RunnableBranch(
    (is_string, to_uppercase),
    (is_number, add_ten),
    unknown,  # Default branch
)

branch.invoke("hello")   # "HELLO"
branch.invoke(5)         # 15
branch.invoke(3.14)      # "Unknown type"
```

### Condition Types

Conditions can be:

- **Callables returning bool**: `lambda x: isinstance(x, str)`
- **Runnables returning bool**: Any `Runnable[Input, bool]`
- **Async callables**: `async def my_condition(x) -> bool:`

The first condition that evaluates to `True` is selected.

### Integration with Chains

`RunnableBranch` is powerful when combined with sequences and parsers:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch, RunnableLambda

model = ChatOpenAI()

# Route based on intent
is_greeting = lambda x: x.get("intent") == "greet"
is_question = lambda x: x.get("intent") == "question"

greeting_chain = ChatPromptTemplate.from_template(
    "Respond warmly to: {input}"
) | model | StrOutputParser()

question_chain = ChatPromptTemplate.from_template(
    "Answer this question: {input}"
) | model | StrOutputParser()

default_chain = RunnableLambda(lambda x: "I don't understand")

router = RunnableBranch(
    (is_greeting, greeting_chain),
    (is_question, question_chain),
    default_chain,
)

result = router.invoke({"intent": "greet", "input": "Hello!"})
```

## Fallback Handling with `.with_fallbacks()`

**Location:** `repo://libs/core/langchain_core/runnables/fallbacks.py#L37-L350`

Fallbacks enable graceful degradation when a Runnable fails. The `.with_fallbacks()` method wraps a Runnable with a list of fallback Runnables to try in sequence.

### Method Signature

```python
def with_fallbacks(
    self,
    fallbacks: Sequence[Runnable[Input, Output]],
    *,
    exceptions_to_handle: tuple[type[BaseException], ...] = (Exception,),
    exception_key: str | None = None,
) -> RunnableWithFallbacks[Input, Output]:
    """Wrap this Runnable with fallbacks.
    
    Args:
        fallbacks: Sequence of fallbacks to try in order.
        exceptions_to_handle: Exception types that trigger fallback.
        exception_key: If set, pass exception to fallbacks under this key.
    
    Returns:
        RunnableWithFallbacks instance.
    """
```

### Basic Usage

```python
from langchain_anthropic import ChatAnthropic
from langchain_openai import ChatOpenAI

# Use Claude, but fallback to GPT if Claude fails
model = ChatAnthropic(model="claude-sonnet-5").with_fallbacks(
    [ChatOpenAI(model="gpt-4-mini")]
)

result = model.invoke("Hello!")  # Tries Claude first, then GPT if needed
```

### Parameters

- **`fallbacks`**: List of Runnables with same input/output types as the primary
- **`exceptions_to_handle`**: Tuple of exception types that trigger fallback (default: all `Exception` subclasses)
- **`exception_key`**: If specified, pass the caught exception to fallbacks as `input[exception_key]` (requires dict input)

### Chain-Level Fallbacks

Fallbacks work at any level of composition:

```python
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda

prompt = PromptTemplate.from_template("Tell me a joke about {topic}")
primary_model = ChatAnthropic(model="claude-sonnet-5")
fallback_model = ChatOpenAI(model="gpt-4-mini")

def hardcoded_joke(inputs):
    return f"Why did the {inputs['topic']} cross the road? To get to the other side!"

chain = (
    prompt
    | primary_model
    | StrOutputParser()
).with_fallbacks(
    [
        prompt | fallback_model | StrOutputParser(),
        RunnableLambda(hardcoded_joke),
    ]
)

# Tries primary model, then fallback model, then hardcoded joke
result = chain.invoke({"topic": "chicken"})
```

### Fallback with Exception Context

When `exception_key` is set, exceptions are passed to fallbacks:

```python
def handle_with_error(input_with_error: dict) -> str:
    error = input_with_error.get("error")
    return f"Error occurred: {error}. Using fallback response."

model = ChatOpenAI().with_fallbacks(
    [RunnableLambda(handle_with_error)],
    exception_key="error",
)

result = model.invoke("prompt")
# If ChatOpenAI fails, fallback receives {"error": <exception>}
```

## Utility Composition Helpers

### `.pick()`: Extract Keys from Dict Output

The `.pick()` method selects specific keys from a dict output:

```python
from langchain_core.runnables import RunnableParallel, RunnableLambda
import json

as_str = RunnableLambda(str)
as_json = RunnableLambda(json.loads)

chain = RunnableParallel(str=as_str, json=as_json)
result = chain.invoke("[1, 2, 3]")
# Output: {"str": "[1, 2, 3]", "json": [1, 2, 3]}

# Pick single key
json_only = chain.pick("json")
json_only.invoke("[1, 2, 3]")  # [1, 2, 3]

# Pick multiple keys
subset = chain.pick(["str", "json"])
subset.invoke("[1, 2, 3]")  # {"str": "[1, 2, 3]", "json": [1, 2, 3]}
```

### `RunnablePassthrough`: Pass Input Unchanged

`RunnablePassthrough` returns its input unchanged, useful for preserving values in parallel branches:

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough, RunnableLambda

chain = RunnableParallel(
    original=RunnablePassthrough(),  # Returns input as-is
    modified=lambda x: x + 1,
)

result = chain.invoke(1)
# Output: {"original": 1, "modified": 2}
```

### `RunnablePassthrough.assign()`: Add Keys to Dict

The `.assign()` method merges computed values into a dict input:

```python
from langchain_core.runnables import RunnablePassthrough

chain = {
    "llm1": lambda x: "output1",
    "llm2": lambda x: "output2",
} | RunnablePassthrough.assign(
    total_chars=lambda inputs: len(inputs["llm1"] + inputs["llm2"]),
)

result = chain.invoke("dummy")
# Output: {"llm1": "output1", "llm2": "output2", "total_chars": 16}
```

## Coercion and Type Conversion

### `coerce_to_runnable()` Function

**Location:** `repo://libs/core/langchain_core/runnables/base.py#L6627-L6652`

The `coerce_to_runnable()` function automatically converts callable-like objects into Runnables:

```python
def coerce_to_runnable(thing: RunnableLike[Input, Output]) -> Runnable[Input, Any]:
    """Convert a Runnable-like object into a Runnable.
    
    Args:
        thing: A Runnable, callable, dict, generator, or async generator.
    
    Returns:
        A Runnable.
    
    Raises:
        TypeError: If the object is not Runnable-like.
    """
```

**Coercion Rules:**

| Input Type | Output |
|---|---|
| `Runnable[A, B]` | Returned unchanged |
| `Callable[[A], B]` | Wrapped in `RunnableLambda` |
| `Callable[[A], Awaitable[B]]` | Wrapped in `RunnableLambda` |
| `Callable[[Iterator[A]], Iterator[B]]` | Wrapped in `RunnableLambda` |
| `Callable[[AsyncIterator[A]], AsyncIterator[B]]` | Wrapped in `RunnableLambda` |
| Generator function | Wrapped in `RunnableGenerator` |
| Async generator function | Wrapped in `RunnableGenerator` |
| `dict[str, Runnable\|Callable]` | Converted to `RunnableParallel` |
| Unsupported type | `TypeError` raised |

This coercion enables flexible mixing of Runnables and callables in composition expressions without explicit wrapping.

## Composition Examples

### Example 1: Simple Pipeline

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

# Sequential composition with pipe operator
chain = (
    ChatPromptTemplate.from_template("Tell me a {length} joke about {topic}")
    | ChatOpenAI(model="gpt-4-mini")
    | StrOutputParser()
)

result = chain.invoke({"length": "short", "topic": "programming"})
```

### Example 2: Parallel Chains

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.runnables import RunnableParallel

model = ChatOpenAI()

# Create two independent chains
summary_chain = (
    ChatPromptTemplate.from_template("Summarize: {text}")
    | model
)
sentiment_chain = (
    ChatPromptTemplate.from_template("Analyze sentiment: {text}")
    | model
)

# Run both in parallel
analysis = RunnableParallel(
    summary=summary_chain,
    sentiment=sentiment_chain,
)

result = analysis.invoke({"text": "This product is amazing!"})
```

### Example 3: Conditional Routing

```python
from langchain_core.runnables import RunnableBranch, RunnableLambda
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI()

# Route based on input type
string_chain = (
    ChatPromptTemplate.from_template("Process string: {input}")
    | model
)
number_chain = (
    ChatPromptTemplate.from_template("Process number: {input}")
    | model
)
default = RunnableLambda(lambda x: "Unknown type")

router = RunnableBranch(
    (lambda x: isinstance(x, str), string_chain),
    (lambda x: isinstance(x, (int, float)), number_chain),
    default,
)

result = router.invoke("hello")
```

### Example 4: Complex Composition with All Patterns

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
import json

model = ChatOpenAI()

# Step 1: Parallel preprocessing
preprocess = {
    "original": RunnablePassthrough(),
    "uppercase": lambda x: x.upper(),
    "word_count": lambda x: len(x.split()),
}

# Step 2: Sequential processing
process = ChatPromptTemplate.from_template(
    "Process: original={original}, uppercase={uppercase}, words={word_count}"
) | model | StrOutputParser()

# Step 3: Conditional branching
is_long = lambda x: len(x) > 100
short_branch = RunnableLambda(lambda x: f"Short: {x[:50]}...")
long_branch = RunnableLambda(lambda x: f"Long: {x}")

branch = RunnableBranch(
    (is_long, long_branch),
    short_branch,
)

# Step 4: Assemble with fallback
chain = (
    RunnableLambda(lambda x: x)
    | RunnableParallel(
        preprocessing=preprocess,
        processing=process,
    )
    | RunnableLambda(lambda x: x["processing"])
    | branch
).with_fallbacks(
    [RunnableLambda(lambda x: "Fallback response")]
)

result = chain.invoke("hello world")
```

## Key Principles

1. **Composability is Universal**: All Runnables—prompts, models, parsers, tools—compose via `|` and dict operators.

2. **Automatic Mode Support**: Any composition automatically inherits sync, async, batch, and streaming support from its components.

3. **Type Safety**: Overloaded operators enable static type checking in IDEs and type checkers to catch misalignments early.

4. **Lazy Evaluation**: Composition creates new Runnable instances but does not execute them; execution happens only when `invoke`, `ainvoke`, `batch`, or `stream` are called.

5. **Streaming Preservation**: End-to-end streaming works when all components in a sequence implement `transform`.

6. **Error Handling Options**: Use `RunnableBranch` for conditional logic, `.with_fallbacks()` for recovery, and middleware for cross-cutting concerns.
