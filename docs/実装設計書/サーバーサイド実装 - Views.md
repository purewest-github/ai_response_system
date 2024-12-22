# サーバーサイド実装 - Views

## 1. 質問関連ビュー

```python
# questions/views.py
from django.views.generic import CreateView, UpdateView, DeleteView, DetailView, ListView
from django.contrib.auth.mixins import LoginRequiredMixin
from django.http import JsonResponse
from django.urls import reverse_lazy
from django.shortcuts import get_object_or_404
from django.core.exceptions import PermissionDenied
from django.db import transaction
from django.utils import timezone

from .models import Question, Response
from .forms import QuestionForm
from .services import QuestionService, AIService
from core.mixins import AjaxResponseMixin

class QuestionCreateView(LoginRequiredMixin, AjaxResponseMixin, CreateView):
    model = Question
    form_class = QuestionForm
    template_name = 'questions/create.html'
    success_url = reverse_lazy('questions:list')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['is_edit'] = False
        context['remaining_questions'] = self.request.user.get_remaining_questions()
        return context

    @transaction.atomic
    def form_valid(self, form):
        # ユーザーの質問制限チェック
        if not self.request.user.can_create_question():
            return self.json_error({'error': '本日の質問制限に達しました'})

        try:
            # 質問の作成と初期処理
            form.instance.user = self.request.user
            question = form.save()

            # 非同期処理の開始
            service = QuestionService()
            transaction.on_commit(lambda: service.process_question.delay(question.id))

            if self.request.is_ajax():
                return self.json_response({
                    'redirect_url': self.get_success_url()
                })
            return super().form_valid(form)

        except Exception as e:
            return self.json_error({'error': str(e)})

class QuestionUpdateView(LoginRequiredMixin, AjaxResponseMixin, UpdateView):
    model = Question
    form_class = QuestionForm
    template_name = 'questions/create.html'

    def get_queryset(self):
        return super().get_queryset().filter(
            user=self.request.user,
            status='draft'
        )

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['is_edit'] = True
        return context

    @transaction.atomic
    def form_valid(self, form):
        try:
            question = form.save()

            if self.request.is_ajax():
                return self.json_response({
                    'redirect_url': reverse_lazy('questions:detail', kwargs={'pk': question.pk})
                })
            return super().form_valid(form)

        except Exception as e:
            return self.json_error({'error': str(e)})

class QuestionDetailView(LoginRequiredMixin, DetailView):
    model = Question
    template_name = 'questions/detail.html'

    def get_queryset(self):
        return super().get_queryset().filter(
            user=self.request.user
        ).select_related('user').prefetch_related('responses')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        responses = self.object.responses.all()

        # レスポンスの分類
        context.update({
            'claude_response': responses.filter(ai_service='claude').first(),
            'chatgpt_response': responses.filter(ai_service='chatgpt').first(),
            'final_response': responses.filter(ai_service='final').first(),
            'total_processing_time': sum(r.processing_time or 0 for r in responses),
            'total_tokens': sum(r.token_count or 0 for r in responses)
        })
        return context

class QuestionListView(LoginRequiredMixin, ListView):
    model = Question
    template_name = 'questions/list.html'
    context_object_name = 'questions'
    paginate_by = 10

    def get_queryset(self):
        queryset = Question.objects.filter(
            user=self.request.user
        ).select_related('user')

        # 検索条件の適用
        search_query = self.request.GET.get('q')
        status = self.request.GET.get('status')
        date_range = self.request.GET.get('date_range')

        if search_query:
            queryset = queryset.search(search_query)

        if status:
            queryset = queryset.filter(status=status)

        if date_range:
            queryset = self.apply_date_range_filter(queryset, date_range)

        return queryset.order_by('-created_at')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context.update({
            'status_choices': Question.STATUS_CHOICES,
            'search_form': self.request.GET
        })
        return context

    def apply_date_range_filter(self, queryset, date_range):
        today = timezone.now().date()

        if date_range == 'today':
            return queryset.filter(created_at__date=today)
        elif date_range == 'week':
            return queryset.filter(created_at__date__gte=today - timezone.timedelta(days=7))
        elif date_range == 'month':
            return queryset.filter(created_at__date__gte=today - timezone.timedelta(days=30))
        elif date_range == 'custom':
            date_from = self.request.GET.get('date_from')
            date_to = self.request.GET.get('date_to')

            if date_from:
                queryset = queryset.filter(created_at__date__gte=date_from)
            if date_to:
                queryset = queryset.filter(created_at__date__lte=date_to)

        return queryset

class QuestionDeleteView(LoginRequiredMixin, DeleteView):
    model = Question
    success_url = reverse_lazy('questions:list')

    def get_queryset(self):
        return super().get_queryset().filter(user=self.request.user)

    def delete(self, request, *args, **kwargs):
        try:
            response = super().delete(request, *args, **kwargs)
            if request.is_ajax():
                return JsonResponse({'status': 'success'})
            return response

        except Exception as e:
            if request.is_ajax():
                return JsonResponse({
                    'status': 'error',
                    'error': str(e)
                }, status=400)
            raise

```

