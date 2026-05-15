---
marp: true
theme: default
paginate: true
backgroundColor: #ffffff
color: #333
style: |
  section {
    font-family: "Hiragino Kaku Gothic ProN", "Noto Sans JP", sans-serif;
    font-size: 30px;
    padding: 40px 60px;
  }
  section.title {
    text-align: center;
    display: flex;
    flex-direction: column;
    justify-content: center;
  }
  section.title h1 {
    font-size: 2.0em;
    color: #ff7a00;
    line-height: 1.4;
  }
  section.title p {
    color: #666;
    font-size: 0.9em;
  }
  h1 {
    color: #ff7a00;
    border-bottom: 2px solid #ff7a0033;
    padding-bottom: 10px;
  }
  h2 {
    color: #2a9d8f;
  }
  code {
    background: #fff4e6;
    color: #cc5200;
    padding: 2px 8px;
    border-radius: 4px;
  }
  pre {
    background: #fff8f0 !important;
    border-radius: 8px;
    padding: 18px !important;
    font-size: 0.78em;
  }
  pre code {
    background: transparent;
    color: #333;
    padding: 0;
  }
  strong {
    color: #e63946;
  }
  table {
    font-size: 0.82em;
  }
  th {
    background: #ff7a00;
    color: #fff;
  }
  td, th {
    border-color: #ddd !important;
  }
  blockquote {
    border-left: 4px solid #ff7a00;
    background: #fff4e6;
    padding: 10px 20px;
    border-radius: 0 8px 8px 0;
    color: #444;
  }
  a {
    color: #ff7a00;
  }
  img[alt~="center"] {
    display: block;
    margin: 0 auto;
  }
  .columns {
    display: flex;
    gap: 40px;
  }
  .columns > div {
    flex: 1;
  }
  ul {
    line-height: 1.7;
  }
  section.end {
    text-align: center;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }
---

<!-- _class: title -->

# AlarmKitで<br>明後日起きれる<br>アラームアプリを作る

trickart

---

# 自己紹介

- **trickart** / POSレジアプリ開発
- どうしても明後日起きたくて作った<br>アプリの話です

![bg fit right:38%](bo.png)

---

# こんな経験ありませんか？

- 出張・旅行で **明後日の朝早く起きたい**
- しかしiOS標準アラームで設定しようとすると…
  - 基本24時間先までしか設定できない
    - 明後日7:00に起きたいのに7:00と設定すると明日7:00になってしまう
  - 「明日設定しよう」→忘れる

> **明後日を設定できるアラームアプリが欲しい**

---

# あさってアラーム<br>DayAfterTomorrow

## 明後日のアラームが設定できるアプリ

- 開くと2日後の日付がもう選ばれている
- iOS 26+ / AlarmKit 製

![bg fit right:38%](app_screen.png)

---

# AlarmKitとは？

WWDC2025（Session 230）で発表されたフレームワーク（iOS 26+）

- サードパーティアプリから**OS標準アラームと同等**のアラームを鳴らせる
  - サイレント・集中モード**貫通**
  - Apple Watch鳴動

---

# 明後日を指定する - `.fixed` モードによる日時指定

AlarmKitの `Alarm.Schedule` には3つのスケジュール方式がある

- `.relative(repeats: .never)` … 24時間以内・単発
- `.relative(repeats: .weekly([.monday]))` … 毎週月曜
- **`.fixed(Date)` … 特定の日の特定時刻** ← あさってアラームはこれ

```swift
// 明後日の日付 + 7:00 を組み立てる
var components = Calendar.current.dateComponents(
    [.year, .month, .day], from: dayAfterTomorrow
)
components.hour = 7
components.minute = 0

// Alarm.Schedule生成
let schedule: Alarm.Schedule = .fixed(Calendar.current.date(from: components)!)
```

---

# Alarm Configurationにscheduleを渡して<br>スケジュールする

```swift
let configuration = AlarmManager.AlarmConfiguration(
    countdownDuration: .init(preAlert: nil, postAlert: 5 * 60), // Snoozeの長さ
    schedule: schedule,                                         // ← さっきの .fixed
    attributes: AlarmAttributes(presentation: .init(alert: alertContent),
                                tintColor: .orange),
    stopIntent: StopAlarmIntent(alarmID: id.uuidString),       // LiveActivityIntent
    secondaryIntent: SnoozeAlarmIntent(alarmID: id.uuidString) // LiveActivityIntent
)

try await AlarmManager.shared.schedule(id: id, configuration: configuration)
```

- `attributes` タイトルや色を設定できる

---

# これでよいかと思いきや…落とし穴

## タイムゾーン問題

`.fixed(Date)` は **UTC絶対時刻** として保存される

- 東京（UTC+9）で「4/12 7:00」設定 → UTC `4/11 22:00`
- ニューヨーク（UTC-4）に移動すると…
  → **現地 4/11 18:00** に鳴る ❌

APIの仕様としては自然だが、自分のユースケースとしてはちょっと困る

→ タイムゾーン変更を検知して **再スケジュール** が必要

---

# 3層補正アーキテクチャ

アプリ起動時・アプリが閉じられているとき・アプリ起動中の3つでカバーする

| 対策 | 手段 | メリット・デメリット |
|:-:|---|---|
| **1** | 起動時に `lastKnownTimeZone` と比較 | 権限不要・確実<br>開かないと直らない |
| **2** | `BGAppRefreshTask` で定期チェック | 起動しなくても動く<br>実行保証なし |
| **3** | `NSSystemTimeZoneDidChange` を購読 | 即時反映<br>起動中限定 |

---

# まとめ

## AlarmKitにはiOS標準のアラーム同等の機能がある

- サイレント貫通・Live Activity・Dynamic Island
- 24時間先まで・weeklyで鳴らせる

## 使いこなせばiOS標準のアラーム"以上"の事ができる

- **24時間以上先**：`Alarm.Suchedule.fixed(Date)`で明後日でも確実に起きられる
- **3層TZ補正**：起動時 / BGAppRefresh / システム通知 を重ねる

---

<!-- _class: end -->

# Thank you!

![](qr.png)

ref:
[AlarmKit – WWDC2025 Session 230](https://developer.apple.com/jp/videos/play/wwdc2025/230/)
[AlarmKit | Apple Developer Documentation](https://developer.apple.com/documentation/alarmkit)
