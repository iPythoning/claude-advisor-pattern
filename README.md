# Claude Advisor Pattern

**Sonnet executes, Opus advises.** A Python implementation of the advisor pattern for the Anthropic Claude API using standard custom tools.

> The Claude API doesn't have a native `advisor` tool type. This library implements the same behavior: the executor model (Sonnet) autonomously decides when to consult a senior advisor (Opus) during task execution, using a standard tool-use loop.

## How It Works

```
User Task
    │
    ▼
┌─────────────────────────────────────────────┐
│  Executor (Sonnet 4.6)                      │
│                                             │
│  "I need creative direction for this..."    │
│        │                                    │
│        ▼ ask_advisor()                      │
│  ┌─────────────────┐                        │
│  │ Advisor (Opus)   │                        │
│  │ "Focus on        │                        │
│  │  contrast and    │                        │
│  │  emotional arc"  │                        │
│  └────────┬────────┘                        │
│           │                                  │
│  "Got it, here's the refined plan..."       │
│                                             │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
                Final Output
```

The executor decides **when** to consult (not every turn), **what** to ask, and **how** to incorporate the advice. The advisor budget (`max_advisor_calls`) prevents overuse.

## Quick Start

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."

# CLI
python advisor.py "Design a go-to-market strategy for an AI video tool" --max-calls 3

# With JSON output
python advisor.py "..." --json
```

### Python API

```python
from advisor import AdvisorClient

client = AdvisorClient()
result = client.run(
    "Create a 3-shot video storyboard for a smartphone ad",
    executor="claude-sonnet-4-6",   # fast, cheap — does the work
    advisor="claude-opus-4-6",      # smart, expensive — gives direction
    max_advisor_calls=3,            # budget: max 3 consultations
)

print(result.text)                  # final output
print(result.advisor_calls)         # how many times Sonnet asked Opus
print(result.advisor_log)           # full Q&A log
print(result.total_input_tokens)    # combined token usage
```

## When Sonnet Consults Opus

The executor autonomously decides when to use the advisor. Typical patterns:

| Situation | Sonnet's behavior |
|-----------|-------------------|
| Creative direction needed | Asks advisor before generating content |
| Ambiguous requirements | Asks advisor to clarify strategy |
| Quality review | Asks advisor to evaluate a draft |
| Trade-off decision | Asks advisor to weigh options |
| Routine execution | Proceeds without consulting |

## Configuration

```python
client = AdvisorClient(
    api_key="sk-ant-...",              # or set ANTHROPIC_API_KEY env var
    advisor_system="You are a senior creative director..."  # customize advisor persona
)

result = client.run(
    prompt="...",
    executor="claude-sonnet-4-6",      # any Claude model
    advisor="claude-opus-4-6",         # any Claude model (usually more capable)
    max_advisor_calls=3,               # budget control
    max_turns=10,                      # loop safety limit
    max_tokens=4096,                   # per-turn output limit
    system="You are a marketing expert",  # executor system prompt
    extra_tools=[...],                 # additional tools for executor
)
```

## Examples

### Video Storyboard (integrated with ai-video-studio)

```python
result = client.run(
    "Create a storyboard for a 15-second product video. "
    "The product is a wireless earbuds called 'AirPulse'. "
    "Target audience: young professionals. "
    "Output as JSON with shots array.",
    system="You are a video production assistant. Output storyboards as JSON.",
    max_advisor_calls=2,
)
# Sonnet will consult Opus on creative direction, then produce JSON storyboard
```

### Code Review

```python
result = client.run(
    f"Review this code for bugs and security issues:\n```python\n{code}\n```",
    advisor_system="You are a senior security engineer. Focus on OWASP top 10.",
    max_advisor_calls=1,  # one strategic review, then Sonnet details the findings
)
```

### Strategic Planning

```python
result = client.run(
    "Create a 90-day launch plan for entering the Japanese market "
    "with our B2B SaaS product (CRM for small businesses).",
    max_advisor_calls=3,  # budget for strategy, positioning, and risk review
)
```

## Cost Optimization

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Sonnet 4.6 | $3 | $15 |
| Opus 4.6 | $15 | $75 |

With `max_advisor_calls=3`, a typical task costs:
- ~90% Sonnet tokens (cheap execution)
- ~10% Opus tokens (expensive but targeted advice)

**Net effect**: ~80% cheaper than running everything on Opus, with comparable quality on strategic decisions.

## AdvisorResult

```python
@dataclass
class AdvisorResult:
    text: str                    # final output text
    advisor_calls: int           # times advisor was consulted
    advisor_log: list[dict]      # [{call, question, context, advice}, ...]
    total_input_tokens: int      # combined across all models
    total_output_tokens: int
    turns: int                   # conversation turns
```

## License

MIT

## Built With

- [Anthropic Claude API](https://docs.anthropic.com)
- Inspired by the advisor pattern concept for multi-model orchestration
