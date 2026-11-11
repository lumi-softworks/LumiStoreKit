# LumiStoreKit

📖 [中文版](README_zh.md) | [English](README.md)

Coffic Lab 应用的 StoreKit 2 生命周期工具集。

`LumiStoreKit` 是应用内购买的**业务无关工具层**。它遵循 Coffic Lab 其余包
相同的分层纪律：

| 层 | 包 | 知晓范围 |
| --- | --- | --- |
| 契约 | `ProviderStore`（位于 [LumiProviders](https://github.com/lumi-softworks/LumiProviders)） | 订阅层级、权益快照、能力协议 |
| 工具 | `KitStoreKit`（本包） | 仅 StoreKit 机制 |
| 功能 | 各应用的 `PluginStore` | 产品目录、购买界面、设置页面 |

## 本包不知道什么

- **订阅层级。** Pro / Ultimate 是各应用不同的产品语义，因此属于
  `ProviderStore` 契约，而不是这里。
- **任何应用的产品 ID。** 产品 ID 列表通过 `StoreKitConfiguration`
  按应用注入。

这保持了依赖方向 `Provider → Kit`。本包不依赖任何 Provider。

## 功能特性

- **交易监听器** — `Transaction.updates` 循环，幂等启动与显式停止。
- **权益校准** — 读取 `Transaction.currentEntitlements`，跳过未验证交易。
- **订阅聚合** — `StoreKitSubscriptionAggregator` 从交易摘要推导当前活跃的
  订阅持有（哪些产品 ID 活跃、最晚何时过期）。它是纯函数，因此有完整的
  单元测试覆盖。
- **产品获取** — 对配置的产品 ID 列表做带缓存的请求。
- **购买与恢复** — 已验证的购买流程与基于 `AppStore.sync()` 的恢复。
- **交易验证** — 未验证结果一律拒绝。
- **权益持久化** — 可插拔存储（默认 `UserDefaults`）。

## 用法

应用提供自己的产品列表并接收权益变化：

```swift
import KitStoreKit

let configuration = StoreKitConfiguration(
    productIDs: [
        "com.example.pro.monthly",
        "com.example.pro.yearly",
    ],
    entitlementStorageKey: "store.purchase",
    lastCheckedAtStorageKey: "store.lastCheckedAt"
)

let service = StoreKitService(configuration: configuration) { holding in
    // Called on the main thread whenever entitlements change.
    print(holding.activeSubscriptionProductIDs, holding.latestExpiration as Any)
}

await service.bootstrap()
```

随后应用用 `ProviderStore` 契约把活跃产品 ID 映射到自己的层级：

```swift
import ProviderStore

struct MyStoreCatalog: StoreProductCatalog {
    let productTiers: [String: StoreTier] = [
        "com.example.pro.monthly": .pro,
        "com.example.pro.yearly": .pro,
    ]
}

let tier = holding.activeSubscriptionProductIDs
    .map { catalog.tier(for: $0) }
    .max() ?? .none
```

## Requirements

- Swift tools 6.0+
- macOS 14+
- iOS 17

## Tests

```sh
swift test
```
