# System One API

`POST /v1/systemone` evaluates typed questions against one state. The request and response follow the [Cloudflare Jev API](https://developers.cloudflare.com/ai/models/typesafe/jev/), with the probability and confidence rules below. This is an API and scoring implementation, not Jev model weights or Jev's probability calibration.

## Running

```sh
llama-server -m model.gguf --ctx-size 8192
```

Use a causal language model with a suitable chat template. Models keep their normal GGUF loading, device, and quantization options. For images or audio, add `--mmproj projector.gguf` and use a model/projector pair that supports the requested modality. A text-only model cannot evaluate media. Router mode forwards this endpoint using the usual `model` field.

```sh
curl http://localhost:8080/v1/systemone -H "Content-Type: application/json" -d '{
  "state": "My payouts have failed for three days. Please help today.",
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which team should handle this?",
      "criteria": {"billing": "Payments and refunds", "technical": "Bugs and integrations"}
    },
    "frustration": {
      "type": "score",
      "instructions": "How frustrated is the customer?",
      "criteria": ["Calm", "Frustrated", "Very angry"]
    },
    "urgent": {
      "type": "noul",
      "instructions": "Is this urgent?",
      "criteria": {"true": "Time-sensitive", "false": "No urgency"}
    }
  }
}'
```

`state` accepts a normal string or structured JSON (object, array, or null). `instructions` and individual criteria accept strings, objects, arrays, or null, as in the Cloudflare schema. Questions have non-empty names. Unknown request/question fields are rejected. `model` is an optional extension for the standard server router; a single-model server returns the loaded model name.

`choice.criteria` is an object with 1-26 entries. `score.criteria` is an ordered array with 2-26 entries. `noul.criteria` is optional or null; if provided, its only keys are `true` and `false`, and omitted values default to null. A single choice returns probability and confidence 1. An empty `questions` object returns empty answers and zero usage.

## Media state

In a state object, an array at `content` is reserved for the server's chat content parts. Other state fields remain structured evidence. Supported parts are `text`, `image_url`, and `input_audio`:

```json
{
  "state": {
    "ticket_id": "A-104",
    "content": [
      {"type": "text", "text": "Assess the visible damage and the spoken complaint."},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}},
      {"type": "input_audio", "input_audio": {"data": "...", "format": "wav"}}
    ]
  },
  "questions": {
    "damaged": {"type": "noul", "instructions": "Is the item damaged?"}
  }
}
```

Image URLs, audio data, supported encodings, and `--media-path` follow the existing chat API. `input_audio.data` can contain raw base64 or a data URI; `input_audio.url` is also accepted. Local files require `--media-path`. Media is passed to the projector, never silently converted to text. Unsupported modalities, malformed parts, and unknown part types fail the request. A model must support both modalities to accept mixed image/audio input.

## Scoring

The scorer ports OpenJev's direct next-token readout. Each question gets one prompt containing evidence, instructions, and letter-labelled options. It applies the loaded model's chat template with thinking disabled, evaluates the prompt, and reads the final position's native logits. It does not generate an answer, use a grammar, or sample tokens. Sampling options such as temperature, top-k, penalties, and logit bias are not applied.

Options use only uppercase `A` through `Z`, with a maximum of 26. Each label must add exactly one distinct token to the actual prompt, without changing its prefix tokenization. An incompatible model/template produces HTTP 400 naming the label. Multi-token labels and truncated prompts are not approximated. Choice keys and score items retain request order. Ties select the first mapped option.

For the selected logits `l[i]`, calculate in double precision:

```text
p[i] = exp(l[i] - max(l)) / sum_j(exp(l[j] - max(l)))
best = argmax(p)
confidence = p[best] * product_{i != best}(1 - p[i])
```

- `choice`: returns `type`, the original key in `choice`, `probabilities` keyed by the original choices, and `confidence`.
- `score`: numbers the items `0`, `1`, ..., and returns `score = sum(i * p[i])`, `confidence`, `probabilities`, and `legend`. Legend values are strings; structured descriptions are serialized as JSON.
- `noul`: maps `yes` and `no` to the two letter slots and returns only `type` and `noul = (p[yes] + 1 - p[no]) * 0.5`. This equals `p[yes]` because these two probabilities sum to 1.

The response has `model`, `answers`, and `usage`. `usage.input_tokens` sums the question prompts, including media tokens; shared evidence is counted for each question. `usage.output_tokens` is zero because no tokens are generated. Questions are queued independently and can share server batches when slots are available; this is not a single joint forward pass for all questions. There is no streaming response.

## Design and verification

The HTTP layer validates JSON, formats prompts with the existing chat/media parser, validates answer boundaries, and assembles answers. `SERVER_TASK_TYPE_SYSTEMONE` uses the existing queue, cancellation, slots, and batching. At prompt completion the inference thread copies only the requested logits, returns them, and releases the slot. Softmax and response formatting run in the HTTP thread. Prompt caching is disabled for these tasks.

Regression coverage lives in `tests/unit/test_chat_completion.py` under `test_systemone_*`, using the existing server test infrastructure. It checks all answer types and arithmetic, string/object states, 26 candidates, invalid inputs, unsupported media, repeat requests, subsequent ordinary completion, and agreement with native full-vocabulary probabilities.

Manual acceptance checklist for deployment models:

- Run the example with the intended GGUF and verify all fields and usage.
- Evaluate known labelled cases; measure decision quality separately from API correctness. These probabilities are conditional option scores, not calibrated Jev confidence.
- Submit 26 candidates and confirm the model/template accepts all uppercase labels.
- With a vision model/projector, compare real image inputs and verify image changes affect the scores.
- With an audio model/projector, submit real WAV input and verify audio changes affect the scores.
- For mixed input, use a model that supports both image and audio.
- Check concurrent questions, client cancellation, and router `model` selection under deployment load.
- Verify the request fails when the context is too small or a modality is unsupported.
