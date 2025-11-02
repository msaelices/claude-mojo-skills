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

# Available Skills

This repository currently includes the following Mojo development skills:

## mojo-kernels - GPU Kernel Development

Write high-performance GPU kernels in Mojo using the MAX framework. This skill provides:

## mojo-updater - Version Migration Tool

Systematically update Mojo code to newer versions using a test-driven approach. This skill provides:

# Using These Skills

## In Claude Code

To use a skill from this repository in Claude Code:

1. Clone this repository locally or download specific skill folders
2. Copy the skill folder to your Claude Code skills directory:
   ```bash
   cp -r /path/to/claude-mojo-skills/mojo-kernels ~/.claude/skills/
   cp -r /path/to/claude-mojo-skills/mojo-updater ~/.claude/skills/
   ```
3. Use the skill by mentioning it in conversation:
   - "Use the mojo-kernels skill to help me write a matrix multiplication kernel"
   - "Use the mojo-updater skill to update my code to Mojo 0.25.6"

To install directly from the plugin marketplace, run the following commands in Claude Code:
```
/plugin marketplace add msaelices/claude-mojo-skills
/plugin install mojo-skills
```

## In Claude.ai

Follow the instructions in [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude#h_a4222fa77b) to upload individual skills.

## Via Claude API

You can upload custom skills via the Claude API. See the [Skills API Quickstart](https://docs.claude.com/en/api/skills-guide#creating-a-skill) for more information.

# License

This repository is open source under the MIT license. See [LICENSE](LICENSE) for details.

