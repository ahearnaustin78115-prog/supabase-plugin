---
title: Log pg_cron Runs in an Application-Owned History Table
impact: MEDIUM-HIGH
impactDescription: Preserve durable success and failure history beyond scheduler diagnostics
tags: pg-cron, job-logging, monitoring, observability, error-handling
---

## Log Scheduled Jobs Through a Small Wrapper

`cron.job_run_details` is useful for inspecting scheduler activity, but an
application-owned history table can preserve the business-level job name,
status, timing, and error details needed for alerting and long-term reporting.
Wrap the actual work so every run records both success and failure explicitly.

**Incorrect (schedule the work without durable application logging):**

```sql
-- The scheduler has some diagnostics, but the application has no stable
-- record for reporting, alerting, or correlating a run with its work.
select cron.schedule(
  'refresh-reporting-data',
  '0 * * * *',
  $$select reporting.refresh_data();$$
);
```

**Correct (record each run and retain scheduler diagnostics):**

```sql
create schema if not exists ops;

create table if not exists ops.cron_run_log (
  id bigint generated always as identity primary key,
  job_name text not null,
  started_at timestamptz not null default clock_timestamp(),
  finished_at timestamptz,
  status text not null check (status in ('running', 'succeeded', 'failed')),
  error_message text,
  error_sqlstate text,
  constraint cron_run_log_finished_status_check check (
    (status = 'running' and finished_at is null)
    or (status in ('succeeded', 'failed') and finished_at is not null)
  )
);

create index if not exists cron_run_log_job_started_idx
  on ops.cron_run_log (job_name, started_at desc);

create or replace function ops.run_refresh_reporting_data()
returns void
language plpgsql
as $$
declare
  v_run_id bigint;
begin
  insert into ops.cron_run_log (job_name)
  values ('refresh-reporting-data')
  returning id into v_run_id;

  begin
    perform reporting.refresh_data();

    update ops.cron_run_log
    set status = 'succeeded',
        finished_at = clock_timestamp()
    where id = v_run_id;
  exception
    when others then
      update ops.cron_run_log
      set status = 'failed',
          finished_at = clock_timestamp(),
          error_message = sqlerrm,
          error_sqlstate = sqlstate
      where id = v_run_id;

      raise;
  end;
end;
$$;

select cron.schedule(
  'refresh-reporting-data',
  '0 * * * *',
  $$select ops.run_refresh_reporting_data();$$
);
```

The `raise` preserves pg_cron's failed-run status and return message while the
wrapper preserves a durable application-level record. Do not swallow the
exception: a job that appears successful to the scheduler cannot be alerted
on reliably.

Query the two sources together when diagnosing a run:

```sql
-- Application-owned history, including the business job name and SQLSTATE
select *
from ops.cron_run_log
where job_name = 'refresh-reporting-data'
order by started_at desc
limit 20;

-- Scheduler-owned execution details
select
  jobid,
  runid,
  status,
  return_message,
  start_time,
  end_time
from cron.job_run_details
where command like '%ops.run_refresh_reporting_data%'
order by start_time desc
limit 20;
```

Keep the history table bounded. Error messages may contain sensitive values,
so restrict access to the operations role and retain only the period required
for troubleshooting:

```sql
revoke all on ops.cron_run_log from public;

-- Run periodically as an operations-only maintenance job.
delete from ops.cron_run_log
where started_at < clock_timestamp() - interval '90 days';
```

The wrapper adds one insert and one update per run, so it is best suited to
jobs where operational visibility matters. Keep the work function itself
idempotent: operators may retry failed jobs, and scheduled executions can
overlap. Logging does not provide a distributed lock. Use an advisory lock or
another job-specific coordination mechanism when overlapping runs are unsafe.

References:

- [Supabase: pg_cron](https://supabase.com/docs/guides/database/extensions/pg_cron)
- [pg_cron: Job run details](https://github.com/citusdata/pg_cron#viewing-job-run-details)
- [PostgreSQL: PL/pgSQL error trapping](https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html)
