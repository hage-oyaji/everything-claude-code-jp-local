---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript コーディングスタイル

> このファイルは [common/coding-style.md](../common/coding-style.md) を TypeScript/JavaScript 固有の内容で拡張します。

## 型とインターフェース

パブリック API、共有モデル、コンポーネントの props を明示的、読みやすく、再利用可能にするために型を使用。

### パブリック API

- エクスポートされた関数、共有ユーティリティ、パブリッククラスメソッドにパラメータ型と戻り値型を追加
- 明白なローカル変数の型は TypeScript の推論に任せる
- 繰り返されるインラインオブジェクト形状は名前付き型またはインターフェースに抽出

```typescript
// WRONG: Exported function without explicit types
export function formatUser(user) {
  return `${user.firstName} ${user.lastName}`
}

// CORRECT: Explicit types on public APIs
interface User {
  firstName: string
  lastName: string
}

export function formatUser(user: User): string {
  return `${user.firstName} ${user.lastName}`
}
```

### インターフェース vs 型エイリアス

- 拡張または実装される可能性のあるオブジェクト形状には `interface` を使用
- ユニオン、インターセクション、タプル、マップ型、ユーティリティ型には `type` を使用
- 相互運用性のために `enum` が必要でない限り、文字列リテラルユニオンを `enum` よりも優先

```typescript
interface User {
  id: string
  email: string
}

type UserRole = 'admin' | 'member'
type UserWithRole = User & {
  role: UserRole
}
```

### `any` を避ける

- アプリケーションコードでは `any` を避ける
- 外部または信頼できない入力には `unknown` を使い、安全にナローイング
- 値の型が呼び出し元に依存する場合はジェネリクスを使用

```typescript
// WRONG: any removes type safety
function getErrorMessage(error: any) {
  return error.message
}

// CORRECT: unknown forces safe narrowing
function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message
  }

  return 'Unexpected error'
}
```

### React Props

- コンポーネントの props は名前付き `interface` または `type` で定義
- コールバック props は明示的に型付け
- 特別な理由がない限り `React.FC` は使わない

```typescript
interface User {
  id: string
  email: string
}

interface UserCardProps {
  user: User
  onSelect: (id: string) => void
}

function UserCard({ user, onSelect }: UserCardProps) {
  return <button onClick={() => onSelect(user.id)}>{user.email}</button>
}
```

### JavaScript ファイル

- `.js` および `.jsx` ファイルでは、型が明確さを向上させ TypeScript への移行が現実的でない場合に JSDoc を使用
- JSDoc をランタイムの動作と一致させる

```javascript
/**
 * @param {{ firstName: string, lastName: string }} user
 * @returns {string}
 */
export function formatUser(user) {
  return `${user.firstName} ${user.lastName}`
}
```

## イミュータビリティ

イミュータブルな更新にはスプレッド演算子を使用：

```typescript
interface User {
  id: string
  name: string
}

// WRONG: Mutation
function updateUser(user: User, name: string): User {
  user.name = name // MUTATION!
  return user
}

// CORRECT: Immutability
function updateUser(user: Readonly<User>, name: string): User {
  return {
    ...user,
    name
  }
}
```

## エラーハンドリング

async/await と try-catch を使い、unknown エラーを安全にナローイング：

```typescript
interface User {
  id: string
  email: string
}

declare function riskyOperation(userId: string): Promise<User>

function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message
  }

  return 'Unexpected error'
}

const logger = {
  error: (message: string, error: unknown) => {
    // Replace with your production logger (for example, pino or winston).
  }
}

async function loadUser(userId: string): Promise<User> {
  try {
    const result = await riskyOperation(userId)
    return result
  } catch (error: unknown) {
    logger.error('Operation failed', error)
    throw new Error(getErrorMessage(error))
  }
}
```

## 入力バリデーション

スキーマベースのバリデーションには Zod を使い、スキーマから型を推論：

```typescript
import { z } from 'zod'

const userSchema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150)
})

type UserInput = z.infer<typeof userSchema>

const validated: UserInput = userSchema.parse(input)
```

## Console.log

- プロダクションコードに `console.log` 文を残さない
- 代わりに適切なロギングライブラリを使用
- 自動検出についてはフックを参照
