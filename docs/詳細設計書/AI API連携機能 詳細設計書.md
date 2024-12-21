# AI API連携機能 詳細設計書

## 1. AIクライアント実装

### 1.1 基底クライアント

```python
# core/ai/base.py
import httpx
from typing import Dict, Optional
from abc import ABC, abstractmethod
from django.conf import settings
from accounts.services import APIKeyService

class BaseAIClient(ABC):
    def __init__(self):
        self.api_key_service = APIKeyService()
        self.client = httpx.AsyncClient(timeout=180.0)

    async def __aenter__(self):
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.client.aclose()

    @abstractmethod
    async def generate_response(
        self,
        prompt: str,
        max_tokens: Optional[int] = None
    ) -> Dict:
        pass

    async def _make_request(
        self,
        url: str,
        payload: Dict,
        headers: Dict,
        retry_count: int = 3
    ) -> Dict:
        """API リクエスト実行（リトライ機能付き）"""
        for attempt in range(retry_count):
            try:
                response = await self.client.post(
                    url,
                    json=payload,
                    headers=headers
                )
                response.raise_for_status()
                return response.json()
            except httpx.HTTPStatusError as e:
                if e.response.status_code == 429:  # Rate limit
                    if attempt < retry_count - 1:
                        await asyncio.sleep(2 ** attempt)
                        continue
                raise

```

### 1.2 Claude APIクライアント

```python
# core/ai/claude.py
from typing import Dict, Optional
from .base import BaseAIClient

class ClaudeClient(BaseAIClient):
    API_BASE_URL = "<https://api.anthropic.com/v1>"

    async def generate_response(
        self,
        prompt: str,
        max_tokens: Optional[int] = None
    ) -> Dict:
        """Claude APIによる回答生成"""
        api_key = await self.api_key_service.get_active_key(
            user_id, 'claude'
        )

        headers = {
            "X-API-Key": api_key,
            "Content-Type": "application/json"
        }

        payload = {
            "prompt": prompt,
            "max_tokens_to_sample": max_tokens or 1000,
            "temperature": 0.7,
            "model": "claude-2"
        }

        response = await self._make_request(
            f"{self.API_BASE_URL}/complete",
            payload,
            headers
        )

        return {
            'content': response['completion'],
            'token_count': len(response['completion'].split()),
            'model': response['model']
        }

    async def verify_response(
        self,
        question: str,
        original_response: str,
        verification_response: str
    ) -> Dict:
        """回答の最終検証"""
        prompt = self._create_verification_prompt(
            question,
            original_response,
            verification_response
        )

        return await self.generate_response(prompt)

    def _create_verification_prompt(
        self,
        question: str,
        original_response: str,
        verification_response: str
    ) -> str:
        return f"""
        質問: {question}

        初回回答:
        {original_response}

        検証回答:
        {verification_response}

        上記の2つの回答を比較・分析し、最も適切な最終回答を生成してください。
        """

```

### 1.3 ChatGPT APIクライアント

```python
# core/ai/chatgpt.py
from typing import Dict, Optional
from .base import BaseAIClient

class ChatGPTClient(BaseAIClient):
    API_BASE_URL = "<https://api.openai.com/v1>"

    async def generate_response(
        self,
        prompt: str,
        max_tokens: Optional[int] = None
    ) -> Dict:
        """ChatGPT APIによる回答生成"""
        api_key = await self.api_key_service.get_active_key(
            user_id, 'chatgpt'
        )

        headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

        payload = {
            "model": "gpt-4",
            "messages": [
                {
                    "role": "user",
                    "content": prompt
                }
            ],
            "max_tokens": max_tokens or 1000,
            "temperature": 0.7
        }

        response = await self._make_request(
            f"{self.API_BASE_URL}/chat/completions",
            payload,
            headers
        )

        return {
            'content': response['choices'][0]['message']['content'],
            'token_count': response['usage']['total_tokens'],
            'model': response['model']
        }

    async def verify_response(
        self,
        question: str,
        original_response: str
    ) -> Dict:
        """Claudeの回答を検証"""
        prompt = self._create_verification_prompt(
            question,
            original_response
        )

        response = await self.generate_response(prompt)
        response['validation_status'] = self._analyze_validation(
            response['content']
        )

        return response

    def _create_verification_prompt(
        self,
        question: str,
        original_response: str
    ) -> str:
        return f"""
        以下の質問と回答を検証してください。

        質問:
        {question}

        回答:
        {original_response}

        回答について、以下の点を評価してください：
        1. 正確性
        2. 完全性
        3. 明確性
        4. 改善点
        """

    def _analyze_validation(self, content: str) -> str:
        """検証結果の分析"""
        # 簡易的な判定ロジック
        lower_content = content.lower()
        if '誤り' in lower_content or 'incorrect' in lower_content:
            return 'invalid'
        elif '改善' in lower_content or 'improvement' in lower_content:
            return 'needs_improvement'
        else:
            return 'valid'

```

## 2. Token管理

