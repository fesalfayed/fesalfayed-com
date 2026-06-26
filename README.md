# fesalfayed.com

Personal site for Fesal Fayed — Data & ML engineer focused on agentic systems,
tool-use fine-tuning, and applied ML.

Static HTML + Tailwind/CDN-free inline CSS. Deployed through Cloudflare Pages.

## Public proof surfaced

- 9k+ Hugging Face downloads across the `gpt-oss-20b` Hermes-style tool-use fine-tune family.
- Fesal-authored Anthropic adapter fix landed in `NousResearch/hermes-agent`.
- Merged `hermes-lcm` context-role fix.
- Amazon image-quality + BSR modeling case study.

## Deploy

Push to `main` → Cloudflare Pages auto-builds & ships. No build step (pure static).

## Local preview

```bash
python3 -m http.server 8000
# open http://localhost:8000
```
