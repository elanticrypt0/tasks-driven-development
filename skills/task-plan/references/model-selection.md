# Model selection for analysis

The capability-based YAML is the source of truth; brand names below are an aid that ages. Verified on 2026-07-14.

```yaml
models:
  primary: best_available               # decisions, plan, and main code
  agent:   best_available_for_coding    # subtasks / parallel execution
  review:  best_available_for_reasoning # critical and security review
  fast:    fastest_available            # trivial or high-volume tasks
```

## Rule for the analysis phase

The analysis and plan-creation phase always uses the **most powerful model the provider offers**. Weaker models may implement; they must not analyze.

| Provider | First choice | Fallback | If unavailable |
|----------|--------------|----------|----------------|
| Anthropic | Fable | Opus | ask the user which model to use |
| OpenAI | SOL (highest variant available) | — | ask the user which model to use |
| Other providers | — | — | ask the user which model to use |

- "Ask the user" means: stop and ask explicitly which model should run the analysis and plan creation. Do not silently downgrade.
- If the current session already runs on a qualifying model, proceed. If not, state the mismatch before continuing.

## Maintenance

When a provider releases a new top model or renames a family, update this table and renew the verification date.
