---
title: マルチモデルルーティング実装入門：複数LLMを賢く使い分けるアーキテクチャ設計
tags:
  - Python
  - アーキテクチャ
  - AI
  - OpenAI
  - LLM
private: false
updated_at: '2026-05-04T09:25:46+09:00'
id: 3f91489b5565d2424d14
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに：「どのモデルを使うか」の意思決定をコードに落とす

2026年、LLMの選択肢は爆発的に増えた。GPT-5.5、Claude Opus 4.7、Gemini Ultra 2……それぞれ得意領域が異なり、コスト・レイテンシ・精度のトレードオフも様々だ。

「とりあえずGPT-4に投げる」という時代は終わりつつある。現場では既に「タスクの性質に応じてモデルを切り替える」マルチモデルルーティングが実務的な課題になっている。

本記事では、**Python で実際に動くマルチモデルルーターを実装しながら**、設計上の判断ポイントを解説する。

---

## なぜマルチモデルルーティングが必要か

### コスト構造の非対称性

| ユースケース | 適切なモデル | 理由 |
|---|---|---|
| 定型文の分類・要約 | 小型・安価モデル | 過剰スペックは無駄 |
| 複雑なコード生成 | 高性能モデル | 精度が直接ビジネス影響 |
| ストリーミング応答UI | 低レイテンシモデル | UX優先 |
| 数学・論理推論 | 推論特化モデル | 能力差が顕著 |

全リクエストを最高性能モデルに投げると、コストは最適比で **10〜100倍**になることも珍しくない。

### 可用性とフォールバック

特定プロバイダーのAPIが落ちたとき、代替モデルへ自動切り替えできる仕組みは運用上必須だ。

---

## アーキテクチャ設計

```
[Request]
    ↓
[Router Layer]
  ├─ Task Classifier（タスク種別判定）
  ├─ Cost Controller（コスト制限チェック）
  └─ Availability Checker（死活監視）
    ↓
[Model Pool]
  ├─ Provider A (OpenAI)
  ├─ Provider B (Anthropic)
  └─ Provider C (Local/OSS)
    ↓
[Response Normalizer]（インターフェース統一）
    ↓
[Response]
```

ルーターの責務は「どのモデルに投げるか」の決定だけ。モデルの呼び出し実装とは分離する。

---

## 実装：ステップバイステップ

### Step 1: 統一インターフェースの定義

まず、モデルごとの差異を吸収するアダプター層を作る。

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class LLMRequest:
    prompt: str
    max_tokens: int = 1024
    temperature: float = 0.7
    system_prompt: Optional[str] = None

@dataclass
class LLMResponse:
    content: str
    model_used: str
    input_tokens: int
    output_tokens: int
    latency_ms: float

class LLMProvider(ABC):
    @abstractmethod
    async def complete(self, request: LLMRequest) -> LLMResponse:
        pass

    @abstractmethod
    def estimate_cost(self, request: LLMRequest) -> float:
        """USD単位でのコスト見積もり"""
        pass
```

### Step 2: プロバイダー実装（OpenAI例）

```python
import time
import asyncio
from openai import AsyncOpenAI

class OpenAIProvider(LLMProvider):
    # 2026年5月時点の参考レート（要確認・更新）
    PRICING = {
        "gpt-4o": {"input": 2.5e-6, "output": 10e-6},
        "gpt-4o-mini": {"input": 0.15e-6, "output": 0.6e-6},
    }

    def __init__(self, model: str = "gpt-4o-mini"):
        self.client = AsyncOpenAI()
        self.model = model

    async def complete(self, request: LLMRequest) -> LLMResponse:
        start = time.monotonic()
        messages = []
        if request.system_prompt:
            messages.append({"role": "system", "content": request.system_prompt})
        messages.append({"role": "user", "content": request.prompt})

        response = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            max_tokens=request.max_tokens,
            temperature=request.temperature,
        )

        latency = (time.monotonic() - start) * 1000
        return LLMResponse(
            content=response.choices[0].message.content,
            model_used=self.model,
            input_tokens=response.usage.prompt_tokens,
            output_tokens=response.usage.completion_tokens,
            latency_ms=latency,
        )

    def estimate_cost(self, request: LLMRequest) -> float:
        pricing = self.PRICING.get(self.model, {"input": 1e-5, "output": 3e-5})
        # トークン数を文字数/4で粗く見積もる
        est_input = len(request.prompt) / 4
        est_output = request.max_tokens * 0.5
        return est_input * pricing["input"] + est_output * pricing["output"]
```

### Step 3: タスク分類器

ルーティングの判断基準となるタスク分類。シンプルにルールベースから始めるのが現実的だ。

```python
from enum import Enum
import re

class TaskType(Enum):
    SIMPLE_QA = "simple_qa"          # 短い質問応答
    CODE_GENERATION = "code_gen"     # コード生成
    REASONING = "reasoning"          # 複雑な推論
    SUMMARIZATION = "summarization"  # 要約
    CREATIVE = "creative"            # 創作系

