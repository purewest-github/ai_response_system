# AI回答管理システム基本設計書

## 改訂履歴

| バージョン | 日付 | 改訂内容 | 承認者 |
| --- | --- | --- | --- |
| 1.0 | 2024/12/20 | 初版作成 | - |

## 目次

1. システム概要
2. 環境構成設計
3. アプリケーション構成設計
4. データベース設計
5. セキュリティ設計
6. 画面設計
7. インターフェース設計
8. 運用設計

## 1. システム概要

### 1.1 システム方式設計

### 開発言語・フレームワーク

```yaml
バックエンド:
  言語: Python 3.11
  フレームワーク: Django 4.2
  テンプレートエンジン: Django Templates

フロントエンド:
  CSS: TailwindCSS 3.0
  JavaScript: Vanilla JS
  マークダウン: django-markdownx

データベース:
  RDBMS: MySQL 8.0
  キャッシュ: Redis 7.0

```

### アーキテクチャ

```yaml
アプリケーションアーキテクチャ:
  - MVTパターン（Django標準）
  - レイヤードアーキテクチャ

```

### 1.2 コーディング規約

### Pythonコーディング規約

```yaml
スタイルガイド:
  - PEP 8準拠
  - 最大行長: 119文字
  - インデント: 4スペース

命名規則:
  クラス名: UpperCamelCase
  関数・変数名: snake_case
  定数: UPPER_SNAKE_CASE

```

### HTMLコーディング規約

```yaml
規約:
  - HTML5準拠
  - セマンティックHTMLの使用
  - WAI-ARIA対応

```

## 2. 環境構成設計

### 2.1 システム構成

### 開発環境

```yaml
ローカル環境:
  - Docker Desktop
  - Docker Compose
  - Visual Studio Code

コンテナ構成:
  - Djangoアプリケーション
  - MySQL 8.0
  - Redis
  - nginx

```

### 本番環境（AWS）

```yaml
構成要素:
  - AWS Fargate
  - Amazon RDS (MySQL 8.0)
  - Amazon ElastiCache (Redis)
  - Application Load Balancer
  - Amazon S3

```

### 2.2 ネットワーク構成

### 開発環境

```yaml
ポート構成:
  - 8000: Djangoアプリケーション
  - 3306: MySQL
  - 6379: Redis
  - 80: nginx

```

### 本番環境

```yaml
VPC設計:
  - パブリックサブネット: ALB配置
  - プライベートサブネット: アプリケーション・DB配置

```

## 3. アプリケーション構成設計

### 3.1 ディレクトリ構成

```
ai_response_system/
├── manage.py
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── local.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── core/            # 共通機能
│   ├── accounts/        # ユーザー管理
│   ├── questions/       # 質問・回答管理
│   ├── api_management/  # API管理
│   └── notifications/   # 通知管理
├── static/
│   ├── css/
│   ├── js/
│   └── images/
└── templates/
    ├── base.html
    └── includes/

```

### 3.2 アプリケーション機能設計

### core アプリケーション

```python
# core/models.py
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class SingletonModel(models.Model):
    class Meta:
        abstract = True

    @classmethod
    def load(cls):
        obj, _ = cls.objects.get_or_create(pk=1)
        return obj

```

### accounts アプリケーション

```python
# accounts/models.py
class CustomUser(AbstractUser):
    is_admin = models.BooleanField(default=False)
    api_request_count = models.IntegerField(default=0)
    last_request_time = models.DateTimeField(null=True)

    class Meta:
        db_table = 'custom_users'

```

### 3.3 エラー処理設計

```python
# core/exceptions.py
class APIKeyError(Exception):
    pass

class ValidationError(Exception):
    pass

# core/middleware.py
class ErrorHandlingMiddleware:
    def process_exception(self, request, exception):
        if isinstance(exception, APIKeyError):
            # APIキーエラーの処理
            pass

```

## 4. データベース設計

### 4.1 データベース基本設定

```sql
-- データベース作成
CREATE DATABASE ai_response_db
    DEFAULT CHARACTER SET utf8mb4
    DEFAULT COLLATE utf8mb4_unicode_ci;

-- ユーザー作成
CREATE USER 'ai_response_app'@'%'
    IDENTIFIED BY 'secure_password';

-- 権限付与
GRANT SELECT, INSERT, UPDATE, DELETE
    ON ai_response_db.*
    TO 'ai_response_app'@'%';

```

### 4.2 テーブル設計

### ユーザーテーブル

