---
title: Control pg_cron Run-Detail Logging Per Job
impact: LOW
impactDescription: Prevents high-frequency, low-value job history from crowding run-detail records
tags: pg-cron, scheduled-jobs, logging
---

## Control pg_cron Run-Detail Logging Per Job

`cron.job_run_details` is useful for diagnosing scheduled work, but recording every run of a high-frequency job can create a large volume of low-value history. Keep logging enabled for jobs where run status and failure details matter; disable it only for high-volume jobs whose history is not needed, and retain an appropriate cleanup policy for logged rows.

**Incorrect (turning off run logging globally to quiet one job):**

```conf
# Suppresses run-detail logging for every job, including jobs that need diagnostics.
cron.log_run = off
```

**Correct (disable logging only for a job, when supported by the installed pg_cron version):**

```sql
-- The per-job log_run argument is available only in pg_cron versions that implement it.
select cron.schedule(
  'frequent_maintenance',
  '1 second',
  'select 1',
  log_run := false
);
```

With per-job logging available, the job's `log_run` setting controls whether its runs are recorded, while `cron.log_run` remains a cluster-wide override: turning the global setting off suppresses run details for all jobs. The per-job setting defaults to `true`, so existing jobs continue logging after an upgrade. Disabling logging also removes run-detail visibility for that job, including information useful for investigating failures; enable it for jobs where that history is important.

Before using the per-job argument or changing an existing job with `cron.alter_job`, verify that the deployed pg_cron version supports the feature. Updating `cron.alter_job` to add the argument can require grants on its old signature to be applied again to the new signature.

Reference: [pg_cron per-job run logging proposal](https://github.com/citusdata/pg_cron/pull/442)
