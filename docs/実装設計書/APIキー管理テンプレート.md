# APIキー管理テンプレート

## 1. APIキー一覧画面

```html
<!-- templates/accounts/api_keys/list.html -->
{% extends 'base.html' %}

{% block title %}APIキー管理 | {{ block.super }}{% endblock %}

{% block content %}
<div class="max-w-4xl mx-auto">
    <!-- ヘッダー -->
    <div class="flex justify-between items-center mb-6">
        <h1 class="text-2xl font-bold">APIキー管理</h1>
        <a href="{% url 'accounts:api_key_create' %}"
           class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors">
            新規APIキーの登録
        </a>
    </div>

    <!-- 検索フィルター -->
    <div class="bg-white shadow-md rounded-lg p-6 mb-6">
        <form method="get" class="space-y-4">
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                {{ search_form.service_name }}
                {{ search_form.status }}
            </div>
            <div class="flex justify-end">
                <button type="submit"
                        class="px-4 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700 transition-colors">
                    検索
                </button>
            </div>
        </form>
    </div>

    <!-- APIキーリスト -->
    <div class="bg-white shadow-md rounded-lg overflow-hidden">
        {% if api_keys %}
            <div class="divide-y divide-gray-200">
                {% for api_key in api_keys %}
                    <div class="p-6">
                        <div class="flex justify-between items-start">
                            <div>
                                <h2 class="text-lg font-medium">
                                    {{ api_key.get_service_name_display }}
                                </h2>
                                <div class="mt-2 space-y-1">
                                    <p class="text-sm text-gray-600">
                                        作成日: {{ api_key.created_at|date:"Y/m/d H:i" }}
                                    </p>
                                    <p class="text-sm {% if api_key.expires_at < now %}text-red-600{% else %}text-gray-600{% endif %}">
                                        有効期限: {{ api_key.expires_at|date:"Y/m/d H:i" }}
                                        {% if api_key.expires_at < now %}
                                            （期限切れ）
                                        {% endif %}
                                    </p>
                                </div>
                            </div>

                            <div class="flex space-x-2">
                                <a href="{% url 'accounts:api_key_update' api_key.id %}"
                                   class="px-3 py-1 bg-blue-100 text-blue-700 rounded hover:bg-blue-200 transition-colors">
                                    更新
                                </a>
                                <button type="button"
                                        data-modal-target="delete-modal-{{ api_key.id }}"
                                        class="px-3 py-1 bg-red-100 text-red-700 rounded hover:bg-red-200 transition-colors">
                                    無効化
                                </button>
                            </div>
                        </div>

                        {% if api_key.last_used_at %}
                            <div class="mt-2 text-sm text-gray-600">
                                最終使用: {{ api_key.last_used_at|date:"Y/m/d H:i" }}
                            </div>
                        {% endif %}
                    </div>

                    <!-- 削除確認モーダル -->
                    <div id="delete-modal-{{ api_key.id }}" class="hidden">
                        <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
                            <div class="bg-white rounded-lg max-w-md w-full mx-4">
                                <div class="p-6">
                                    <h3 class="text-lg font-bold mb-4">APIキーの無効化確認</h3>
                                    <p class="text-gray-600 mb-6">
                                        このAPIキーを無効化してもよろしいですか？この操作は取り消せません。
                                    </p>
                                    <div class="flex justify-end space-x-4">
                                        <button type="button"
                                                data-modal-close
                                                class="px-4 py-2 bg-gray-200 rounded-lg hover:bg-gray-300">
                                            キャンセル
                                        </button>
                                        <form method="post"
                                              action="{% url 'accounts:api_key_delete' api_key.id %}">
                                            {% csrf_token %}
                                            <button type="submit"
                                                    class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700">
                                                無効化する
                                            </button>
                                        </form>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                {% endfor %}
            </div>
        {% else %}
            <div class="p-6 text-center text-gray-600">
                登録されているAPIキーはありません
            </div>
        {% endif %}
    </div>
</div>
{% endblock %}

{% block extra_js %}
<script>
document.addEventListener('DOMContentLoaded', function() {
    // モーダル制御
    document.querySelectorAll('[data-modal-target]').forEach(button => {
        button.addEventListener('click', () => {
            const modalId = button.dataset.modalTarget;
            const modal = document.getElementById(modalId);
            modal.classList.remove('hidden');
        });
    });

    document.querySelectorAll('[data-modal-close]').forEach(button => {
        button.addEventListener('click', () => {
            const modal = button.closest('.fixed');
            modal.closest('.hidden').classList.add('hidden');
        });
    });
});
</script>
{% endblock %}

```

## 2. APIキー作成/更新フォーム