```python
# core/ai/token_counter.py
from typing import Dict
import tiktoken

class TokenCounter:
    def __init__(self):
        self.claude_encoder = tiktoken.get_encoding("cl100k_base")
        self.gpt_encoder = tiktoken.encoding_for_model("gpt-4")

    def count_tokens(
        self,
        text: str,
        model: str = 'claude'
    ) -> int:
        """トークン数のカウント"""
        encoder = (
            self.claude_encoder if model == 'claude'
            else self.gpt_encoder
        )
        return len(encoder.encode(text))

    def check_token_limit(
        self,
        text: str,
        model: str = 'claude'
    ) -> Dict:
        """トークン制限のチェック"""
        token_count = self.count_tokens(text, model)
        limit = 200000 if model == 'claude' else 128000

        return {
            'within_limit': token_count <= limit,
            'token_count': token_count,
            'token_limit': limit
        }

```

## 3. エラーハンドリング

```python
# core/ai/exceptions.py
class AIClientError(Exception):
    """AI API関連の基底例外クラス"""
    pass

class TokenLimitError(AIClientError):
    """トークン制限超過エラー"""
    pass

class APIKeyError(AIClientError):
    """APIキー関連エラー"""
    pass

class RateLimitError(AIClientError):
    """レート制限エラー"""
    pass

class ResponseValidationError(AIClientError):
    """回答検証エラー"""
    pass

# core/ai/error_handler.py
import logging
from typing import Optional, Callable
from functools import wraps

logger = logging.getLogger(__name__)

def handle_ai_errors(
    retry_count: int = 3,
    fallback_handler: Optional[Callable] = None
):
    """AI API エラーハンドリングデコレータ"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_error = None

            for attempt in range(retry_count):
                try:
                    return await func(*args, **kwargs)
                except RateLimitError:
                    await asyncio.sleep(2 ** attempt)
                    continue
                except TokenLimitError as e:
                    logger.error(f"Token limit exceeded: {str(e)}")
                    raise
                except APIKeyError as e:
                    logger.error(f"API key error: {str(e)}")
                    raise
                except Exception as e:
                    last_error = e
                    logger.error(
                        f"AI API error (attempt {attempt + 1}): {str(e)}"
                    )
                    if attempt == retry_count - 1:
                        if fallback_handler:
                            return await fallback_handler(*args, **kwargs)
                        raise

            raise last_error
        return wrapper
    return decorator

```

## 4. メトリクス収集

```python
# core/ai/metrics.py
import time
from dataclasses import dataclass
from typing import Optional
from django.core.cache import cache

@dataclass
class APIMetrics:
    total_requests: int = 0
    total_tokens: int = 0
    total_time: float = 0
    error_count: int = 0
    average_response_time: float = 0

class MetricsCollector:
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.cache_key = f"ai_metrics_{service_name}"

    async def record_request(
        self,
        tokens: int,
        response_time: float,
        error: Optional[Exception] = None
    ):
        """メトリクスの記録"""
        metrics = self._get_current_metrics()

        metrics.total_requests += 1
        metrics.total_tokens += tokens
        metrics.total_time += response_time

        if error:
            metrics.error_count += 1

        metrics.average_response_time = (
            metrics.total_time / metrics.total_requests
        )

        await self._save_metrics(metrics)

    def _get_current_metrics(self) -> APIMetrics:
        """現在のメトリクスを取得"""
        cached = cache.get(self.cache_key)
        return cached if cached else APIMetrics()

    async def _save_metrics(self, metrics: APIMetrics):
        """メトリクスの保存"""
        cache.set(self.cache_key, metrics, timeout=86400)  # 24時間

```

## 5. テスト

```python
# core/ai/tests/test_clients.py
import pytest
from unittest.mock import patch, AsyncMock
from core.ai.claude import ClaudeClient
from core.ai.chatgpt import ChatGPTClient
from core.ai.exceptions import TokenLimitError

@pytest.mark.asyncio
class TestClaudeClient:
    async def test_generate_response(self):
        with patch('httpx.AsyncClient.post') as mock_post:
            mock_post.return_value.json.return_value = {
                'completion': 'Test response',
                'model': 'claude-2'
            }

            client = ClaudeClient()
            response = await client.generate_response(
                "Test prompt"
            )

            assert response['content'] == 'Test response'
            assert 'token_count' in response

    async def test_token_limit_error(self):
        with patch('core.ai.token_counter.TokenCounter.check_token_limit') as mock_check:
            mock_check.return_value = {
                'within_limit': False,
                'token_count': 250000,
                'token_limit': 200000
            }

            client = ClaudeClient()
            with pytest.raises(TokenLimitError):
                await client.generate_response(
                    "Test prompt" * 100000
                )

# core/ai/tests/test_metrics.py
@pytest.mark.asyncio
class TestMetricsCollector:
    async def test_record_metrics(self):
        collector = MetricsCollector('test')

        await collector.record_request(
            tokens=100,
            response_time=1.5
        )

        metrics = collector._get_current_metrics()
        assert metrics.total_requests == 1
        assert metrics.total_tokens == 100
        assert metrics.average_response_time == 1.5

```

この詳細設計では、以下の点に特に注意を払っています：

1. エラーハンドリング
- 包括的な例外処理
- リトライ機構の実装
- フォールバック処理の提供
1. パフォーマンス
- 非同期処理の活用
- トークン管理の最適化
- メトリクス収集による監視
1. 保守性
- モジュール化された設計
- テストの充実
- ログ管理の実装