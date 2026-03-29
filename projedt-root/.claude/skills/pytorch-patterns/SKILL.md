---
name: pytorch-patterns
description: PyTorchディープラーニングパターンとベストプラクティス。堅牢で効率的かつ再現可能なトレーニングパイプライン、モデルアーキテクチャ、データローディングの構築に対応。
origin: ECC
---

# PyTorch開発パターン

堅牢で効率的かつ再現可能なディープラーニングアプリケーションを構築するためのPyTorchパターンとベストプラクティス。

## アクティベーション条件

- 新しいPyTorchモデルやトレーニングスクリプトを書くとき
- ディープラーニングコードをレビューするとき
- トレーニングループやデータパイプラインのデバッグ時
- GPUメモリ使用量やトレーニング速度の最適化時
- 再現可能な実験のセットアップ時

## 基本原則

### 1. デバイス非依存コード

デバイスをハードコードせず、CPUとGPUの両方で動作するコードを常に書きます。

```python
# Good: デバイス非依存
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = MyModel().to(device)
data = data.to(device)

# Bad: デバイスのハードコード
model = MyModel().cuda()  # Crashes if no GPU
data = data.cuda()
```

### 2. 再現性の優先

再現可能な結果のためにすべてのランダムシードを設定します。

```python
# Good: 完全な再現性セットアップ
def set_seed(seed: int = 42) -> None:
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    np.random.seed(seed)
    random.seed(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

# Bad: シード制御なし
model = MyModel()  # Different weights every run
```

### 3. 明示的な形状管理

テンソルの形状を常にドキュメント化し検証します。

```python
# Good: 形状アノテーション付きフォワードパス
def forward(self, x: torch.Tensor) -> torch.Tensor:
    # x: (batch_size, channels, height, width)
    x = self.conv1(x)    # -> (batch_size, 32, H, W)
    x = self.pool(x)     # -> (batch_size, 32, H//2, W//2)
    x = x.view(x.size(0), -1)  # -> (batch_size, 32*H//2*W//2)
    return self.fc(x)    # -> (batch_size, num_classes)

# Bad: 形状追跡なし
def forward(self, x):
    x = self.conv1(x)
    x = self.pool(x)
    x = x.view(x.size(0), -1)  # What size is this?
    return self.fc(x)           # Will this even work?
```

## モデルアーキテクチャパターン

### 整理されたnn.Module構造

```python
# Good: よく整理されたモジュール
class ImageClassifier(nn.Module):
    def __init__(self, num_classes: int, dropout: float = 0.5) -> None:
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
        )
        self.classifier = nn.Sequential(
            nn.Dropout(dropout),
            nn.Linear(64 * 16 * 16, num_classes),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.features(x)
        x = x.view(x.size(0), -1)
        return self.classifier(x)

# Bad: すべてがforward内
class ImageClassifier(nn.Module):
    def __init__(self):
        super().__init__()

    def forward(self, x):
        x = F.conv2d(x, weight=self.make_weight())  # Creates weight each call!
        return x
```

### 適切な重み初期化

```python
# Good: 明示的な初期化
def _init_weights(self, module: nn.Module) -> None:
    if isinstance(module, nn.Linear):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
        if module.bias is not None:
            nn.init.zeros_(module.bias)
    elif isinstance(module, nn.Conv2d):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
    elif isinstance(module, nn.BatchNorm2d):
        nn.init.ones_(module.weight)
        nn.init.zeros_(module.bias)

model = MyModel()
model.apply(model._init_weights)
```

## トレーニングループパターン

### 標準トレーニングループ

```python
# Good: ベストプラクティスを含む完全なトレーニングループ
def train_one_epoch(
    model: nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    criterion: nn.Module,
    device: torch.device,
    scaler: torch.amp.GradScaler | None = None,
) -> float:
    model.train()  # Always set train mode
    total_loss = 0.0

    for batch_idx, (data, target) in enumerate(dataloader):
        data, target = data.to(device), target.to(device)

        optimizer.zero_grad(set_to_none=True)  # More efficient than zero_grad()

        # 混合精度トレーニング
        with torch.amp.autocast("cuda", enabled=scaler is not None):
            output = model(data)
            loss = criterion(output, target)

        if scaler is not None:
            scaler.scale(loss).backward()
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            scaler.step(optimizer)
            scaler.update()
        else:
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()

        total_loss += loss.item()

    return total_loss / len(dataloader)
```

### バリデーションループ

```python
# Good: 適切な評価
@torch.no_grad()  # torch.no_grad()ブロックで囲むより効率的
def evaluate(
    model: nn.Module,
    dataloader: DataLoader,
    criterion: nn.Module,
    device: torch.device,
) -> tuple[float, float]:
    model.eval()  # Always set eval mode — disables dropout, uses running BN stats
    total_loss = 0.0
    correct = 0
    total = 0

    for data, target in dataloader:
        data, target = data.to(device), target.to(device)
        output = model(data)
        total_loss += criterion(output, target).item()
        correct += (output.argmax(1) == target).sum().item()
        total += target.size(0)

    return total_loss / len(dataloader), correct / total
```

## データパイプラインパターン

### カスタムDataset

```python
# Good: 型ヒント付きの整理されたDataset
class ImageDataset(Dataset):
    def __init__(
        self,
        image_dir: str,
        labels: dict[str, int],
        transform: transforms.Compose | None = None,
    ) -> None:
        self.image_paths = list(Path(image_dir).glob("*.jpg"))
        self.labels = labels
        self.transform = transform

    def __len__(self) -> int:
        return len(self.image_paths)

    def __getitem__(self, idx: int) -> tuple[torch.Tensor, int]:
        img = Image.open(self.image_paths[idx]).convert("RGB")
        label = self.labels[self.image_paths[idx].stem]

        if self.transform:
            img = self.transform(img)

        return img, label
```

