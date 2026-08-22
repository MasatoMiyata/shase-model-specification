# 水-水熱交換器 計算仕様書（エナジープラス プラントヒートエクスチェンジャー）

## 0.  文書基本情報

| 項目 | 内容 |
|---|---|
| 対象コード | `PlantHeatExchanger.f90` |
| 対象モジュール | `PlantHeatExchangerFluidToFluid` |
| 対象オブジェクト | `HeatExchanger:FluidToFluid` |
| モデル名称 | 汎用プラント流体-流体熱交換器 |
| 目的 | 2つのプラントループを熱的に結合し、流れ形式、UA、流量、流体物性および制御方式から熱量・出口温度・運転状態を算定する。 |
| 計算状態 | 定常状態。熱交換器壁の蓄熱、圧力損失、外部熱損失および相変化は扱わない。 |

## 1.  基本情報

### 1.1 適用範囲・前提

- 供給側ループ（Supply Side）と需要側ループ（Demand Side）の単相液体どうしの顕熱交換を扱う。
- 流体物性は各ループの流体名および入口温度から `GetSpecificHeatGlycol()` により取得する。従って純水だけでなく、エナジープラスで定義されたグリコール水溶液にも適用できる。
- 熱計算は有効度-NTU法で行う。対向流・並流・4種類の直交流・理想熱交換器を選択できる。
- 制御は可用性スケジュール、負荷、設定温度、デッドバンド、温度差又は関連コンポーネントの状態により、両側流量をON/OFF又は変調する。

### 1.2 処理の流れ

1. `GetFluidHeatExchangerInput` で入力を読み込み、`InitFluidHeatExchanger` でノード・流量上限下限を初期化する。
2. 必要時は `SizeFluidHeatExchanger` で設計流量・UA・容量をサイジングする。
3. `ControlFluidHeatExchanger` で可用性、制御方式、負荷又は設定値から流量を設定する。
4. `CalcFluidHeatExchanger` で有効度、熱移動量、両側出口温度を算定する。
5. `UpdateFluidHeatExchanger` で出口ノード温度を更新し、`ReportFluidHeatExchanger` で熱量、エネルギー、運転状態を報告する。

### 1.3 単位・記号

| 記号 | 対応変数 | 単位 | 定義 |
|---|---|---|---|
| `T_s`, `T_d` | `SupSideLoopInletTemp`, `DmdSideLoopInletTemp` | ℃ | 供給側・需要側入口温度 |
| `m_s`, `m_d` | `SupSideMdot`, `DmdSideMdot` | kg/s | 供給側・需要側質量流量 |
| `cp_s`, `cp_d` | `SupSideLoopInletCp`, `DmdSideLoopInletCp` | J/(kg K) | 各側入口温度における定圧比熱 |
| `C_s`, `C_d` | `SupSideCapRate`, `DmdSideCapRate` | W/K | 熱容量流量（m × cp） |
| `C_min`, `C_max` | `MinCapRate`, `MaxCapRate` | W/K | 最小・最大熱容量流量 |
| `Cr` | `CapRatio` | \- | 熱容量流量比（C_min/C_max） |
| `NTU` | `NTU` | \- | 移動単位数（UA/C_min） |
| `ε` | `Effectiveness` | \- | 熱交換有効度 |
| `Q` | `HeatTransferRate` | W | 熱移動量。正値は供給側から需要側への移動を示す。 |
| `E` | `HeatTransferEnergy` | J | 計算ステップの熱移動エネルギー |

## 2.  入出力インターフェイス

### 2.1 状態・制御入力

| 分類 | 入力 | 単位 | 定義 |
|---|---|---|---|
| ノード | 需要側・供給側の入口／出口ノード | ℃, kg/s | 各側の温度、質量流量、設定温度を保持する。 |
| 設計値 | 需要側・供給側設計体積流量 | m3/s | 初期化・サイジング時に質量流量上限下限へ変換する。 |
| 性能 | `UA` | W/K | 総括伝熱性能。`Ideal`以外で必須。 |
| 流れ形式 | `HeatExchangeModelType` | \- | 直交流4形式、対向流、並流、理想を選択する。 |
| 可用性 | `AvailSchedNum` | \- | スケジュール値が正のときのみ運転可能。 |
| 制御 | `ControlMode` | \- | 無制御、負荷追従、設定値、デッドバンド等の方式。 |
| 設定値 | `SetpointNodeNum`, `TempControlTol` | ℃ | 温度制御の目標値と許容差。 |
| 運転制限 | `MinOperationTemp`, `MaxOperationTemp` | ℃ | 制御信号温度の運転許容範囲。 |

### 2.2 出力

| 出力 | 単位 | 定義 |
|---|---|---|
| `SupplySideLoop.OutletTemp` | ℃ | 供給側出口温度 |
| `DemandSideLoop.OutletTemp` | ℃ | 需要側出口温度 |
| `HeatTransferRate` | W | 熱移動量 |
| `HeatTransferEnergy` | J | `HeatTransferRate × TimeStepSys × SecInHour` |
| `Effectiveness` | \- | 有効度（最大1） |
| `OperationStatus` | \- | `abs(Q)>SmallLoad` かつ両側流量が正なら1、それ以外0 |
| `MinLoad`, `MaxLoad`, `OptLoad` | W | 運転スキーム用の供給可能容量。`OptLoad=0.9×MaxLoad`。 |

### 2.3 主なパラメータ

