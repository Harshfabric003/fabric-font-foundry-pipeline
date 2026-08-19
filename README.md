## Orchestration

The three medallion-layer notebooks (Bronze → Silver → Gold) are chained into a 
single Fabric Data Pipeline (`font_foundry_pipeline`), with sequential 
"on success" dependencies — each stage only runs if the previous one completes 
without error.

- **Bronze** (`01_bronze_ingest`) → **Silver** (`02_silver_clean_match`) → **Gold** (`03_gold_aggregate`)
- Configured with a daily schedule via Fabric's built-in Schedule trigger 
  (disabled by default in this portfolio project to avoid unnecessary compute usage)

<img width="1302" height="924" alt="image" src="https://github.com/user-attachments/assets/1e62a223-f101-436e-bbf1-83654006e914" />
<img width="884" height="833" alt="Screenshot 2026-08-19 at 14 03 54" src="https://github.com/user-attachments/assets/d1473182-b223-47b6-93a0-727bdc679c32" />
