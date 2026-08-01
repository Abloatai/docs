# Physical-world state

An agent about to act on a physical thing needs four answers, and a database gives it one:

| The agent asks | Answered by |
| --- | --- |
| What is true about this object right now? | an observation, with a time and a source |
| How stale is that, and may I act on it? | a validity interval, not a timestamp alone |
| Am I allowed to change it, and does anyone else own it? | authority and a claim |
| Did my last command actually land? | an acknowledgement, then an independent observation |

This is the same settlement problem as a row write, with one asymmetry that changes everything: a
database update rolls back, and a robot has already moved. The door is open, the arm has swung,
the shipment is on the truck.

## The four states

| State | Meaning | Owner |
| --- | --- | --- |
| **Desired** | what an authorised actor wants to become true | the application |
| **Command** | what instruction was durably issued to an actuator | the coordination layer |
| **Acknowledged** | what the device reports it accepted or completed | the device |
| **Observed** | what sensors or independent observers report is true now | the observer |

`desired = on` is not `observed = emitting_light`. `command = move` is not a completed safe
motion. Collapsing any pair is how a system comes to believe a door is locked because somebody
asked for it to be.

Every assertion carries observer identity, observation time and ingestion time, uncertainty,
validity interval, causal relation to a command or prior observation, the authorisation chain,
supersession or contradiction, and settlement status.

## Freshness is part of the value

A physical reading without an expiry is indistinguishable from a current one, which is why the
robotics standards publish state on a schedule as well as on change. VDA 5050 vehicles emit a
state message when something meaningful changes **and** on a heartbeat, so the controller always
has a bound on how stale its picture is
([spec](https://github.com/VDA5050/VDA5050/blob/main/VDA5050_EN.md)). The heartbeat is not
redundancy, it is the freshness guarantee: without it, silence and no-change are the same
message.

For an agent that reasons for thirty seconds before acting, this is the whole game. The premise
it read may have expired mid-inference, which is the physical version of the stale-generation
anomaly in [shared-state-concurrency](../research/shared-state-concurrency.md).

## What already exists

| Layer | What it standardises | What it leaves open |
| --- | --- | --- |
| [MQTT 5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html) | transport, three QoS levels, retained messages, sessions | whether the message corresponds to a physical fact |
| [ROS 2 / DDS](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html) | reliability, durability, history, deadline, lifespan, liveliness, per topic | authority. Any node on the graph can publish |
| [VDA 5050](https://github.com/VDA5050/VDA5050) | AGV and AMR fleets: `order`, `state`, `instantActions`, `connection`, `factsheet`, over MQTT with JSON | one fleet controller's view. Not cross-vendor authority, not an audit record |
| [W3C WoT Thing Description](https://w3c.github.io/wot-thing-description/) | a Thing's properties, actions and events, with protocol bindings | who may invoke an action, and what happened when two actors did |
| [Eclipse Ditto](https://eclipse.dev/ditto/intro-overview.html) | digital twins: a Thing with features carrying state properties, telemetry and commands | settlement. A twin can be confidently wrong |
| [OPC UA](https://eclipse.dev/ditto/basic-wot-integration.html) | industrial device information models | the same gaps, in a factory |

The pattern matches the software neighbours in [01](../01-the-space.md#who-else-is-in-this-space-and-where-they-stop):
the transports move bytes with real delivery semantics, the twin frameworks model what a Thing
has, and none of them decides who was allowed to change it or what to believe when two sources
disagree.

## The command lifecycle

```text
proposed -> authorised -> issued -> accepted -> completed
                    \-> rejected
                    \-> expired
                    \-> superseded
                    \-> uncertain -> reconciled
```

The command record is never overwritten by the observation. The observation refers to the
command and states what evidence exists for it. `uncertain` is the state most systems lack and
most need: the command went out, the acknowledgement never came back, and the truth is
recoverable only by looking.

Safety-critical control stays local. A coordination layer that puts itself inside a motor loop
has failed at its own job; what it owns is authority, the command record, and the reconciliation
afterwards.

## Convergence is not truth

CRDTs converge replicas ([Shapiro et al.](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F21/handouts/papers/Shapiro-CRDT.pdf)).
Two replicas can agree perfectly on a desired value while the device is unreachable, and a sensor
can report a fact no actor requested. Convergence is a property of replicas, truth is a property
of the world, and a system that needs both needs two columns.

## See it yourself

The gap between delivery and completion takes two terminals:

```sh
mosquitto_sub -t 'lamp/+/state' -q 2 -v
mosquitto_pub -t 'lamp/1/set' -q 2 -m 'on'
```

The publish returns once the broker completes the QoS 2 handshake, which is the strongest
delivery guarantee MQTT offers. Nothing in that acknowledgement says a lamp is lit. Unplug the
lamp and repeat: the protocol still succeeds.

Then add the missing half. Have the subscriber echo a state message with a timestamp on a two
second heartbeat, stop it, and watch how long your system keeps believing the last value. That
interval is your unmeasured staleness, and it is the number an agent needs before it acts.

## Go deeper

| Read | For |
| --- | --- |
| [VDA 5050](https://github.com/VDA5050/VDA5050) | a shipped, manufacturer-independent state and order interface for robot fleets. Read the state message first |
| [W3C WoT Thing Description](https://w3c.github.io/wot-thing-description/) | the closest thing to a schema for a real-world object's state |
| [Eclipse Ditto](https://eclipse.dev/ditto/intro-overview.html) | digital twins as running software, including the WoT Thing Model integration |
| [ROS 2 QoS](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html) | six independent delivery policies, a richer vocabulary than most cloud systems have |
| [Embodied AI Agents: Modeling the World](https://arxiv.org/abs/2506.22355) | why world models sit at the centre of embodied agent reasoning, from a large multi-institution group |
| [Atomix](https://arxiv.org/abs/2602.14849) | irreversible effects treated as their own class, in the agent-tooling context |

## Still open

- Which of the four states Ablo represents today, and which are conflated.
- What the authoritative observation is when two sensors disagree, and who decides.
- How a validity interval is expressed and enforced, so an agent cannot act on an expired premise.
- What stops a disconnected lease holder from acting, given a fence stops it from being
  authorised rather than from moving.
- Whether the unit of coordination for a physical thing is the object, the zone, or the task.
  Fleet standards answer zone, and a general layer has to answer all three.
