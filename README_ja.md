# llama.cpp with Jev API

**[llama.cppオリジナルのREADMEはこちら](README_original.md)**

## 概要

[llama.cpp](https://github.com/ggml-org/llama.cpp)の`llama-server`に[TypeSafe AI](https://typesafe.ai/)の[JevのAPI](https://docs.typesafe.ai/api)を追加したものです。

特徴：
- 通常のLLMのggufが利用可能です。
- **[Jevの公式API](https://docs.typesafe.ai/api)と互換性**があるため、URLの変更のみで動作します。
- 上記に加え、**マルチモーダル**（**画像**・**音声**）に対応しています。
- 機能の詳細は[SYSTEMONE.md](tools/server/SYSTEMONE.md)を参照してください。
- `llama-server`の他の機能（**OpenAI互換API**など）もそのまま同時に使用できます。
- ビルド方法は通常の`llama-server`のビルド方法に準拠します。
- `llama-server`以外の変更点はありません。

注意点：
- 確率は正確ではありません。
- 項目数に上限はありませんが、多い場合は動作速度に影響が出る場合があります。62項目以下を推奨します。
- 動作速度・正確性は測定しておりません。
- いわゆるバイブコーディングです。コードは十分に整理されているとは限りません。

## 参考

- [llama.cpp](https://github.com/ggml-org/llama.cpp)
- [Jev](https://typesafe.ai/)
- [SemIf](https://github.com/TheoLeeCJ/SemIf) - 動作方法を参照させていただきました（[MIT license](licenses/LICENSE-SemIf)）