```sql
CREATE TABLE custom_users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(150) NOT NULL,
    email VARCHAR(254) NOT NULL,
    password VARCHAR(255) NOT NULL,
    is_admin BOOLEAN DEFAULT FALSE,
    api_request_count INT UNSIGNED DEFAULT 0,
    last_request_time DATETIME,
    is_active BOOLEAN DEFAULT TRUE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_username (username),
    UNIQUE KEY uk_email (email),
    INDEX idx_api_request (api_request_count, last_request_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

```

### 質問テーブル

```sql
CREATE TABLE questions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    author_id BIGINT UNSIGNED NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY fk_author (author_id) REFERENCES custom_users(id),
    INDEX idx_author_status (author_id, status),
    FULLTEXT INDEX ft_content (title, content)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

```

### 4.3 Django Database設定

```python
# settings/base.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'ai_response_db',
        'USER': 'ai_response_app',
        'PASSWORD': 'secure_password',
        'HOST': 'mysql',
        'PORT': '3306',
        'OPTIONS': {
            'charset': 'utf8mb4',
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'",
            'isolation_level': 'read committed',
        },
    }
}

```

## 5. セキュリティ設計

### 5.1 認証・認可設計

### セッション管理

```python
# settings/base.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
SESSION_COOKIE_AGE = 1209600  # 2週間
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True

```

### 認証バックエンド

```python
# accounts/backends.py
class CustomAuthBackend:
    def authenticate(self, request, username=None, password=None):
        try:
            user = CustomUser.objects.get(username=username)
            if user.check_password(password):
                return user
        except CustomUser.DoesNotExist:
            return None

```

### 5.2 セキュリティ対策

### XSS対策

```python
# settings/base.py
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True

# テンプレートでのエスケープ
{{ user_input|escape }}

```

### CSRF対策

```python
# settings/base.py
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',
]

# テンプレート
{% csrf_token %}

```

### クリックジャッキング対策

```python
# settings/base.py
X_FRAME_OPTIONS = 'DENY'
SECURE_HSTS_SECONDS = 31536000

```

## 6. 画面設計

### 6.1 画面一覧

```yaml
認証系画面:
  - ログイン画面
  - パスワードリセット画面
  - ユーザー登録画面

メイン画面:
  - ダッシュボード
  - 質問一覧
  - 質問詳細・回答
  - API Key管理
  - 通知一覧

```

### 6.2 レイアウト設計

### 共通レイアウト

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{% block title %}{% endblock %}</title>
    {% tailwind_css %}
</head>
<body>
    {% include 'includes/header.html' %}
    <main class="container mx-auto px-4">
        {% block content %}{% endblock %}
    </main>
    {% include 'includes/footer.html' %}
</body>
</html>

```

### 6.3 レスポンシブ対応

```yaml
ブレークポイント:
  - sm: 640px
  - md: 768px
  - lg: 1024px
  - xl: 1280px

対応方針:
  - モバイルファースト
  - Tailwind CSSのユーティリティクラス活用
  - フレックスボックスレイアウト

```

## 7. インターフェース設計

### 7.1 API設計

### AI API連携

```python
# core/services/ai_service.py
class AIService:
    def generate_response(self, question: str) -> dict:
        claude_response = self._call_claude_api(question)
        chatgpt_response = self._call_chatgpt_api(question)
        return {
            'claude': claude_response,
            'chatgpt': chatgpt_response
        }

    def _call_claude_api(self, question: str) -> str:
        # Claude APIの呼び出し実装
        pass

    def _call_chatgpt_api(self, question: str) -> str:
        # ChatGPT APIの呼び出し実装
        pass

```

### 7.2 非同期処理設計

```python
# core/tasks.py
from celery import shared_task

@shared_task
def process_ai_response(question_id: int):
    question = Question.objects.get(id=question_id)
    service = AIService()
    response = service.generate_response(question.content)
    # 回答の保存処理

```

## 8. 運用設計

### 8.1 ログ設計

```python
# settings/base.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': 'debug.log',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file'],
            'level': 'INFO',
            'propagate': True,
        },
    },
}

```

### 8.2 監視設計

```yaml
監視項目:
  システムメトリクス:
    - CPU使用率
    - メモリ使用率
    - ディスク使用率
  アプリケーションメトリクス:
    - レスポンスタイム
    - エラー率
    - API使用量

監視ツール:
  - CloudWatch
  - CloudWatch Logs
  - X-Ray

```

### 8.3 バックアップ設計

```yaml
データベースバックアップ:
  - 日次フルバックアップ
  - 差分バックアップ（6時間ごと）
  - バイナリログのバックアップ

ファイルバックアップ:
  - S3でのバージョニング
  - クロスリージョンレプリケーション

```