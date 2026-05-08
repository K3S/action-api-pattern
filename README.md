# Action API Pattern

Source for [action-api-pattern.k3s.com](https://action-api-pattern.k3s.com) — a free, open-source guide to decomposing IBM i green-screen applications into single-purpose, callable APIs that work for web, mobile, and AI agents.

Sponsored by [King III Solutions](https://k3s.com).

## What this is

A pattern for taking the business logic out of green-screen RPG applications and packaging each individual action as a self-contained, callable API. The same RPG program then serves a web app, a mobile client, an AI agent, or another RPG program.

This is the pattern K3S has used in our supply chain replenishment software since the late 2000s, and the pattern that powers our R7 web platform.

## Reading this locally

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

## Related K3S tutorials

- [RPG Tutorial](https://rpgtutorial.k3s.com) — Learning RPGLE and CL with VS Code
- [IBM i AI Workers](https://ibmi-ai-workers.k3s.com) — Calling LLMs at scale from RPG batch jobs
- **Action API Pattern** — *this site*

## License

This project is dual-licensed:

- **Code** (any sample programs, snippets, configuration files): MIT License — see [LICENSE-CODE](LICENSE-CODE)
- **Content** (prose, documentation, diagrams): Creative Commons Attribution-ShareAlike 4.0 International — see [LICENSE-CONTENT](LICENSE-CONTENT)

## Contributing

Contributions are welcome. Spot a typo, want to add an example, want to argue with a recommendation? [Open an issue or PR](https://github.com/K3S/action-api-pattern). Each page also has an "Edit this page on GitHub" link in the footer for one-click PRs.
