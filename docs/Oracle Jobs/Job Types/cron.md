---
layout: nodes.liquid
date: Last Modified
title: "Cron"
permalink: "docs/jobs/job-types/cron/"
---

Executes a job on a schedule. Does not rely on any kind of external trigger.

**Spec format**

```toml
type            = "cron"
schemaVersion   = 1
schedule        = "0 0 0 1 1 *"
externalJobID       = "0EEC7E1D-D0D2-476C-A1A8-72DFB6633F01"
observationSource   = """
    ds          [type=http method=GET url="https://chain.link/ETH-USD"]
    ds_parse    [type=jsonparse path="data,price"]
    ds_multiply [type=multiply times=100]

    ds -> ds_parse -> ds_multiply
"""
```

**Shared fields**
See [shared fields](/docs/jobs/#shared-fields).

**Unique fields**

- `schedule`: the frequency with which the job is to be run, specified in traditional UNIX cron format (but with 6 fields, not 5. The extra field allows for "seconds" granularity). You can also use e.g. `@every 1h` if you just want something executed on a regular interval, but in this format, note that the wall clock time is not taken into account. It simply begins counting the moment that the job is added to the node (or the moment that the node starts, if it has already been added).

For all supported schedules, please refer to the [cron library documentation](https://pkg.go.dev/github.com/robfig/cron?utm_source=godoc).

**Job type specific pipeline variables**

- `$(jobSpec.databaseID)`: the ID of the job spec in the local database. You shouldn't need this in 99% of cases.
- `$(jobSpec.externalJobID)`: the globally-unique job ID for this job. Used to coordinate between node operators in certain cases.
- `$(jobSpec.name)`: the local name of the job.
- `$(jobRun.meta)`: a map of metadata that can be sent to a bridge, etc.
