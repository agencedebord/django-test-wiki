
## Codefidence (auto-managed)

### Before each non-trivial task
1. Read `.wiki/_index.md` for the overview
2. Read the domain notes related to the task
3. Account for documented business decisions
4. Check the dependency graph if multiple domains are involved

### After each task that modifies behavior
1. Update or create notes in `.wiki/domains/`
2. Document any non-obvious business decision in `.wiki/decisions/`
3. Update "Dependencies" and "Referenced by" sections
4. Regenerate `.wiki/_graph.md`
5. Update `.wiki/_index.md`
6. Separate commit with "wiki:" prefix

### What to document
- Counter-intuitive or client-specific business decisions
- Behaviors intentionally different from standard conventions
- Business rules not obvious from reading the code
- Domain architecture and responsibilities

### What NOT to document
- Standard framework/library behavior
- Implementation details readable in the code
- Trivial bugfixes with no behavioral impact

### Confidence rules
- `[confirmed]` or `[verified]`: trust as source of truth
- `[inferred]` or `[needs-validation]`: ALWAYS verify in code before relying on it
- If wiki contradicts code: code wins, update the wiki
- If `[inferred]` info drives a structural decision: ask the user for confirmation
