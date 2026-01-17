# Contributing to SharePasswords

Thank you for your interest in contributing to SharePasswords! This document provides guidelines and instructions for contributing.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Ways to Contribute](#ways-to-contribute)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Pull Request Guidelines](#pull-request-guidelines)
- [Code Style](#code-style)
- [Security](#security)

## Code of Conduct

By participating in this project, you agree to maintain a respectful and inclusive environment. We expect all contributors to:

- Be respectful and constructive in discussions
- Welcome newcomers and help them get started
- Focus on what's best for the community
- Show empathy towards other community members

## Ways to Contribute

### 1. Submit Suggestions

Have an idea for a new feature or improvement? See [SUGGESTIONS.md](SUGGESTIONS.md) for how to submit feature requests via Pull Requests.

### 2. Improve Documentation

- Fix typos or clarify existing documentation
- Add examples or tutorials
- Translate documentation to other languages

### 3. Report Bugs

Found a bug? Please open an issue with:

- Clear description of the problem
- Steps to reproduce
- Expected vs actual behavior
- Environment details (OS, browser, etc.)

### 4. Security Vulnerabilities

**Do NOT open public issues for security vulnerabilities.**

Please report security issues privately to: security@sharepasswords.com

See [Security](#security) section for more details.

### 5. Code Contributions

Want to contribute code? Great! See [Development Workflow](#development-workflow) below.

## Getting Started

### Prerequisites

- Node.js 22.x or higher
- pnpm 10.x or higher
- Git
- MongoDB (for local development)

### Setting Up the Development Environment

1. **Fork the repository** on GitHub

2. **Clone your fork**

   ```bash
   git clone https://github.com/YOUR_USERNAME/sharepasswords.git
   cd sharepasswords
   ```

3. **Install dependencies**

   ```bash
   pnpm install
   ```

4. **Set up environment variables**

   ```bash
   cp packages/backend/.env.example packages/backend/.env
   cp packages/frontend/.env.example packages/frontend/.env
   ```

5. **Start development servers**

   ```bash
   # Start all services
   pnpm dev

   # Or individually
   pnpm dev:backend   # Backend on port 3000
   pnpm dev:frontend  # Frontend on port 5173
   ```

## Development Workflow

### 1. Create a Branch

```bash
# For features
git checkout -b feature/your-feature-name

# For bug fixes
git checkout -b fix/bug-description

# For documentation
git checkout -b docs/what-you-changed
```

### 2. Make Your Changes

- Write clean, readable code
- Follow the existing code style
- Add tests for new functionality
- Update documentation as needed

### 3. Test Your Changes

```bash
# Run all tests
pnpm test

# Run linting
pnpm lint

# Build to check for errors
pnpm build
```

### 4. Commit Your Changes

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```bash
# Format: <type>(<scope>): <description>

git commit -m "feat(api): add rate limiting to secrets endpoint"
git commit -m "fix(frontend): resolve decryption error on Safari"
git commit -m "docs: update API reference with new endpoints"
```

**Types:**

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Code style (formatting, etc.) |
| `refactor` | Code refactoring |
| `test` | Adding or updating tests |
| `chore` | Maintenance tasks |

### 5. Push and Create PR

```bash
git push origin your-branch-name
```

Then open a Pull Request on GitHub.

## Pull Request Guidelines

### Before Submitting

- [ ] Code follows the project's style guidelines
- [ ] Tests pass locally (`pnpm test`)
- [ ] Linting passes (`pnpm lint`)
- [ ] Build succeeds (`pnpm build`)
- [ ] Documentation updated (if applicable)
- [ ] Commit messages follow Conventional Commits

### PR Description Template

```markdown
## Description

[Describe your changes]

## Type of Change

- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Code refactoring
- [ ] Other (please describe)

## Testing

[Describe how you tested your changes]

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review
- [ ] I have added tests (if applicable)
- [ ] I have updated documentation (if applicable)
```

### Review Process

1. A maintainer will review your PR
2. They may request changes or ask questions
3. Once approved, your PR will be merged
4. Your contribution will be credited in the release notes

## Code Style

### TypeScript/JavaScript

- Use TypeScript for all new code
- Use meaningful variable and function names
- Prefer `const` over `let`; avoid `var`
- Use async/await over callbacks
- Document complex logic with comments

```typescript
// Good
async function createEncryptedSecret(plaintext: string): Promise<Secret> {
  const key = await generateKey();
  const encrypted = await encrypt(plaintext, key);
  return { id: generateId(), content: encrypted };
}

// Avoid
function createSecret(p, cb) {
  genKey().then(k => {
    enc(p, k).then(e => cb(null, { id: genId(), content: e }));
  });
}
```

### Svelte Components

- Use Svelte 5 runes (`$state`, `$derived`, etc.)
- Keep components focused and small
- Use TypeScript in script blocks

### CSS/Styling

- Use Tailwind CSS utility classes
- Follow mobile-first approach
- Maintain consistent spacing

## Project Structure

```
packages/
├── backend/           # NestJS API
│   ├── src/
│   │   ├── modules/   # Feature modules
│   │   ├── core/      # Core utilities
│   │   └── infra/     # Infrastructure
│   └── test/          # Tests
├── frontend/          # Svelte SPA
│   ├── src/
│   │   ├── components/ # UI components
│   │   └── lib/       # Utilities
│   └── tests/         # Tests
├── core/              # Shared logic
└── types/             # Shared types
```

## Security

### Reporting Vulnerabilities

Please report security vulnerabilities **privately** to:

**Email:** security@sharepasswords.com

Include:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We will acknowledge receipt within 48 hours and work with you to address the issue.

### Security Guidelines for Contributors

When contributing code, please ensure:

1. **No plaintext secrets** - Never log, store, or transmit unencrypted secrets
2. **Maintain zero-knowledge** - The server must never have access to encryption keys
3. **Use secure defaults** - Secure configurations should be the default
4. **Validate all input** - Never trust user input
5. **Follow OWASP guidelines** - Be aware of common vulnerabilities

### Dependencies

- Only add dependencies from trusted sources
- Check for known vulnerabilities before adding
- Keep dependencies updated

## Questions?

- **General questions:** Open a Discussion or Issue
- **Contribution questions:** Tag a maintainer in your PR
- **Security questions:** Email security@sharepasswords.com

## Recognition

All contributors will be recognized in:

- Release notes
- Contributors list
- Special thanks section (for significant contributions)

Thank you for contributing to SharePasswords!
