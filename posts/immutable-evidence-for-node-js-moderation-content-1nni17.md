# Immutable Evidence for Node.js Moderation: Content, Image, JSON Schema, Chat APIs

The least complex safe design is a quarantine boundary: persist an immutable copy of the submitted text and image, ask a chat-completions model for one JSON Schema-constrained decision, validate it locally, and publish only the bytes named by that decision. An OpenAI-compatible API is a transport contract here, not the policy authority.

Short answer: make the moderation result a typed, versioned record tied to a content digest; timeouts, invalid JSON, and incomplete image reads must leave the upload quarantined rather than quietly converting uncertainty into approval.

This framing matters for Node.js services because the attractive part of the example is the API call, while the dangerous part sits on either side of it. The service can validate a response perfectly and still publish the wrong object if an image URL is mutable. It can also classify the right object, retry after an ambiguous acknowledgment, and apply the same publication side effect twice. I don't trust a green HTTP response to settle either question.

## How should Node.js content moderation check text and image inputs?

Freeze the candidate before classification. Assign a submission ID, store the exact UTF-8 text and image bytes, calculate a digest over a canonical manifest, and attach the policy version that will judge it. The moderation request and the later publication transaction must both refer to that immutable identity. If the product accepts a remote image URL, fetch it into controlled storage first; otherwise the URL owner can replace the image between the safety check and publication.

The invariant is short: **the bytes checked are the bytes published.**

Keep three meanings separate. Transport success means a complete response arrived. Contract success means the response parsed and passed local schema validation. Policy success means application code accepted the model's typed findings under a named policy version. Collapsing those meanings into one `ok` boolean makes incident review nearly impossible, and it encourages a particularly bad fallback: treating a parser failure as an empty set of findings.

Preflight the media outside the model call. Verify the declared media type against decoded content, enforce limits selected for your own system, reject an empty or truncated object, and normalize text once. Those limits aren't universal facts; they are deployment constraints, so put them in code and tests rather than implying that a prompt will enforce them. I'm not sure there is a defensible universal confidence threshold either. A labeled evaluation set, policy-specific loss assumptions, and human overturn data are what would resolve that choice.

This also defines the Node.js boundary cleanly. The request handler accepts a submission and returns a durable ID; a worker reads the frozen manifest, invokes the compatible chat API, and writes a decision; a compare-and-set transition exposes the object only when the expected digest and policy version still match. The implementation language used for a probe doesn't alter that state machine.

## Derive the JSON Schema contract before making the chat completion

A moderation contract should be smaller than the policy document. The model needs to return a decision, zero or more findings from a closed category set, the policy version, and the submission digest. Explanations can help reviewers, but they must not become executable instructions. Unknown fields should fail validation because silent schema expansion is indistinguishable from an unreviewed policy change.

The following Python probe shows the wire payload even when the production caller lives in Node.js. `CHAT_COMPLETIONS_URL` is the complete endpoint supplied by the chosen compatible service, so the example doesn't assume that every provider exposes the same route. The image URL should identify the already frozen object, not a mutable public upload.

```python
import os

import requests
from jsonschema import Draft202012Validator


DECISION_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["submission_digest", "policy_version", "decision", "findings"],
    "properties": {
        "submission_digest": {"type": "string", "minLength": 1},
        "policy_version": {"type": "string", "minLength": 1},
        "decision": {"type": "string", "enum": ["allow", "review", "block"]},
        "findings": {
            "type": "array",
            "items": {
                "type": "object",
                "additionalProperties": False,
                "required": ["category", "reason"],
                "properties": {
                    "category": {
                        "type": "string",
                        "enum": ["sexual", "violence", "self_harm", "hate", "other"],
                    },
                    "reason": {"type": "string"},
                },
            },
        },
    },
}


def request_decision(text, frozen_image_url, digest, policy_version):
    payload = {
        "model": os.environ["MODERATION_MODEL"],
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify the supplied content under the named policy. "
                    "Return only the requested structured decision."
                ),
            },
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": text},
                    {"type": "image_url", "image_url": {"url": frozen_image_url}},
                    {
                        "type": "text",
                        "text": f"digest={digest}; policy_version={policy_version}",
                    },
                ],
            },
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": {
                "name": "moderation_decision",
                "strict": True,
                "schema": DECISION_SCHEMA,
            },
        },
    }

    response = requests.post(
        os.environ["CHAT_COMPLETIONS_URL"],
        headers={"Authorization": f"Bearer {os.environ['API_KEY']}"},
        json=payload,
        timeout=(5, 30),
    )
    response.raise_for_status()
    decision = response.json()["choices"][0]["message"]["content"]
    return decision
```

Compatibility needs to be tested, not inferred from a label. The deployed model and endpoint must accept image content parts and strict structured output in the combination shown; some compatible interfaces expose different subsets. A client abstraction can make configuration easier, but it cannot prove that a model supports a capability. The LangChain `ChatOpenAI` integration documentation is useful evidence for the client boundary, while conformance tests remain the evidence for the deployment.

