# Event Broadcast Automation Samples

This repo collects the key pieces that power the WhatsApp automation used by the Comunidade-AWS-BSB event platform. Each markdown file captures the Amplify schema or Lambda wiring so the automation can be explained quickly.

## Files
- `schema-scheduler.md` – the `EventBroadcast` model plus the `scheduleBroadcast` resolver that enqueues an EventBridge Scheduler job.
- `start-broadcast.md` – the Lambda that is triggered by the scheduler, plus its Evolution API environment and delivery flow.
