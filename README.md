# Jobsearcher

Enable job seekers to track job openings at companies they care about.

## Documentation
- Product Requirements: see the PRD section in the repository root (this README). The MVP focuses on tracking career pages, showing only jobs from the past 30 days, and removing closed roles automatically.
- Architecture Overview: [docs/architecture.md](docs/architecture.md)

## MVP Highlights
- Authenticated users add companies by name or careers URL; validation ensures the target page is reachable.
- Twice-daily crawls fetch public career pages, apply deterministic parsing, and fall back to LLM extraction on low-confidence results.
- LLM usage is tier-gated; deterministic validation still governs whether extracted jobs are stored.
- Lifecycle rules create jobs when first seen, delete roles missing from subsequent crawls, and expire listings older than 30 days.
- Users can view per-company or aggregated feeds, sorted by posted date with filters for company, location, job type, and keywords.
- Posted date transparency: display whether the date was scraped from the page or inferred by the system.
