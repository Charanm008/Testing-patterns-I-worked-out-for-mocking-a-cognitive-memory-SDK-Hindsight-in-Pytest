**Title:** Testing patterns I worked out for mocking a cognitive memory SDK (Hindsight) in Pytest — singleton isolation, SDK object trees, and failure boundaries

**Suggested Flair:** Discussion

**Post Body:**

I've been working on a deal intelligence tool that uses LLM agents to analyze sales pipelines. The agents pull context from a persistent memory store (Hindsight SDK) to make recommendations — things like flagging failed strategies and suggesting course corrections.

The testing problem hit us pretty fast: our CI suite was calling the real memory API, which meant 45-second runs, flaky failures on latency spikes, and burning API tokens on every push. We needed full offline mocking.

Turns out mocking a cognitive memory SDK is meaningfully different from mocking a standard REST client. Here are the three patterns we landed on:

**Pattern 1: Mirror the SDK's object tree, not just primitives**

The Hindsight SDK returns typed response objects with `.results` containing memory items that expose `.text`, `.type`, `.metadata`. If you mock with raw dicts, your downstream code that does `getattr(mem, 'text')` either breaks or hits a fallback branch you didn't intend to test.

We built hierarchical mocks instead:

    mock_mem1 = MagicMock()
    mock_mem1.text = 'Customer rejected cloud hosting'
    mock_mem1.type = 'objection'
    mock_mem1.metadata = {'customer_id': '123'}

    mock_result = MagicMock()
    mock_result.results = [mock_mem1]
    mock_client.recall.return_value = mock_result

This exercises the actual attribute-extraction path in our recall layer rather than silently bypassing it.

**Pattern 2: Singleton teardown in setUp()**

Our memory client uses `__new__`-based singleton pattern to avoid connection thrashing across FastAPI handlers. Great in production, terrible for tests — Test A's configured client bleeds into Test B.

Fix is dead simple but easy to forget:

    def setUp(self):
        HindsightMemoryClient._instance = None

Without this, testing missing env vars or cold-start errors becomes impossible because `__new__` just returns the already-initialized instance.

**Pattern 3: Test failure boundaries, not just happy paths**

The most useful test we wrote simulates a `ConnectionError` from the memory layer and verifies the primary API still returns 200:

    @patch('memory.retain.retain_memory')
    def test_interaction_survives_hindsight_failure(mock_retain, client, test_deal):
        mock_retain.side_effect = ConnectionError("Hindsight API unreachable")

        response = client.post("/interaction", json=payload)
        assert response.status_code == 200

This proved our fire-and-forget memory hook (try/except around the retention call) actually works. Business data gets committed to SQLite regardless of what the memory layer does.

**Results:** 38 tests, 0.42 seconds, fully offline, zero token burn.

The takeaway for anyone integrating external memory or retrieval SDKs into an agent system: don't just mock the call — mock the response structure. And always test what happens when memory is down, because that's when your agent's graceful degradation matters most.

Full test suite and agent code: https://github.com/Pranav-173/Precedent

---

## r/sideproject

**Title:** I built a sales intelligence tool with AI agents and persistent memory — here's the testing wall I hit and how I got past it

**Suggested Flair:** Share your project!

**Post Body:**

A few of us built Precedent, a deal intelligence platform for B2B sales teams. The idea: instead of static CRM dashboards, have AI agents actively analyze deal pipelines, flag risks, and generate tactical next steps for sales reps.

The core architecture has four pieces — a FastAPI/SQLAlchemy backend managing deals and stakeholders, a competitor intelligence engine, a cognitive memory layer (using Hindsight SDK for persistent vector memory), and an agent orchestrator that reasons across all of it.

The interesting part isn't the feature set though. It's the testing wall we slammed into.

**The problem**

Our agent system depends on an external memory API. Every time we recalled past interactions or stored new deal context, we were hitting a live service. CI runs went from "run before every commit" to "maybe run before a PR if we feel patient." 45 seconds, flaky on network hiccups, and burning API tokens.

