# RobotLabelBridge

[English](README.md) | [日本語](README_jp.md)

ロボットの joint / link 名を、プロジェクト内で共通に使う短い正式名——いわゆる **カノニカル短名**——にそろえるための Python モジュールです。  
`LeftHip` や `hip_pitch_joint`、`l_hip` のように表記は違っても同じ軸を指すことがよくあります。ニックネームがいくつあっても名簿では本名を決めておくと安心なのと同じで、こちらもプロジェクト内の共通語を一つに寄せます。

URDF・MJCF・ROS・製品固有フォーマットなど、由来の違う名前を  
「意味は同じなのに表記が違う」まま放置すると、変換ミス・検索漏れ・制御の取り違えが起きます。  
RobotLabelBridge はその橋渡し（bridge）を担当します。

見た目は「名前変換ライブラリ」ですが、中では部品なのか、動かす軸なのか、ただの別名なのか、親子でどうつながるのかを分けて持つ、軽い意味の整理——いわゆる **オントロジー**——があります。  
学術向けの巨大な仕組みではなく、名前を間違えにくくするための実務向けの意味モデルです。普段触るのは短いカノニカル名と変換 API で、裏側のその整理が誤変換を抑えます。

| ファイル | 役割 |
|---|---|
| `RobotLabelBridge.py` | 変換 API・オントロジー registry・検証・リネーム・Viewer |
| `RobotLabelBridge_Master.json` | カノニカル定義・別名・リンク階層などのマスタ知識 |

- Python API: **2.2.0**
- Master: `schema_version` **2.2** / `ontology_version` **1.2**

---

## 何のためのものか

### 目的

1. **名前の標準化**  
   `LeftHip` / `hip_pitch_joint` / `l_hip` などを、共通ルールの短名（例: `l_hipjoint_yp`）へ寄せる
2. **意味の取り違え防止**  
   「見た目が似てる名前」と「実際に指す関節・リンク」を分けて扱う
3. **モデル変換の自動化**  
   URDF/MJCF 上の joint 名を、衝突チェック付きで一括リネームできる
4. **LegacyMotionEditor との連携**  
   エディタ上のロボットモデルに対して、同じ規則で rename plan を作れる

### 向いている用途

- 複数メーカー・複数フォーマットのヒューマノイド / 動物型モデルを扱う
- モーションデータや制御ロジックを、機種をまたいで再利用したい
- 「この別名はどの正式名？」を機械的に解決したい
- リンクの親子関係を見て、曖昧な名前を絞り込したい

### 向いていない用途

- 物理シミュレーションそのもの
- 学術用途の完全 formal OWL オントロジー基盤そのものの代替
- 曖昧な候補を勝手に 1 つに確定すること（意図的に避けています）

---

## 内部で何を区別しているか

RobotLabelBridge は、単なる文字列置換表ではありません。  
Master JSON と `OntologyRegistry` の中で、たとえば次のような意味の違いを分けて持っています。

| 内部概念 | かんたんな意味 | 表に出る名前の例 |
|---|---|---|
| **Canonical label** | アプリで使う共通の短い名前 | `l_knee_yp` |
| **Alias** | 外から来た別表記 | `LeftKnee`, `knee_pitch_joint` |
| **FunctionalJoint** | 「左ひざ」など機能としての関節 | （内部 id: `joint:l:knee`） |
| **DegreeOfFreedom** | 実際に動かす 1 軸 | `l_knee_yp` |
| **KinematicJoint** | リンク同士をつなぐ物理的な継手 | 駆動軸でも、モーターなしでも可 |
| **PhysicalLink / VirtualLink** | 剛体パーツ、または軸分解の中間節 | `l_leg_upper` |
| **link_tree** | 親子の木構造（輪にしない） | parent / children |
| **KinematicLoop など** | 閉リンク機構（必要なときだけ） | `l_knee_lp` |

こう分けておくと:

- 表記が違っても同じ関節、似ていても別物、を扱いやすい
- 機能としての関節・実際に動かす軸・物理的な継手を混同しにくい
- 変換結果に根拠（provenance）を残せる
- Master の整合を validator で機械検査できる

OWL や rdflib は必須ではありません。必要なら `iter_semantic_triples()` で関係を書き出せます。  
普段は意識しなくてもよく、`canonicalize_*` や `convert_with_provenance` が内部の整理を使って名前を解決します。中身を直接見るときは `get_ontology_registry()` です。

