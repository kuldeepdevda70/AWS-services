# Kubernetes Job

A **Job** is a Kubernetes workload that runs a task **one or more times** and ensures it completes successfully. Once the task finishes, the Job stops and is **not restarted** unless it fails or is configured to retry.

### Use Cases
- Database backup
- Data migration
- Batch processing
- Sending emails
- Running scripts

---

# Kubernetes CronJob

A **CronJob** is a Kubernetes resource that creates **Jobs on a scheduled basis** using a cron expression. It is used to automate recurring tasks such as backups, cleanup, or report generation.

### Use Cases
- Daily database backups
- Log cleanup
- Scheduled report generation
- Periodic data synchronization
- Health checks

---

## Difference Between Job and CronJob

| Job | CronJob |
|------|----------|
| Runs a task once. | Runs tasks repeatedly based on a schedule. |
| Executes immediately after creation. | Executes according to a cron schedule. |
| Used for one-time tasks. | Used for recurring or scheduled tasks. |
| Example: Database migration. | Example: Daily database backup at midnight. |
