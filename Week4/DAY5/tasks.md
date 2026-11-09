# Week 4, Day 5 - Presentation Day
### Tasks

---

**Presentation: 3:00 PM today.**

**Goal for today:** a finished README, your Week 3 presentation extended with what you added this week, and a smooth demo.

**Time:** about 4 hours of preparation. Aim to be ready by 2:30 PM.

There is no new building today. Do not change the pipeline.

---

## Morning

### Task 1: Write the README (45 minutes)
Keep it to one page. Put it in the repo as `README.md`.

1. What the project does, in two sentences
2. The architecture picture (`docs/architecture.png`)
3. How to run it: the `load_date` parameter and how to start the pipeline
4. What was added in Week 4: incremental load, retries and checks, alerts, Key Vault and managed identity, SQL on gold
5. Results: the timing table and the cost per run from Day 4
6. How to clean up: delete the resource group

### Task 2: Update the architecture picture (15 minutes)
Open your Week 3 sketch and add what is new: the `load_date` parameter, Azure Monitor alert, and Git. Export it again as `architecture.png`.

### Task 3: Extend your Week 3 presentation (45 minutes)
Keep your Week 3 slides. Add a few slides after them, one per topic:

| Slide | Say |
|---|---|
| What changed | Week 3 pipeline versus Week 4 pipeline, one sentence each |
| Incremental load | The `load_date` parameter, and the same row count after running twice |
| Reliability | Retries, checks, and the failed run with the alert email screenshot |
| Security and code | Key Vault, managed identity, GitHub repo |
| Results | A query result, the OPTIMIZE timing, and the cost per run |

Use the screenshots you saved on Days 1 to 4. Keep text short.

---

## Before Lunch

### Task 4: Prepare the demo (45 minutes)
1. Start your cluster and run the pipeline once for a normal date, so everything is warm. Do not leave the first run for the live demo.
2. Open these tabs and leave them open: Data Factory Monitor, the silver table count query, the GitHub repo, your slides.
3. Save backup screenshots of a successful run, the failed run with its alert, and the repeated-run row counts. If something breaks during the demo, show the screenshots.
4. Check that the cluster will not run out of auto-terminate time before 3 PM. Restart it at 2:30 if needed.

---

## Afternoon

### Task 5: Rehearse (30 minutes)
Present it out loud twice, with a timer. Practice the demo part once on screen. Note the two or three points you want to remember.

### Task 6: Be ready (by 2:30 PM)
- Slides open
- Demo tabs open
- Cluster running
- Water, charger, and internet checked

---

## 3:00 PM - Present

---

## After the Presentation

### Task 7: Clean up
Do this only after the presentation, and only once the repo has everything you want to keep.

1. Confirm GitHub has the notebooks, Data Factory definitions, SQL file, README and diagram, and no secrets.
2. Delete the resource group:

```powershell
az group delete -n rg-bigdata-dev --yes --no-wait
```

3. Check it later with `az group exists -n rg-bigdata-dev`. It should say `false`.
4. Revoke your Kaggle token from your Kaggle account settings.

---

## End-of-Day Checklist

- [ ] README is in the repo
- [ ] Architecture picture is updated
- [ ] Week 3 presentation extended with the Week 4 slides
- [ ] I ran the demo once before presenting
- [ ] I have backup screenshots
- [ ] I rehearsed twice
- [ ] After presenting: the resource group is deleted and the Kaggle token revoked