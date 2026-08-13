# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }

from genlayer import *
import json
import typing


class MultiSourceEventResolver(gl.Contract):
    """
    Reusable multi-source event resolver for prediction markets and adjudication.
    Fetches independent sources, validates them, extracts outcomes via LLM,
    aggregates results, and reaches consensus with thoughtful equivalence.
    """

    events: TreeMap[str, str]       # event_id -> json
    event_counter: u256

    def __init__(self):
        self.events = TreeMap[str, str]()
        self.event_counter = 0

    @gl.public.write
    def create_event(self, question: str, source_urls: str) -> str:
        """
        question: natural language question (e.g. "Did Team X win the match on 2026-08-10?")
        source_urls: comma-separated list of independent sources (recommended 2-5)
        """
        if not question or not source_urls:
            raise gl.vm.UserError("question and source_urls are required")

        urls = [u.strip() for u in source_urls.split(",") if u.strip()]
        if len(urls) < 1:
            raise gl.vm.UserError("At least one source URL required")

        self.event_counter += 1
        event_id = str(self.event_counter)

        event_data = {
            "question": question,
            "source_urls": urls,
            "status": "open",                 # open | resolving | resolved | failed | disputed
            "source_results": [],
            "final_outcome": None,            # "yes" / "no" / "unknown" / "failed"
            "final_confidence": 0.0,
            "final_reasoning": "",
            "created_by": str(gl.message.sender_address),
            "challenged": False
        }

        self.events[event_id] = json.dumps(event_data)
        return event_id

    @gl.public.write
    def resolve_event(self, event_id: str) -> typing.Any:
        if event_id not in self.events:
            raise gl.vm.UserError("Event does not exist")

        event = json.loads(self.events[event_id])

        if event["status"] not in ["open", "resolving", "disputed"]:
            raise gl.vm.UserError(f"Event already finalized: {event['status']}")

        event["status"] = "resolving"
        question = event["question"]
        urls = event["source_urls"]

        def leader_fn() -> dict:
            source_results = []
            valid_count = 0
            yes_count = 0
            no_count = 0
            total_confidence = 0.0

            for url in urls:
                try:
                    response = gl.nondet.web.get(url)

                    if response.status_code >= 400:
                        source_results.append({
                            "url": url,
                            "status": "fetch_failed",
                            "outcome": None,
                            "confidence": 0.0,
                            "reasoning": f"HTTP {response.status_code}"
                        })
                        continue

                    content = response.body.decode("utf-8")[:9000]
                    if len(content.strip()) < 80:
                        source_results.append({
                            "url": url,
                            "status": "insufficient_content",
                            "outcome": None,
                            "confidence": 0.0,
                            "reasoning": "Content too short"
                        })
                        continue

                    prompt = f"""
You are a neutral fact-extractor for GenLayer.
Answer the question using ONLY the provided source content.
Be strict: if the source does not clearly support a yes or no, choose "unknown".

Question: {question}

Source content:
{content}

Respond with EXACTLY this JSON (no extra text):
{{
  "outcome": "yes" or "no" or "unknown",
  "confidence": 0.0 to 1.0,
  "reasoning": "one short paragraph"
}}
"""
                    raw = gl.nondet.exec_prompt(prompt, response_format="json")
                    outcome = raw.get("outcome", "unknown")
                    conf = float(raw.get("confidence", 0.0))

                    source_results.append({
                        "url": url,
                        "status": "success",
                        "outcome": outcome,
                        "confidence": conf,
                        "reasoning": raw.get("reasoning", "")
                    })

                    valid_count += 1
                    total_confidence += conf
                    if outcome == "yes":
                        yes_count += 1
                    elif outcome == "no":
                        no_count += 1

                except Exception as e:
                    source_results.append({
                        "url": url,
                        "status": "error",
                        "outcome": None,
                        "confidence": 0.0,
                        "reasoning": str(e)[:180]
                    })

            # Aggregation logic (substantive)
            if valid_count == 0:
                final_outcome = "failed"
                final_confidence = 0.0
                final_reasoning = "No valid sources could be processed"
            else:
                avg_conf = total_confidence / valid_count
                if yes_count > no_count and yes_count >= (valid_count / 2):
                    final_outcome = "yes"
                elif no_count > yes_count and no_count >= (valid_count / 2):
                    final_outcome = "no"
                else:
                    final_outcome = "unknown"

                final_confidence = round(avg_conf, 3)
                final_reasoning = (
                    f"Valid sources: {valid_count}. "
                    f"Yes: {yes_count}, No: {no_count}. "
                    f"Avg confidence: {final_confidence}"
                )

            return {
                "source_results": source_results,
                "final_outcome": final_outcome,
                "final_confidence": final_confidence,
                "final_reasoning": final_reasoning,
                "valid_count": valid_count
            }

        result = gl.eq_principle.prompt_comparative(
            leader_fn,
            principle=(
                "final_outcome must be identical. "
                "valid_count may differ by at most 1. "
                "final_confidence may differ by at most 0.15. "
                "Number of source_results must match."
            )
        )

        event["source_results"] = result["source_results"]
        event["final_outcome"] = result["final_outcome"]
        event["final_confidence"] = result["final_confidence"]
        event["final_reasoning"] = result["final_reasoning"]
        event["status"] = "resolved" if result["final_outcome"] != "failed" else "failed"

        self.events[event_id] = json.dumps(event)
        return result

    @gl.public.write
    def challenge_event(self, event_id: str, reason: str) -> str:
        if event_id not in self.events:
            raise gl.vm.UserError("Event does not exist")

        event = json.loads(self.events[event_id])
        if event["status"] not in ["resolved", "failed"]:
            raise gl.vm.UserError("Only finalized events can be challenged")

        event["challenged"] = True
        event["status"] = "disputed"
        event["challenge_reason"] = reason
        event["challenged_by"] = str(gl.message.sender_address)

        self.events[event_id] = json.dumps(event)
        return "Event challenged and set to disputed"

    @gl.public.view
    def get_event(self, event_id: str) -> typing.Any:
        if event_id not in self.events:
            return None
        return json.loads(self.events[event_id])

    @gl.public.view
    def get_event_count(self) -> u256:
        return self.event_counter