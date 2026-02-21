# LangExtract Guardrails Provider

A provider plugin for [LangExtract](https://github.com/google/langextract) that wraps any `BaseLanguageModel` with output validation and automatic retry with corrective prompts. Subsumes retry-optimisation and verification provider concepts.

> **Note**: This is a third-party provider plugin for LangExtract. For the main LangExtract library, visit [google/langextract](https://github.com/google/langextract).

## Installation

Install from source:

```bash
git clone <repo-url>
cd langextract-guardrails
pip install -e .
```

## Features

- **Validation + retry loop** — validates LLM output against pluggable validators, retries with corrective prompts on failure
- **JSON Schema validation** — built-in `JsonSchemaValidator` checks syntax and schema compliance
- **Regex validation** — built-in `RegexValidator` for pattern matching
- **Corrective prompts** — automatically constructs retry prompts that include the original request, the invalid output, and the validation error
- **Configurable retries** — set `max_retries` (default: 3) per provider instance
- **Custom correction templates** — override the retry prompt template
- **Markdown fence stripping** — automatically handles ` ```json ` wrapped responses
- **Batch-independent retries** — each prompt in a batch retries independently

## Usage

### JSON Schema Validation

```python
import langextract as lx
from langextract_guardrails import GuardrailLanguageModel, JsonSchemaValidator

# Create the inner provider
inner_config = lx.factory.ModelConfig(
    model_id="litellm/azure/gpt-4o",
    provider="LiteLLMLanguageModel",
)
inner_model = lx.factory.create_model(inner_config)

# Define the expected output schema
schema = {
    "type": "object",
    "properties": {
        "parties": {
            "type": "array",
            "items": {"type": "string"},
        },
        "effective_date": {"type": "string"},
        "term_years": {"type": "integer"},
    },
    "required": ["parties", "effective_date"],
}

# Wrap with guardrails
guard_model = GuardrailLanguageModel(
    model_id="guardrails/gpt-4o",
    inner=inner_model,
    validators=[JsonSchemaValidator(schema=schema)],
    max_retries=3,
)

# Use as normal — invalid outputs are retried automatically
result = lx.extract(
    text_or_documents="Contract text...",
    model=guard_model,
    prompt_description="Extract parties and dates as JSON.",
)
```

### Regex Validation

```python
from langextract_guardrails import GuardrailLanguageModel, RegexValidator

guard_model = GuardrailLanguageModel(
    model_id="guardrails/gpt-4o",
    inner=inner_model,
    validators=[RegexValidator(r'"parties"\s*:', description="parties field")],
    max_retries=2,
)
```

### Combining Multiple Validators

Validators are applied in order. The first failure triggers a retry:

```python
from langextract_guardrails import (
    GuardrailLanguageModel,
    JsonSchemaValidator,
    RegexValidator,
)

guard_model = GuardrailLanguageModel(
    model_id="guardrails/gpt-4o",
    inner=inner_model,
    validators=[
        JsonSchemaValidator(schema=my_schema),       # Must be valid JSON matching schema
        RegexValidator(r'\d{4}-\d{2}-\d{2}', "date format"),  # Must contain a date
    ],
    max_retries=3,
)
```

### Custom Correction Template

```python
template = (
    "The output was invalid.\n"
    "Original request: {original_prompt}\n"
    "Your response: {invalid_output}\n"
    "Error: {error_message}\n"
    "Please fix the output."
)

guard_model = GuardrailLanguageModel(
    model_id="guardrails/gpt-4o",
    inner=inner_model,
    validators=[JsonSchemaValidator()],
    correction_template=template,
)
```

### Custom Validators

Implement the `GuardrailValidator` interface:

```python
from langextract_guardrails import GuardrailValidator, ValidationResult

class MaxLengthValidator(GuardrailValidator):
    def __init__(self, max_chars: int) -> None:
        self._max = max_chars

    def validate(self, output: str) -> ValidationResult:
        if len(output) <= self._max:
            return ValidationResult(valid=True)
        return ValidationResult(
            valid=False,
            error_message=f"Output exceeds {self._max} characters ({len(output)} chars)",
        )
```

### Async Usage

```python
results = await guard_model.async_infer(["prompt1", "prompt2"])
# Each prompt independently validates and retries
```

## How It Works

1. The original prompt is sent to the inner provider
2. The response is validated against all configured validators (in order)
3. If validation passes, the result is returned as-is
4. If validation fails:
   a. A corrective prompt is constructed with the original prompt, invalid output, and error message
   b. The corrective prompt is sent to the inner provider
   c. Steps 2-4 repeat up to `max_retries` times
5. If all retries are exhausted, the last result is returned with `score=0.0`

## Development

```bash
pip install -e ".[dev]"
pytest
```

## License

Apache 2.0
