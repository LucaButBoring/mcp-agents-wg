# The Case for MCP Server-Embedded Subagents: Overcoming Context Blindness in the Model Context Protocol

## Executive Summary

The Model Context Protocol (MCP) has become the de facto standard for connecting LLM-based agents to external tools and services, with over 16,000 community-contributed servers by late 2025. However, a fundamental architectural asymmetry exists: MCP servers are designed as **stateless, context-blind executors** that have no awareness of the agent's configuration, the model driving the session, or the contents and constraints of the context window. This report evaluates the thesis that MCP servers would be more effective if they could expose their own subagents with delegated control over a portion of the agent session's context window, analyzing both the structural limitations that motivate this idea and the emerging architectural patterns that point toward it.[^1][^2][^3]

## The Structural Blindness of MCP Servers

### Servers as Stateless Tools

The MCP specification explicitly enforces isolation: servers "should not be able to read the whole conversation, nor 'see into' other servers" and receive "only necessary contextual information". In the standard architecture, the central LLM decomposes tasks and issues instructions to servers that act as stateless executors with no access to a global context. This design choice prioritizes composability and security but creates what researchers call "inter-agent awareness" limitations — servers cannot leverage shared context across tasks.[^2][^4][^1]

In the traditional MCP flow, all coordination flows through the LLM, which must operate within a fixed context window. This creates a bottleneck: the LLM handles every intermediate result, every routing decision, and every context-management task, while the servers themselves remain oblivious to the broader workflow they participate in.[^5]

### No Awareness of Agent or Model Configuration

MCP servers receive only the parameters of a specific tool call. They have no knowledge of:

- **Which model** is driving the agent session (its capabilities, token limits, reasoning style)
- **Agent-level policies** such as tenant constraints, user preferences, or safety rules
- **The current state of the context window** — how much space remains, what other tools have been called, or what the user's conversational history contains
- **Other servers** in the session or their outputs[^6][^4]

This means a server exposing 40 tools has no mechanism to suggest which subset is relevant, no way to adapt its behavior to the model's strengths, and no ability to compress or prioritize its own outputs based on available context budget.[^7]

### The Context Window Tax

The practical cost of this blindness is severe. Tool definitions from MCP servers alone can consume 16%+ of a 200K-token context window before a single user message is processed. When intermediate results pass through the model (e.g., a Google Drive transcript being forwarded to Salesforce), the same data may flow through the context window multiple times — a 2-hour meeting transcript can add 50,000+ tokens of redundant processing. Research on multi-MCP agents found that cumulative token footprints "routinely exceed the 128K–200K boundaries of state-of-the-art LLMs".[^8][^9][^10]

An IEEE study benchmarking six optimization families across multiple datasets reported a mean token reduction of 37% and a mean accuracy gain of 23 percentage points when context-management strategies are applied, confirming that the unmanaged context consumption problem is both measurable and solvable.[^8]

## Emerging Evidence: Servers Need More Autonomy

### Context-Aware MCP (CA-MCP)

The most direct academic validation of this thesis comes from the CA-MCP framework (Jayanti & Han, January 2026). CA-MCP introduces a **Shared Context Store (SCS)** that allows MCP servers to read from and write to shared memory, coordinating autonomously rather than funneling everything through the central LLM. In experiments on the TravelPlanner and REALM-Bench benchmarks, CA-MCP reduced the number of LLM calls required for complex tasks and decreased response failures when task conditions were unmet.[^1][^2]

The key architectural shift: servers become "autonomous reactors" that monitor shared state and coordinate in real time, rather than "puppets controlled by LLM". The central LLM's role is limited to initial planning and final summarization, while servers handle intermediate coordination independently.[^2][^5]

| Architecture | Server Role | Context Mechanism | LLM Involvement |
|---|---|---|---|
| Traditional MCP | Stateless executor | None (all via LLM) | Every subtask[^5] |
| CA-MCP | Autonomous reactor | Shared Context Store | Init + summarize only[^2] |
| Code Execution MCP | Code API surface | Filesystem-based | On-demand discovery[^9] |
| Proposed Subagent MCP | Embedded subagent | Delegated context partition | Scoped delegation |

### Code Execution as Partial Solution

Anthropic's own engineering team acknowledged the problem in November 2025, proposing that agents interact with MCP servers via **code execution** rather than direct tool calls. By presenting MCP servers as code APIs on a filesystem, agents load only the tool definitions they need (progressive disclosure) and process data in the execution environment before returning results to the model. Their example showed a reduction from 150,000 tokens to 2,000 tokens — a 98.7% savings.[^9]