続いて、API関連のビューを示させていただきましょうか？

# サーバーサイド実装 - API Views

```python
# questions/api_views.py
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods
from django.contrib.auth.decorators import login_required
from django.views.decorators.csrf import ensure_csrf_cookie
from django.core.exceptions import PermissionDenied
from django.shortcuts import get_object_or_404
from django.utils.decorators import method_decorator
from django.views import View

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from rest_framework.throttling import UserRateThrottle

from .models import Question, Response as QuestionResponse
from .services import QuestionService, AIService
from .serializers import QuestionSerializer, ResponseSerializer
from core.utils import markdown_to_html
from core.decorators import handle_api_errors

class QuestionAPIThrottle(UserRateThrottle):
    rate = '30/hour'

class QuestionStatusView(APIView):
    permission_classes = [IsAuthenticated]
    throttle_classes = [QuestionAPIThrottle]

    @handle_api_errors
    def get(self, request, pk):
        question = get_object_or_404(Question, pk=pk, user=request.user)

        responses = []
        for response in question.responses.all():
            responses.append({
                'ai_service': response.ai_service,
                'content': response.content,
                'processing_time': response.processing_time,
                'token_count': response.token_count,
                'validation_status': response.validation_status
            })

        return Response({
            'status': question.status,
            'error_message': question.error_message,
            'responses': responses
        })

class QuestionRetryView(APIView):
    permission_classes = [IsAuthenticated]
    throttle_classes = [QuestionAPIThrottle]

    @handle_api_errors
    def post(self, request, pk):
        question = get_object_or_404(Question, pk=pk, user=request.user)

        if question.status != 'error':
            return Response(
                {'error': '再試行できるのはエラー状態の質問のみです'},
                status=status.HTTP_400_BAD_REQUEST
            )

        service = QuestionService()
        service.process_question.delay(question.id)

        return Response({'status': 'processing'})

class ResponseFeedbackView(APIView):
    permission_classes = [IsAuthenticated]

    @handle_api_errors
    def post(self, request):
        response_id = request.data.get('response_id')
        feedback_type = request.data.get('feedback_type')

        if not response_id or not feedback_type:
            return Response(
                {'error': 'response_idとfeedback_typeは必須です'},
                status=status.HTTP_400_BAD_REQUEST
            )

        response = get_object_or_404(
            QuestionResponse,
            id=response_id,
            question__user=request.user
        )

        response.add_feedback(feedback_type)
        return Response({'status': 'success'})

class MarkdownPreviewView(APIView):
    permission_classes = [IsAuthenticated]
    throttle_classes = [UserRateThrottle]

    @handle_api_errors
    def post(self, request):
        content = request.data.get('content', '')
        html = markdown_to_html(content)
        return Response({'html': html})

@method_decorator(login_required, name='dispatch')
class QuestionAutosaveView(View):
    @handle_api_errors
    def post(self, request):
        if not request.is_ajax():
            raise PermissionDenied

        question_id = request.POST.get('id')
        content = request.POST.get('content')
        title = request.POST.get('title')

        if question_id:
            # 既存の質問の更新
            question = get_object_or_404(
                Question,
                id=question_id,
                user=request.user,
                status='draft'
            )
            if content:
                question.content = content
            if title:
                question.title = title
            question.save()
        else:
            # 下書きとして新規作成
            question = Question.objects.create(
                user=request.user,
                title=title or '無題の質問',
                content=content or '',
                status='draft'
            )

        return JsonResponse({
            'status': 'success',
            'question_id': question.id
        })

@login_required
@require_http_methods(['POST'])
@handle_api_errors
def cancel_processing(request, pk):
    """処理中の質問をキャンセルする"""
    question = get_object_or_404(Question, pk=pk, user=request.user)

    if question.status != 'processing':
        return JsonResponse(
            {'error': 'キャンセルできるのは処理中の質問のみです'},
            status=400
        )

    service = QuestionService()
    service.cancel_processing(question.id)

    return JsonResponse({'status': 'success'})

# WebSocket関連のコンシューマー
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class QuestionStatusConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.question_id = self.scope['url_route']['kwargs']['question_id']
        self.user = self.scope['user']

        # 認証チェック
        if not self.user.is_authenticated:
            await self.close()
            return

        # 質問の所有者チェック
        try:
            question = await Question.objects.aget(
                id=self.question_id,
                user=self.user
            )
        except Question.DoesNotExist:
            await self.close()
            return

        await self.channel_layer.group_add(
            f'question_{self.question_id}',
            self.channel_name
        )
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            f'question_{self.question_id}',
            self.channel_name
        )

    async def receive_json(self, content):
        # クライアントからのメッセージは現時点では使用しない
        pass

    async def question_update(self, event):
        """質問の状態が更新された際の通知"""
        await self.send_json({
            'type': 'question.update',
            'question': {
                'status': event['status'],
                'error_message': event.get('error_message'),
                'responses': event.get('responses', [])
            }
        })

    async def response_update(self, event):
        """回答が生成された際の通知"""
        await self.send_json({
            'type': 'response.update',
            'response': event['response']
        })

```

