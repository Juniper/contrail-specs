# 1. Introduction
Contrail fabric management currently supports many types of jobs and workflows.
Some jobs are short-lived while others can take a substantial amount of time to complete.
We want to add the ability to abort long-running jobs.

# 2. Problem statement
As of today there is no way to abort fabric jobs. The user at times may want
to stop a job because they changed their minds or because the workflow encounters
long timeouts. Yet we ideally want to leave the system in a stable state after
abort. Most of the time the user will want to attempt a graceful abort.
Other times the user will want to forcefully abort if the graceful abort doesn't work.
We want to provide the ability to gracefully or forcefully abort any currently running job.

# 3. Proposed solution
The job-abort support in Contrail will include the following changes/additions:

1) Add a new API to select between graceful or forceful abort.
2) Add support for graceful abort in the backend. Abort will occur only after
completion of the current playbook.
3) Add support for forceful abort in the backend. Abort will occur immediately
by killing the task.

# 4. User workflow impact
At a minimum, the user should be able to go to the job summary page,
select a job, and have the option to abort.

Details of UI impact are still TBD.

# 5. API schema changes

#### Abort-job API
There will be a new action URL with the following input format:

```
  "input_schema": {
    "title": "Abort Job",
    "$schema": "http://json-schema.org/draft-06/schema#",
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "job_execution_id": {
        "type": "string",
        "description": "Job execution ID of job to abort"
      },
      "abort_mode": {
        "type": "string",
        "description": "Choice of 'graceful' or 'force'"
      }
    }
  }
```


# 6. Implementation

#### Abort-job API
There will be a new action URL with the following format:

POST http://{controller-ip}:8082/abort-job

with json input {'input': {...}} and the input schema defined above.

For example:

POST http://{controller-ip}:8082/abort-job -d {'input': {'job_execution_id': 'cf369d42-a1c4-11e9-be52-c4544444d663', 'abort_mode': 'force'}}

Note that the new abort-job format is similar to execute-job, except abort-job
does not require 'job_template_fq_name' since there is no job template in this case.
Only 'input' is required for abort-job.

# 7. Testing
(Insert link to test plan here)

# 8. Documentation Impact
Requires new section on aborting job

