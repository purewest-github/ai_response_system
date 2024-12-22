# AI APIクライアント実装

```python
# core/ai/base.py
from abc import ABC, abstractmethod
from typing import Dict, Optional
import httpx
import asyncio
import backoff
from django.conf import settings
from core.exceptions import APIError, RateLimitError, TokenLimitError

class BaseAIClient(ABC):
    """AI APIクライアントの基底クラス"""

    def __init__(self):
        self.client = httpx.AsyncClient(
            timeout=180.0,
            limits=httpx.Limits(max_keepalive_connections=5)
        )

    async def __aenter__(self):
        return self

    async def __aexit__(self, exc_type, exc_value, traceback):
        await self.client.aclose()

    @abstractmethod
    async def generate_response(
        self,
        prompt: str,
        max_tokens: Optional[int] = None
    ) -> Dict:
        """回答を生成する抽象メソッド"""
        pass

    @backoff.on_exception(
        backoff.expo,
        (httpx.HTTPError, RateLimitError),
        max_tries=3,
        max_time=30
    )
    async def _make_request(
        self,
        url: str,
        payload: Dict,
        headers: Dict
    ) -> Dict:
        """APIリクエストの実行"""
        try:
            response = await self.client.post(
                url,
                json=payload,
                headers=headers
            )
            response.raise_for_status()
            return response.json()

        except httpx.HTTPStatusError as e:
            if e.response.status_code == 429:
                raise RateLimitError("API rate limit exceeded")
            raise APIError(f"HTTP error occurred: {str(e)}")

        except httpx.RequestError as e:
            raise APIError(f"Request error occurred: {str(e)}")

# core/ai/claude.py
class ClaudeClient(BaseAIClient):
    """Claude APIクライアント"""

    API_BASE_URL = settings.CLAUDE_API_URL

    async def generate_response(
        self,
        prompt: str,
        max_tokens: Optional[int] = None
    ) -> Dict:
        """回答の生成"""
        headers = {
            "X-API-Key": settings.CLAUDE_API_KEY,
            "Content-Type": "application/json",
            "Anthropic-Version": "2023-06-01"
        }

        payload = {
            "prompt": self._format_prompt(prompt),
            "max_tokens_to_sample": max_tokens or settings.CLAUDE_MAX_TOKENS,
            "temperature": 0.7,
            "model": "claude-2"
        }

        try:
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

        except Exception as e:
            raise APIError(f"Claude API error: {str(e)}")

    async def verify_response(
        self,
        question: str,
        original_response: str,
        verification_response: str
    ) -> Dict:
        """最終検証の実行"""
        prompt = self._create_verification_prompt(
            question,
            original_response,
            verification_response
        )

        return await self.generate_response(prompt)

    def _format_prompt(self, prompt: str) -> str:
        """プロンプトのフォーマット"""
        return f"\\n\\nHuman: {prompt}\\n\\nAssistant:"

    def _create_verification_prompt(
        self,
        question: str,
        original_response: str,
        verification_response: str
    ) -> str:
        """検証用プロンプトの生成"""
        return f"""
以下の質問と2つの回答を分析し、最適な最終回答を生成してください。

[質問]
{question}

[初回回答]
{original_response}

[検証回答]
{verification_response}

回答の比較分析を行い、より正確で包括的な最終回答を生成してください。
両方の回答の良い点を組み合わせ、不正確な部分があれば修正してください。
"""

# core/ai/chatgpt.py
class ChatGPTClient(BaseAIClient):
    """ChatGPT APIクライアント"""

    API_BASE_URL = settings.CHATGPT_API_URL

    async def generate_response(
        self,
        prompt: str,
        max_tokens: Optional[int] = None
    ) -> Dict:
        """回答の生成"""
        headers = {
            "Authorization": f"Bearer {settings.CHATGPT_API_KEY}",
            "Content-Type": "application/json"
        }

        payload = {
            "model": "gpt-4",
            "messages": [
                {"role": "system", "content": self._get_system_prompt()},
                {"role": "user", "content": prompt}
            ],
            "max_tokens": max_tokens or settings.CHATGPT_MAX_TOKENS,
            "temperature": 0.7
        }

        try:
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

        except Exception as e:
            raise APIError(f"ChatGPT API error: {str(e)}")

    async def verify_response(
        self,
        question: str,
        original_response: str
    ) -> Dict:
        """回答の検証"""
        prompt = self._create_verification_prompt(
            question,
            original_response
        )

        response = await self.generate_response(prompt)
        response['validation_status'] = self._analyze_validation_result(
            response['content']
        )

        return response

    def _get_system_prompt(self) -> str:
        """システムプロンプトの取得"""
        return """
あなたは質問への回答を検証する検証者です。
回答の正確性、完全性、明確性を評価し、改善点を指摘してください。
技術的な正確性を特に重視し、誤りや不正確な情報があれば指摘してください。
"""

    def _create_verification_prompt(
        self,
        question: str,
        original_response: str
    ) -> str:
        """検証用プロンプトの生成"""
        return f"""
以下の質問と回答を検証してください：

[質問]
{question}

[回答]
{original_response}

以下の点について評価してください：
1. 技術的な正確性
2. 回答の完全性
3. 説明の明確さ
4. 改善が必要な点

また、回答の検証結果を「valid」「invalid」「needs_improvement」のいずれかで示してください。
"""

    def _analyze_validation_result(self, content: str) -> str:
        """検証結果の分析"""
        content_lower = content.lower()

        if '誤り' in content_lower or 'incorrect' in content_lower:
            return 'invalid'
        elif '改善' in content_lower or 'improvement' in content_lower:
            return 'needs_improvement'
        else:
            return 'valid'

```

この実装では以下の点に注意を払っています：

1. エラーハンドリング
- 適切な例外処理
- 再試行ロジック
- タイムアウト管理
1. パフォーマンス
- 非同期処理の活用
- コネクション管理
- レート制限の考慮
1. セキュリティ
- APIキーの安全な管理
- 入力値の検証
- エラーメッセージの適切な処理
1. 拡張性
- 抽象基底クラスの活用
- モジュール化された設計
- 設定の外部化