この実装では、以下の点に注意を払っています：

1. 非同期処理
- WebSocket対応での状態通知
- 非同期タスクの管理
- キャンセル処理の実装
1. セキュリティ
- 適切な認証・認可チェック
- レート制限の実装
- CSRF対策
1. エラーハンドリング
- 包括的なエラー処理
- 適切なエラーメッセージ
- 例外処理の一元化
1. パフォーマンス
- データベースクエリの最適化
- キャッシュの活用
- 非同期処理の活用

次は、Models部分の実装を示させていただきましょうか？

# サーバーサイド実装 - Models

```python
# questions/models.py
from django.db import models
from django.conf import settings
from django.utils import timezone
from django.core.exceptions import ValidationError
from core.models import TimeStampedModel
from core.managers import SearchableQuerySet

class QuestionQuerySet(SearchableQuerySet):
    def search(self, query):
        """全文検索を行う"""
        if query:
            return self.filter(
                models.Q(title__icontains=query) |
                models.Q(content__icontains=query)
            )
        return self

    def for_user(self, user):
        """特定ユーザーの質問を取得"""
        return self.filter(user=user)

    def active(self):
        """アクティブな質問のみを取得"""
        return self.exclude(status='deleted')

class Question(TimeStampedModel):
    STATUS_CHOICES = [
        ('draft', '下書き'),
        ('processing', '処理中'),
        ('completed', '完了'),
        ('error', 'エラー'),
        ('deleted', '削除済み')
    ]

    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='questions'
    )
    title = models.CharField(
        '質問タイトル',
        max_length=200
    )
    content = models.TextField(
        '質問内容'
    )
    status = models.CharField(
        'ステータス',
        max_length=20,
        choices=STATUS_CHOICES,
        default='draft'
    )
    error_message = models.TextField(
        'エラーメッセージ',
        blank=True,
        null=True
    )
    processing_started_at = models.DateTimeField(
        '処理開始日時',
        null=True,
        blank=True
    )

    objects = QuestionQuerySet.as_manager()

    class Meta:
        verbose_name = '質問'
        verbose_name_plural = '質問'
        indexes = [
            models.Index(fields=['user', 'status']),
            models.Index(fields=['created_at']),
        ]

    def __str__(self):
        return self.title

    def clean(self):
        """バリデーション"""
        if len(self.title) < 5:
            raise ValidationError({
                'title': '質問のタイトルは5文字以上で入力してください。'
            })
        if len(self.content) < 20:
            raise ValidationError({
                'content': '質問内容は20文字以上で入力してください。'
            })

    def start_processing(self):
        """処理開始"""
        self.status = 'processing'
        self.processing_started_at = timezone.now()
        self.save()

    def complete_processing(self):
        """処理完了"""
        self.status = 'completed'
        self.save()

    def mark_as_error(self, message):
        """エラー状態にする"""
        self.status = 'error'
        self.error_message = message
        self.save()

    def can_be_edited(self):
        """編集可能かどうかを判定"""
        return self.status in ['draft', 'error']

    def get_processing_time(self):
        """処理時間を計算"""
        if self.processing_started_at and self.status == 'completed':
            return (self.updated_at - self.processing_started_at).total_seconds()
        return None

class Response(TimeStampedModel):
    AI_SERVICE_CHOICES = [
        ('claude', 'Claude'),
        ('chatgpt', 'ChatGPT'),
        ('final', '最終回答')
    ]

    VALIDATION_STATUS_CHOICES = [
        ('valid', '有効'),
        ('invalid', '無効'),
        ('needs_improvement', '改善が必要')
    ]

    question = models.ForeignKey(
        Question,
        on_delete=models.CASCADE,
        related_name='responses'
    )
    ai_service = models.CharField(
        'AIサービス',
        max_length=20,
        choices=AI_SERVICE_CHOICES
    )
    content = models.TextField(
        '回答内容'
    )
    validation_status = models.CharField(
        '検証状態',
        max_length=20,
        choices=VALIDATION_STATUS_CHOICES,
        null=True,
        blank=True
    )
    processing_time = models.IntegerField(
        '処理時間（ミリ秒）',
        null=True,
        blank=True
    )
    token_count = models.IntegerField(
        'トークン数',
        null=True,
        blank=True
    )
    feedback = models.JSONField(
        'フィードバック',
        default=dict,
        blank=True
    )

    class Meta:
        verbose_name = '回答'
        verbose_name_plural = '回答'
        indexes = [
            models.Index(fields=['question', 'ai_service']),
            models.Index(fields=['created_at']),
        ]

    def __str__(self):
        return f'{self.get_ai_service_display()}の回答 - {self.question.title}'

    def add_feedback(self, feedback_type, user=None):
        """フィードバックを追加"""
        if not self.feedback:
            self.feedback = {}

        timestamp = timezone.now().isoformat()
        self.feedback[timestamp] = {
            'type': feedback_type,
            'user_id': user.id if user else None
        }
        self.save()

    def get_feedback_summary(self):
        """フィードバックの集計"""
        if not self.feedback:
            return {
                'helpful': 0,
                'not_helpful': 0
            }

        helpful = sum(1 for f in self.feedback.values() if f['type'] == 'helpful')
        not_helpful = sum(1 for f in self.feedback.values() if f['type'] == 'not_helpful')

        return {
            'helpful': helpful,
            'not_helpful': not_helpful
        }

class QuestionHistory(TimeStampedModel):
    """質問の変更履歴"""
    question = models.ForeignKey(
        Question,
        on_delete=models.CASCADE,
        related_name='histories'
    )
    title = models.CharField(
        '質問タイトル',
        max_length=200
    )
    content = models.TextField(
        '質問内容'
    )
    changed_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        related_name='question_changes'
    )

    class Meta:
        verbose_name = '質問履歴'
        verbose_name_plural = '質問履歴'
        ordering = ['-created_at']

    def __str__(self):
        return f'{self.question.title}の履歴 - {self.created_at}'

# シグナル定義
from django.db.models.signals import pre_save
from django.dispatch import receiver

@receiver(pre_save, sender=Question)
def create_question_history(sender, instance, **kwargs):
    """質問が更新される際に履歴を作成"""
    if instance.pk:  # 既存の質問の場合のみ
        try:
            old_instance = Question.objects.get(pk=instance.pk)
            if (old_instance.title != instance.title or
                old_instance.content != instance.content):
                QuestionHistory.objects.create(
                    question=instance,
                    title=old_instance.title,
                    content=old_instance.content,
                    changed_by=instance.user
                )
        except Question.DoesNotExist:
            pass

```

