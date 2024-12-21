# テーマ・UIコンポーネント詳細設計

## 1. テーマカスタマイズ

### 1.1 Tailwindの設定カスタマイズ

```jsx
// tailwind.config.js
const colors = require('tailwindcss/colors')

module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#f0f9ff',
          100: '#e0f2fe',
          200: '#bae6fd',
          300: '#7dd3fc',
          400: '#38bdf8',
          500: '#0ea5e9',  // ベースカラー
          600: '#0284c7',
          700: '#0369a1',
          800: '#075985',
          900: '#0c4a6e',
        },
        secondary: colors.gray,
        success: colors.green,
        warning: colors.yellow,
        danger: colors.red,
        info: colors.blue,
      },
      spacing: {
        '72': '18rem',
        '84': '21rem',
        '96': '24rem',
      },
      fontSize: {
        'xxs': '0.625rem',
      },
      maxWidth: {
        'xxs': '16rem',
      },
      minHeight: {
        'modal': '60vh',
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}

```

### 1.2 カスタムCSS変数

```css
/* static/css/custom-properties.css */
:root {
  /* カラー変数 */
  --color-primary: theme('colors.primary.500');
  --color-primary-dark: theme('colors.primary.700');
  --color-primary-light: theme('colors.primary.300');

  /* スペーシング変数 */
  --spacing-base: theme('spacing.4');
  --spacing-large: theme('spacing.6');
  --spacing-xlarge: theme('spacing.8');

  /* アニメーション変数 */
  --transition-speed: 150ms;
  --transition-timing: cubic-bezier(0.4, 0, 0.2, 1);
}

/* ダークモード設定 */
@media (prefers-color-scheme: dark) {
  :root {
    --color-primary: theme('colors.primary.400');
    --color-primary-dark: theme('colors.primary.600');
    --color-primary-light: theme('colors.primary.200');
  }
}

```

## 2. カスタムコンポーネント

### 2.1 アラートコンポーネント

```html
<!-- templates/components/alert.html -->
{% macro alert type="info" dismissible=false %}
<div class="rounded-lg p-4 mb-4 {% alert_class type %}" role="alert">
    <div class="flex items-center">
        <!-- アイコン -->
        <div class="flex-shrink-0">
            {% if type == "success" %}
                <svg class="h-5 w-5 text-green-400" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
                </svg>
            {% elif type == "error" %}
                <svg class="h-5 w-5 text-red-400" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd" />
                </svg>
            {% else %}
                <svg class="h-5 w-5 text-blue-400" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clip-rule="evenodd" />
                </svg>
            {% endif %}
        </div>

        <!-- メッセージ -->
        <div class="ml-3">
            <p class="text-sm font-medium text-{% alert_text_color type %}">
                {{ content|safe }}
            </p>
        </div>

        <!-- 閉じるボタン -->
        {% if dismissible %}
        <div class="ml-auto pl-3">
            <button type="button" class="alert-dismiss inline-flex rounded-md p-1.5 text-{% alert_text_color type %} hover:bg-{% alert_hover_color type %} focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-{% alert_ring_color type %}">
                <span class="sr-only">閉じる</span>
                <svg class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
                </svg>
            </button>
        </div>
        {% endif %}
    </div>
</div>

```

### 2.2 ページネーションコンポーネント

