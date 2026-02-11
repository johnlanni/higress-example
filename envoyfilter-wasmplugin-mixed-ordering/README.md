# EnvoyFilter 与 WasmPlugin 混合排序功能指南

## 背景

在 Higress/Istio 中，WasmPlugin 和 EnvoyFilter 都可以用于向 HTTP filter chain 中插入自定义过滤器。然而，在此功能之前，它们是分开处理的：

- **WasmPlugin** 按照 `phase` 和 `priority` 排序
- **EnvoyFilter** 使用 `INSERT_BEFORE`/`INSERT_AFTER` 等操作定位

这导致无法精确控制 EnvoyFilter 添加的 filter 与 WasmPlugin 之间的相对位置。

## 新功能：混合排序

通过在 EnvoyFilter 中引入 `wasmPhase` 和 `wasmPriority` 字段，现在可以让 EnvoyFilter 与 WasmPlugin 参与统一的排序。

### 新增字段

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: my-custom-filter
spec:
  wasmPhase: AUTHN    # 新增：指定参与混合排序的 phase
  wasmPriority: 50    # 新增：在同 phase 内的优先级（数值越大越靠前）
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_OUTBOUND
        listener:
          portNumber: 8080
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: ADD
        value:
          name: my-custom-filter
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
            # ... filter config
```

### Phase 类型

| Phase | 顺序 | 说明 |
|-------|------|------|
| `AUTHN` | 1 | 认证阶段 |
| `AUTHZ` | 2 | 授权阶段 |
| `STATS` | 3 | 统计阶段 |
| `UNSPECIFIED_PHASE` | 4 | 未指定（不参与混合排序） |

### 排序规则

在同一个 phase 内，按以下优先级排序：

1. **priority 值**：数值越大越靠前
2. **创建时间**：priority 相同时，较早创建的在前
3. **名称**：创建时间也相同时，按名称字典序

## 使用示例

### 场景：在认证插件之后、授权插件之前插入自定义 filter

参考 [example.yaml](./example.yaml)

```yaml
# WasmPlugin - JWT 认证 (AUTHN phase, priority 100)
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: jwt-auth
  namespace: istio-system
spec:
  phase: AUTHN
  priority: 100
  url: oci://example.com/jwt-auth:v1
---
# WasmPlugin - RBAC 授权 (AUTHZ phase, priority 100)
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: rbac
  namespace: istio-system
spec:
  phase: AUTHZ
  priority: 100
  url: oci://example.com/rbac:v1
---
# EnvoyFilter - 自定义 filter (AUTHN phase, priority 50)
# 会排在 jwt-auth 之后（因为 priority 50 < 100）
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: custom-authn-filter
  namespace: istio-system
spec:
  wasmPhase: AUTHN
  wasmPriority: 50
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_OUTBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: ADD
        value:
          name: custom-authn-filter
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
            inline_code: |
              function envoy_on_request(handle)
                handle:logInfo("Custom AUTHN filter executed")
              end
```

**最终 filter 顺序：**

```
1. jwt-auth (WasmPlugin, AUTHN, priority=100)
2. custom-authn-filter (EnvoyFilter, AUTHN, priority=50)
3. [内置 authn filters]
4. rbac (WasmPlugin, AUTHZ, priority=100)
5. [内置 authz filters]
...
```

### 场景：多个 EnvoyFilter 与 WasmPlugin 混合

```yaml
# WasmPlugin A - priority 100
# EnvoyFilter B - priority 80
# EnvoyFilter C - priority 60

# 最终顺序: wasm-a → filter-b → filter-c
```

## 注意事项

1. **仅对 ADD 操作生效**：`wasmPhase` 和 `wasmPriority` 仅对 `operation: ADD` 的 HTTP_FILTER patch 生效

2. **不指定 wasmPhase 时**：EnvoyFilter 使用传统的 `INSERT_BEFORE`/`INSERT_AFTER` 定位方式，不参与混合排序

3. **Gateway 和 Sidecar 都支持**：此功能同时支持 Gateway 和 Sidecar 场景

4. **Waypoint 支持**：Ambient mesh 的 Waypoint 也支持此功能

## 兼容性

- 需要 Higress 使用包含此功能的 istio 版本
- 向后兼容：不使用新字段的 EnvoyFilter 行为不变

## 相关链接

- [Feature PR: higress-group/istio#48](https://github.com/higress-group/istio/pull/48)
- [API PR: higress-group/api#5](https://github.com/higress-group/api/pull/5)
