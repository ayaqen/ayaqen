# Existing Gist Inspection

- Owner: ayaqen
- Public: true
- Description: LangGraph supervisor template with budget guards, retries, and resumable human approval
- Created: 2026-07-20T19:01:28Z
- Updated: 2026-07-29T16:06:17Z
- Comments: 0
- Forks returned: 0

## Files

### `gistfile1.txt`

```
"""
supervisor_graph.py - a production-shaped LangGraph multi-agent template.

Most LangGraph examples stop at "supervisor routes to workers." That part is easy.
The parts that actually break in production are scattered across a dozen doc pages:

  1. Routing that can't hallucinate a node name  -> structured output over a Literal
  2. Parallel branches clobbering shared state   -> a reducer on every multi-writer field
  3. One flaky call killing a 40-step run        -> per-node RetryPolicy
  4. A supervisor loop quietly burning $200      -> hop + token budget enforced in code
  5. Crash recovery and human sign-off           -> checkpointer + interrupt()

This file wires all five into one runnable graph. It is a skeleton, not a demo:
swap the worker prompts and you have a real pipeline.

Tested against:
    langgraph 0.3.x, langchain-anthropic 0.3.x, langgraph-checkpoint-sqlite 2.x

Setup:
    pip install langgraph langchain-anthropic langgraph-checkpoint-sqlite pydantic
    export ANTHROPIC_API_KEY=...
    python supervisor_graph.py
"""

from __future__ import annotations

import operator
import os
from typing import Annotated, Literal, TypedDict

from langchain_anthropic import ChatAnthropic
from langchain_core.messages import AIMessage, AnyMessage, HumanMessage, SystemMessage
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.types import Command, RetryPolicy, interrupt
from pydantic import BaseModel, Field

# Nothing below is model-specific. Point this anywhere.
MODEL = os.environ.get("AGENT_MODEL", "claude-sonnet-4-5")

# Hard ceilings. These are checked in code, never delegated to the model --
# a supervisor asked to reason about its own budget will reason its way into
# one more loop every single time.
MAX_HOPS = 12
MAX_TOKENS = 120_000

WORKERS = ("researcher", "analyst", "writer")


# --------------------------------------------------------------------------
# State
# --------------------------------------------------------------------------

class Finding(TypedDict):
    agent: str
    content: str


class AgentState(TypedDict):
    """Rule of thumb: any field more than one node can write in the same
    superstep needs a reducer, or LangGraph raises InvalidUpdateError the first
    time you fan out. `draft` and `approved` have exactly one writer each, so
    they're safe as last-write-wins."""

    messages: Annotated[list[AnyMessage], add_messages]
    findings: Annotated[list[Finding], operator.add]
    tokens_used: Annotated[int, operator.add]
    hops: Annotated[int, operator.add]
    task: str
    draft: str
    approved: bool


class Route(BaseModel):
    """The Literal is the entire point. The model physically cannot emit a node
    name that isn't in the graph, which kills the single most common failure
    mode in string-parsed supervisors."""

    next: Literal["researcher", "analyst", "writer", "FINISH"]
    reason: str = Field(description="One sentence. This is what you read in traces at 2am.")


def _text(msg: AIMessage) -> str:
    """Anthropic returns content as a block list the moment tools or extended
    thinking are in play. Never index msg.content[0] -- flatten it."""
    if isinstance(msg.content, str):
        return msg.content
    return "\n".join(
        b.get("text", "")
        for b in msg.content
        if isinstance(b, dict) and b.get("type") == "text"
    )


def _digest(findings: list[Finding]) -> str:
    if not findings:
        return "(nothing yet -- you're first)"
    return "\n\n".join(f"[{f['agent']}]\n{f['content']}" for f in findings)


# --------------------------------------------------------------------------
# Workers
# --------------------------------------------------------------------------

WORKER_PROMPTS = {
    "researcher": (
        "You gather raw material. Surface concrete facts, numbers, and sources. "
        "Do not interpret, do not recommend. If you don't know something, say so plainly."
    ),
    "analyst": (
        "You pressure-test what the researcher found. Name the weakest assumption, "
        "the missing counter-evidence, and the one thing that would change the conclusion."
    ),
    "writer": (
        "You produce the deliverable. Plain language, no preamble, no summary of "
        "your own process. Lead with the answer."
    ),
}


def make_worker(name: str):
    llm = ChatAnthropic(model=MODEL, max_tokens=2048)

    def worker(state: AgentState) -> dict:
        reply = llm.invoke([
            SystemMessage(WORKER_PROMPTS[name]),
            HumanMessage(
                f"Task: {state['task']}\n\n"
                f"What the team has so far:\n{_digest(state['findings'])}"
            ),
        ])
        body = _text(reply)
        usage = reply.usage_metadata or {}

        out: dict = {
            "findings": [Finding(agent=name, content=body)],
            "messages": [AIMessage(content=body, name=name)],
            "tokens_used": usage.get("total_tokens", 0),
        }
        if name == "writer":
            out["draft"] = body
        return out

    return worker


# --------------------------------------------------------------------------
# Supervisor
# --------------------------------------------------------------------------

def supervisor(
    state: AgentState,
) -> Command[Literal["researcher", "analyst", "writer", "approval", "__end__"]]:
    # Guard first, LLM second. Cheaper and unbypassable.
    if state["hops"] >= MAX_HOPS or state["tokens_used"] >= MAX_TOKENS:
        return Command(
            goto="approval" if state.get("draft") else END,
            update={"messages": [SystemMessage("Budget ceiling hit. Wrapping up.")]},
        )

    router = ChatAnthropic(model=MODEL, max_tokens=512).with_structured_output(Route)
    route: Route = router.invoke([
        SystemMessage(
            "You coordinate three specialists: researcher (gathers), analyst "
            "(critiques), writer (produces the deliverable). Pick who goes next. "
            "Choose FINISH only once a draft exists and the analyst has seen it. "
            "Do not send work to an agent whose input hasn't changed since last turn."
        ),
        HumanMessage(
            f"Task: {state['task']}\n"
            f"Hops used: {state['hops']}/{MAX_HOPS}\n\n"
            f"Transcript:\n{_digest(state['findings'])}"
        ),
    ])

    if route.next == "FINISH":
        return Command(goto="approval" if state.get("draft") else "writer")

    return Command(
        goto=route.next,
        update={"hops": 1, "messages": [SystemMessage(f"-> {route.next}: {route.reason}")]},
    )


# --------------------------------------------------------------------------
# Human gate
# --------------------------------------------------------------------------

def approval(state: AgentState) -> Command[Literal["supervisor", "__end__"]]:
    # GOTCHA: on resume, LangGraph re-executes this node from the top. Every
    # line above interrupt() runs twice. Keep DB writes, emails, and payments
    # strictly below it.
    decision = interrupt({
        "draft": state.get("draft", ""),
        "findings": len(state["findings"]),
        "tokens_used": state["tokens_used"],
        "how_to_resume": "{'approved': true} to ship, or {'approved': false, 'notes': '...'}",
    })

    if decision.get("approved"):
        return Command(goto=END, update={"approved": True})

    return Command(
        goto="supervisor",
        update={"messages": [HumanMessage(f"Revision requested: {decision.get('notes', '')}")]},
    )


# --------------------------------------------------------------------------
# Wiring
# --------------------------------------------------------------------------

def build_graph(checkpointer):
    b = StateGraph(AgentState)

    b.add_node("supervisor", supervisor)
    b.add_node("approval", approval)
    for name in WORKERS:
        b.add_node(
            name,
            make_worker(name),
            # On langgraph < 0.2.60 this kwarg is `retry=` instead.
            retry_policy=RetryPolicy(
                max_attempts=3,
                initial_interval=1.0,
                backoff_factor=2.0,
                retry_on=(TimeoutError, ConnectionError),
            ),
        )

    b.add_edge(START, "supervisor")
    for name in WORKERS:
        b.add_edge(name, "supervisor")  # workers always report back

    # No edges out of supervisor or approval -- Command(goto=...) carries the
    # routing. The Command[Literal[...]] return annotations are what let
    # graph.get_graph().draw_mermaid() still render those hops.
    return b.compile(checkpointer=checkpointer)


def main() -> None:
    task = "Should a two-person team self-host Postgres or pay for a managed instance?"
    cfg = {"configurable": {"thread_id": "demo-001"}}

    seed: AgentState = {
        "messages": [],
        "findings": [],
        "tokens_used": 0,
        "hops": 0,
        "task": task,
        "draft": "",
        "approved": False,
    }

    with SqliteSaver.from_conn_string("checkpoints.sqlite") as saver:
        graph = build_graph(saver)

        for update in graph.stream(seed, cfg, stream_mode="updates"):
            for node, payload in update.items():
                print(f"  [{node}] {str(payload)[:160]}")

        snapshot = graph.get_state(cfg)
        if snapshot.next:  # parked on the interrupt
            payload = snapshot.tasks[0].interrupts[0].value
            print("\n--- awaiting human ---")
            print(payload["draft"][:800])
            print(f"({payload['tokens_used']:,} tokens, {payload['findings']} findings)\n")

            # Kill the process here and rerun with the same thread_id: it picks
            # up exactly at this line. That is the whole value of the checkpointer.
            for update in graph.stream(Command(resume={"approved": True}), cfg, stream_mode="updates"):
                for node, p in update.items():
                    print(f"  [{node}] {str(p)[:160]}")

        print("\nfinal:", graph.get_state(cfg).values["draft"][:800])


if __name__ == "__main__":
    main()


# --------------------------------------------------------------------------
# Things that cost me time, in rough order of pain:
#
# - InvalidUpdateError on fan-out almost always means a shared state field is
#   missing a reducer, not that your graph shape is wrong.
# - interrupt() replays the node from the top. Side effects go after it.
# - Checkpoints are keyed on thread_id. Reusing one silently resumes an old
#   run; generate a fresh id per job unless resumption is the point.
# - RetryPolicy wraps the node, not the LLM call, so a retried node re-emits
#   its full state update. Keep nodes idempotent.
# - MemorySaver is fine for tests and useless in prod -- it dies with the
#   process. Sqlite for single-box, Postgres for anything real.
# - recursion_limit in config is a separate ceiling from your hop budget. Set
#   both; they fail differently and you want to know which one tripped.
# --------------------------------------------------------------------------
```