This approach partially addresses context efficiency but does not solve the fundamental asymmetry: the server still has no awareness of the agent session. The code execution environment is agent-controlled, not server-controlled. The server cannot proactively optimize its own behavior based on session state.

### Subagent Patterns in Production

Production systems have already converged on subagent delegation as a core architectural pattern, though currently implemented at the agent framework level rather than at the MCP server level:

- **Claude Code** implements subagents with independent context windows that receive scoped task descriptions, operate in isolation, and return structured summaries. Subagents prevent "context poisoning" where detailed implementation work clutters the main conversation.[^11][^12]
- **Jeremy Daly's commercial agent framework** uses scoped delegation where the parent assembles a minimal context package per subagent: task description, applicable policies, and explicitly selected artifacts. Each subagent runs in an isolated context.[^13]
- **OpenAI Agents SDK** supports agent handoffs and delegation patterns where specialized agents seamlessly hand off tasks to each other, with MCP servers as capability sources for individual agents.[^14]

The critical gap is that these subagents are defined and managed by the **host/client** side. The MCP server itself has no agency in defining how delegation should work for its domain.

## The Thesis: Server-Exposed Subagents

### Core Proposal

The proposal extends MCP so that a server can expose not just tools, resources, and prompts, but also a **subagent specification** — a self-contained agent definition that includes:

1. **A system prompt** tailored to the server's domain expertise
2. **A model preference or constraint** (e.g., "requires reasoning model" or "optimized for GPT-4-class")
3. **A context budget request** (e.g., "needs 8K tokens of working space")
4. **Tool subset selection** — which of its own tools the subagent will use
5. **Input/output contract** — what context it needs from the parent and what it returns

This would allow the server to define how it is best used, rather than relying entirely on the host's LLM to figure out optimal invocation patterns.

### Architectural Advantages

**Domain-specific context management.** A database MCP server knows that query results can be enormous and should be filtered before returning. A code analysis server knows it needs to see the full file but only needs to return a summary. Today, the host LLM must learn these patterns through trial and error. A server-exposed subagent encodes this knowledge structurally.[^7][^9]

**Context window partitioning.** Instead of tool definitions and intermediate results flowing through the main context window, the subagent operates in an isolated partition. The parent agent delegates a task and receives a structured summary. This mirrors the pattern that Claude Code already implements at the framework level, but pushes the isolation boundary down to the server.[^12][^13]

**Model-aware behavior.** A subagent specification could declare model requirements or preferences. A complex reasoning task might request a larger model, while a simple data retrieval might specify a smaller, faster one. Current MCP servers have no mechanism to express this.[^15][^16]

**Reduced LLM call overhead.** CA-MCP demonstrated that giving servers more autonomy reduces total LLM calls. Server-exposed subagents would formalize this: the server defines multi-step workflows internally rather than requiring the host to orchestrate each step.[^1]

### Protocol-Level Implementation Considerations

Extending MCP to support subagent exposure would require additions to the protocol's capability negotiation system:[^4]

- A new **`subagents`** capability that servers can declare during initialization
- A **`subagents/list`** method analogous to `tools/list` that returns available subagent specifications
- A **`subagents/invoke`** method that delegates a task with a scoped context payload
- **Context budget negotiation** — the host and server agree on token allocation before invocation
- **Scope inheritance rules** — tenant ID, user ID, and policy version must propagate from parent to subagent as security invariants[^13]

The MCP spec's 2025-11-25 revision already introduced async Tasks with task hints and progress tracking, which provides the transport foundation for long-running subagent operations. The existing sampling capability, where servers can request LLM completions through the client, represents a nascent form of server-initiated intelligence.[^17][^4]

### Security and Isolation Considerations

The MCP specification's principle that servers "should not be able to read the whole conversation" would need to be preserved. Server-exposed subagents would operate under strict constraints:[^4]

- **Scoped input only:** The subagent receives only what the parent explicitly passes, not the full conversation[^13]
- **Output contracts:** Returns must conform to a declared schema, preventing information leakage
- **Tenant boundary inheritance:** All memory and data access must inherit the parent's tenant and user scope[^13]
- **No recursive delegation:** Subagents should not spawn further subagents, enforcing a single-level hierarchy to prevent uncontrolled depth[^13]
- **Audit trail:** Delegation events must be captured in the parent's trace envelope for replay and compliance[^13]

Current MCP security concerns around cross-tenant data leakage and session management flaws would need to be addressed before adding this delegation surface.[^18]

## Supporting and Countervailing Evidence

### Evidence Supporting the Thesis

