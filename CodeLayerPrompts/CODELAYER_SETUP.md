# CodeLayer Research Setup

## Quick Start

### 1. Clone Chartsmith and add your PRDs
```bash
git clone https://github.com/replicatedhq/chartsmith
cd chartsmith
mkdir -p docs/prds docs/research
```

### 2. Copy your PRD files into the repo
```bash
# Copy all your PRD files to docs/prds/
cp ~/path/to/PR1_Product_PRD.md docs/prds/
cp ~/path/to/PR1_Tech_PRD.md docs/prds/
cp ~/path/to/PR1_INSTRUCTIONS.md docs/prds/
cp ~/path/to/PR2_Product_PRD.md docs/prds/
cp ~/path/to/PR2_Tech_PRD.md docs/prds/
cp ~/path/to/PR2_GAPS_ANALYSIS.md docs/prds/
cp ~/path/to/PR2_INSTRUCTIONS.md docs/prds/
cp ~/path/to/PR2_SUPPLEMENTAL_DETAILS.md docs/prds/
```

### 3. Launch CodeLayer
```bash
codelayer
# or open from Finder/Spotlight
```

---

## Run PR1 Research

Copy-paste into CodeLayer:

```
/research_codebase

Research this codebase to answer these questions for our Vercel AI SDK migration:

1. Where is `@anthropic-ai/sdk` imported and how is it used?
2. Where is the main chat UI component and how is state managed?
3. What tools/function calling exists for the LLM today?
4. Where are API routes and how do they interact with Go?
5. What tests exist and what frameworks are used?
6. What env vars are required (especially ANTHROPIC_API_KEY)?

Context: Migrating to Vercel AI SDK with OpenRouter for multi-provider support.

Reference: Read docs/prds/PR1_*.md for full specs.

Output: Create docs/research/PR1_CODEBASE_ANALYSIS.md with findings and file:line references.
```

---

## Run PR2 Gap Analysis Research

Copy-paste into CodeLayer:

```
/research_codebase

Answer the gap analysis questions in docs/prds/PR2_GAPS_ANALYSIS.md:

Critical:
1. How does Next.js communicate with Go backend? (ports, proxy, env vars)
2. Is there a "current chart" context mechanism?
3. What tools exist today and where are they defined?

Medium:
4. Where are system prompts defined?
5. How do tool results flow back to the LLM?
6. What test charts exist in test_chart/ or testdata/?

Context: Adding a validateChart tool that calls Go backend for helm lint/template/kube-score.

Reference: Read all docs/prds/PR2_*.md files.

Output: 
- Update docs/prds/PR2_GAPS_ANALYSIS.md with ✅/⚠️/❌ status per gap
- Create docs/research/PR2_CODEBASE_FINDINGS.md with architecture details
```

---

## File Reference

| Your File | Purpose | When Referenced |
|-----------|---------|-----------------|
| PR1_Product_PRD.md | What to build for SDK migration | PR1 research |
| PR1_Tech_PRD.md | How to build SDK migration | PR1 research |
| PR1_INSTRUCTIONS.md | Step-by-step implementation | PR1 research |
| PR2_Product_PRD.md | What to build for validation | PR2 research |
| PR2_Tech_PRD.md | How to build validation | PR2 research |
| PR2_GAPS_ANALYSIS.md | Questions to answer | PR2 research |
| PR2_INSTRUCTIONS.md | Step-by-step implementation | PR2 research |
| PR2_SUPPLEMENTAL_DETAILS.md | Component pseudocode | PR2 research |
