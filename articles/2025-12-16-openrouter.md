---
title: "いま OpenRouter を始めるべき 5 つの理由"
emoji: "🐈‍⬛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenRouter", "LLM", "AI"]
publication_name: "japagate"
published: true
---

:::message
この記事は、ジャパゲートシステムズ Advent Calendar 2025 16 日目の記事です。
:::

AI アプリケーションの開発において、どの LLM プロバイダーを使うか・複数のモデルをどう管理するかは悩ましい問題です。
各プロバイダーの API を個別に管理し、それぞれのレート制限や障害に対応するのは大変な作業になります。

本記事では、私たちのプロダクト開発で実際に OpenRouter を導入して得られた知見をもとに、OpenRouter をおすすめする理由をご紹介します。

## モデルレイヤーで抽象化できる

AI SDK などのライブラリは実装レイヤーでプロバイダーを抽象化しますが、OpenRouter は**モデルレイヤーで抽象化**を提供します。

2025/12/16 現在、OpenRouter は 600 種類弱のモデルをサポートしており、コードの変更なしにモデルを切り替えたり複数のモデルを組み合わせたりできます。

| 観点 | AI SDK (実装レベル抽象化) | OpenRouter (モデルレベル抽象化) |
|------|--------------------------|-------------------------------|
| API キー | プロバイダーごとに必要 | 1つで OK |
| レート制限 | プロバイダーごとに管理 | OpenRouter 側で管理 |
| 新モデル対応 | SDK のアップデートが必要 | 即座に利用可能 |

実際、OpenRouter は [Vercel AI SDK のプロバイダーとしても利用でき](https://openrouter.ai/docs/guides/community/vercel-ai-sdk)、AI SDK の使い勝手と OpenRouter のモデルレベル抽象化の両方の恩恵を受けることができます。

```typescript
import { createOpenRouter } from '@openrouter/ai-sdk-provider';
import { streamText } from 'ai';
import { z } from 'zod';
export const getLasagnaRecipe = async (modelName: string) => {
  const openrouter = createOpenRouter({
    apiKey: '${API_KEY_REF}',
  });
  const response = streamText({
    model: openrouter(modelName),
    prompt: 'Write a vegetarian lasagna recipe for 4 people.',
  });
  await response.consumeStream();
  return response.text;
};
```

レート制限とも被りますが、Tier からの解放も大きなメリットです。

OpenAI などの API を直接使う場合、アカウントのそれまでの使用量に応じて [Tier](https://platform.openai.com/docs/guides/rate-limits) が決まり、レート制限が大きく異なります。
一方 OpenRouter の場合は、free モデルの場合と有料モデルの場合で異なるのですが、『[DDOS レベルのものを緩めに弾く](https://openrouter.ai/docs/api/reference/limits)』程度のものだと理解しています。

## 既存の SDK をそのまま利用できる

**OpenRouter は OpenAI API 互換のエンドポイント**を提供しているため、既存の OpenAI SDK での実装がそのまま使えます。
（[SDK](https://openrouter.ai/docs/guides/community/openai-sdk) も使用できます。）

```typescript
import OpenAI from "openai"

const openai = new OpenAI({
  baseURL: "https://openrouter.ai/api/v1",
  apiKey: "${API_KEY_REF}",
  defaultHeaders: {
    ${getHeaderLines().join('\n        ')}
  },
})

async function main() {
  const completion = await openai.chat.completions.create({
    model: "${Model.GPT_4_Omni}",
    messages: [
      { role: "user", content: "Say this is a test" }
    ],
  })

  console.log(completion.choices[0].message)
}
main();
```

## プラグインで機能を拡張できる

OpenRouter には独自の[プラグイン機能](https://openrouter.ai/docs/guides/features/plugins/overview)があり、現在では以下の 3 つが提供されています。

- [Web Search](https://openrouter.ai/docs/guides/features/plugins/web-search)
- [PDF Inputs](https://openrouter.ai/docs/guides/overview/multimodal/pdfs)
- [Response Healing](https://openrouter.ai/docs/guides/features/plugins/response-healing)

**モデルに依存しない形で機能を提供している**ため、外部の LLM プロバイダーとは別軸で拡張が可能です。

特に Response Healing は面白く、response_format で JSON schema を指定した場合に、スキーマに合致しない出力を自動修正してくれます。

- 閉じブラケットがない
- テキストと JSON が混在している
- 末尾のカンマがついてしまっている
- フィールド名が quote されてない
- markdown code block を消す

など、様々なケースに対して修正を試みてくれます。
（詳しくは公式ドキュメントをご覧ください。）

## モデルフォールバック機能

[モデルフォールバック機能](https://openrouter.ai/docs/guides/routing/model-fallbacks)を使うと、特定のモデルプロバイダーが down してる場合に、別のモデルへ自動的に切り替えることができます。

```typescript
const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer <OPENROUTER_API_KEY>',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    // 配列の 1 つ目から順に試し、エラーがなくなるまで自動的にフォールバック。
    models: ['anthropic/claude-3.5-sonnet', 'gryphe/mythomax-l2-13b'],
    messages: [
      {
        role: 'user',
        content: 'What is the meaning of life?',
      },
    ],
  }),
});

const data = await response.json();
console.log(data.choices[0].message.content);
```

他にもフォールバックを提供するライブラリ等はありますが、**API キーを用意する必要がない分、導入コストが非常に低い**のが魅力です。

## 無料モデルで手軽に試せる

OpenRouter には[無料で使えるモデル（モデル id の末尾に ":free" が付くもの）](https://openrouter.ai/models?fmt=cards&q=free)が大量に用意されています。
（[公開日初日から大量の Activity](https://openrouter.ai/nvidia/nemotron-3-nano-30b-a3b:free/activity) が見られて面白いです。）

結構な頻度で追加されては消えているのですが、コストを気にせず試すことができるのは大きなメリットです。

## まとめ

OpenRouter の魅力は、**1 つの API キーで複数プロバイダーのモデルにアクセスできる点**と、**モデルフォールバックによる可用性向上を手軽に実現できる点**にあると感じています。

AI/LLM の分野は移り変わりが非常に早く、OpenRouter 自体も頻繁に機能追加やモデル対応が行われているため、引き続きキャッチアップしていきます。
