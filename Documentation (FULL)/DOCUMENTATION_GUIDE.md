# Documentation Guide

Formula Vision documentation is organized around reader intent instead of implementation folders.

## Sources used for the documentation structure

The repository documentation follows three external references:

1. GitHub repository and README guidance: explain what the project does, why it is useful, how to get started, and where to find more help.
2. Diataxis: separate explanation, how-to guidance, reference material, and learning/history-oriented documentation.
3. Google developer documentation style guidance: prefer clear, concise, unambiguous language and consistent terminology.

References:

- https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes
- https://docs.github.com/en/repositories/creating-and-managing-repositories/best-practices-for-repositories
- https://diataxis.fr/
- https://developers.google.com/style/translation

## Repository documentation model

### README.md

The first-stop overview:

- what Formula Vision is;
- why v2 exists;
- current architecture at a glance;
- basic local commands;
- links to deeper documentation.

### Explanation

Use for system concepts and why the project works the way it does.

Examples:

- ENGINEERING_OVERVIEW.md
- ARCHITECTURE.md
- PROJECT_EVOLUTION.md
- SPATIAL_CAMERA_ARCHITECTURE.md
- UI_REDESIGN_BLUEPRINT.md

### How-to and operations

Use for procedures that a maintainer follows to accomplish a task.

Examples:

- PRODUCTION_RUNBOOK.md
- LIBRARY_BACKFILL.md
- CLOUDFLARE_DELIVERY.md
- CUTOVER_CHECKLIST.md
- R2_SETUP.md

### Reference

Use for stable facts, contracts, profiles, and policies.

Examples:

- DATA_SOURCE_POLICY.md
- DATA_ATTRIBUTION.md
- DELIVERY_BENCHMARK_FINDINGS.md
- canonical profile documents

### Decision records

Use ADRs for important architectural choices, alternatives, and consequences.

### Milestone records

Milestone docs are historical evidence. Do not rewrite old milestone branches to make them look like the current system.

## Writing rules

- State the purpose of a document near the top.
- Prefer direct sentences and active voice.
- Use the same term for the same concept everywhere.
- Separate current behavior from future ideas.
- Distinguish authoritative race data from presentation/inference.
- Use relative repository links.
- Put procedures in numbered steps when order matters.
- Keep commands copyable.
- Keep current-state docs timeless where possible; use date-relative wording in historical records only.
- Put the most important fact at the start of a section or paragraph so the document scans well.
- Prefer concrete evidence, constraints, measurements, and tradeoffs over adjectives such as "advanced", "enterprise", or "production-grade".
- For portfolio signal, explain ownership and engineering decisions through the system itself rather than inserting resume language into technical sections.
- Record important failure behavior, not only happy paths.
- Avoid vague words such as "magic", "seamless", and "robust" unless the document explains what they mean.
- Avoid em dashes.
- Update docs in the same change that changes architecture, public behavior, or operations.

## Current documentation ownership

When changing:

- source/canonical contracts: update pipeline docs and architecture/reference docs;
- publication/storage: update architecture, runbook, and relevant ADR;
- UI/product interaction: update frontend docs and design docs;
- camera/spatial behavior: update SPATIAL_CAMERA_ARCHITECTURE.md;
- operational commands: update the relevant runbook immediately;
- project history/status: update PROJECT_EVOLUTION.md or ROADMAP.md, not historical milestone branches;
- cross-cutting project scope: update ENGINEERING_OVERVIEW.md when a major capability changes more than one subsystem.