class TaskClassifier:
    CODE_PATTERNS = re.compile(
        r'\b(コード|実装|関数|クラス|デバッグ|Python|TypeScript|SQL)\b',
        re.IGNORECASE
    )
    REASONING_PATTERNS = re.compile(
        r'\b(なぜ|理由|分析|比較|証明|論理|推論|考えて)\b'
    )

    def classify(self, prompt: str) -> TaskType:
        if self.CODE_PATTERNS.search(prompt):
            return TaskType.CODE_GENERATION
        if self.REASONING_PATTERNS.search(prompt):
            return TaskType.REASONING
        if len(prompt) > 500 and ('要約' in prompt or 'まとめ' in prompt):
            return TaskType.SUMMARIZATION
        return TaskType.SIMPLE_QA
```

### Step 4: ルーター本体

```python
from typing import Dict, List
import logging

logger = logging.getLogger(__name__)

class ModelRouter:
    def __init__(self, providers: Dict[str, LLMProvider]):
        self.providers = providers
        self.classifier = TaskClassifier()
        # タスク種別 → 優先プロバイダーのマッピング
        self.routing_rules: Dict[TaskType, List[str]] = {
            TaskType.SIMPLE_QA:       ["gpt-4o-mini", "claude-haiku"],
            TaskType.CODE_GENERATION: ["gpt-4o", "claude-opus"],
            TaskType.REASONING:       ["claude-opus", "gpt-4o"],
            TaskType.SUMMARIZATION:   ["gpt-4o-mini", "claude-haiku"],
            TaskType.CREATIVE:        ["claude-opus", "gpt-4o"],
        }

    async def route(
        self,
        request: LLMRequest,
        force_model: Optional[str] = None,
        max_cost_usd: float = 0.05,
    ) -> LLMResponse:
        if force_model:
            return await self._call_with_fallback([force_model], request)

        task_type = self.classifier.classify(request.prompt)
        priority_list = self.routing_rules.get(task_type, ["gpt-4o-mini"])

        # コスト上限チェックで高コストモデルをフィルタ
        affordable = [
            name for name in priority_list
            if name in self.providers
            and self.providers[name].estimate_cost(request) <= max_cost_usd
        ]

        if not affordable:
            logger.warning("コスト上限でフィルタ後に候補なし。最初のモデルで強行します")
            affordable = priority_list[:1]

        logger.info(f"TaskType={task_type.value}, routing to {affordable[0]}")
        return await self._call_with_fallback(affordable, request)

    async def _call_with_fallback(
        self, model_names: List[str], request: LLMRequest
    ) -> LLMResponse:
        last_exc = None
        for name in model_names:
            if name not in self.providers:
                continue
            try:
                return await self.providers[name].complete(request)
            except Exception as e:
                logger.warning(f"{name} failed: {e}. Trying next...")
                last_exc = e
        raise RuntimeError(f"All providers failed. Last: {last_exc}")
```

### Step 5: 動作確認

```python
import asyncio

async def main():
    providers = {
        "gpt-4o":      OpenAIProvider(model="gpt-4o"),
        "gpt-4o-mini": OpenAIProvider(model="gpt-4o-mini"),
        # Anthropicプロバイダーも同様に実装して追加
    }

    router = ModelRouter(providers)

    # シンプルなQA → gpt-4o-mini にルーティングされるはず
    req1 = LLMRequest(prompt="Pythonのリスト内包表記を教えて")
    resp1 = await router.route(req1)
    print(f"[{resp1.model_used}] {resp1.content[:80]}...")
    print(f"  コスト概算: ${resp1.input_tokens * 0.15e-6:.6f}")

    # コード生成 → gpt-4o にルーティング
    req2 = LLMRequest(prompt="Pythonで非同期HTTPクライアントを実装してください")
    resp2 = await router.route(req2)
    print(f"[{resp2.model_used}] {resp2.content[:80]}...")

asyncio.run(main())
```

---

## 設計上の注意点

### ルーター自体を複雑にしすぎない

タスク分類に高性能LLMを使うと**ルーティングコスト > 節約コスト**になる逆転現象が起きる。最初はルールベースで十分。分類精度が問題になってから機械学習的アプローチを検討すること。

### メトリクス収集を最初から組み込む

```python
# 各レスポンスにルーティング理由を付与するだけでデバッグが激的に楽になる
@dataclass
class LLMResponse:
    ...
    routing_reason: str = ""  # 例: "task=code_gen, selected=gpt-4o"
```

### フォールバックのループに注意

AがBにフォールバック → BがAにフォールバック、という設定を防ぐため、フォールバックチェーンは有向グラフで管理するのが安全だ。

---

## まとめ

マルチモデルルーティングの要点を整理する：

1. **統一インターフェース**でプロバイダー差異を吸収する
2. **タスク分類はシンプルに始める**（ルールベース → ML）
3. **コスト見積もり**をルーティング判断に組み込む
4. **フォールバックチェーン**で可用性を担保する
5. **メトリクス**を最初から収集し、ルーティング品質を継続改善する

「特定モデルへの依存」は技術的負債になりやすい。抽象化層を設けておくことで、新モデルへの移行コストを大幅に下げられる。今のうちにルーター層を設計しておくことが、2026年以降のAI活用において競争優位につながるだろう。