```html
<!-- templates/components/pagination.html -->
{% macro pagination page_obj %}
{% if page_obj.paginator.num_pages > 1 %}
<nav class="flex items-center justify-between border-t border-gray-200 px-4 py-3 sm:px-6" aria-label="ページネーション">
    <div class="hidden sm:block">
        <p class="text-sm text-gray-700">
            全{{ page_obj.paginator.count }}件中
            <span class="font-medium">{{ page_obj.start_index }}</span>
            -
            <span class="font-medium">{{ page_obj.end_index }}</span>
            件を表示
        </p>
    </div>

    <div class="flex-1 flex justify-center sm:justify-end">
        <div>
            <span class="relative z-0 inline-flex rounded-md shadow-sm">
                <!-- 前へ -->
                {% if page_obj.has_previous %}
                <a href="?page={{ page_obj.previous_page_number }}" class="relative inline-flex items-center px-2 py-2 rounded-l-md border border-gray-300 bg-white text-sm font-medium text-gray-500 hover:bg-gray-50">
                    <span class="sr-only">前へ</span>
                    <svg class="h-5 w-5" xmlns="<http://www.w3.org/2000/svg>" viewBox="0 0 20 20" fill="currentColor">
                        <path fill-rule="evenodd" d="M12.707 5.293a1 1 0 010 1.414L9.414 10l3.293 3.293a1 1 0 01-1.414 1.414l-4-4a1 1 0 010-1.414l4-4a1 1 0 011.414 0z" clip-rule="evenodd" />
                    </svg>
                </a>
                {% endif %}

                <!-- ページ番号 -->
                {% for i in page_obj.paginator.page_range %}
                {% if page_obj.number == i %}
                <span class="relative inline-flex items-center px-4 py-2 border border-gray-300 bg-blue-50 text-sm font-medium text-blue-600">
                    {{ i }}
                </span>
                {% elif i > page_obj.number|add:"-3" and i < page_obj.number|add:"3" %}
                <a href="?page={{ i }}" class="relative inline-flex items-center px-4 py-2 border border-gray-300 bg-white text-sm font-medium text-gray-700 hover:bg-gray-50">
                    {{ i }}
                </a>
                {% endif %}
                {% endfor %}

                <!-- 次へ -->
                {% if page_obj.has_next %}
                <a href="?page={{ page_obj.next_page_number }}" class="relative inline-flex items-center px-2 py-2 rounded-r-md border border-gray-300 bg-white text-sm font-medium text-gray-500 hover:bg-gray-50">
                    <span class="sr-only">次へ</span>
                    <svg class="h-5 w-5" xmlns="<http://www.w3.org/2000/svg>" viewBox="0 0 20 20" fill="currentColor">
                        <path fill-rule="evenodd" d="M7.293 14.707a1 1 0 010-1.414L10.586 10 7.293 6.707a1 1 0 011.414-1.414l4 4a1 1 0 010 1.414l-4 4a1 1 0 01-1.414 0z" clip-rule="evenodd" />
                    </svg>
                </a>
                {% endif %}
            </span>
        </div>
    </div>
</nav>
{% endif %}
{% endmacro %}

```

### 2.3 タブコンポーネント

```html
<!-- templates/components/tabs.html -->
{% macro tabs %}
<div x-data="{ activeTab: '{{ active_tab }}' }" class="w-full">
    <!-- タブヘッダー -->
    <div class="border-b border-gray-200">
        <nav class="-mb-px flex space-x-8" aria-label="タブ">
            {% for tab in tabs %}
            <button @click="activeTab = '{{ tab.id }}'"
                    :class="{ 'border-primary-500 text-primary-600': activeTab === '{{ tab.id }}',
                             'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300': activeTab !== '{{ tab.id }}' }"
                    class="whitespace-nowrap py-4 px-1 border-b-2 font-medium text-sm">
                {{ tab.label }}
            </button>
            {% endfor %}
        </nav>
    </div>

    <!-- タブコンテンツ -->
    <div class="mt-4">
        {% for tab in tabs %}
        <div x-show="activeTab === '{{ tab.id }}'"
             x-transition:enter="transition ease-out duration-200"
             x-transition:enter-start="opacity-0"
             x-transition:enter-end="opacity-100">
            {{ tab.content|safe }}
        </div>
        {% endfor %}
    </div>
</div>
{% endmacro %}

```

## 3. JavaScript拡張

### 3.1 コンポーネント制御

