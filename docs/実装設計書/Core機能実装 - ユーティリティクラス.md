# Core機能実装 - ユーティリティクラス

```python
# core/utils.py
import re
import json
import bleach
import markdown
from typing import Any, Dict, Optional
from django.conf import settings
from django.core.cache import cache
from django.utils.text import slugify
from cryptography.fernet import Fernet

class Encryptor:
    """データ暗号化ユーティリティ"""

    def __init__(self):
        self.fernet = Fernet(settings.ENCRYPTION_KEY)

    def encrypt(self, data: str) -> str:
        """文字列の暗号化"""
        return self.fernet.encrypt(data.encode()).decode()

    def decrypt(self, encrypted_data: str) -> str:
        """文字列の復号化"""
        return self.fernet.decrypt(encrypted_data.encode()).decode()

    @classmethod
    def generate_key(cls) -> bytes:
        """新しい暗号化キーの生成"""
        return Fernet.generate_key()

class MarkdownConverter:
    """マークダウン変換ユーティリティ"""

    ALLOWED_TAGS = [
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'p', 'div', 'span',
        'ul', 'ol', 'li',
        'blockquote', 'pre', 'code',
        'hr', 'br',
        'table', 'thead', 'tbody', 'tr', 'th', 'td',
        'a', 'img',
        'strong', 'em', 'del'
    ]

    ALLOWED_ATTRIBUTES = {
        'a': ['href', 'title', 'target'],
        'img': ['src', 'alt', 'title'],
        'code': ['class'],
        '*': ['class']
    }

    def __init__(self):
        self.md = markdown.Markdown(
            extensions=[
                'extra',
                'codehilite',
                'tables',
                'fenced_code',
                'toc'
            ]
        )

    def convert(self, content: str) -> str:
        """マークダウンをHTMLに変換"""
        # マークダウンの変換
        html = self.md.convert(content)

        # HTMLのサニタイズ
        clean_html = bleach.clean(
            html,
            tags=self.ALLOWED_TAGS,
            attributes=self.ALLOWED_ATTRIBUTES,
            strip=True
        )

        # コードブロックのシンタックスハイライト用クラスを保持
        clean_html = self._preserve_code_classes(clean_html)

        return clean_html

    def _preserve_code_classes(self, html: str) -> str:
        """コードブロックのクラスを保持"""
        code_pattern = r'<code class="([^"]+)">'
        preserved_html = re.sub(
            code_pattern,
            lambda m: f'<code class="{m.group(1)}">',
            html
        )
        return preserved_html

class CacheManager:
    """キャッシュ管理ユーティリティ"""

    @staticmethod
    def get_or_set(
        key: str,
        func: callable,
        timeout: int = 3600,
        force_refresh: bool = False
    ) -> Any:
        """キャッシュの取得または設定"""
        if force_refresh:
            cache.delete(key)

        value = cache.get(key)
        if value is None:
            value = func()
            cache.set(key, value, timeout)

        return value

    @staticmethod
    def invalidate_pattern(pattern: str) -> None:
        """パターンに一致するキャッシュの無効化"""
        keys = cache.keys(pattern)
        cache.delete_many(keys)

class RateLimiter:
    """レート制限ユーティリティ"""

    def __init__(
        self,
        key_prefix: str,
        limit: int,
        period: int
    ):
        self.key_prefix = key_prefix
        self.limit = limit
        self.period = period

    def is_allowed(self, identifier: str) -> bool:
        """レート制限のチェック"""
        cache_key = f"{self.key_prefix}:{identifier}"
        count = cache.get(cache_key, 0)

        if count >= self.limit:
            return False

        if count == 0:
            cache.set(cache_key, 1, self.period)
        else:
            cache.incr(cache_key)

        return True

    def get_remaining(self, identifier: str) -> int:
        """残りのリクエスト数を取得"""
        cache_key = f"{self.key_prefix}:{identifier}"
        count = cache.get(cache_key, 0)
        return max(0, self.limit - count)

class Validator:
    """バリデーションユーティリティ"""

    @staticmethod
    def validate_email(email: str) -> bool:
        """メールアドレスの検証"""
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))

    @staticmethod
    def validate_password(password: str) -> Dict[str, bool]:
        """パスワード強度の検証"""
        return {
            'length': len(password) >= 8,
            'uppercase': bool(re.search(r'[A-Z]', password)),
            'lowercase': bool(re.search(r'[a-z]', password)),
            'number': bool(re.search(r'\\d', password)),
            'special': bool(re.search(r'[!@#$%^&*(),.?":{}|<>]', password))
        }

    @staticmethod
    def sanitize_filename(filename: str) -> str:
        """ファイル名のサニタイズ"""
        # 拡張子の取得と保存
        name, ext = os.path.splitext(filename)

        # ファイル名のスラグ化
        safe_name = slugify(name)

        # 拡張子の再付加
        return f"{safe_name}{ext.lower()}"

class Logger:
    """ログ管理ユーティリティ"""

    def __init__(self, name: str):
        self.logger = logging.getLogger(name)

    def info(self, message: str, extra: Optional[Dict] = None):
        """情報ログの記録"""
        self._log('info', message, extra)

    def error(self, message: str, extra: Optional[Dict] = None):
        """エラーログの記録"""
        self._log('error', message, extra)

    def warning(self, message: str, extra: Optional[Dict] = None):
        """警告ログの記録"""
        self._log('warning', message, extra)

    def _log(
        self,
        level: str,
        message: str,
        extra: Optional[Dict] = None
    ):
        """ログの記録"""
        if extra is None:
            extra = {}

        # タイムスタンプの追加
        extra['timestamp'] = timezone.now().isoformat()

        # ログレベルに応じた記録
        log_func = getattr(self.logger, level)
        log_func(message, extra=extra)

class Metrics:
    """メトリクス収集ユーティリティ"""

    def __init__(self, prefix: str):
        self.prefix = prefix

    def increment(self, metric: str, value: int = 1) -> None:
        """メトリクスの増加"""
        key = f"{self.prefix}:{metric}"
        cache.incr(key, value)

    def gauge(self, metric: str, value: float) -> None:
        """ゲージメトリクスの設定"""
        key = f"{self.prefix}:{metric}"
        cache.set(key, value, timeout=None)

    def timing(self, metric: str, value: float) -> None:
        """タイミングメトリクスの記録"""
        key = f"{self.prefix}:{metric}:timing"
        timings = cache.get(key) or []
        timings.append(value)
        cache.set(key, timings[-100:], timeout=None)  # 直近100件を保持

    def get_stats(self, metric: str) -> Dict:
        """メトリクスの統計情報を取得"""
        counter_key = f"{self.prefix}:{metric}"
        timing_key = f"{self.prefix}:{metric}:timing"

        counter = cache.get(counter_key, 0)
        timings = cache.get(timing_key, [])

        return {
            'count': counter,
            'avg_timing': sum(timings) / len(timings) if timings else 0,
            'min_timing': min(timings) if timings else 0,
            'max_timing': max(timings) if timings else 0
        }

```

この実装では以下の点に注意を払っています：

1. セキュリティ
- データの暗号化
- 入力のサニタイズ
- HTMLの安全な処理
1. パフォーマンス
- キャッシュの効率的な利用
- レート制限の実装
- 最適化された処理
1. 拡張性
- モジュール化された設計
- 設定の柔軟性
- インターフェースの一貫性
1. 監視・デバッグ
- ログ機能の充実
- メトリクスの収集
- エラー追跡の容易さ