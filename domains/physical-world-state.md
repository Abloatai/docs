# Authoritative physical-world state

## The problem

A database update can be rolled back. A robot has already moved, a door has already unlocked,
and a lamp is either emitting light or it is not. Every assertion about the physical world is a
claim by some observer at some time, with uncertainty attached, and a coordination layer that
stores it in one mutable boolean has thrown away the part that mattered.

## The four states

This is the model, and it is the single reason the rest of this file exists. For any physical
thing, keep these distinct:

1. **Desired**: what an authorised actor wants to become true
2. **Command**: what instruction was durably issued to an actuator
3. **Acknowledged**: what the device or controller reports it accepted or completed
4. **Observed**: what sensors or independent observers report is currently true

`desired = on` is not `observed = emitting_light`. `command = move` is not a safe completed
motion. `requested` is not `settled`. Collapsing any pair of these is how systems come to
believe a door is locked because someone asked for it to be.

Each assertion also needs metadata: observer identity, observation time and ingestion time,
uncertainty or confidence, validity interval, causal relation to a command or prior observation,
the authorisation chain behind it, any supersession or contradiction relation, and its
acknowledgement or settlement status.

## The command lifecycle

A first-class actuation transaction carries capability, command identity, fence, acknowledgement
and reconciliation state. That gives an auditable control history under retry and disconnection
without claiming the cloud is a real-time control loop.

```text
proposed -> authorised -> issued -> accepted -> completed
                    \-> rejected
                    \-> expired
                    \-> superseded
                    \-> uncertain -> reconciled
```

The command record is never overwritten by the observation. The observation refers to the
command and states what evidence exists for it. Safety-critical local control stays local; Ablo
coordinates authority and accountable state, not motor loops.

## What the field knows

Device ecosystems already carry several distinct meanings of "reliable", and composing them
requires keeping the boundaries explicit:

- **MQTT** distinguishes at-most-once, at-least-once and protocol-level exactly-once delivery
  between a sender and a receiver, with retained messages, session state and defined
  retransmission behaviour.
- **DDS and ROS 2** expose reliability, durability, history depth, deadline, lifespan and
  liveliness as separate policies, which is a far richer vocabulary than most cloud systems
  offer and worth borrowing from.
- a device controller may run its own hard-real-time safety loop that answers to nobody upstream
- an application maintains desired state and retries commands
- an organisational system governs which actor was allowed to issue the command in the first
  place

A QoS acknowledgement is not an actuator completion. An actuator completion is not an
independent observation. A sensor reading is not necessarily fresh. Ablo's proposed role sits
above all of these transports: transaction identity, authority, settlement and reconciliation
across them.

## The relationship to convergence

CRDTs and local-first systems solve a neighbouring problem well and a different one. Two
replicas can converge on the same desired value while the physical device stays unreachable, and
a sensor can report a fact no authorised actor requested. Convergence is a property of replicas.
Truth is a property of the world. A world-state layer needs a representation for both, and needs
them to be different columns.

## Questions this raises

- Which of the four states does Ablo represent today, and which are conflated in the current
  schema?
- What is the authoritative observation for a given fact when two sensors disagree, and who
  decides?
- How long is an observation valid? Without a validity interval, a stale reading is
  indistinguishable from a current one.
- What happens to a lease when the holder is disconnected but still physically able to act? The
  fence stops it from being *authorised*, but what stops it from *acting*?
- Which side effects are compensable, and which are not? The design differs sharply between them.
- If the cloud is unreachable, what is the device permitted to do, and who wrote that policy
  down?
- Is there a real customer for this today, or is it a direction? The answer changes what should
  be built now versus modelled now.