```jsx
// static/js/components.js

// アラートコンポーネントの制御
class Alert {
    constructor(element) {
        this.element = element;
        this.dismissButton = element.querySelector('.alert-dismiss');

        if (this.dismissButton) {
            this.dismissButton.addEventListener('click', () => this.dismiss());
        }
    }

    dismiss() {
        this.element.classList.add('opacity-0');
        setTimeout(() => {
            this.element.remove();
        }, 150);
    }
}

// タブコンポーネントの制御
class Tabs {
    constructor(element) {
        this.element = element;
        this.tabs = this.element.querySelectorAll('[role="tab"]');
        this.panels = this.element.querySelectorAll('[role="tabpanel"]');

        this.tabs.forEach(tab => {
            tab.addEventListener('click', () => this.switchTab(tab));
        });
    }

    switchTab(tab) {
        // 現在のアクティブタブを非アクティブに
        const currentTab = this.element.querySelector('[aria-selected="true"]');
        currentTab.setAttribute('aria-selected', 'false');
        currentTab.classList.remove('border-primary-500', 'text-primary-600');
        currentTab.classList.add('border-transparent', 'text-gray-500');

        // 新しいタブをアクティブに
        tab.setAttribute('aria-selected', 'true');
        tab.classList.remove('border-transparent', 'text-gray-500');
        tab.classList.add('border-primary-500', 'text-primary-600');

        // パネルの切り替え
        const targetId = tab.getAttribute('aria-controls');
        this.panels.forEach(panel => {
            panel.classList.add('hidden');
            if (panel.id === targetId) {
                panel.classList.remove('hidden');
            }
        });
    }
}

// コンポーネントの初期化
document.addEventListener('DOMContentLoaded', () => {
    // アラートの初期化
    document.querySelectorAll('[role="alert"]').forEach(element => {
        new Alert(element);
    });

    // タブの初期化
    document.querySelectorAll('[role="tablist"]').forEach(element => {
        new Tabs(element);
    });

    // モーダルの初期化
    document.querySelectorAll('[data-modal]').forEach(element => {
        new Modal(element);
    });

    // ドロップダウンの初期化
    document.querySelectorAll('[data-dropdown]').forEach(element => {
        new Dropdown(element);
    });

    // ツールチップの初期化
    document.querySelectorAll('[data-tooltip]').forEach(element => {
        new Tooltip(element);
    });
});

// Modalクラスの実装
class Modal {
    constructor(element) {
        this.element = element;
        this.targetId = element.dataset.modal;
        this.modal = document.getElementById(this.targetId);
        this.closeButtons = this.modal.querySelectorAll('[data-modal-close]');

        this.bindEvents();
    }

    bindEvents() {
        // モーダルを開く
        this.element.addEventListener('click', () => this.open());

        // モーダルを閉じる
        this.closeButtons.forEach(button => {
            button.addEventListener('click', () => this.close());
        });

        // 背景クリックで閉じる
        this.modal.addEventListener('click', (e) => {
            if (e.target === this.modal) this.close();
        });

        // ESCキーで閉じる
        document.addEventListener('keydown', (e) => {
            if (e.key === 'Escape' && !this.modal.classList.contains('hidden')) {
                this.close();
            }
        });
    }

    open() {
        this.modal.classList.remove('hidden');
        document.body.classList.add('overflow-hidden');
    }

    close() {
        this.modal.classList.add('hidden');
        document.body.classList.remove('overflow-hidden');
    }
}

// Dropdownクラスの実装
class Dropdown {
    constructor(element) {
        this.element = element;
        this.targetId = element.dataset.dropdown;
        this.dropdown = document.getElementById(this.targetId);

        this.bindEvents();
    }

    bindEvents() {
        // ドロップダウンの表示・非表示
        this.element.addEventListener('click', (e) => {
            e.stopPropagation();
            this.toggle();
        });

        // 外部クリックで閉じる
        document.addEventListener('click', () => {
            this.close();
        });
    }

    toggle() {
        this.dropdown.classList.toggle('hidden');
    }

    close() {
        this.dropdown.classList.add('hidden');
    }
}

// Tooltipクラスの実装
class Tooltip {
    constructor(element) {
        this.element = element;
        this.content = element.dataset.tooltip;
        this.tooltip = this.createTooltip();

        this.bindEvents();
    }

    createTooltip() {
        const tooltip = document.createElement('div');
        tooltip.className = 'absolute z-50 px-2 py-1 text-sm text-white bg-gray-900 rounded hidden';
        tooltip.textContent = this.content;
        document.body.appendChild(tooltip);
        return tooltip;
    }

    bindEvents() {
        this.element.addEventListener('mouseenter', () => this.show());
        this.element.addEventListener('mouseleave', () => this.hide());
    }

    show() {
        const rect = this.element.getBoundingClientRect();
        this.tooltip.style.top = `${rect.top - this.tooltip.offsetHeight - 5}px`;
        this.tooltip.style.left = `${rect.left + (rect.width - this.tooltip.offsetWidth) / 2}px`;
        this.tooltip.classList.remove('hidden');
    }

    hide() {
        this.tooltip.classList.add('hidden');
    }
}

// FormValidationクラスの実装
class FormValidation {
    constructor(form) {
        this.form = form;
        this.fields = form.querySelectorAll('[data-validate]');

        this.bindEvents();
    }

    bindEvents() {
        this.fields.forEach(field => {
            field.addEventListener('blur', () => this.validateField(field));
        });

        this.form.addEventListener('submit', (e) => {
            if (!this.validateAll()) {
                e.preventDefault();
            }
        });
    }

    validateField(field) {
        const rules = field.dataset.validate.split(',');
        let isValid = true;
        let errorMessage = '';

        rules.forEach(rule => {
            if (!this.runValidation(field, rule.trim())) {
                isValid = false;
                errorMessage = this.getErrorMessage(rule.trim());
            }
        });

        this.showFieldError(field, isValid, errorMessage);
        return isValid;
    }

    validateAll() {
        let isValid = true;
        this.fields.forEach(field => {
            if (!this.validateField(field)) {
                isValid = false;
            }
        });
        return isValid;
    }

    runValidation(field, rule) {
        const value = field.value.trim();

        switch (rule) {
            case 'required':
                return value !== '';
            case 'email':
                return /^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/.test(value);
            case 'number':
                return !isNaN(value) && value !== '';
            default:
                return true;
        }
    }

    getErrorMessage(rule) {
        const messages = {
            required: '必須項目です',
            email: '有効なメールアドレスを入力してください',
            number: '数値を入力してください'
        };
        return messages[rule] || 'Invalid input';
    }

    showFieldError(field, isValid, message) {
        const errorElement = this.getErrorElement(field);

        if (!isValid) {
            field.classList.add('border-red-500');
            errorElement.textContent = message;
            errorElement.classList.remove('hidden');
        } else {
            field.classList.remove('border-red-500');
            errorElement.classList.add('hidden');
        }
    }

    getErrorElement(field) {
        let errorElement = field.nextElementSibling;
        if (!errorElement || !errorElement.classList.contains('error-message')) {
            errorElement = document.createElement('p');
            errorElement.className = 'error-message text-red-500 text-sm mt-1 hidden';
            field.parentNode.insertBefore(errorElement, field.nextSibling);
        }
        return errorElement;
    }
}

// 入力フィールドのライブプレビュー
class LivePreview {
    constructor(input, preview) {
        this.input = input;
        this.preview = preview;
        this.markdown = new marked.Marked();

        this.bindEvents();
    }

    bindEvents() {
        this.input.addEventListener('input', () => this.update());
    }

    update() {
        const content = this.input.value;
        const html = this.markdown.parse(content);
        this.preview.innerHTML = html;
    }
}

```

