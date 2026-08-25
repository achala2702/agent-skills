# 🛠️ Agent Skills (`agent-skills`)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skill Standard](https://img.shields.io/badge/Standard-SKILL.md-blue.svg)](https://skills.sh)

A production-grade, installable agent skills designed for any AI agent that supports the Skills standard. This skill encodes modern software engineering practices, testing frameworks, and domain-specific knowledge directly into your agentic workflows.

---


## 📦 Available Skills

| Skill Name | Description | Target Stack | Status |
|---|---|---|---|
| [`spock-spring-boot-testing`](skills/spock-spring-boot-testing/SKILL.md) | Comprehensive guidelines for Spring Boot unit and integration testing using Groovy Spock specs. | Java / Spring Boot / Groovy | ✅ Active |
| More coming soon... | Additional developer toolkits and architectural skills will be added here. | Various | 🔜 Planned |

---

## 🧪 spock-spring-boot-testing

Encodes production-tested standards for writing, reviewing, and refactoring unit and integration tests in Spring Boot applications using the **Spock Framework** (`*.groovy` Specification classes) combined with `spring-boot-starter-test`.

### Key Highlights & Capabilities

- **Idiomatic BDD Specs** — Standardizes `given:`, `when:`, `then:`, and `where:` blocks for readable, parameter-driven Groovy tests.
- **Spring Boot Integration** — Guidelines for bean mocking, `@SpringBean`, `@SpringBootTest`, and slice testing (Controller, Service, Repository layers).
- **JUnit Migration** — Built-in guidance for converting legacy JUnit suites into clean Spock specifications.
- **CI & Coverage** — Best practices for coverage reporting with Spock and continuous integration execution.

### Installation

```bash
npx skills add achala2702/agent-skills --skill spock-spring-boot-testing
```

---

## 📂 Repository Layout

This repository follows the standard multi-skill directory architecture:

```
agent-skills/
├── LICENSE
├── README.md
└── skills/
    └── spock-spring-boot-testing/
        └── SKILL.md
```

---

## 🤝 Contributing

Got an idea for a new skill or improvements to existing ones?

1. Fork the repository.
2. Create your feature branch (`git checkout -b skill/my-new-skill`).
3. Add your skill directory with a `SKILL.md` file following the frontmatter convention.
4. Commit your changes (`git commit -m 'feat(skills): add my-new-skill'`).
5. Open a Pull Request into `main`.

---

## 📄 License

This repository is licensed under the **MIT License**. You are free to use, modify, and distribute these skills in personal, open-source, or commercial projects.