# Authoritative physical-world state

A database update rolls back. A robot has already moved, a door has already unlocked, a lamp is
either emitting light or it is not.

## The four states

| State | Meaning | Owner |
| --- | --- | --- |
| **Desired** | what an authorised actor wants to become true | the application |
| **Command** | what instruction was durably issued to an actuator | the coordination layer |
| **Acknowledged** | what the device reports it accepted or completed | the device |
| **Observed** | what sensors or independent observers report is true now | the observer |

`desired = on` is not `observed = emitting_light`. `command = move` is not a completed safe
motion. `requested` is not `settled`. Collapsing any pair is how a system comes to believe a
door is locked because somebody asked for it to be.

Every assertion also carries: observer identity, observation time and ingestion time,
uncertainty, validity interval, causal relation to a command or prior observation, the
authorisation chain, supersession or contradiction, and settlement status.

## The command lifecycle

```text
proposed -> authorised -> issued -> accepted -> completed
                    \-> rejected
                    \-> expired
                    \-> superseded
                    \-> uncertain -> reconciled
```

The command record is never overwritten by the observation. The observation refers to the
command and states what evidence exists. Safety-critical control stays local; the coordination
layer owns authority and accountable state, not motor loops.

## Reliability already means several things

| Layer | Vocabulary | Proves |
| --- | --- | --- |
| MQTT | at-most-once, at-least-once, exactly-once delivery; retained messages; sessions ([spec](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)) | a message reached a receiver |
| DDS and ROS 2 | reliability, durability, history, deadline, lifespan, liveliness as separate policies ([QoS](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html)) | delivery expectations, per topic |
| Device controller | its own hard-real-time safety loop | that the machine stayed safe |
| Application | desired state and command retry | that intent persisted |
| Organisation | who was allowed to issue the command | authority |

A QoS acknowledgement is not an actuator completion. An actuator completion is not an
independent observation. A sensor reading is not necessarily fresh.

## Convergence is not truth

CRDTs converge replicas ([Shapiro et al.](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F21/handouts/papers/Shapiro-CRDT.pdf)).
Two replicas can agree on a desired value while the device is unreachable, and a sensor can
report a fact no actor requested. Convergence is a property of replicas; truth is a property of
the world. Both need a representation, and they need different columns.

## See it yourself

MQTT makes the gap between delivery and completion tangible in two terminals:

```sh
mosquitto_sub -t 'lamp/+/state' -q 2 -v
mosquitto_pub -t 'lamp/1/set' -q 2 -m 'on'
```

The publish returns once the broker completes the QoS 2 handshake. Nothing in that
acknowledgement says a lamp is lit. Unplug the lamp and repeat: the protocol still succeeds.

## Go deeper

| Read | For |
| --- | --- |
| [ROS 2 QoS](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html) | six independent delivery policies, a richer vocabulary than most cloud systems have |
| [MQTT 5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html) | what exactly-once means between exactly two parties, and no further |
| [Local-First Software](https://www.inkandswitch.com/essay/local-first/) | ownership and offline operation as product properties |
| [Atomix](https://arxiv.org/abs/2602.14849) | irreversible effects treated as a separate class from reversible ones |

## Still open

- Which of the four states Ablo represents today, and which are conflated.
- What the authoritative observation is when two sensors disagree, and who decides.
- How long an observation stays valid, since without an interval a stale reading and a current
  one are indistinguishable.
- What stops a disconnected lease holder from acting, given the fence only stops it from being
  authorised.
- Whether there is a customer for this today, or a direction. The answer changes what to build
  against what to model.