このJavaScriptコードでは、以下のような機能を実装しています：

1. 各種コンポーネントの初期化
2. モーダルダイアログの制御
3. ドロップダウンメニューの制御
4. ツールチップの実装
5. フォームバリデーション
6. マークダウンのライブプレビュー

# コンポーネント使用例

## 1. モーダルダイアログの使用

```html
<!-- templates/questions/confirm_delete.html -->
<!-- モーダルトリガーボタン -->
<button data-modal="deleteModal"
        class="bg-red-600 text-white px-4 py-2 rounded-lg hover:bg-red-700">
    削除
</button>

<!-- モーダル本体 -->
<div id="deleteModal" class="fixed inset-0 bg-black bg-opacity-50 hidden flex items-center justify-center">
    <div class="bg-white rounded-lg max-w-md w-full mx-4">
        <div class="p-6">
            <h3 class="text-lg font-bold mb-4">質問の削除確認</h3>
            <p class="text-gray-600 mb-6">
                この質問を削除してもよろしいですか？この操作は取り消せません。
            </p>
            <div class="flex justify-end space-x-4">
                <button data-modal-close
                        class="px-4 py-2 bg-gray-200 rounded-lg hover:bg-gray-300">
                    キャンセル
                </button>
                <form method="post" action="{% url 'questions:delete' question.id %}">
                    {% csrf_token %}
                    <button type="submit"
                            class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700">
                        削除する
                    </button>
                </form>
            </div>
        </div>
    </div>
</div>

```

## 2. ドロップダウンメニューの使用

```html
<!-- templates/includes/user_menu.html -->
<div class="relative">
    <!-- ドロップダウントリガー -->
    <button data-dropdown="userMenu"
            class="flex items-center space-x-2 text-gray-700 hover:text-gray-900">
        <span>{{ user.username }}</span>
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
        </svg>
    </button>

    <!-- ドロップダウンメニュー -->
    <div id="userMenu"
         class="hidden absolute right-0 mt-2 w-48 bg-white rounded-lg shadow-lg py-1">
        <a href="{% url 'accounts:profile' %}"
           class="block px-4 py-2 text-gray-700 hover:bg-gray-100">
            プロフィール
        </a>
        <a href="{% url 'accounts:api_keys' %}"
           class="block px-4 py-2 text-gray-700 hover:bg-gray-100">
            APIキー管理
        </a>
        <hr class="my-1">
        <form method="post" action="{% url 'accounts:logout' %}">
            {% csrf_token %}
            <button type="submit"
                    class="block w-full text-left px-4 py-2 text-gray-700 hover:bg-gray-100">
                ログアウト
            </button>
        </form>
    </div>
</div>

```

## 3. フォームバリデーションの使用

