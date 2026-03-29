---
description: 要件を再確認し、リスクを評価し、ステップバイステップの実装プランを作成。コードに触れる前にユーザーの確認を待つ。
---

# プランコマンド

このコマンドは **planner** エージェントを呼び出し、コードを書く前に包括的な実装プランを作成します。

## このコマンドの動作

1. **要件の再確認** - 何を構築する必要があるかを明確化
2. **リスクの特定** - 潜在的な問題やブロッカーを洗い出す
3. **ステッププランの作成** - 実装をフェーズに分解
4. **確認を待つ** - 進行前にユーザーの承認を必ず受ける

## 使用するタイミング

`/plan` は以下の場合に使用します：
- 新しい機能の開始時
- 重要なアーキテクチャ変更を行う場合
- 複雑なリファクタリングに取り組む場合
- 複数のファイル/コンポーネントが影響を受ける場合
- 要件が不明確または曖昧な場合

## 動作の仕組み

Plannerエージェントは以下を行います：

1. **リクエストを分析** し、要件を明確な言葉で再確認
2. **フェーズに分解** 具体的で実行可能なステップに
3. **依存関係を特定** コンポーネント間の
4. **リスクを評価** 潜在的なブロッカーを
5. **複雑さを見積もり** （高/中/低）
6. **プランを提示** し、明示的な確認を待つ

## 使用例

```
User: /plan I need to add real-time notifications when markets resolve

Agent (planner):
# Implementation Plan: Real-Time Market Resolution Notifications

## Requirements Restatement
- Send notifications to users when markets they're watching resolve
- Support multiple notification channels (in-app, email, webhook)
- Ensure notifications are delivered reliably
- Include market outcome and user's position result

## Implementation Phases

### Phase 1: Database Schema
- Add notifications table with columns: id, user_id, market_id, type, status, created_at
- Add user_notification_preferences table for channel preferences
- Create indexes on user_id and market_id for performance

### Phase 2: Notification Service
- Create notification service in lib/notifications.ts
- Implement notification queue using BullMQ/Redis
- Add retry logic for failed deliveries
- Create notification templates

### Phase 3: Integration Points
- Hook into market resolution logic (when status changes to "resolved")
- Query all users with positions in market
- Enqueue notifications for each user

### Phase 4: Frontend Components
- Create NotificationBell component in header
- Add NotificationList modal
- Implement real-time updates via Supabase subscriptions
- Add notification preferences page

## Dependencies
- Redis (for queue)
- Email service (SendGrid/Resend)
- Supabase real-time subscriptions

## Risks
- HIGH: Email deliverability (SPF/DKIM required)
- MEDIUM: Performance with 1000+ users per market
- MEDIUM: Notification spam if markets resolve frequently
- LOW: Real-time subscription overhead

## Estimated Complexity: MEDIUM
- Backend: 4-6 hours
- Frontend: 3-4 hours
- Testing: 2-3 hours
- Total: 9-13 hours

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## 重要な注意事項

**重要**: Plannerエージェントは、「yes」「proceed」などの肯定的な応答で明示的にプランを確認するまで、**コードを一切書きません**。

変更を希望する場合は、以下のように応答してください：
- "modify: [変更内容]"
- "different approach: [代替案]"
- "skip phase 2 and do phase 3 first"

## 他のコマンドとの連携

プランニング後：
- テスト駆動開発で実装するには `/tdd` を使用
- ビルドエラーが発生した場合は `/build-fix` を使用
- 完成した実装をレビューするには `/code-review` を使用

## 関連エージェント

このコマンドはECCが提供する `planner` エージェントを呼び出します。

手動インストールの場合、ソースファイルは以下にあります：
`agents/planner.md`