```python
from RobotLabelBridge import get_ontology_registry

reg = get_ontology_registry()
reg.dofs_by_label["l_knee_yp"]       # DegreeOfFreedom
reg.links_by_label["l_leg_upper"]    # Link
reg.functional_joints                # 機能関節の一覧
```

---

## 何が嬉しいか（利点）

| 利点 | 内容 |
|---|---|
| **短い・一貫した名前** | `l_shoulder_yp` のように、左右・部位・軸が名前から読める |
| **意味モデルがある** | 別名・関節・軸・リンク・親子を、ただの文字列以上として扱える |
| **別名に強い** | Master の alias 辞書で、レガシー名や製品名を吸収できる |
| **曖昧さを隠さない** | 候補が複数あるときは AMBIGUOUS として返せる（誤変換しにくい） |
| **根拠が残る** | どの alias / トポロジ / ヒューリスティックで決めたかを追跡できる |
| **木構造を壊さない** | 通常の親子関係は `link_tree`、閉リンクは別管理 |
| **ヘッドレス利用可** | 変換・検証だけなら PySide6 なしでも使える |
| **Editor 連携** | LegacyMotionEditor から同じ規則で一括 rename できる |

要するに、**「名前の方言」を共通語に翻訳し、しかも裏側の意味モデルで誤訳しにくくする**ための道具です。

---

## 命名ルール（短い正式名）

ここが RobotLabelBridge の中心です。プロジェクト内で共通に使う短い正式名の形を決めます。

### 基本形

```text
関節軸 (DOF):  [l|r|c]_<部位>_[任意の番号]_<軸>
リンク:        [l|r|c]_<部位やセグメント>
```

| 部品 | 意味 | 例 |
|---|---|---|
| `l` / `r` / `c` | 左 / 右 / 中央 | `l_...`, `c_pelvis` |
| `<部位>` | 機能的な場所 | `shoulder`, `knee`, `hipjoint` |
| `_01`, `_02` … | 同じ部位が複数あるときの番号 | `l_spine_01_yp` |
| `<軸>` | 回転軸の短号 | `xr` / `yp` / `zy` |

### 軸トークン

| 短名 | 意味 | おおよその軸 |
|---|---|---|
| `xr` | roll（縦軸まわりに近いロール） | X |
| `yp` | pitch（前後に倒すピッチ） | Y |
| `zy` | yaw（左右に振るヨー） | Z |

正式な長い形は `xroll` / `ypitch` / `zyaw` ですが、  
**カノニカル短名では `xr` / `yp` / `zy` を使います。**

### 具体例

| 種類 | カノニカル短名 | 読み方 |
|---|---|---|
| DOF | `l_shoulder_yp` | 左肩のピッチ軸 |
| DOF | `l_hipjoint_yp` | 左股関節のピッチ軸 |
| DOF | `c_head_zy` | 頭部ヨー |
| Link | `l_arm_upper` | 左上腕リンク |
| Link | `l_leg_lower` | 左下腿リンク |
| Link | `c_pelvis` | 骨盤（中央） |

### よくある注意

- **`hip` や `waist` をカノニカル名にそのまま使わない**  
  - 股関節 → `hipjoint`  
  - 腰まわりの根 → `pelvis` / `pelvis_root` など
- **左右プレフィックスは `l_` / `r_` / `c_`**  
  `left_` / `right_` はレガシー扱い（alias では受けられる）
- **リンク名に `_link` を無理に付けない短名もある**  
  Master 上の canonical target は短名中心。ROS 用の長い名前は別フィールド
- **番号の省略ルール（singleton）**  
  同じ部位が 1 段だけなら `_01` を省略できる。  
  2 段目が増えたら、既存を `_01`、新規を `_02` にする

### 閉リンクがある場合の名前

通常の関節・リンク名は変えません。  
閉リンク機構そのものには別の短い名前を付けます。

```text
閉リンク本体:  [l|r|c]_<landmark>_lp[_NN]
経路 (branch): <loop>_bNN
閉じ拘束:      <loop>_cl[_NN]
```

例:

```text
l_knee_lp
l_knee_lp_b01
l_knee_lp_b02
l_knee_lp_cl
```

