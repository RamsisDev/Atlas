# {agent-name}-state.md
# Universal Ralph State File — copy this template, never edit the template itself.
# Every agent in every department uses this exact structure.
# The NEXT field is the contract between iterations — it must be self-contained.

---

## metadata

```
agent: {name} | feature: {feature-slug} | iteration: {N} of {max}
context-usage: ~{pct}% | status: IN_PROGRESS | PARTIAL | EARLY_EXIT_WARNING
timestamp: {ISO-8601}
```

---

## completed-this-iteration

- [x] {completed item 1}
- [x] {completed item 2}

---

## pending

- [ ] {remaining item 1} (priority: HIGH)
- [ ] {remaining item 2} (priority: MEDIUM)

---

## filesystem-state

```
files-modified: [{path1}, {path2}]
files-created:  [{path3}]
files-deleted:  []
last-command:   {last bash command or tool call run}
```

---

## NEXT

{
  Exactly what the next fresh instance must do first.
  Must be specific enough that it needs only this file and the filesystem to continue.
  No conversation history. No vague instructions like "continue" or "keep going".

  Good example:
    "Read src/Domain/Auth/UserCredential.cs. The interface is defined but the
     Validate() method body is empty. Implement it using the rules in
     openspec/specs/jwt-auth-spec.md section 3.2. Then run the unit tests."

  Bad example:
    "Continue implementing the authentication feature."
}

---

## notes

<!-- Optional. Append observations the next instance should be aware of.
     Never delete previous notes — append only. -->
