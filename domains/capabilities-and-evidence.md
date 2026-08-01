# Credentials and authority

Ablo's credential model is the Stripe key and token shape. The sync-server is a **data plane,
not an auth authority**: it verifies no external end-user tokens, and there is no JWT, no JWKS
and no trusted-issuer path. Every consumer, including Ablo's own web app, presents a bearer key
or a backend-minted session token.

## The key kinds

| Kind | Prefix | Held by | Authority |
| --- | --- | --- | --- |
| secret | `sk_` | the customer's backend | full org |
| restricted | `rk_` | agents, and minted agent session tokens | scoped: sync groups, operations, TTL |
| ephemeral | `ek_` | minted user session tokens, browser | typed operation grant inside the user's org, TTL |
| publishable | `pk_` | public browser reads | read-only, server-enforced scope |

An agent gets an `rk_`, never an `sk_`. A browser that writes gets a minted `ek_`.

## Minting is a backend call

```ts
// customer backend, where sk_ lives
const session = await ablo.sessions.create({
  user: { id: userId },
  can: { tasks: ['read', 'update'] },
  ttlSeconds: 900,
});

const agent = await ablo.agents.create({
  name: 'task-writer',
  can: { tasks: ['read', 'update'] },
});
```

`can` is typed off the schema's model names and serialised to a concrete wire allowlist, so
`{ Task: ['update'] }` becomes `task.update`, enforced per commit against
`${model.toLowerCase()}.${op}`. User sessions mint through `POST /v1/ephemeral_keys`, agent
sessions through `POST /v1/capabilities`, both authenticated by the `sk_`.

## Two properties worth noticing

**Scope stays live.** A key carries baked base sync groups (`org:`, `user:`, `team:`), and at
connect the server unions them with relation-driven membership resolved server-side. Scope is
therefore not frozen at mint time.

**Revocation is instant without server-side session state.** Session tokens are short-lived, 15
minutes and auto-refreshed, sitting under a long-lived revocable upstream session that the
customer's platform owns. Revoke upstream and the next refresh fails. The sync-server holds no
session state to invalidate.

## The credential is half of it

| Question | Answered by |
| --- | --- |
| May this actor act? | the key's scope and operation allowlist |
| Are two authorised actors both acting? | the claim |
| Is this holder stale? | the fence |
| Is this a retry or a second action? | the idempotency key |
| What happened? | the receipt |
| Bounded by what? | the org, sync groups, and the scoped DML role in the customer's database |

Definitions in [02-the-contract.md](../02-the-contract.md#vocabulary). A token that validates
says nothing about whether another actor already owns the row, which is why authority and
coordination are separate primitives here.

## In the code

| | |
| --- | --- |
| Key kinds | [auth/credentialKind.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/credentialKind.ts), [auth/apiKey.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/apiKey.ts) |
| Minting | [auth/sessionMint.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/sessionMint.ts) |
| Scope and lifecycle | [auth/capability.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/capability.ts), [auth/capabilityLifecycle.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/capabilityLifecycle.ts) |
| Identity | [auth/identity.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/identity.ts) |
| Browser limits | [auth/browserCredentialSafety.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/browserCredentialSafety.ts) |

The verification path itself runs in the engine, which is not public.

## See it yourself

Mint a read-only session and try to write with it. The refusal happens on the server against the
allowlist baked into the key, not in the client:

```ts
const session = await ablo.sessions.create({
  user: { id: 'u_1' },
  can: { orders: ['read'] },
  ttlSeconds: 900,
});

const scoped = Ablo({ schema, apiKey: session.token });

await scoped.orders.get({ id });                 // allowed
await scoped.orders.update({ id, data: { status: 'approved' } });  // refused
```

Widen `can` to `['read', 'update']`, mint again, and the same call succeeds. Nothing about the
client changed.

## Still open

- Revocation latency for an agent that is already mid-operation when its authority is pulled.
  The refresh boundary bounds it, and the bound has not been measured.
- What a signed receipt should commit to for a counterparty: the request, the resulting delta,
  or the database LSN. Only the last ties a signature to settlement, and none of it is built.
- Delegation chains, where an agent spawns a sub-agent. Scope narrows correctly by construction;
  attribution across the chain is less clear.