The returned `content` may be a JSON string depending on the client boundary. Parse once, validate with the same schema, and then verify the values that the schema alone cannot bind: `submission_digest` must equal the frozen manifest, and `policy_version` must equal the requested version. Don't ask the model to police its own identity binding.

```python
import json


def validate_and_bind(raw_content, expected_digest, expected_policy):
    value = json.loads(raw_content) if isinstance(raw_content, str) else raw_content
    Draft202012Validator(DECISION_SCHEMA).validate(value)

    if value["submission_digest"] != expected_digest:
        raise ValueError("decision is bound to another content digest")
    if value["policy_version"] != expected_policy:
        raise ValueError("decision is bound to another policy version")
    return value
```

No valid object, no publish.

## Failure behavior is the real architecture comparison

The model call is read-like, but the surrounding workflow changes durable state: it records a submission, appends a decision, may enqueue human review, and may expose an object. Retries therefore need a stable operation key derived from the content digest, policy version, and model configuration. A unique decision record makes two workers converge. A compare-and-set publication transition ensures that a late result for an old policy cannot approve a newer submission state.

Consider a concrete retry sequence. Worker A obtains a valid `allow` object and commits decision `dec_41`, but loses the acknowledgment. Worker B sees the retry, receives another valid answer, and tries to commit the same operation key. The unique constraint returns the existing decision; B reads it and attempts the state transition. Exactly one transition from `quarantined` to `published` can succeed. Without those two storage constraints, duplicate queue messages can create duplicate audit rows, counters, notifications, or publication jobs even though both model answers agree. An HTTP `429` is easier to see than this ambiguity, because a rate-limit response clearly authorizes a later attempt while a missing acknowledgment says nothing about whether the previous write committed.

Here is the design review I would use. It names the unsafe state rather than grading SDK ergonomics.

| Shape | Best fit | Failure mode to test | Limitation |
|---|---|---|---|
| In-request classification | Low-risk content that need not be retained | Client disconnect loses the pending result | Not suitable when replay or appeal evidence matters |
| Persist, quarantine, classify | Public uploads with delayed publication | Queue age becomes hidden user latency | Adds storage lifecycle and worker operations |
| Deterministic rules before a model | Stable structural exclusions | Rule and model policy versions drift | Poor fit when meaning depends heavily on context |
| Model plus human review | Uncertain decisions with material consequences | Review backlog silently becomes an allow path | Requires staffing, access control, and reviewer calibration |

Persist-and-quarantine is a sound default for public multimodal uploads because it preserves evidence and makes uncertainty explicit. The catch is that it is **not suitable when retaining unreviewed material is prohibited**. In that case, stick with an ephemeral synchronous path, tightly control memory and logs, and accept weaker replay and appeal capability. A low-risk internal text tool may also justify a synchronous gate because the queue, retention policy, and review operation would cost more complexity than the risk warrants.

Streaming rarely changes the final moderation latency because policy code still needs the complete JSON object. Server-Sent Events deliver a stream of events over a persistent connection, but a prefix that appears to contain `allow` is not a valid decision. If streaming is required, buffer until the protocol's completion condition, parse once, validate once, and commit once; test split JSON tokens, duplicate events, reconnects, cancellation, and a stream that ends before a complete object. Otherwise, a non-streaming request leaves fewer states to reason about.

## Roll out from shadow evidence to an enforced gate

Start by writing immutable submission manifests, digests, quarantine states, and policy versions without changing publication. Then run the classifier in shadow mode, retain typed decisions, and compare them with the existing outcome on a labeled set. Measure false allows, false blocks, malformed objects, latency percentiles, quarantine age, retry counts, duplicate-key conflicts, and human-review overturns. Cost belongs beside those measures as usage per resolved submission; a nominal per-call figure doesn't include retries, image processing, or review labor.

Next, enforce one narrow content class with compare-and-set transitions. Route uncertainty to review, never to implicit approval. Canary model and policy changes independently because they answer different questions: the model change alters classifier behavior, while the policy change alters what the application permits. Keep old decisions append-only so an appeal can reconstruct what policy judged which digest at the time.

Finally, test recovery as a first-class path. Pause workers, deliver duplicate jobs, rotate a policy while work is queued, remove access to a frozen image, and replay a valid old result against a new manifest. The expected outcome in each case is boring: the object stays quarantined, one current decision owns the transition, and operators can see why progress stopped. Your mileage may vary on retention duration and review thresholds, because law, product risk, and staffing differ; document those choices rather than burying them in a prompt.

The compact migration rule is this: establish identity and durable states first, collect shadow evidence second, and enable side effects last. The model may classify content. The storage transaction decides what becomes visible.

## References

- MDN, "Using server-sent events": https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- LangChain, "ChatOpenAI integration": https://python.langchain.com/docs/integrations/chat/openai/

## Further reading

- Server-Sent Events behavior and event-stream handling: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Chat model client integration and compatibility configuration: https://python.langchain.com/docs/integrations/chat/openai/