`four_bar` や `pantograph` といった機構種別名は、名前には入れず属性として持ちます。  
「四節リンクのひざ」でも、関節軸の正式名は従来どおり `l_knee_yp` のままです。

---

## 名前のレイヤ（混乱しやすい点）

同じ「関節」という言葉でも、中では別物として持ちます。

| 呼び方 | 何を指すか | 例 |
|---|---|---|
| **カノニカル短名** | アプリ内の共通名 | `l_knee_yp` |
| **別名 (alias)** | 外から来た表記 | `LeftKnee`, `knee_pitch_joint` |
| **FunctionalJoint** | 「左ひざ」という機能のまとまり | `joint:l:knee` |
| **DegreeOfFreedom** | 実際に動かす 1 軸（servo 軸） | `l_knee_yp` |
| **KinematicJoint** | リンク同士をつなぐ物理的な接続 | 駆動軸でも、モーターなしの継手でもよい |
| **Link** | 剛体パーツ | `l_leg_upper` |

特に重要な分離:

- **DOF ≠ すべての継手**  
  モーターなし（passive）の継手を、無理に servo 名へ変換しない
- **機能関節 ≠ 物理継手**  
  「ひざ」1 つが、機構上は複数の継手で実現されることがある
- **親子ツリー ≠ 閉リンク**  
  通常の体幹・手足の親子は木構造。輪になる機構は別データで持つ

この分離があるから、名前変換は「文字列マッチ」以上の判断ができます。

---

## 使い方

### 1. いちばん簡単な変換

```python
from RobotLabelBridge import (
    canonicalize_best_effort,
    canonicalize_strict,
)

# いちばん有力な候補を返す（便利だが取り違えの余地あり）
canonicalize_best_effort("l_hip")
# -> "l_hipjoint_yp"

# 一意に決まるときだけ返す。曖昧なら None
canonicalize_strict("ShoulderJoint", entity="joint")
# -> None（候補が複数など）
```

`entity` には `"joint"` / `"link"` / `"loop"` / `"auto"` が使えます。

### 2. 根拠付きで変換する（推奨）

```python
from RobotLabelBridge import convert_with_provenance, NameConverter

result = convert_with_provenance(
    "HipJoint",
    entity="joint",
    parent="c_pelvis",
    child="l_leg_upper",
    axis=[0, 1, 0],
    morphology="humanoid",
)

print(result.status)            # resolved / ambiguous / unresolved
print(result.target)            # 採択されたカノニカル名（あれば）
print(result.reason_codes)      # なぜそう判断したか
print(result.score_components)  # スコア内訳
print(result.candidates)        # 候補一覧
```

親リンク・子リンク・軸ベクトルなど、分かる情報を渡すほど精度が上がります。

### 3. トポロジを参照する

```python
from RobotLabelBridge import parent_link, ancestor_links, children_links

parent_link("l_leg_upper")     # 例: c_pelvis など
ancestor_links("l_foot")       # 根までの祖先
children_links("c_pelvis")     # 直接の子リンク
```

### 4. モデルファイルを変換する

```python
from RobotLabelBridge import convert_model_file

# URDF / MJCF を読み、リネーム案を作る
report = convert_model_file("/path/to/robot.urdf")
```

LegacyMotionEditor 上では、読み込み済みモデルに対して  
`plan_joint_rename` → 確認 → 適用、という流れでも使えます。

### 5. Master を検証する

```python
from RobotLabelBridge import validate_master

raw = validate_master(stage="raw")        # ディスク上のデータの健全性
mig = validate_master(stage="migrated")  # 正規化後の参照整合

print(raw.ok, mig.ok)
print(mig.summary())
```

### 6. コマンドライン

```bash
# Viewer（別名検索 / リンクツリー / 閉リンク / 変換プレビュー）
python RobotLabelBridge.py

# Master 検証（RAW と MIGRATED を両方表示）
python RobotLabelBridge.py --validate-master

# 内蔵セルフテスト
python RobotLabelBridge.py --self-test

# 機械的に直せる不整合を Master へ書き戻し
python RobotLabelBridge.py --migrate-master

# 意味関係の triple をサンプル表示
python RobotLabelBridge.py --dump-triples
```

Viewer だけ PySide6 が必要です。  
変換・検証・registry 利用は GUI なしでも動作します。

