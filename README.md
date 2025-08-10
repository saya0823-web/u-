# SNS投稿の「いいね」予測・示唆（1枚サマリ） — for Saya
*作成日: 2025-08-09 / データ: サンプル60件（SNS投稿CSV）*

## 目的
- 投稿の **「いいね（likes）」を予測** し、運用に活かせる **配信タイミング/施策** の当たりをつける。

## 手順（超要約）
1. 前処理：`media_type`, `weekday` は One-Hot（drop='first'）、数値は標準化。
2. モデル：線形回帰 & Ridge 回帰（CVでα自動選択）。train/test=80/20。
3. 妥当性チェック：平均予測（ベースライン）比較、`impressions`除外の検証、係数/Permutationでドライバー把握。

## 主な結果
- **MAE（平均予測）= 208.40 → 線形= 88.68（R²=0.820）**
- **Ridge= 63.62（R²=0.909, α=4.28）**  ← 係数が安定しやすく、**最終採用**
- 集計より：**18時が最も平均エンゲージメント高い**（夜帯強め）
- ドライバー概観：**reach ＞ 曜日差 ＞ media_type**

## 示唆（Saya向け）
- **夜18時中心で投稿比率↑**、`reel`起点の企画を重点チェック。
- **到達の質（保存/コメント率）**にフォーカス。`reach`の増加が likes に直結。
- `impressions`は冗長気味 → **レポートは率（likes/imp）も並記**して質を監視。

## 図（Ridge: 実測 vs 予測）
![Actual vs Predicted](./scatter_ridge_actual_vs_pred.png)

---
### 付録：数表
| 指標 | 値 |
|---|---:|
| MAE（平均予測） | 208.40 |
| MAE / R²（線形） | 88.68 / 0.820 |
| MAE / R²（Ridge, α=4.28） | 63.62 / 0.909 |

> 一言：**Ridgeで精度と解釈のバランス良好。reachが主因 → 夜帯×保存/コメント設計で伸ばす。**
>
> ## TL;DR
Ridge回帰で likes 予測（R²=0.909 / MAE=63.6）。主要因は reach。夜18時中心×保存/コメント設計を推奨。

## Contact
お仕事/ご相談: ✉️ <あなたの連絡先 or SNSリンク>

## TL;DR
Ridge回帰で「いいね」を予測（R²=0.909 / MAE=63.6）。主要因は reach。夜18時中心×保存/コメント設計で「到達の質」を上げると効果が見込める。

## Contact
お仕事/ご相談: ✉️ tianzhongzaoji80@gmail.com ｜ X: https://x.com/1046vsaki_saya

# 1) 特徴量作成
import numpy as np, pandas as pd
df["save_rate"] = (df["saves"] / df["impressions"]).clip(0, 1)

# 2) 学習：Ridge回帰（impressionsは目的の分母なので使わない）
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.linear_model import RidgeCV
from sklearn.metrics import mean_absolute_error, r2_score

X = df[["hour","media_type","weekday","reach"]]  # 分母のimpressionsは除外
y = df["save_rate"]

Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.2, random_state=42)

pre = ColumnTransformer([
    ("cat", OneHotEncoder(drop='first', handle_unknown='ignore'), ["media_type","weekday"]),
    ("num", StandardScaler(), ["hour","reach"])
])

model = Pipeline([("prep", pre), ("reg", RidgeCV(alphas=np.logspace(-3,3,20), cv=5))]).fit(Xtr, ytr)
pred = model.predict(Xte)

mae_base = mean_absolute_error(yte, np.full_like(yte, ytr.mean(), dtype=float))
mae = mean_absolute_error(yte, pred); r2 = r2_score(yte, pred)
model.named_steps["reg"].alpha_, mae_base, mae, r2

# 3) 可視化：実測 vs 予測（保存率）
import matplotlib.pyplot as plt
plt.figure()
plt.scatter(yte, pred, alpha=0.7)
mn, mx = float(min(yte.min(), pred.min())), float(max(yte.max(), pred.max()))
plt.plot([mn, mx], [mn, mx])
plt.xlabel("Actual save_rate"); plt.ylabel("Predicted save_rate"); plt.title("Actual vs Predicted (save_rate)")
plt.show()

# 4) 重要度ざっくり：Permutation Importance（列単位）
from sklearn.inspection import permutation_importance
cols = ["hour","media_type","weekday","reach"]
r = permutation_importance(model, Xte, yte, n_repeats=20, random_state=42)
pd.Series(r.importances_mean, index=cols).sort_values(ascending=False)

### 保存率の予測（結果）
Ridgeで save_rate を回帰。ベースラインMAE→モデルMAEで○○%改善。重要因子は reach／weekday／media_type の順。→ 夜帯×保存を促す構成（チェックリスト/スワイプ解説/CTA）を強化。

### 保存率（save_rate）の予測
- モデル：Ridge（重みなし）
- 指標：MAE(平均予測)=**0.0041** → モデル=**0.0041**（改善 **0%**）／ R²=**-0.015**
- 解釈：現行の特徴（hour/weekday/media_type/reach）だけでは保存率の説明力がほぼ出ず、**データ生成のランダム性**の影響が大きい。

**次の打ち手**
- impressionsで**重み付きRidge**に（試行回数を重視）
- もしくは **二項GLM（ロジット）**で `saves ~ impressions` を正しく扱う
- コンテンツ特徴を追加（キャプション長・ハッシュタグ数・媒体ごとの固定効果）

<!-- 画像を出力＆アップしていれば下を有効化 -->
<!-- ![Actual vs Predicted (save_rate)](./scatter_save_rate.png) -->
<!-- ![Permutation importance (save_rate)](./permimp_save_rate.png) -->

## 保存率（save_rate）の予測

- **モデル**：Ridge（重みなし）
- **指標**：MAE(平均予測)=**0.0041** → モデル=**0.0041**（**改善 0%**）／ **R²=-0.015**
- **解釈**：現行の特徴（hour / weekday / media_type / reach）だけでは保存率の説明力がほぼ出ず、データ生成のランダム性の影響が大きい。

**次の打ち手**
- impressionsで**重み付きRidge**に（試行回数を重視）
- または **二項GLM（ロジット）**で `saves ~ impressions` を正しく扱う
- **コンテンツ特徴**を追加（キャプション長、ハッシュタグ数、媒体ごとの固定効果 など）

<img width="640" height="480" alt="interaction_hour_media_curves" src="https://github.com/user-attachments/assets/59748916-34a5-47c0-944b-f8fc7bc78f49" />
<img width="640" height="480" alt="coeffs_interaction" src="https://github.com/user-attachments/assets/14fa0433-be55-4573-ba33-c48fd8560ac1" />

### 時刻×媒体の交互作用（likes）

- **モデル**：Ridge（交互作用 `hour×media_type`、共変量 `reach`）
- **指標**：**R²=0.906 / MAE=68.5**
- **結果**：予測曲線より **夜×carousel の傾きが最も大きい**。reel は中位、image は控えめ。
- **施策**：夜は **carousel を厚め**、昼は image / reel のABテスト。保存・コメント導線で“質”を上げる。

<p align="center">
  <img width="720" alt="時間×メディアカーブ" src="https://github.com/user-attachments/assets/59748916-34a5-47c0-944b-f8fc7bc78f49" />
</p>
<p align="center">
  <img width="720" alt="係数" src="https://github.com/user-attachments/assets/14fa0433-be55-4573-ba33-c48fd8560ac1" />
</p>