### 効率的なDataLoader設定

```python
# Good: 最適化されたDataLoader
dataloader = DataLoader(
    dataset,
    batch_size=32,
    shuffle=True,            # トレーニング時はシャッフル
    num_workers=4,           # 並列データロード
    pin_memory=True,         # 高速なCPU→GPU転送
    persistent_workers=True, # エポック間でワーカーを維持
    drop_last=True,          # BatchNorm用の一定バッチサイズ
)

# Bad: 遅いデフォルト
dataloader = DataLoader(dataset, batch_size=32)  # num_workers=0, no pin_memory
```

### 可変長データ用カスタムcollate

```python
# Good: collate_fnでシーケンスをパディング
def collate_fn(batch: list[tuple[torch.Tensor, int]]) -> tuple[torch.Tensor, torch.Tensor]:
    sequences, labels = zip(*batch)
    # バッチ内の最大長にパディング
    padded = nn.utils.rnn.pad_sequence(sequences, batch_first=True, padding_value=0)
    return padded, torch.tensor(labels)

dataloader = DataLoader(dataset, batch_size=32, collate_fn=collate_fn)
```

## チェックポイントパターン

### チェックポイントの保存と読み込み

```python
# Good: すべてのトレーニング状態を含む完全なチェックポイント
def save_checkpoint(
    model: nn.Module,
    optimizer: torch.optim.Optimizer,
    epoch: int,
    loss: float,
    path: str,
) -> None:
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "loss": loss,
    }, path)

def load_checkpoint(
    path: str,
    model: nn.Module,
    optimizer: torch.optim.Optimizer | None = None,
) -> dict:
    checkpoint = torch.load(path, map_location="cpu", weights_only=True)
    model.load_state_dict(checkpoint["model_state_dict"])
    if optimizer:
        optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
    return checkpoint

# Bad: モデルの重みのみ保存（トレーニング再開不可）
torch.save(model.state_dict(), "model.pt")
```

## パフォーマンス最適化

### 混合精度トレーニング

```python
# Good: AMPとGradScaler
scaler = torch.amp.GradScaler("cuda")
for data, target in dataloader:
    with torch.amp.autocast("cuda"):
        output = model(data)
        loss = criterion(output, target)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
    optimizer.zero_grad(set_to_none=True)
```

### 大規模モデル向け勾配チェックポイント

```python
# Good: 計算とメモリのトレードオフ
from torch.utils.checkpoint import checkpoint

class LargeModel(nn.Module):
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # メモリ節約のため逆伝播時にアクティベーションを再計算
        x = checkpoint(self.block1, x, use_reentrant=False)
        x = checkpoint(self.block2, x, use_reentrant=False)
        return self.head(x)
```

### 高速化のためのtorch.compile

```python
# Good: 高速実行のためにモデルをコンパイル（PyTorch 2.0以降）
model = MyModel().to(device)
model = torch.compile(model, mode="reduce-overhead")

# Modes: "default"（安全）、"reduce-overhead"（高速）、"max-autotune"（最速）
```

## クイックリファレンス: PyTorchイディオム

| イディオム | 説明 |
|-------|-------------|
| `model.train()` / `model.eval()` | トレーニング/評価前に必ずモードを設定 |
| `torch.no_grad()` | 推論時に勾配を無効化 |
| `optimizer.zero_grad(set_to_none=True)` | より効率的な勾配クリア |
| `.to(device)` | デバイス非依存のテンソル/モデル配置 |
| `torch.amp.autocast` | 2倍速の混合精度 |
| `pin_memory=True` | 高速なCPU→GPUデータ転送 |
| `torch.compile` | 速度向上のためのJITコンパイル（2.0以降） |
| `weights_only=True` | 安全なモデル読み込み |
| `torch.manual_seed` | 再現可能な実験 |
| `gradient_checkpointing` | 計算とメモリのトレードオフ |

## 避けるべきアンチパターン

```python
# Bad: バリデーション時にmodel.eval()を忘れる
model.train()
with torch.no_grad():
    output = model(val_data)  # Dropout still active! BatchNorm uses batch stats!

# Good: 常にevalモードを設定
model.eval()
with torch.no_grad():
    output = model(val_data)

# Bad: autogradを壊すインプレース操作
x = F.relu(x, inplace=True)  # Can break gradient computation
x += residual                  # In-place add breaks autograd graph

# Good: アウトオブプレース操作
x = F.relu(x)
x = x + residual

# Bad: トレーニングループ内でGPUへの繰り返し移動
for data, target in dataloader:
    model = model.cuda()  # Moves model EVERY iteration!

# Good: ループ前にモデルを一度だけ移動
model = model.to(device)
for data, target in dataloader:
    data, target = data.to(device), target.to(device)

# Bad: backward前に.item()を使用
loss = criterion(output, target).item()  # Detaches from graph!
loss.backward()  # Error: can't backprop through .item()

# Good: ロギング用にのみ.item()を呼ぶ
loss = criterion(output, target)
loss.backward()
print(f"Loss: {loss.item():.4f}")  # .item() after backward is fine

# Bad: torch.saveの不適切な使用
torch.save(model, "model.pt")  # Saves entire model (fragile, not portable)

# Good: state_dictを保存
torch.save(model.state_dict(), "model.pt")
```

__覚えておくこと__: PyTorchコードはデバイス非依存、再現可能、メモリ意識の高いものであるべきです。迷ったときは `torch.profiler` でプロファイリングし、`torch.cuda.memory_summary()` でGPUメモリを確認してください。
