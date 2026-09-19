# System One API

`POST /v1/systemone` evaluates typed questions against one state. The request and response follow the [TypeSafe API reference](https://docs.typesafe.ai/api) and [Cloudflare Jev API](https://developers.cloudflare.com/ai/models/typesafe/jev/), with the extensions and probability rules below. This is an API and scoring implementation, not Jev model weights or Jev's probability calibration.

## Running

```sh
llama-server -m model.gguf --alias local-model --ctx-size 8192
```

Use a causal language model with a suitable chat template. Models keep their normal GGUF loading, device, and quantization options. For images or audio, add `--mmproj projector.gguf` and use a model/projector pair that supports the requested modality. A text-only model cannot evaluate media. Router mode forwards this endpoint using the usual `model` field.

```sh
curl http://localhost:8080/v1/systemone -H "Content-Type: application/json" -d '{
  "model": "local-model",
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

`state`, `model`, and `questions` are required. `state` and `instructions` accept strings, objects, or arrays, but not null. Individual criteria also accept null for compatibility with Cloudflare. Questions have non-empty names. Unknown request/question fields are rejected. `model` must be a non-empty string: use the loaded GGUF model's name or alias, or a router model ID. A single-model server returns its loaded model name; specifying `jev-latest` does not load Jev weights.

`choice.criteria` is an object with at least 1 entry. `score.criteria` is an ordered array with at least 2 entries. There is no fixed candidate-count limit; prompts must still fit the slot context and available memory. `noul.criteria` is optional or null; if provided, its only keys are `true` and `false`, and omitted values default to null. A single choice returns probability and confidence 1. An empty `questions` object returns empty answers and zero usage.

## Media state

Ordinary state objects and arrays remain structured evidence, including fields named `content`. In a state object, a `content` array containing an explicit media part enables the chat media extension. Supported parts are `text`, `image_url`, and `input_audio`. Text paths remain available in evidence; media payloads are passed separately to the projector:

```json
{
  "model": "local-model",
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

Options use fixed-width uppercase labels: `A` through `Z` for up to 26 options, `AA`, `BA`, ..., `ZA`, `AB`, `BB`, ... for 27-676 options, and the minimum sufficient width for larger sets. The first character changes fastest. Each uppercase character must add exactly one distinct token to the actual prompt, without changing its prefix tokenization. Later characters are evaluated by appending these character token IDs individually, even if tokenizing the complete label would merge them. An incompatible model/template produces HTTP 422 naming the label. Characters requiring multiple tokens and truncated prompts are not approximated. Choice names and descriptions both enter the prompt; question IDs do not. Choice keys and score items retain request order. Ties select the first mapped option.

For each label position, normalize over the characters used at that position across all labels, regardless of the preceding characters. Evaluate logits after feeding the preceding character tokens into the model. For 27 options, every second-position readout uses both `A` and `B`, including after `B` through `Z`, even though `BB` through `ZB` are not listed options.

Calculate each conditional probability and the label probability in double precision:

```text
conditional(c | prefix) = exp(l[c] - max(l)) / sum_j(exp(l[j] - max(l)))
weight[i] = product_d conditional(label[i][d] | label[i][:d])
p[i] = weight[i] / sum_j(weight[j])
best = argmax(p)
confidence = p[best] * product_{i != best}(1 - p[i])
```

Unlisted combinations participate in the per-position normalization. After multiplying the conditional probabilities, renormalize over the listed candidates so returned probabilities sum to 1, including with 27 or 677 options. Confidence and score use these renormalized probabilities. With 52 options, `weight(BA) = p(B) * p(A | B)`, with the first readout normalized over `A`-`Z` and the second over `A`-`B`.

- `choice`: returns `type`, the original key in `choice`, `probabilities` keyed by the original choices, and `confidence`.
- `score`: numbers the items `0`, `1`, ..., and returns `score = sum(i * p[i])`, `confidence`, `probabilities`, and `legend`. Legend values are strings; structured descriptions are serialized as JSON.
- `noul`: maps `yes` and `no` to the two letter slots and returns only `type` and `noul = (p[yes] + 1 - p[no]) * 0.5`. This equals `p[yes]` because these two probabilities sum to 1.

The response has `model`, `answers`, and `usage`. `usage.input_tokens` sums all evaluated prompts, including media tokens and supplied label prefixes; repeated evidence is counted for every prefix readout. `usage.output_tokens` is zero because no tokens are generated. Question and prefix readouts are queued independently and can share server batches when slots are available; this is not a single joint forward pass for all questions. There is no streaming response.

## Compatibility notes

Input validation returns HTTP 422 with a JSON `error` containing `code`, `type`, and `message`. Schema errors identify the invalid field in the message. Other endpoints retain their existing error status codes.

The official references have some differences. The API table lists string/null choice descriptions and string score legends, while the detailed [choice](https://docs.typesafe.ai/primitives/choice) and [score](https://docs.typesafe.ai/primitives/score) guides show structured descriptions. This endpoint accepts structured criteria and serializes structured score legend values as JSON strings, retaining the Cloudflare/API-reference response shape.

Intentional differences from the hosted service:

- Choice and score have no fixed candidate-count limit and use uppercase character sequences. The official guides describe up to 255 choices and 10 score levels.
- The [official state guide](https://docs.typesafe.ai/concepts/state) describes text input. Image/audio state is a local extension.
- Probabilities and confidence use the requested OpenJev readout and formulas above. They do not reproduce Jev's trained calibration.
- Token usage reflects local per-question prompts and zero generated tokens.
- Authentication follows the server's `--api-key` configuration. Hosted rate limits and service-specific overload responses are not emulated.

## Design and verification

The HTTP layer validates JSON, formats prompts with the existing chat/media parser, validates answer boundaries, and assembles answers. `SERVER_TASK_TYPE_SYSTEMONE` uses the existing queue, cancellation, slots, and batching. At prompt completion the inference thread copies only the requested logits, returns them, and releases the slot. Softmax and response formatting run in the HTTP thread. Prompt caching is disabled for these tasks.

Regression coverage lives in `tests/unit/test_chat_completion.py` under `test_systemone_*`, using the existing server test infrastructure. It checks all answer types and arithmetic, string/object states, 26/27/52/676/677 candidates, conditional multi-character probabilities including unlisted combinations, invalid inputs, unsupported media, repeat requests, subsequent ordinary completion, and agreement with native full-vocabulary probabilities.

Manual acceptance checklist for deployment models:

- Run the example with the intended GGUF and verify all fields and usage.
- Evaluate known labelled cases; measure decision quality separately from API correctness. These probabilities are conditional option scores, not calibrated Jev confidence.
- Submit 26, 27, 52, 676, and 677 candidates and confirm label widths, ordering, and conditional probabilities. Increase `--ctx-size` as needed.
- Confirm partial final groups use the same per-position alphabet for every prefix, then renormalize the returned candidate probabilities to sum to 1.
- Measure latency and memory with realistic candidate counts: every distinct label prefix requires a separate prompt evaluation, with prompt caching disabled.
- With a vision model/projector, compare real image inputs and verify image changes affect the scores.
- With an audio model/projector, submit real WAV input and verify audio changes affect the scores.
- For mixed input, use a model that supports both image and audio.
- Check concurrent questions, client cancellation, and router `model` selection under deployment load.
- Verify the request fails when the context is too small or a modality is unsupported.