| 分類 | 値 | 定義 |
|---|---|---|
| 流れ形式 | `CrossFlowBothUnMixed` | 直交流・両側非混合 |
|  | `CrossFlowBothMixed` | 直交流・両側混合 |
|  | `CrossFlowSupplyMixedDemandUnMixed` | 直交流・供給側混合／需要側非混合 |
|  | `CrossFlowSupplyUnMixedDemandMixed` | 直交流・供給側非混合／需要側混合 |
|  | `CounterFlow`, `ParallelFlow`, `Ideal` | 対向流、並流、理想 |
| 変調ソルバ | `MaxIte=500`, `Acc=1E-3` | ルーラファルシ法の反復上限、温度残差精度 |

## 3.  計算ロジック

### 3.1 停止条件・流量設定

以下のいずれかの場合、`ControlFluidHeatExchanger` は供給側・需要側の流量を0に設定する。

- 可用性スケジュール値が0以下。
- 制御信号温度が `MinOperationTemp` 未満又は `MaxOperationTemp` 超。
- 選択したON/OFF制御で、負荷、設定温度又は温度差が運転条件を満たさない。

流量変調の設定値制御では、供給側流量を与えた上で、需要側流量を最小～最大流量の範囲で解く。目標温度が最小・最大流量時の出口温度で挟まれない場合は、該当する端値流量を用いる。

### 3.2 熱容量流量・無次元係数

``` text
C_s = m_s × cp_s
 C_d = m_d × cp_d
 C_min = min(C_s, C_d)
 C_max = max(C_s, C_d)
 Cr = C_min / C_max
 NTU = UA / C_min
```

いずれかの熱容量流量が0の場合は `ε=0` とする。指数関数を含む相関では、指数の上限・下限を確認してオーバーフローを回避し、`ε≤1` に制限する。

### 3.3 有効度相関

#### 対向流

``` text
ε = [1 - exp{-NTU(1-Cr)}] / [1 - Cr exp{-NTU(1-Cr)}]
```

分母が数値的に1となる場合は `ε = 1-exp(-NTU)` を用い、極端なNTUでは `ε=1` とする。

#### 並流

``` text
ε = [1 - exp{-NTU(1+Cr)}] / (1+Cr)
```

#### 直交流・両側非混合

``` text
ε = 1 - exp{(NTU^0.22/Cr)[exp(-Cr×NTU^0.78)-1]}
```

#### 直交流・両側混合

``` text
ε = 1 / {1/[1-exp(-NTU)] + Cr/[1-exp(-Cr×NTU)] - 1/NTU}
```

#### 直交流・片側混合

混合側が最小熱容量流量か最大熱容量流量かを判定し、次の相関を選択する。

- 混合側が最小熱容量流量：

  ``` text
  ε = [1-exp{Cr×exp(-NTU)-1}]/Cr
  ```

  `Cr=0` の極限値は 0.632。
- 混合側が最大熱容量流量：

  ``` text
  ε = 1-exp{-(1/Cr)[1-exp(-Cr×NTU)]}
  ```

需要側・供給側のどちらが混合側かは、選択した流れ形式により対応付ける。

#### 理想熱交換器

``` text
ε = 1
```

### 3.4 熱量・出口温度

``` text
Q = ε × C_min × (T_s,in - T_d,in)
 T_s,out = T_s,in - Q/C_s
 T_d,out = T_d,in + Q/C_d
```

片側流量が0の場合は、その側の出口温度を入口温度とする。`Q>0` は供給側が冷却され、需要側が加熱されることを示す。

### 3.5 設定値変調

`FindHXDemandSideLoopFlow` は需要側最小・最大流量で `T_s,out` を試算する。目標温度が両端値の間にある場合、以下の残差を0とする需要側流量をルーラファルシ法で求める。

``` text
r(m_d) = T_s,out,set - T_s,out(m_d)
```

反復上限超過時は警告を出し、その時点の計算流量で継続する。根が挟まれない場合は、端値計算結果から線形補間した推定流量を使用する。

### 3.6 制御方式

| 制御方式 | 概要 |
|---|---|
| `UncontrolledOn` | 両側を最大流量で運転する。 |
| `OperationSchemeModulated` | 運転スキームの負荷から、需要側流量を必要量へ変調する。 |
| `OperationSchemeOnOff` | 負荷方向・ループ種別により最大流量運転又は停止する。 |
| `Heating/CoolingSetpointModulated` | 供給側出口設定温度に対して需要側流量を変調する。 |
| `Heating/CoolingSetpointOnOff` | 設定温度と許容差で最大流量運転／停止を判定する。 |
| `DualDeadbandSetpointModulated/OnOff` | 加熱・冷却双方の設定値により変調又はON/OFFを判定する。 |
| `CoolingDifferentialOnOff` | 両側の入口温度差によって冷却運転を判定する。 |
| `CoolingSetpointOnOffWithComponentOverride` | 外気湿球・乾球又はループ温度により関連コンポーネントを上書きする。 |
| `TrackComponentOnOff` | 関連コンポーネントのON/OFF状態に追従する。 |

## 4.  精度検証

## 5.  パラメータ同定

## 6.  Cx・コミッショニングにおける使用場面

## 制約・適用上の留意事項

- 本仕様は `PlantHeatExchanger.f90` の静的解析に基づく。サイジング、ノード、スケジュール、EMS、流体物性および関連コンポーネントの詳細はエナジープラス上位モジュールに依存する。
- 本モジュールは熱計算および流量制御を対象とし、圧力損失・バルブCv・配管熱損失・過渡応答を計算しない。
- `HeatTransferRate` の符号は供給側から需要側へ移動する熱を正とする。計測・外部モデルとの照合では符号規約を統一する。

