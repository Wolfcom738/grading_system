# 评分算法说明

## 1. 匹配逻辑

### 1.1 时间匹配条件
- 在时间窗口（timeToleranceMs）内搜索真值点
- 默认时间窗口：2000ms（可自定义）
- 秒级数据的时间戳默认为该秒的000ms处

### 1.2 空间匹配条件
- 在时间窗口内，只有距离小于匹配空间窗口（distanceToleranceM）的真值点才被视为候选
- 默认空间窗口：150米（可自定义）
- 从候选真值点中选择距离最近的点进行匹配

### 1.3 时间偏移补偿
- 自动检测时间偏移：比较轨迹时间范围与真值时间范围
  - 如果时间范围直接重叠 → 使用0偏移
  - 如果轨迹-8h后与真值重叠 → 使用-8小时偏移
  - 如果轨迹+8h后与真值重叠 → 使用+8小时偏移
- 支持手动设置时间偏移补偿

## 2. 评分指标（共9项，每项10分，总分90分）

### 2.1 3D定位误差
公式：`position_error_m = sqrt(d_h^2 + d_z^2)`
- d_h：水平距离（米）
- d_z：高度差（米）

评分细则：
- ≤10m：10分
- ≤20m：8分
- ≤50m：6分
- ≤100m：4分
- ≤200m：2分
- >200m：0分

### 2.2 虚警率
公式：`false_alarm_rate = false_alarm_count / sensing_point_count`

虚警判定：在给定timeToleranceMs内，若某个感知4D点位在任一真值4D点位周围groundTruthNeighborhoodM范围内都找不到邻点，则记为虚警点。

默认真值邻域（groundTruthNeighborhoodM）：50米（可自定义）

评分细则：
- ≤5%：10分
- ≤10%：8分
- ≤20%：6分
- ≤30%：4分
- ≤50%：2分
- >50%：0分

### 2.3 漏检率
公式：`miss_rate = miss_count / ground_truth_point_count`

漏检判定：在给定timeToleranceMs内，若某个真值4D点位在任一感知4D点位周围groundTruthNeighborhoodM范围内都找不到邻点，则记为漏检点。

评分细则：
- ≤5%：10分
- ≤10%：8分
- ≤20%：6分
- ≤30%：4分
- ≤50%：2分
- >50%：0分

### 2.4 时间误差
（缺少数据，暂时留空，记0分）

### 2.5 速度误差
公式：`speed_error_mps = abs(sensing.GS - truth.GS)`
- GS：地速（m/s）

评分细则：
- ≤5 m/s：10分
- ≤10 m/s：8分
- ≤20 m/s：6分
- ≤30 m/s：4分
- ≤50 m/s：2分
- >50 m/s：0分

### 2.6 采样频率
公式：`observed_sample_rate_hz = 1000 / median_dt_ms`
- median_dt_ms：相邻采样时刻时间差的中位数（毫秒）

评分细则（基于采样频率稳定性，变异系数CV）：
- CV ≤0.1：10分
- CV ≤0.2：8分
- CV ≤0.3：6分
- CV ≤0.5：4分
- CV >0.5：2分

### 2.7 丢包判定
公式：`dt_i > expected_period_ms * packetLossGapFactor`
- expected_period_ms：期望周期（ms），由sampleRateHz计算
- packetLossGapFactor：基线取值为1.5

评分细则（基于丢包率）：
- ≤5%：10分
- ≤10%：8分
- ≤20%：6分
- ≤30%：4分
- ≤50%：2分
- >50%：0分

### 2.8 跳变判定
公式：`apparent_speed_mps = distance_3d_m / delta_t_s`
- distance_3d_m：3D距离（米）
- delta_t_s：时间差（秒）

跳变判定阈值：jumpSpeedThresholdMps = 41.7（基线值）

评分细则（基于跳变率）：
- ≤1%：10分
- ≤5%：8分
- ≤10%：6分
- ≤20%：4分
- >20%：2分

### 2.9 轨迹平滑度
公式：`smoothness_score_01 = 1 / (1 + rms_horizontal_acc_mps2)`
- rms_horizontal_acc_mps2：水平加速度均方根

评分细则：
- ≥0.9：10分
- ≥0.7：8分
- ≥0.5：6分
- ≥0.3：4分
- <0.3：2分

## 3. 参数说明

| 参数名 | 默认值 | 单位 | 说明 |
|--------|--------|------|------|
| timeToleranceMs | 2000 | 毫秒 | 匹配时间窗口 |
| distanceToleranceM | 150 | 米 | 匹配空间窗口 |
| groundTruthNeighborhoodM | 50 | 米 | 真值邻域（用于虚警/漏检判定） |
| timeOffset | 0 | 毫秒 | 时间偏移补偿 |
| packetLossGapFactor | 1.5 | - | 丢包判定系数 |
| jumpSpeedThresholdMps | 41.7 | m/s | 跳变判定阈值 |

## 4. 数据字段要求

### JSON感知数据
- `timestamp`：时间戳（毫秒）
- `latitude`：纬度
- `longitude`：经度
- `height`：高度（米）
- `GS` / `groundSpeed` / `speed`：地速（m/s）
- `trajectoryInfo.sampleRateHz`：声明的采样频率

### CSV真值数据
- `timestamp`：时间戳（毫秒）
- `latitude`：纬度
- `longitude`：经度
- `altitude`：高度（米）
- `GS` / `groundSpeed` / `speed`：地速（m/s）

## 5. 评分流程

1. 读取JSON感知数据和CSV真值数据
2. 用户选择需要评分的轨迹
3. 自动检测时间偏移（或使用手动设置）
4. 对每个感知点进行匹配：
   - 在时间窗口内搜索真值点
   - 在空间窗口内筛选候选点
   - 选择距离最近的真值点进行匹配
5. 匹配后验证：检查时间差是否超过1秒，超过则丢弃
6. 计算9项评分指标
7. 输出评分结果和警告信息
