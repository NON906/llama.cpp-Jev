# llama.cpp with Jev API

**[Read the original llama.cpp README here](README_original.md)**

## Overview

This project adds [TypeSafe AI](https://typesafe.ai/)'s [Jev API](https://docs.typesafe.ai/api) to [llama.cpp](https://github.com/ggml-org/llama.cpp)'s `llama-server`.

Features:

- Standard LLM GGUF models can be used.
- **Compatible with the [official Jev API](https://docs.typesafe.ai/api)**, so only the URL needs to be changed.
- Also supports **multimodal** input (**images** and **audio**).
- See [SYSTEMONE.md](tools/server/SYSTEMONE.md) for feature details.
- Other `llama-server` features, such as the **OpenAI-compatible API**, remain available and can be used at the same time.
- Follow the standard `llama-server` build instructions.
- There are no changes outside `llama-server`.

Caveats:

- Probabilities are not accurate.
- There is no limit on the number of items, but larger numbers may affect performance. Using 62 items or fewer is recommended.
- Performance and accuracy have not been measured.
- This project was developed through so-called vibe coding. The code may not be fully organized.

## References

- [llama.cpp](https://github.com/ggml-org/llama.cpp)
- [Jev](https://typesafe.ai/)
- [SemIf](https://github.com/TheoLeeCJ/SemIf) - Used as a reference for how the implementation works. See its [MIT license](licenses/LICENSE-SemIf).