この実装では以下の点に注意を払っています：

1. データモデリング
- 適切なリレーション設定
- インデックス最適化
- 履歴管理機能
1. ビジネスロジック
- 状態管理メソッド
- バリデーション
- フィードバック機能
1. パフォーマンス
- クエリの最適化
- 効率的なインデックス
- 適切なリレーション
1. 拡張性
- カスタムマネージャー
- フレキシブルな設定
- 履歴管理

次は、Forms部分の実装を示させていただきましょうか？

# サーバーサイド実装 - Forms

```python
# questions/forms.py
from django import forms
from django.core.exceptions import ValidationError
from django.utils.translation import gettext_lazy as _
from .models import Question, Response
from core.widgets import MarkdownEditorWidget

class QuestionForm(forms.ModelForm):
    """質問作成・編集フォーム"""

    class Meta:
        model = Question
        fields = ['title', 'content']
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
                'placeholder': '質問のタイトルを入力してください',
            }),
            'content': MarkdownEditorWidget(attrs={
                'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
                'placeholder': '質問内容を入力してください\\nMarkdown記法が使用できます',
                'rows': 10,
            })
        }
        error_messages = {
            'title': {
                'required': _('タイトルは必須です'),
                'max_length': _('タイトルは200文字以内で入力してください'),
            },
            'content': {
                'required': _('質問内容は必須です'),
            }
        }

    def clean_title(self):
        """タイトルのバリデーション"""
        title = self.cleaned_data['title']
        if len(title.strip()) < 5:
            raise ValidationError(
                _('タイトルは5文字以上で入力してください')
            )
        return title

    def clean_content(self):
        """質問内容のバリデーション"""
        content = self.cleaned_data['content']
        if len(content.strip()) < 20:
            raise ValidationError(
                _('質問内容は20文字以上で入力してください')
            )
        return content

    def clean(self):
        """フォーム全体のバリデーション"""
        cleaned_data = super().clean()
        title = cleaned_data.get('title', '')
        content = cleaned_data.get('content', '')

        # タイトルと内容が重複していないかチェック
        if title and content and title.strip() in content.strip():
            raise ValidationError(
                _('タイトルは質問内容と重複しないようにしてください')
            )

        return cleaned_data

class QuestionSearchForm(forms.Form):
    """質問検索フォーム"""

    q = forms.CharField(
        required=False,
        widget=forms.TextInput(attrs={
            'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
            'placeholder': 'キーワードで検索',
        })
    )

    status = forms.ChoiceField(
        required=False,
        choices=[('', '全てのステータス')] + Question.STATUS_CHOICES,
        widget=forms.Select(attrs={
            'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
        })
    )

    date_range = forms.ChoiceField(
        required=False,
        choices=[
            ('', '全期間'),
            ('today', '今日'),
            ('week', '過去7日間'),
            ('month', '過去30日間'),
            ('custom', 'カスタム期間')
        ],
        widget=forms.Select(attrs={
            'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
        })
    )

    date_from = forms.DateField(
        required=False,
        widget=forms.DateInput(attrs={
            'type': 'date',
            'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
        })
    )

    date_to = forms.DateField(
        required=False,
        widget=forms.DateInput(attrs={
            'type': 'date',
            'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
        })
    )

    def clean(self):
        cleaned_data = super().clean()
        date_range = cleaned_data.get('date_range')
        date_from = cleaned_data.get('date_from')
        date_to = cleaned_data.get('date_to')

        if date_range == 'custom':
            if not date_from and not date_to:
                raise ValidationError(
                    _('カスタム期間を選択した場合は、開始日または終了日を指定してください')
                )

            if date_from and date_to and date_from > date_to:
                raise ValidationError(
                    _('開始日は終了日より前の日付を指定してください')
                )

        return cleaned_data

class ResponseFeedbackForm(forms.Form):
    """回答フィードバックフォーム"""

    response_id = forms.IntegerField(
        widget=forms.HiddenInput()
    )

    feedback_type = forms.ChoiceField(
        choices=[
            ('helpful', '役に立った'),
            ('not_helpful', '役に立たなかった')
        ],
        widget=forms.RadioSelect(attrs={
            'class': 'form-radio'
        })
    )

    comment = forms.CharField(
        required=False,
        widget=forms.Textarea(attrs={
            'class': 'w-full px-3 py-2 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm',
            'placeholder': 'コメントがあれば入力してください',
            'rows': 3,
        })
    )

class BaseQuestionFormSet(forms.BaseModelFormSet):
    """質問フォームセット基底クラス"""

    def clean(self):
        """重複チェックなどの追加バリデーション"""
        if any(self.errors):
            return

        titles = []
        for form in self.forms:
            if self.can_delete and self._should_delete_form(form):
                continue

            title = form.cleaned_data.get('title')
            if title in titles:
                raise ValidationError(
                    _('同じタイトルの質問が複数含まれています')
                )
            titles.append(title)

QuestionFormSet = forms.modelformset_factory(
    Question,
    form=QuestionForm,
    formset=BaseQuestionFormSet,
    extra=1,
    can_delete=True
)

```