The MemTool framework directly addresses the problem of managing tool and MCP server contexts across multi-turn conversations, finding that fixed context windows severely limit effectiveness and proposing dynamic tool management architectures. The MCP Maturity Model identifies "disconnected models" — maintaining coherent context across agent handoffs — as "the number one failure mode in production systems". Multiple benchmarks (MCPToolBench++, MCP-Universe, MCPAgentBench) confirm that even state-of-the-art models struggle with complex multi-step tool invocations, with GPT-5 achieving only 43.72% on realistic MCP tasks.[^19][^16][^20][^21][^15]

### Potential Counterarguments

**Complexity risk.** Adding subagent capabilities to MCP servers increases implementation complexity. Anthropic's design principle that "servers should be extremely easy to build" would be challenged. Mitigation: subagent exposure would be an optional capability, not a requirement.[^4]

**Security surface expansion.** Every additional delegation layer is a potential attack vector. MCP already faces security challenges with authentication, session management, and prompt injection. Mitigation: strict scope inheritance, output contracts, and the existing host-controlled permission model.[^22][^18]

**Redundancy with agent frameworks.** Frameworks like LangGraph, CrewAI, and AutoGen already implement multi-agent coordination. Mitigation: the proposal is complementary — framework-level orchestration handles agent-to-agent coordination, while server-exposed subagents handle domain-specific task encapsulation.[^20]

**Statefulness concerns.** MCP servers being stateless is a feature for scalability. Subagents would introduce session-scoped state. Mitigation: subagent sessions would be ephemeral and task-scoped (created and destroyed per invocation), similar to Claude Code's subagent lifecycle.[^23][^11]

## Toward Implementation: A Maturity Path

Realizing server-exposed subagents could follow a phased approach building on existing MCP primitives:

1. **Phase 1 — Enhanced server metadata.** Servers declare model preferences, recommended context budgets, and tool groupings via extended capability negotiation. No protocol changes required; this extends existing patterns.
2. **Phase 2 — Prompt templates as proto-subagents.** MCP's existing `prompts` primitive already allows servers to expose pre-defined templates. Enriching these with execution context requirements and output schemas creates a lightweight subagent-like experience.
3. **Phase 3 — Formal subagent primitive.** A new protocol primitive enabling full subagent specifications with context budget negotiation, model preferences, and isolated execution. This requires spec-level changes and SDK support.
4. **Phase 4 — Bidirectional context coordination.** Subagents can read from and contribute to a shared context store (à la CA-MCP), enabling multi-server coordination without routing everything through the host LLM.[^1]

## Conclusion

The thesis that MCP servers suffer from being unaware of agent configuration, model configuration, and context window state is well-supported by both academic research and production experience. Tool definitions consuming 16%+ of context windows, intermediate results flowing redundantly through model context, and servers operating as blind executors while the LLM handles all coordination represent measurable, documented inefficiencies.[^10][^9][^5]

The proposal to let MCP servers expose their own subagents with delegated context control is architecturally novel but directionally consistent with where the ecosystem is heading. CA-MCP demonstrates that server autonomy reduces LLM overhead. Claude Code and commercial agent systems already implement subagent isolation at the framework level. The MCP spec's async Tasks and sampling capabilities provide protocol-level foundations.[^17][^12][^4][^1][^13]

The key insight is that **the server is the domain expert** — it knows how its tools should be composed, what context its workflows need, and how to compress its outputs. Encoding that expertise as a subagent specification, rather than expecting every host LLM to rediscover it, represents a shift from tool-centric to capability-centric integration. This would move MCP from being a universal tool connector to being a universal capability delegation protocol.

---

## References

