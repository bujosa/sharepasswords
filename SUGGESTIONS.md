# Suggestions & Feature Requests

We welcome community input to help shape the future of SharePasswords! This document explains how to submit suggestions, feature requests, and ideas via Pull Requests.

## How to Submit a Suggestion

### 1. Check Existing Suggestions

Before submitting, please check the existing suggestions in the [suggestions/](suggestions/) folder to avoid duplicates.

### 2. Create Your Suggestion File

Create a new markdown file in the `suggestions/` folder following this naming convention:

```
suggestions/YYYY-MM-DD-short-description.md
```

Example: `suggestions/2024-01-15-password-generator.md`

### 3. Use the Suggestion Template

Copy the template below into your new file:

```markdown
# Suggestion: [Title]

**Author:** [Your GitHub username]
**Date:** [YYYY-MM-DD]
**Status:** Proposed

## Summary

[One paragraph describing the suggestion]

## Problem Statement

[What problem does this solve? Why is it needed?]

## Proposed Solution

[Detailed description of your proposed solution]

## Alternatives Considered

[What other approaches did you consider?]

## Additional Context

[Any other context, mockups, or examples]

---

## Discussion

[Leave this section empty - maintainers and community will add comments here]
```

### 4. Submit a Pull Request

1. Fork this repository
2. Create a new branch: `suggestion/your-suggestion-name`
3. Add your suggestion file to `suggestions/`
4. Open a Pull Request with the title: `[Suggestion] Your suggestion title`

## Suggestion Categories

When creating your suggestion, consider which category it falls into:

| Category | Description |
|----------|-------------|
| **Feature** | New functionality |
| **Enhancement** | Improvement to existing features |
| **Security** | Security-related improvements |
| **UX** | User experience improvements |
| **API** | API changes or additions |
| **Documentation** | Documentation improvements |
| **Integration** | Third-party integrations |

## Suggestion Workflow

```
┌─────────────────┐
│    Proposed     │  ← Initial submission
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Under Review   │  ← Maintainers reviewing
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────────┐
│Accepted│ │  Declined │
└───┬───┘ └───────────┘
    │
    ▼
┌─────────────────┐
│  In Progress    │  ← Being implemented
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Completed     │  ← Shipped!
└─────────────────┘
```

## Status Labels

| Status | Description |
|--------|-------------|
| `Proposed` | Newly submitted, awaiting review |
| `Under Review` | Being evaluated by maintainers |
| `Accepted` | Approved for implementation |
| `In Progress` | Currently being implemented |
| `Completed` | Implemented and released |
| `Declined` | Not accepted (with explanation) |
| `Duplicate` | Already suggested |
| `Needs Info` | More information needed |

## What Makes a Good Suggestion?

### Do:

- **Be specific** - Clearly describe what you want and why
- **Provide context** - Explain the problem you're trying to solve
- **Consider security** - SharePasswords is security-focused; consider implications
- **Include examples** - Mockups, diagrams, or code examples help
- **Research first** - Check if similar features exist or were discussed

### Don't:

- Submit vague or incomplete suggestions
- Request features that compromise zero-knowledge security
- Duplicate existing suggestions
- Submit multiple unrelated suggestions in one PR

## Example Suggestions

### Good Example

```markdown
# Suggestion: Password Strength Indicator

**Author:** @johndoe
**Date:** 2024-01-15
**Status:** Proposed

## Summary

Add a visual password strength indicator when users create secrets,
helping them understand if their password is weak before sharing.

## Problem Statement

Users sometimes share weak passwords without realizing it. A strength
indicator would provide immediate feedback and encourage better security
practices.

## Proposed Solution

Add a strength meter below the secret input that analyzes:
- Length (minimum 12 characters recommended)
- Character variety (uppercase, lowercase, numbers, symbols)
- Common patterns to avoid

Display as a colored bar: Red (weak) → Yellow (fair) → Green (strong)

## Alternatives Considered

1. **Server-side validation** - Rejected because it would require
   sending plaintext to server, breaking zero-knowledge
2. **Popup warning** - Less user-friendly than inline indicator

## Additional Context

This would be purely client-side to maintain zero-knowledge architecture.
Similar to the strength indicators on services like 1Password or Bitwarden.
```

### Bad Example

```markdown
# Suggestion: Make it better

**Author:** @someone

## Summary

The app should be better.

## Proposed Solution

Improve it.
```

## Questions?

If you're unsure whether your idea is appropriate or how to format it, feel free to:

1. Open a Discussion (if enabled)
2. Create an Issue asking for guidance
3. Check existing suggestions for examples

## Current Roadmap Ideas

These are features we're already considering. Feel free to submit detailed suggestions for any of these:

- [ ] User accounts and authentication
- [ ] Personal password vault
- [ ] Team/organization workspaces
- [ ] Browser extensions
- [ ] CLI tool
- [ ] API keys for programmatic access
- [ ] Slack/Teams integrations
- [ ] Email notifications on view
- [ ] IP-based access restrictions
- [ ] Custom branding (enterprise)
- [ ] Audit logging

## License

By submitting a suggestion, you agree that your contribution may be used in SharePasswords under the project's license.