この実装では以下の点に注意を払っています：

1. バリデーション
- 複数層でのバリデーション
- カスタムバリデーションルール
- エラーメッセージの国際化
1. ユーザー体験
- 適切なウィジェット選択
- 分かりやすいエラーメッセージ
- プレースホルダーテキスト
1. セキュリティ
- XSS対策
- CSRF対策
- 入力値の適切なサニタイズ
1. 再利用性
- フォームセットの活用
- 共通コンポーネント化
- 拡張性の確保

これらのフォームは、各ビューで適切に利用され、ユーザーからの入力を安全に処理します。次は、Servicesレイヤーの実装を示させていただきましょうか？

# サーバーサイド実装 - Services

```python
# questions/services.py
import asyncio
import time
from typing import Dict, Optional
from django.conf import settings
from django.utils import timezone
from django.core.cache import cache
from django.db import transaction
from celery import shared_task
from asgiref.sync import async_to_sync

from .models import Question, Response
from core.ai import ClaudeClient, ChatGPTClient
from core.exceptions import AIServiceError, ValidationError

class QuestionService:
    """質問処理に関するサービスクラス"""

    def __init__(self):
        self.claude_client = ClaudeClient()
        self.chatgpt_client = ChatGPTClient()

    @shared_task(bind=True, max_retries=3)
    def process_question(self, question_id: int) -> None:
        """質問処理の実行"""
        question = Question.objects.get(id=question_id)

        try:
            # 処理開始
            question.start_processing()
            self._notify_status_update(question, 'processing')

            # 各AIからの回答を取得
            claude_response = async_to_sync(self._generate_claude_response)(question)
            self._notify_response_created(question, claude_response)

            chatgpt_response = async_to_sync(self._verify_with_chatgpt)(
                question, claude_response
            )
            self._notify_response_created(question, chatgpt_response)

            final_response = async_to_sync(self._generate_final_response)(
                question, claude_response, chatgpt_response
            )
            self._notify_response_created(question, final_response)

            # 処理完了
            question.complete_processing()
            self._notify_status_update(question, 'completed')

        except Exception as e:
            error_message = str(e)
            question.mark_as_error(error_message)
            self._notify_status_update(question, 'error', error_message)
            raise

    async def _generate_claude_response(self, question: Question) -> Response:
        """Claudeによる回答生成"""
        start_time = time.time()

        try:
            response_data = await self.claude_client.generate_response(
                question.content,
                max_tokens=settings.CLAUDE_MAX_TOKENS
            )

            processing_time = int((time.time() - start_time) * 1000)

            return Response.objects.create(
                question=question,
                ai_service='claude',
                content=response_data['content'],
                processing_time=processing_time,
                token_count=response_data.get('token_count')
            )

        except Exception as e:
            raise AIServiceError(f"Claude API Error: {str(e)}")

    async def _verify_with_chatgpt(
        self,
        question: Question,
        claude_response: Response
    ) -> Response:
        """ChatGPTによる検証"""
        start_time = time.time()

        try:
            response_data = await self.chatgpt_client.verify_response(
                question.content,
                claude_response.content,
                max_tokens=settings.CHATGPT_MAX_TOKENS
            )

            processing_time = int((time.time() - start_time) * 1000)

            return Response.objects.create(
                question=question,
                ai_service='chatgpt',
                content=response_data['content'],
                validation_status=response_data.get('validation_status'),
                processing_time=processing_time,
                token_count=response_data.get('token_count')
            )

        except Exception as e:
            raise AIServiceError(f"ChatGPT API Error: {str(e)}")

    async def _generate_final_response(
        self,
        question: Question,
        claude_response: Response,
        chatgpt_response: Response
    ) -> Response:
        """最終回答の生成"""
        start_time = time.time()

        try:
            response_data = await self.claude_client.generate_final_response(
                question.content,
                claude_response.content,
                chatgpt_response.content,
                max_tokens=settings.CLAUDE_MAX_TOKENS
            )

            processing_time = int((time.time() - start_time) * 1000)

            return Response.objects.create(
                question=question,
                ai_service='final',
                content=response_data['content'],
                processing_time=processing_time,
                token_count=response_data.get('token_count')
            )

        except Exception as e:
            raise AIServiceError(f"Final Response Error: {str(e)}")

    def cancel_processing(self, question_id: int) -> None:
        """処理のキャンセル"""
        question = Question.objects.get(id=question_id)

        if question.status != 'processing':
            raise ValidationError("処理中の質問のみキャンセルできます")

        # 処理を中断してエラー状態にする
        question.mark_as_error("ユーザーによってキャンセルされました")
        self._notify_status_update(question, 'error', "ユーザーによってキャンセルされました")

    def _notify_status_update(
        self,
        question: Question,
        status: str,
        error_message: Optional[str] = None
    ) -> None:
        """質問状態の更新を通知"""
        from channels.layers import get_channel_layer
        channel_layer = get_channel_layer()

        async_to_sync(channel_layer.group_send)(
            f'question_{question.id}',
            {
                'type': 'question.update',
                'status': status,
                'error_message': error_message
            }
        )

    def _notify_response_created(
        self,
        question: Question,
        response: Response
    ) -> None:
        """回答生成を通知"""
        from channels.layers import get_channel_layer
        channel_layer = get_channel_layer()

        async_to_sync(channel_layer.group_send)(
            f'question_{question.id}',
            {
                'type': 'response.update',
                'response': {
                    'id': response.id,
                    'ai_service': response.ai_service,
                    'content': response.content,
                    'validation_status': response.validation_status,
                    'processing_time': response.processing_time,
                    'token_count': response.token_count
                }
            }
        )

class ResponseService:
    """回答に関するサービスクラス"""

    def add_feedback(
        self,
        response_id: int,
        feedback_type: str,
        user_id: int
    ) -> None:
        """フィードバックの追加"""
        response = Response.objects.get(id=response_id)

        # 同じユーザーからの重複フィードバックをチェック
        cache_key = f'feedback_{response_id}_{user_id}'
        if cache.get(cache_key):
            raise ValidationError("既にフィードバックを送信済みです")

        response.add_feedback(feedback_type, user_id)
        cache.set(cache_key, True, timeout=86400)  # 24時間有効

        # フィードバック統計の更新
        self._update_feedback_stats(response)

    def _update_feedback_stats(self, response: Response) -> None:
        """フィードバック統計の更新"""
        stats = response.get_feedback_summary()
        cache_key = f'feedback_stats_{response.id}'
        cache.set(cache_key, stats, timeout=3600)  # 1時間有効

    @classmethod
    def get_feedback_stats(cls, response_id: int) -> Dict:
        """フィードバック統計の取得"""
        cache_key = f'feedback_stats_{response_id}'
        stats = cache.get(cache_key)

        if stats is None:
            response = Response.objects.get(id=response_id)
            stats = response.get_feedback_summary()
            cache.set(cache_key, stats, timeout=3600)

        return stats

```

この実装では以下の点に注意を払っています：

1. 非同期処理
- async/awaitの活用
- WebSocketによる通知
- タスクキュー処理
1. エラーハンドリング
- 適切な例外処理
- エラー状態の管理
- ユーザーへの通知
1. パフォーマンス
- キャッシュの活用
- 適切なトランザクション管理
- 効率的なデータ処理
1. 拡張性
- モジュール化された設計
- 設定の外部化
- インターフェースの一貫性