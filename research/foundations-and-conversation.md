# Classical foundations, and the conversation wave

Both of these bodies of work supply vocabulary that the newer shared-state research assumes, and
both answer questions that would otherwise get rediscovered from scratch. They operate one layer
above Ablo's problem: they decide what actors intend and say, where Ablo decides what actually
commits.

## Classical foundations: commitment, plans, delegation

Modern LLM-agent work often rediscovers questions that multi-agent systems research studied
without language models.

| Work | Coordination model | Why it still matters |
| --- | --- | --- |
| [The Contract Net Protocol](https://reidgsmith.com/The_Contract_Net_Protocol_Dec-1980.pdf) (Smith, 1980) | managers announce tasks, contractors bid, the manager awards work | a foundation for dynamic delegation and capability-based allocation. It coordinates assignment, not concurrent mutation |
| [Teamwork](https://www.sri.com/publication/teamwork/) (Cohen and Levesque, 1991) | a team is held together by joint persistent goals, plus an obligation to communicate when a goal is achieved, impossible, or no longer relevant | collaboration needs explicit commitments and termination conditions, not several actors with similar prompts |
| [Collaborative Plans for Complex Group Action](https://u.cs.biu.ac.il/~sarit/data/articles/20.pdf) (Grosz and Kraus, 1996) | SharedPlans represent partial plans, partial knowledge, joint action and subcontracting | the right frame for plans that are still incomplete while work is already underway |
| [Towards Flexible Teamwork](https://arxiv.org/abs/cs/9709101) (Tambe, 1997) | STEAM operationalises joint intentions while communicating selectively under changing and inconsistent views | directly about the cost of keeping several actors aligned without broadcasting everything |

These formalise goals, beliefs, plans, roles and messages. They do not provide isolation,
durable authority, replay or settlement. They do expose a recurring requirement: a collaboration
protocol has to say what actors are jointly committed to, what information must be shared, and
what event ends or revises that commitment.

## The conversation wave

The first large wave of LLM multi-agent systems treated structured conversation as the
coordination substrate:

- [AutoGen](https://arxiv.org/abs/2308.08155) represents applications as conversations among
  configurable agents, tools and humans. Its contribution is architectural composability, not
  concurrency control over a shared mutable world.
- [CAMEL](https://proceedings.neurips.cc/paper/2023/hash/a3621ee907def47c1b952ade25c67698-Abstract-Conference.html)
  studies role-playing between communicative agents, and how roles and inception prompts sustain
  cooperation.
- [MetaGPT](https://proceedings.iclr.cc/paper_files/paper/2024/file/6507b115562bb0a305f1958ccc87355a-Paper-Conference.pdf)
  encodes software-development standard operating procedures into specialised roles, and makes
  intermediate artifacts part of the coordination.
- [ChatDev](https://aclanthology.org/2024.acl-long.810/) models a software organisation as
  specialised agents on a chat chain, using communication to refine and validate output.
- [Exploring Collaboration Mechanisms for LLM Agents](https://aclanthology.org/2024.acl-long.782/)
  evaluates collaboration through a social-psychology lens, and is a useful warning that more
  discussion rounds do not reliably produce better collaboration.

This literature asks whether more model calls, roles or perspectives improve a cognitive result.
Ablo's question sits lower in the stack: once an actor has decided to act, how do humans,
agents, services and devices share consequential state with explicit authority and recoverable
ordering. A conversation transcript can explain an intention. It is not a commit protocol.

## Debate and aggregation

Multi-agent debate and repeated sampling coordinate beliefs or candidate answers, usually
without any durable shared world.

- [Improving Factuality and Reasoning through Multiagent Debate](https://composable-models.github.io/llm_debate/)
  reports gains from agents exchanging arguments over several rounds.
- [Should We Be Going MAD?](https://proceedings.mlr.press/v235/smit24a.html) finds that debate
  does not uniformly improve outcomes, and analyses when simpler baselines stay competitive.
- [More Agents Is All You Need](https://arxiv.org/abs/2402.05120) scales inference through
  independent agents and aggregation, which is closer to sampling and voting than to
  shared-state collaboration.
- [SOTOPIA](https://proceedings.iclr.cc/paper_files/paper/2024/file/b3075b88e583a0e98d8b24338a613060-Paper-Conference.pdf)
  evaluates language agents in interactive social scenarios, providing evidence about
  negotiation rather than transactional consistency.

The practical lesson is that agent count is not a coordination architecture. Any evaluation of
an Ablo-backed system has to separate the benefit caused by additional inference compute from
the benefit caused by the state, authority and concurrency protocol. The measurement discipline
for that is in [evaluation-and-failure.md](evaluation-and-failure.md).