```html
<!-- templates/accounts/api_keys/form.html -->
{% extends 'base.html' %}

{% block title %}
    {% if form.instance.pk %}APIキーの更新{% else %}新規APIキーの登録{% endif %} | {{ block.super }}
{% endblock %}

{% block content %}
<div class="max-w-2xl mx-auto">
    <div class="bg-white shadow-md rounded-lg p-6">
        <h1 class="text-2xl font-bold mb-6">
            {% if form.instance.pk %}APIキーの更新{% else %}新規APIキーの登録{% endif %}
        </h1>

        <form method="post" class="space-y-6">
            {% csrf_token %}

            <!-- サービス選択 -->
            <div class="form-group">
                <label class="block text-gray-700 text-sm font-bold mb-2" for="{{ form.service_name.id_for_label }}">
                    サービス
                </label>
                {{ form.service_name }}
                {% if form.service_name.errors %}
                    <p class="text-red-600 text-sm mt-1">
                        {{ form.service_name.errors.0 }}
                    </p>
                {% endif %}
            </div>

            <!-- APIキー入力 -->
            <div class="form-group">
                <label class="block text-gray-700 text-sm font-bold mb-2" for="{{ form.key_value.id_for_label }}">
                    APIキー
                </label>
                <div class="relative">
                    {{ form.key_value }}
                    <button type="button"
                            id="toggleVisibility"
                            class="absolute right-2 top-1/2 transform -translate-y-1/2 text-gray-500 hover:text-gray-700">
                        <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z" />
                        </svg>
                    </button>
                </div>
                {% if form.key_value.errors %}
                    <p class="text-red-600 text-sm mt-1">
                        {{ form.key_value.errors.0 }}
                    </p>
                {% endif %}
                {% if form.key_value.help_text %}
                    <p class="text-gray-600 text-sm mt-1">
                        {{ form.key_value.help_text }}
                    </p>
                {% endif %}
            </div>

            <div class="flex justify-end space-x-4">
                <a href="{% url 'accounts:api_key_list' %}"
                   class="px-4 py-2 bg-gray-200 text-gray-800 rounded-lg hover:bg-gray-300 transition-colors">
                    キャンセル
                </a>
                <button type="submit"
                        class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors">
                    {% if form.instance.pk %}更新する{% else %}登録する{% endif %}
                </button>
            </div>
        </form>
    </div>
</div>
{% endblock %}

{% block extra_js %}
<script>
document.addEventListener('DOMContentLoaded', function() {
    // APIキーの表示/非表示切り替え
    const toggleButton = document.getElementById('toggleVisibility');
    const keyInput = document.getElementById('{{ form.key_value.id_for_label }}');

    toggleButton.addEventListener('click', () => {
        const type = keyInput.type;
        keyInput.type = type === 'password' ? 'text' : 'password';

        // アイコンの切り替え
        const svg = toggleButton.querySelector('svg');
        if (type === 'password') {
            svg.innerHTML = `
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z" />
            `;
        } else {
            svg.innerHTML = `
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13.875 18.825A10.05 10.05 0 0112 19c-4.478 0-8.268-2.943-9.543-7a9.97 9.97 0 011.563-3.029m5.858.908a3 3 0 114.243 4.243M9.878 9.878l4.242 4.242M9.88 9.88l-3.29-3.29m7.532 7.532l3.29 3.29M3 3l3.59 3.59m0 0A9.953 9.953 0 0112 5c4.478 0 8.268 2.943 9.543 7a10.025 10.025 0 01-4.132 5.411m0 0L21 21" />
            `;
        }
    });

    // サービス選択に応じたバリデーション
    const serviceSelect = document.getElementById('{{ form.service_name.id_for_label }}');
    serviceSelect.addEventListener('change', () => {
        const service = serviceSelect.value;
        const keyInput = document.getElementById('{{ form.key_value.id_for_label }}');

        if (service === 'claude') {
            keyInput.placeholder = 'sk-で始まるAPIキーを入力';
        } else if (service === 'chatgpt') {
            keyInput.placeholder = 'Bearer で始まるAPIキーを入力';
        }
    });
});
</script>
{% endblock %}

```

これらのテンプレートでは、以下の点に特に注意を払っています：

1. セキュリティ
- APIキーの安全な表示/非表示切り替え
- 適切な確認モーダルの実装
- フォームのバリデーション
1. ユーザー体験
- 直感的な操作性
- 適切なフィードバック
- 状態に応じた表示切り替え
1. アクセシビリティ
- セマンティックなHTML
- キーボード操作対応
- 適切なラベル付け
1. レスポンシブ対応
- モバイルフレンドリーなレイアウト
- 適切なスペーシング
- フレキシブルなグリッドシステム