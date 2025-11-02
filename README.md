# Mojo Development Skills for Claude Code

Skills for developing, refactoring, analyzing, and migrating [Mojo](https://www.modular.com/mojo) code. These skills help Claude Code assist with Mojo's unique features, rapid evolution, and performance-critical use cases like GPU kernel development.

For more information about skills, check out:
- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills)
- [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)

# About This Repository

This repository contains Claude Code skills specifically designed for Mojo development. Mojo is a high-performance programming language that combines Python's usability with systems programming and metaprogramming capabilities. Since Mojo is evolving rapidly with frequent breaking changes, these skills help Claude Code stay current with best practices and assist with common migration challenges.

**Why Mojo-specific skills?** Mojo's unique combination of features requires specialized knowledge:
- **Rapid evolution**: Mojo updates frequently with breaking API changes
- **GPU/hardware programming**: Low-level kernel development and SIMD operations
- **Type system**: Advanced parametric types and compile-time metaprogramming
- **Python interop**: Bridging Python libraries with high-performance Mojo code
- **Performance optimization**: Memory management, vectorization, and parallelization

Each skill is self-contained in its own directory with a `SKILL.md` file containing the instructions and metadata that Claude uses.

**Note:** These skills are designed to be practical tools for real-world Mojo development. As Mojo evolves, these skills may need updates to reflect the latest language features and best practices.

## Disclaimer

**These skills are provided for educational and development purposes.** Mojo is a rapidly evolving language, and syntax, APIs, and best practices may change between versions. Always verify against the [official Mojo documentation](https://docs.modular.com/mojo/) and test thoroughly. The implementations shown here reflect current understanding but may need adaptation as the language evolves.

# Example Skills

This repository will include skills demonstrating different aspects of Mojo development:

## GPU & Hardware Programming
- **gpu-kernel-development** - Write high-performance GPU kernels using Mojo's SIMD types, parallel primitives, and memory management
- **simd-optimization** - Optimize code using SIMD vectors and vectorization strategies
- **hardware-intrinsics** - Work with low-level hardware features and intrinsics

## Version Migration & Compatibility
- **version-migration** - Migrate Mojo code between versions, handling breaking changes and deprecated APIs
- **api-compatibility** - Check compatibility across Mojo versions and suggest modern alternatives
- **migration-patterns** - Common patterns for refactoring code to use newer Mojo features

## Code Quality & Patterns
- **mojo-best-practices** - Apply Mojo-specific best practices for performance, safety, and maintainability
- **python-interop** - Integrate Python libraries and migrate Python code to Mojo
- **type-system-guide** - Use Mojo's advanced type system including parametric types and traits

## Development & Tooling
- **performance-analysis** - Profile and optimize Mojo code for maximum performance
- **testing-mojo** - Write tests for Mojo code using current testing approaches
- **debugging-guide** - Debug Mojo programs and understand common error patterns

# Using These Skills

## In Claude Code

To use a skill from this repository in Claude Code:

1. Clone this repository locally or download specific skill folders
2. Copy the skill folder to your Claude Code skills directory:
   ```bash
   cp -r /path/to/claude-mojo-skills/skill-name ~/.claude/skills/
   ```
3. Use the skill by mentioning it in conversation: "Use the gpu-kernel-development skill to help me write a matrix multiplication kernel"

Alternatively, if this repository becomes available as a Claude Code plugin marketplace:
```
/plugin marketplace add your-username/claude-mojo-skills
/plugin install mojo-skills
```

## In Claude.ai

Follow the instructions in [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b) to upload individual skills.

## Via Claude API

You can upload custom skills via the Claude API. See the [Skills API Quickstart](https://docs.claude.com/en/api/skills-guide#creating-a-skill) for more information.

# Creating a Mojo Skill

Skills are simple to create - just a folder with a `SKILL.md` file containing YAML frontmatter and instructions:

```markdown
---
name: my-mojo-skill
description: A clear description of what this skill does for Mojo development
---

# My Mojo Skill

[Add your Mojo-specific instructions here that Claude will follow when this skill is active]

## Key Mojo Concepts
- Concept 1 (e.g., ownership and lifetimes)
- Concept 2 (e.g., parametric types)

## Common Patterns
- Pattern 1 with code example
- Pattern 2 with code example

## Version-Specific Notes
- Note any version-specific behaviors or breaking changes

## Resources
- Links to relevant Mojo documentation
```

The frontmatter requires only two fields:
- `name` - A unique identifier for your skill (lowercase, hyphens for spaces)
- `description` - A complete description of what the skill does and when to use it

For more details, see [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills).

# Contributing

Contributions are welcome! If you've created a useful Mojo skill or have improvements to existing ones:

1. Ensure your skill includes clear instructions and examples
2. Test the skill with current Mojo versions
3. Document any version-specific behavior
4. Submit a pull request with a description of what the skill does

# Resources

- [Mojo Official Documentation](https://docs.modular.com/mojo/)
- [Mojo GitHub Repository](https://github.com/modularml/mojo)
- [Modular Developer Community](https://www.modular.com/community)
- [Claude Code Documentation](https://docs.claude.com/claude-code)

# License

This repository is open source under the Apache 2.0 license. See [LICENSE](LICENSE) for details.

---

**About Mojo**: Mojo is a programming language designed for AI infrastructure, combining Python's ease of use with C-level performance and low-level control. It's developed by Modular and is actively evolving with frequent updates and improvements.