1. [Enhancing Model Context Protocol (MCP) with Context-Aware Server Collaboration](https://arxiv.org/abs/2601.11595) - The Model Context Protocol (MCP) (MCP Community, 2025) has emerged as a widely used framework for en...

2. [Enhancing Model Context Protocol (MCP) with Context-Aware ...](https://arxiv.org/html/2601.11595v2) - The CA-MCP architecture introduces a Shared Context Store (SCS) into the MCP workflow, fundamentally...

3. [Model Context Protocol: Simplifying AI Tool Integration - Aerospike](https://aerospike.com/blog/model-context-protocol/) - Discover how the Model Context Protocol (MCP) standardizes AI integration with tools, databases, and...

4. [Architecture - Model Context Protocol](https://modelcontextprotocol.io/specification/2025-06-18/architecture) - The Model Context Protocol (MCP) follows a client-host-server architecture where each host can run m...

5. [Enhancing Model Context Protocol (MCP) with Context-Aware ...](https://arxiv.org/html/2601.11595v1) - This stateless separation simplifies tool integration but could also limit inter-agent awareness—pre...

6. [Understanding the Limitations of Anthropic's Model Context Protocol ...](https://dev.to/ramkey982/beyond-the-hype-understanding-the-limitations-of-anthropics-model-context-protocol-for-tool-48kk) - This report will examine the current limitations of MCP, including its stateful communication requir...

7. [MCP and Context Overload: Why More Tools Make Your AI Agent ...](https://eclipsesource.com/blogs/2026/01/22/mcp-context-overload/) - Don't give your agent access to full MCP servers if it doesn't need them. If you're designing agents...

8. [Scaling Multi-MCP AI Agents Beyond Context Limits](https://ieeexplore.ieee.org/document/11294866/) - Transformer models cannot reason over unbounded interactions; every request must fit within a finite...

9. [Code execution with MCP: building more efficient AI agents - Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp) - Learn how code execution with the Model Context Protocol enables agents to handle more tools while u...

10. [The Hidden Cost of MCPs and Custom Instructions on Your Context ...](https://selfservicebi.co.uk/analytics%20edge/improve%20the%20experience/2025/11/23/the-hidden-cost-of-mcps-and-custom-instructions-on-your-context-window.html) - Large context windows sound limitless—until you bolt on a few MCP servers and custom instructions. H...

11. [Enhancing Claude Code with MCP Servers and Subagents](https://dev.to/oikon/enhancing-claude-code-with-mcp-servers-and-subagents-29dd) - This architecture enables parent agents to delegate tasks to subagents for execution, with parallel ...

12. [Understanding Claude Code's Full Stack: MCP, Skills, Subagents ...](https://alexop.dev/posts/understanding-claude-code-full-stack/) - Each subagent has its own system prompt, allowed tools, and separate context window. When Claude enc...

13. [Context Engineering for Commercial Agent Systems - Jeremy Daly](https://www.jeremydaly.com/context-engineering-for-commercial-agent-systems/) - This architecture gives you: Replayability; Deletion guarantees; Poisoning containment; Cross-tenant...

14. [How to build AI agents with MCP: 12 framework comparison (2025)](https://clickhouse.com/blog/how-to-build-ai-agents-mcp-12-frameworks) - Compare 12 AI agent frameworks with MCP support. Complete guide with code examples for Claude SDK, O...

15. [MCPToolBench++: A Large Scale AI Agent Model Context Protocol MCP Tool Use Benchmark](https://arxiv.org/abs/2508.07575) - LLMs'capabilities are enhanced by using function calls to integrate various data sources or API resu...

16. [MCP-Universe: Benchmarking Large Language Models with Real-World Model Context Protocol Servers](https://arxiv.org/abs/2508.14704) - The Model Context Protocol has emerged as a transformative standard for connecting large language mo...

17. [MCP 2025-11-25 is here: async Tasks, better OAuth, extensions ...](https://workos.com/blog/mcp-2025-11-25-spec-update) - Yesterday, the Model Context Protocol (MCP) shipped a new spec revision: 2025-11-25—right on the one...

18. [The State of MCP Security in 2025: Key Risks, Attack Vectors, and ...](https://datasciencedojo.com/blog/mcp-security-risks-and-challenges/) - A flaw in the MCP server's handling of authentication and session management allowed one customer's ...

19. [MCPAgentBench: A Real-world Task Benchmark for Evaluating LLM Agent MCP Tool Use](https://arxiv.org/abs/2512.24565) - Large Language Models (LLMs) are increasingly serving as autonomous agents, and their utilization of...

20. [The MCP Maturity Model: Evaluating Your Multi-Agent Context ...](https://subhadipmitra.com/blog/2025/mcp-maturity-model/) - Makes agent selection harder; consumes context window space unnecessarily, Keep tools focused and we...

21. [MemTool: Optimizing Short-Term Memory Management for Dynamic Tool Calling in LLM Agent Multi-Turn Conversations](https://arxiv.org/abs/2507.21428) - Large Language Model (LLM) agents have shown significant autonomous capabilities in dynamically sear...

22. [Memory in AI: MCP, A2A & Agent Context Protocols | Orca Security](https://orca.security/resources/blog/bringing-memory-to-ai-mcp-a2a-agent-context-protocols/) - Explore how MCP, A2A, and other agent context protocols bring memory to AI—along with the security r...

23. [Why MCP Servers Are a Nightmare for Engineers](https://utcp.io/blog/mcp-servers-nightmare) - It Forces Statefulness on Stateless Problems​ ... This is great for a complex, multi-turn copilot, b...

