# Sample Slack prompt

Realistic first message for a software engineer portfolio. Mention your bot the same way SuperPlane’s Slack integration expects (app mention).

## Initial request

```text
@PortfolioFactory Build a personal portfolio website for Jordan Lee, a mid-level software engineer in Austin, TX.

Role: Full-stack engineer focused on TypeScript, React, and Go.
Tone: Clean, modern, slightly technical — dark charcoal background, electric blue accents, Inter-like UI typography feel (use web-safe / system stacks if needed).

Sections:
1. Hero with name, title, one-line pitch, and links to GitHub + LinkedIn
2. About — 2 short paragraphs (placeholder OK)
3. Experience — 3 roles with company, dates, and 2–3 bullets each
4. Projects — 3 project cards with name, 1-sentence description, stack tags, and a “View project” placeholder link
5. Skills — grouped chips (Languages, Frontend, Backend, Tools)
6. Contact — email placeholder and a simple CTA button

Projects to feature:
- OrbitTasks — kanban-style project tracker (React, Node, Postgres)
- SignalBoard — real-time ops dashboard (Go, WebSockets, Redis)
- Lenscribe — AI meeting notes web app (TypeScript, OpenAI API, Next-style UX in single page)

Experience ideas:
- Platform Engineer @ Northwave (2023–present)
- Software Engineer @ Brightline Labs (2021–2023)
- SWE Intern @ Harbor Systems (2020)

Include subtle hover transitions, sticky nav, and a responsive mobile layout. Single self-contained index.html only.
```

## Follow-up refinements (same Slack thread)

After a preview, reply in the **same thread** so conversational memory applies:

```text
@PortfolioFactory Make the hero more minimal, drop the Skills chip cloud into a compact two-column list, and add a “Speaking” section with three placeholder conference talks.
```

```text
@PortfolioFactory Use a warmer palette — off-white background, navy text, terracotta accent — and rename the CTA to “Hire me”.
```

## Optional GitHub repo override

If you need a different repository than the canvas default:

```text
@PortfolioFactory repo:YOUR_GITHUB_USERNAME/YOUR_REPOSITORY Build a software engineer portfolio for Jordan Lee with a dark monochrome theme and three project cards.
```