**What we tried first (and why it didn't work)**

Naive mocking — `client.recall.return_value = [{"text": "..."}]` — looked fine until tests started passing that should have failed. The real SDK returns nested typed objects, not dicts. Our production code does `mem.text` and `getattr(mem, 'metadata')`, so dict mocks just slid into a fallback code path we weren't trying to test.

Also, our memory client is a singleton. Test isolation was nonexistent — whichever test ran first configured the client, and every subsequent test inherited that state.

**What actually worked**

Three things:

1. Build MagicMock objects that mirror the SDK's response hierarchy. Set `.text`, `.type`, `.metadata` as explicit attributes on mock objects, then nest them inside a parent mock's `.results` list. This exercises the same code path production hits.

2. Reset the singleton in `setUp()` with `HindsightMemoryClient._instance = None`. Simple, but it was the difference between a reliable suite and one where test order determined pass/fail.

3. Test failures explicitly. We simulate `ConnectionError` from the memory SDK and verify the API still returns 200. The deal data saves to SQLite even when memory is completely down. This was probably the most valuable test we wrote — it proved the fire-and-forget pattern actually worked.

**End result:** 38 tests, 0.42 seconds, fully offline. We went from "skip the tests" to "run them reflexively before every commit."

If you want to look at the code or the test suite: https://github.com/Pranav-173/Precedent

Happy to talk about any of the architecture decisions. The agent orchestration layer and the memory integration were both areas where we iterated a lot.

---

## r/aiagents

**Title:** How we unit test agents that depend on a cognitive memory store — mocking patterns, singleton isolation, and testing degraded memory

**Suggested Flair:** Discussion

**Post Body:**

One of the less-discussed aspects of building agent systems is testability. Once your agent pulls context from a persistent memory store before reasoning, you have a dependency that makes unit testing genuinely difficult.

I ran into this building a multi-agent system for deal intelligence. We have a `RiskAnalystAgent` and `NextBestActionAgent` orchestrated together. They pull live deal state, competitor signals, and — critically — recalled memories from past interactions via Hindsight (a cognitive memory SDK). The recalled context directly shapes the agents' recommendations.

The question becomes: how do you write fast, deterministic tests for agent logic when the memory dependency returns complex SDK objects from a remote API?

**The architecture in brief:**

    ┌──────────────────────────────────────────────────┐
    │            Agent Orchestrator                    │
    │  (RiskAnalystAgent + NextBestActionAgent)        │
    └────────────────────┬─────────────────────────────┘
                         │
           ┌─────────────┼─────────────────┐
           ▼                               ▼
    Deal State (SQLite)         Hindsight Memory Layer
                                 - recall_memories
                                 - retain_memory

The agents consume structured context dicts that include recalled memories. Testing the agents themselves is straightforward — you construct the context dict with whatever memory entries you want:

    def test_next_best_action_flags_failed_strategy():
        agent = NextBestActionAgent()
        context = {
            "brief": {
                "deal_id": 101,
                "deal_stage": "Proposal",
                "risk_level": "HIGH",
            },
            "interactions": [
                {
                    "type": "meeting",
                    "strategy": "aggressive-discounting",
                    "outcome": "failed"
                }
            ],
            "stakeholders": []
        }
        result = agent.analyze(context)
        recs = [r.action for r in result.recommendations]
        assert any("Follow up on failed strategy" in a for a in recs)

No live vector DB, no embedding calls, full control over the scenario. The agent either detects the failed strategy and pivots, or it doesn't.

**The harder problem is testing the memory layer itself.**

The Hindsight SDK doesn't return plain dicts — it returns typed objects with `.text`, `.type`, `.metadata` attributes nested inside a response object's `.results` field. If you mock with raw dicts, you're testing a code path that production never hits.

What works:

    mock_mem = MagicMock()
    mock_mem.text = 'Customer rejected cloud hosting'
    mock_mem.type = 'objection'
    mock_mem.metadata = {'customer_id': '123'}

    mock_result = MagicMock()
    mock_result.results = [mock_mem]
    mock_client.recall.return_value = mock_result

Additionally, our memory client is a singleton. Without resetting `_instance = None` in setUp, tests that check for missing API keys will silently reuse a previously configured client and give false passes.

**Testing graceful degradation**

The test I'd argue matters most for agent systems: what happens when memory is unavailable?

    mock_retain.side_effect = ConnectionError("Hindsight API unreachable")
    response = client.post("/interaction", json=payload)
    assert response.status_code == 200

Our agents' data pipeline (deal creation, interaction logging) continues working even when the memory layer is down. Memory retention is wrapped in a try/except block — fire-and-forget. The agent loses historical context but doesn't crash the transaction.

38 tests, 0.42 seconds, no network calls.

If you want to dig into the implementation, the test suite and full agent orchestrator are at https://github.com/Pranav-173/Precedent

---

## r/aimemory

**Title:** Practical mocking patterns for Hindsight SDK in Pytest — dealing with typed response objects, singleton clients, and client-side metadata filtering

**Suggested Flair:** Discussion

**Post Body:**

If you're integrating a persistent memory SDK into an agent system and want fast, deterministic tests, here's what I learned mocking the Hindsight Python SDK (`hindsight-client`) in a real project.

The context: we built a deal intelligence platform where every customer interaction gets stored as a memory entry and recalled later to provide agents with historical context. The memory layer wraps Hindsight's `retain` and `recall` operations with client-side metadata filtering and prompt context formatting.

Testing it properly was harder than expected. Three specific problems came up:

**Problem 1: SDK returns typed objects, not dicts**

`client.recall()` returns a response object with a `.results` attribute. Each item in `.results` is a typed memory object exposing `.text`, `.type`, and `.metadata`. Our recall wrapper extracts these with `getattr()`:

    if hasattr(result, 'results'):
        for mem in result.results:
            memories.append({
                'text': getattr(mem, 'text', ''),
                'type': getattr(mem, 'type', None),
                'metadata': getattr(mem, 'metadata', {})
            })

Mocking with raw dicts (`[{"text": "..."}]`) skips this entire extraction path and hits a fallback branch instead. That means your tests pass but you're not actually testing the code that runs in production.

The fix is to build the mock hierarchy to match the SDK structure:

    mock_mem1 = MagicMock()
    mock_mem1.text = 'hello'
    mock_mem1.type = 'note'
    mock_mem1.metadata = {'src': 'test'}

    mock_result = MagicMock()
    mock_result.results = [mock_mem1]
    mock_client.recall.return_value = mock_result

    result = recall_memories('query', limit=5)
    assert result[0]['text'] == 'hello'
    assert result[0]['metadata'] == {'src': 'test'}

**Problem 2: Singleton memory client poisons test isolation**

Our `HindsightMemoryClient` uses `__new__`-based singleton to prevent connection thrashing. In tests, this means the first test to instantiate the client wins — subsequent tests that need to test initialization errors (missing API key, missing bank ID) silently reuse the existing instance.

Solution: reset the singleton before each test.

    def setUp(self):
        HindsightMemoryClient._instance = None

    @patch('memory.hindsight_client.Hindsight')
    def test_init_missing_api_key(self, MockHindsight):
        os.environ.pop('HINDSIGHT_API_KEY', None)
        with self.assertRaises(ValueError):
            HindsightMemoryClient()

Without the reset, this test gives a false pass because `__new__` returns the already-configured instance from a previous test.

**Problem 3: Memory retention must not block business transactions**

Memory writes happen inline during interaction logging. If Hindsight is unreachable, the interaction still needs to commit to the relational DB. We wrap the retention call in try/except:

    try:
        retain_memory(text=interaction.summary, metadata={...})
    except Exception as exc:
        logger.warning("Memory hook failed: %s", exc)

Testing this with a simulated `ConnectionError`:

    mock_retain.side_effect = ConnectionError("Hindsight API unreachable")
    response = client.post("/interaction", json=payload)
    assert response.status_code == 200

**Practical lessons from doing this:**

- Always mock SDK response objects at the attribute level, not with primitive stand-ins. `MagicMock()` with explicit `.text`, `.type`, `.metadata` beats returning a dict every time.
- Singletons need explicit `_instance = None` resets in test fixtures. This isn't optional — test order bugs are some of the most painful to debug.
- Client-side metadata filtering on top of SDK responses needs its own tests. The SDK's filtering might not match your exact needs, so if you're doing post-processing, test the post-processing in isolation.
- Test the "memory is down" scenario. It's probably the single most important test for any memory-integrated system.

38 tests, 0.42 seconds, completely offline. Full implementation is at https://github.com/Pranav-173/Precedent

---
