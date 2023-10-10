# sidekiq-tool

```
Usage: sidekiq-tool [options]
    -u URL                           Redis URL (default: from REDIS_URL env var)
    -q, --queue QUEUE                apply next commands to specified queue
        --jid JID                    (alias for --job-id)
        --job-id JID                 (can be used multiple times)
        --job-class CLASS            (can be used multiple times)

    -l, --list                       list queues (default)
    -s, --show [RANGE]               show contents of queue
                                     see https://redis.io/commands/lrange/
    -P, --processes                  show processes (respects queue parameter)
    -R, --retries                    show retries (respects queue parameter)
    -S, --scheduled                  show scheduled jobs (respects queue parameter)
    -r, --running-jobs               show currently running jobs (respects queue/jid/job-class)

        --import-jobs                add jobs from STDIN into queue
        --move-jobs [N]              atomically move jobs to another queue
    -Q, --dst-queue QUEUE            destination queue

Destructive commands: (require confirmations)
        --delete-jobs [N]            N limits number of jobs to delete, 0 (default) = delete all
                                     respects --job-id and --job-class parameters
        --export-jobs [N]            same as delete, but job data is written to STDOUT beforehead
        --delete-queue               deletes ALL jobs from queue

        --confirm-delete-jobs        jobs will not be deleted without this option
        --confirm-export-jobs
        --confirm-queue-delete       queue will not be deleted without this option

    -W, --omit-weight                Omit weight from schedule/retries output (easier to parse)
    -v, --[no-]verbose               Run verbosely
    -k                               Bypass SSL verification (for debug/dev)

```