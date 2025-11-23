# EventBroadcast Schema and Scheduler Resolver

The automation is anchored on an `EventBroadcast` record that stores the WhatsApp text, schedule kind, and its emitted `OutboundMessage` log. Only the `ADMINS` group can mutate or read these records because they drive the automation.

```ts
const EventBroadcast = a.model({
  eventId: a.id().required(),
  event: a.belongsTo('Event', 'eventId'),

  templateBody: a.string().required(),

  scheduleKind: a.enum(['NOW', 'AT', 'CRON']),
  scheduledAtIso: a.string(),
  cron: a.string(),

  status: BroadcastStatus,

  messages: a.hasMany('OutboundMessage', 'broadcastId')
}).authorization(allow => [
  allow.group('ADMINS').to(['create', 'read', 'update', 'delete'])
])

const OutboundMessage = a.model({
  broadcastId: a.id().required(),
  broadcast: a.belongsTo('EventBroadcast', 'broadcastId'),

  phone: a.string(),
  providerMessageId: a.string(),
  providerStatus: a.enum(['PENDING', 'SENT', 'RECEIVED']),

  attempts: a.integer(),
  error: a.string(),
  lastUpdateIso: a.string()
}).authorization(allow => [
  allow.group('ADMINS').to(['create', 'read', 'update', 'delete'])
])
```

The GraphQL schema wires two mutations that help the UI schedule and start broadcasts. `scheduleBroadcast` calculates the `ScheduleExpression` based on the `EventBroadcast` record and registers an EventBridge Scheduler job targeting the `start-broadcast` Lambda.

```ts
const schema = a.schema({
  // ...other models omitted for brevity...
  EventBroadcast,
  OutboundMessage,

  startBroadcast: a.mutation()
    .arguments({ broadcastId: a.id().required() })
    .returns(a.ref('StartResult'))
    .authorization(allow => [allow.group('ADMINS')])
    .handler(a.handler.function(startBroadcastFn)),

  scheduleBroadcast: a.mutation()
    .arguments({ broadcastId: a.id().required() })
    .returns(a.boolean())
    .authorization(allow => [allow.group('ADMINS')])
    .handler(a.handler.function(scheduleBroadcastFn))
})
```

And the Lambda handler uses `CreateScheduleCommand` to set up the cron or `at()` execution, flips the broadcast status to `scheduled`, and targets the `start-broadcast` function via the role in the scheduler environment.

```ts
const cmd = new CreateScheduleCommand({
  Name: `broadcast-${broadcastId}`,
  GroupName: 'event-broadcasts',
  FlexibleTimeWindow: { Mode: 'OFF' },
  Target: {
    Arn: env.START_BROADCAST_ARN,
    RoleArn: env.SCHEDULER_ROLE_ARN,
    Input: JSON.stringify({ arguments: { broadcastId } }),
  },
  ScheduleExpressionTimezone: env.TZ_DEFAULT,
  ScheduleExpression:
    broadcast.scheduleKind === 'AT'
      ? `at(${new Date(broadcast.scheduledAtIso!).toISOString()})`
      : broadcast.cron!,
  State: 'ENABLED',
})
```