---

## Master データの見方

`RobotLabelBridge_Master.json` が辞書の本体です。

| セクション | 中身 |
|---|---|
| `servo_targets` | カノニカル DOF（作動軸）の定義 |
| `link_targets` | カノニカル link の定義 |
| `alias_index` | 別名 → カノニカル名 |
| `link_tree` | リンクの親子ツリー（輪にしない） |
| `kinematic_joints` | 明示的な物理継手（特に passive）。無ければ空 |
| `loop_targets` | 閉リンク定義。無ければ空 |

`load_master()` で dict として読めます。  
型付きで扱いたいときは `get_ontology_registry()` を使います。

```python
from RobotLabelBridge import get_ontology_registry

reg = get_ontology_registry()
reg.dofs_by_label["l_knee_yp"]
reg.links_by_label["l_leg_upper"]
reg.get_loop("l_knee_lp")            # 定義がある場合
reg.get_kinematic_joint("...")       # 定義がある場合
```

---

## 曖昧さの扱い方

| API | 曖昧なとき |
|---|---|
| `canonicalize_strict` | `None` を返す（安全） |
| `canonicalize_best_effort` | 最有力候補を返す（便利） |
| `NameConverter.convert` / `convert_with_provenance` | `ambiguous` ステータスと候補一覧 |

運用のおすすめ:

- 研究・自動パイプラインの厳密経路 → `strict`
- 人手確認つきの補助 → `best_effort` または provenance 付き API
- 「たぶんこれ」を黙って正式採用しない

---

## リンクツリーと閉リンク（必要な人向け）

通常のロボットは **親子の木** で足ります。

```text
c_base_link
 └─ c_pelvis
     └─ l_leg_upper
         └─ l_leg_lower
             └─ l_foot
```

一方、四節リンクや平行リンクのように **輪になる機構** がある場合、  
その輪を木の辺として書き足すと、親子探索や「木のバグ検出」が壊れます。

そのため RobotLabelBridge では:

- **木（`link_tree`）** … いつも使う親子関係。輪は作らない
- **閉リンク（`loop_targets`）** … 「どこで輪になるか」を別定義

として分けています。  
閉リンクを使わないモデルでは、このセクションは空のままで問題ありません。

---

## LegacyMotionEditor からの使い方（概要）

Editor 側からは、だいたい次の流れです。

1. ロボットモデルを読み込む
2. RobotLabelBridge で rename plan を作る
3. 衝突や未解決名を確認する
4. 問題なければモデルへ適用する

詳細な UI 操作は Editor 側のメニューに従います。  
規則・辞書・検証の本体はこのモジュールと Master JSON にあります。

---

## 設計上の約束（これを守ると安全）

1. **カノニカル名は短く、左右・部位・軸が一目で分かること**
2. **別名は alias に置き、正式名を増やしすぎないこと**
3. **曖昧な変換を成功扱いにしないこと**
4. **存在しない解剖構造を推測で Master に足さないこと**
5. **閉リンクを理由に、既存の関節・リンク名を勝手に変えないこと**
6. **木構造に輪を書き込まないこと**

---

## バージョン

| 項目 | 値 |
|---|---|
| Python API | 2.2.0 |
| `schema_version` | 2.2 |
| `ontology_version` | 1.2 |

Master を編集したあとは、同じプロセス内なら `reload_master()` を呼んでキャッシュを更新してください。

---

## まとめ

RobotLabelBridge は、ロボットの名前を共通の短い正式名へ翻訳する辞書＋変換器です。  
裏側では、関節・軸・リンク・別名・親子関係の違いを分けて持つ軽い意味の整理があります。

- **目的**: 機種やフォーマットをまたいでも、同じ意味の関節・リンクを同じ名前で扱えるようにする
- **使い方**: `canonicalize_*` / `convert_with_provenance` / CLI / Editor 連携
- **命名**: `[l|r|c]_部位_軸` を基本に、短く一貫させる
- **内部**: 表記と意味を分けて持つ（OWL は必須ではない）
- **利点**: 別名に強く、曖昧さを隠しにくく、根拠付きで変換できる

まずは `canonicalize_best_effort("あなたの関節名")` を試し、  
本番経路では `convert_with_provenance(...)` か `canonicalize_strict(...)` を使うのがおすすめです。
