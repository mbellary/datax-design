<section class="hero">
  <div class="hero__inner">
    <div class="hero__eyebrow">Architecture Knowledge Base</div>
    <h1>DataX Design</h1>
    <p>
      A living design portal for DataX system architecture, request flows,
      container diagrams, operational models, and implementation decisions.
    </p>
    <div class="hero__actions">
      <a href="datax/" class="md-button md-button--primary">Explore DataX</a>
      <a href="codex/00-index/" class="md-button">Read Codex Study</a>
    </div>
  </div>
  <div class="hero__meta">
    <span>Version-controlled design artifacts</span>
    <span>Mermaid-native diagrams</span>
    <span>Published with GitHub Pages</span>
  </div>
</section>

## Design System Map

<div class="grid cards" markdown>

### DataX Design

The primary architecture space for DataX. Use this section for system context,
container architecture, workflows, data flows, APIs, storage, security, and
operations.

[Open DataX design](datax/index.md)

### Codex Design Study

Existing design research for Codex. This remains as a reference architecture
study and a source of patterns for extending DataX.

[Open Codex study](codex/00-index.md)

### Templates

Reusable writing patterns for architecture docs, sequence diagrams, flow
charts, container diagrams, API contracts, data models, and ADRs.

[Open templates](artifact-templates/index.md)

</div>

## Current Status

<div class="status-strip">
  <div>
    <strong>Codex study</strong>
    <span>Documented and wired into the site navigation.</span>
  </div>
  <div>
    <strong>DataX architecture</strong>
    <span>Structured and ready for design artifacts.</span>
  </div>
  <div>
    <strong>Publishing</strong>
    <span>Configured for GitHub Pages from Actions.</span>
  </div>
</div>

## Working Agreement

Design artifacts should be text-first, reviewable, and easy to evolve. Prefer
Markdown plus Mermaid for diagrams unless the artifact needs a richer modeling
tool. Every significant architectural decision should graduate into an ADR once
the trade-off is stable enough to preserve.

```mermaid
flowchart LR
    Discover["Discover requirements"] --> Model["Model architecture"]
    Model --> Review["Review trade-offs"]
    Review --> Decide["Record decision"]
    Decide --> Evolve["Evolve implementation plan"]
    Evolve --> Discover
```
