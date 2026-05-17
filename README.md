# effective-testing-skills

> Agent Skills for writing, reviewing, and auditing automated tests — built on the [Agent Skills](https://agentskills.io) open format and compatible with Claude Code, GitHub Copilot (Agent mode), and any other Agent Skills-compatible client.

## What's in this repo

A single skill that gives an agent practical, opinionated guidance on automated testing — synthesised from multiple practitioner sources and structured around the jobs you're actually doing, not an abstract list of principles.

```
.agents/skills/
└── effective-testing/
    └── SKILL.md
```

## What the skill covers

The skill is organised into four task-focused sections, so the agent jumps straight to what's relevant:

| Task | What the skill does |
|---|---|
| **Writing a new test** | Guides layer selection, test structure, assertions, data isolation, and CI compatibility |
| **Reviewing a PR** | Structured questions covering correctness, trust, stability, readability, and redundancy |
| **Diagnosing a flaky test** | Symptom-to-cause table with concrete fixes for the most common flakiness patterns |
| **Auditing a suite** | Coverage balance check, trust litmus test, redundancy review, and maintenance cost signals |

### Key decisions the skill helps with

- **Which layer to test at** — using a resource-based model (small/medium/large) rather than just "unit vs E2E", informed by Google's engineering practice and the Testing Trophy's ROI perspective
- **What to assert on** — outcomes, not implementation steps
- **How to write failure output** that's self-explanatory without re-running
- **When to delete a test** — a smaller trusted suite beats a large ignored one

## Installation

### Option 1 — npx (recommended)

```bash
npx skills add mannibasi/effective-testing-skills
```

This copies the skill into `.agents/skills/effective-testing/` relative to your project root.

### Option 2 — Git submodule

Keep the skill in sync with upstream changes:

```bash
git submodule add https://github.com/mannibasi/effective-testing-skills .agents/skills/effective-testing
git submodule update --init
```

To pull future updates:

```bash
git submodule update --remote
```

### Option 3 — Manual copy

```bash
mkdir -p .agents/skills/effective-testing
curl -o .agents/skills/effective-testing/SKILL.md \
  https://raw.githubusercontent.com/mannibasi/effective-testing-skills/main/.agents/skills/effective-testing/SKILL.md
```

## Usage

Once installed, skills are auto-discovered when the agent starts a session. No configuration required.

**VS Code (GitHub Copilot Agent mode)**

1. Open Copilot Chat and switch to **Agent** mode
2. Type `/skills` to confirm `effective-testing` appears in the list
3. Ask naturally — the skill activates when relevant:
   - *"Write a test for the checkout flow"*
   - *"Review the tests in this PR"*
   - *"This test is flaky in CI — help me work out why"*
   - *"Audit our test suite, it feels too heavy on E2E"*

**Claude Code**

Skills in `.agents/skills/` are picked up automatically. Work normally and the agent will apply the skill when writing or reviewing tests.

## Design decisions

**Task-oriented, not principle-oriented** — the skill is organised around what you're *doing* (writing, reviewing, diagnosing, auditing) rather than a numbered list of abstract principles. This maps better to how agents are actually used in a conversation.

**Layer selection by resource cost, not label** — "unit vs integration vs E2E" is ambiguous and often argued about. The skill uses Google's small/medium/large framing (defined by what a test is allowed to do — network access, filesystem, etc.) which is more precise and less subjective.

**Integration tests are the backbone** — the skill reflects the Testing Trophy view that integration tests often give the best ROI: they test real component interactions without the full cost and brittleness of E2E. This is a deliberate bias away from "mostly unit tests."

**No tool or framework assumptions** — the skill is language-agnostic. Examples use Python for concreteness but the principles apply to any stack.

## Contributing

Contributions that add gotchas, sharpen examples, or improve the layer-selection guidance with real-world experience are welcome.

1. Fork the repo
2. Edit `.agents/skills/effective-testing/SKILL.md`
3. Open a PR with a short explanation of what you changed and why

Please keep the skill under 500 lines and 5,000 tokens — the [Agent Skills spec](https://agentskills.io/specification) recommends this to avoid overloading the agent's context window.

## Sources

The skill was synthesised from the following sources, each reviewed and evaluated for what they added versus what was already covered:

- [Software Engineering at Google, Chapter 11](https://abseil.io/resources/swe-book/html/ch11.html)
- [The Testing Trophy — Kent C. Dodds](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
- [Write tests. Not too many. Mostly integration. — Kent C. Dodds](https://kentcdodds.com/blog/write-tests)
- [7 Features of a Good Automated Test — testRigor](https://testrigor.com/blog/7-features-of-a-effective-testing/)
- [12 Traits of Highly Effective Tests — Automation Panda / Andy Knight](https://automationpanda.com/2020/07/09/12-traits-of-highly-effective-tests/)
- [10 Key Features of Effective Automated Software Tests — TestWheel](https://www.testwheel.com/blog/10-features-of-successful-test-automation/)

## License

MIT