```html
<!-- templates/questions/create.html -->
<form method="post" class="space-y-6" data-validate>
    {% csrf_token %}

    <div class="form-group">
        <label class="block text-gray-700 text-sm font-bold mb-2">
            タイトル
        </label>
        <input type="text"
               name="title"
               data-validate="required"
               class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500">
    </div>

    <div class="form-group">
        <label class="block text-gray-700 text-sm font-bold mb-2">
            質問内容
        </label>
        <textarea name="content"
                  data-validate="required"
                  rows="6"
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500"></textarea>
    </div>

    <button type="submit"
            class="w-full md:w-auto px-6 py-2 bg-blue-600 text-white rounded-lg
                   hover:bg-blue-700 transition-colors">
        質問を投稿
    </button>
</form>

<!-- JavaScript初期化 -->
<script>
document.addEventListener('DOMContentLoaded', () => {
    const form = document.querySelector('form[data-validate]');
    if (form) {
        new FormValidation(form);
    }
});
</script>

```

## 4. ライブプレビューの使用

```html
<!-- templates/questions/create.html -->
<div class="space-y-4">
    <div class="form-group">
        <label class="block text-gray-700 text-sm font-bold mb-2">
            質問内容
        </label>
        <textarea id="questionContent"
                  name="content"
                  rows="6"
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500"></textarea>
    </div>

    <div class="bg-gray-50 rounded-lg p-4">
        <h3 class="text-sm font-bold text-gray-700 mb-2">プレビュー</h3>
        <div id="preview" class="prose max-w-none"></div>
    </div>
</div>

<script>
document.addEventListener('DOMContentLoaded', () => {
    const input = document.getElementById('questionContent');
    const preview = document.getElementById('preview');

    if (input && preview) {
        new LivePreview(input, preview);
    }
});
</script>

```

## 5. ツールチップの使用

```html
<!-- templates/questions/detail.html -->
<div class="flex items-center space-x-2">
    <span>回答状態:</span>
    <span class="relative"
          data-tooltip="この回答は2つのAIモデルによって検証されています">
        <span class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
            検証済み
        </span>
    </span>
</div>

<!-- APIキー有効期限の表示 -->
<div class="mt-4">
    <span class="relative"
          data-tooltip="期限切れの30日前に通知されます">
        有効期限: {{ api_key.expires_at|date:"Y/m/d" }}
    </span>
</div>

```

## 6. タブインターフェースの使用

```html
<!-- templates/questions/detail.html -->
<div class="mt-8" x-data="{ activeTab: 'claude' }">
    <!-- タブヘッダー -->
    <div class="border-b border-gray-200">
        <nav class="-mb-px flex space-x-8">
            <button @click="activeTab = 'claude'"
                    :class="{ 'border-blue-500 text-blue-600': activeTab === 'claude',
                             'border-transparent text-gray-500': activeTab !== 'claude' }"
                    class="whitespace-nowrap py-4 px-1 border-b-2 font-medium text-sm">
                Claude回答
            </button>
            <button @click="activeTab = 'chatgpt'"
                    :class="{ 'border-blue-500 text-blue-600': activeTab === 'chatgpt',
                             'border-transparent text-gray-500': activeTab !== 'chatgpt' }"
                    class="whitespace-nowrap py-4 px-1 border-b-2 font-medium text-sm">
                ChatGPT検証
            </button>
            <button @click="activeTab = 'final'"
                    :class="{ 'border-blue-500 text-blue-600': activeTab === 'final',
                             'border-transparent text-gray-500': activeTab !== 'final' }"
                    class="whitespace-nowrap py-4 px-1 border-b-2 font-medium text-sm">
                最終回答
            </button>
        </nav>
    </div>

    <!-- タブコンテンツ -->
    <div class="mt-4">
        <div x-show="activeTab === 'claude'" class="prose max-w-none">
            {{ claude_response.content|markdown }}
        </div>
        <div x-show="activeTab === 'chatgpt'" class="prose max-w-none">
            {{ chatgpt_response.content|markdown }}
        </div>
        <div x-show="activeTab === 'final'" class="prose max-w-none">
            {{ final_response.content|markdown }}
        </div>
    </div>
</div>

```

これらのコンポーネントは、必要に応じて組み合わせて使用することができます。また、各コンポーネントはモジュール化されているため、必要な機能だけを選択して実装することも可能です。

特に以下の点に注意を払っています：

1. アクセシビリティ
- 適切なARIAラベルの使用
- キーボード操作のサポート
- スクリーンリーダー対応
1. ユーザビリティ
- 直感的な操作性
- 適切なフィードバック
- レスポンシブ対応
1. パフォーマンス
- 必要なコンポーネントのみの読み込み
- 効率的なイベント処理
- 適切なキャッシュの利用