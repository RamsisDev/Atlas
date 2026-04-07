# model-capabilities.md
# Reference file for Ralph protocol context thresholds.
# Each agent reads this file at startup to determine its 80% context threshold.
# Update this file when new model generations are released — no AGENT.md changes needed.

## context-thresholds

| Model             | Context | 80% Threshold | Best For                                          | Cost Tier |
|-------------------|---------|---------------|---------------------------------------------------|-----------|
| claude-opus-4-6   | 200k    | 160k tokens   | Root agents, complex analysis, architecture review | High      |
| claude-sonnet-4-6 | 200k    | 160k tokens   | Dev agents, QA, most execution tasks              | Medium    |
| claude-haiku-4-5  | 200k    | 160k tokens   | Scout, simple checks, mappers, compilers          | Low       |

## ralph-protocol-rule

Every agent using Ralph must:
1. Read this file at the start of iteration 1.
2. Use the 80% threshold for its declared model to determine when to checkpoint.
3. NEVER hardcode a token count — always derive it from this file.

When context tokens consumed >= threshold: write ralph-state with NEXT field and stop.

## update-instructions

When a new Claude model generation is released:
1. Add a new row to the context-thresholds table above.
2. Update the 80% threshold if context size changed.
3. Do NOT modify any AGENT.md files — model assignments stay the same.
4. The Ralph protocol will automatically use the new threshold on next pipeline run.
