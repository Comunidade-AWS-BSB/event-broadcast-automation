# Evolution WhatsApp Lambda

The `start-broadcast` Lambda is the principal delivery function. It pulls the `EventBroadcast` and its event, lists Cognito users that have a phone number, and loops over EvolutionAPI `/message/sendText/{instance}` to send the template. Each attempt is logged via the `OutboundMessage` model so the team can trace per-phone outcomes.

```ts
const users = await idp.send(new ListUsersCommand({ UserPoolId: env.AMPLIFY_AUTH_USERPOOL_ID }))
const recipients = (users.Users ?? [])
  .map(u => u.Attributes?.find(a => a.Name === 'phone_number')?.Value)
  .filter(Boolean) as string[]

await client.models.EventBroadcast.update({ id: broadcastId, status: 'running' })

const text = broadcast.templateBody
let created = 0

for (const phone of recipients) {
  try {
    const res = await fetch(`${baseUrl}/message/sendText/${instance}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'apiKey': apiKey },
      body: JSON.stringify({ number: phone, text })
    })
    const json = await res.json().catch(() => ({}))

    const providerMessageId = json?.id ?? json?.messageId
    const providerStatus: 'PENDING' | 'SENT' | 'RECEIVED' = res.ok ? 'SENT' : 'PENDING'

    await client.models.OutboundMessage.create({
      broadcastId,
      phone,
      providerMessageId,
      providerStatus,
      attempts: 1,
      lastUpdateIso: new Date().toISOString(),
    })
    created++
  } catch (err) {
    await client.models.OutboundMessage.create({
      broadcastId,
      phone,
      providerStatus: 'PENDING',
      attempts: 1,
      error: String((err as Record<string, unknown>)?.message ?? err),
      lastUpdateIso: new Date().toISOString(),
    })
  }
}

await client.models.EventBroadcast.update({
  id: broadcastId,
  status: created > 0 ? 'done' : 'failed',
})
```

## Environment details
- `EVOLUTION_BASE_URL`: Evolution API gateway base.
- `EVOLUTION_INSTANCE`: injected instance identifier (used in `/message/sendText/{instance}`).
- `EVOLUTION_API_KEY`: Amplify secret added to `apiKey` header.
- `TZ_DEFAULT`: `America/Sao_Paulo` (used when scheduling jobs).
- `AMPlIFY_AUTH_USERPOOL_ID`: for the Cognito user listing.
