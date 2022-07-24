# admission webhook 花式玩法 - kinitiras 发送 http(s) 请求


## 本篇由来

在使用 Admission Webhook 的时候，很可能会涉及到发送 http 请求以获取某些数据。在 v0.1.1 版本中对此进行了支持，本文主要来介绍如何在 [kinitiras](https://github.com/k-cloud-labs/kinitiras) 中发送 http 请求，如果对 kinitiras 还不熟悉，请参考[前篇](../kinitiras)。

## 功能介绍

当前仅支持在使用 CUE 时可以发送 http(s) 请求，相关结构定义在 `processing` 下，其中 `output` 用来定义输出结果，需要与 http response 的结构一致，按需定义即可。`http` 部分和 CUE `http` 结构一致。

示例：

```cue
object: _ @tag(object)

processing: {
  http: {
    method: *"GET" | string
    url: "http://127.0.0.1:8090/api/v1/token?val=test-token"
    request: {
      body ?: bytes
      header: {
        "Accept-Language": "en,nl"
      }
      trailer: {
        "Accept-Language": "en,nl"
        User: "foo"
      }
    }
  }
  output: {
    token?: string
  }
}

validate:{
	reason: "hello cue"
	valid: object.metadata.name == "ut-cue-success-with-parameter" && processing.output.token == "test-token"
}
```

上面这段 CUE 配置的含义是传入一个 k8s 资源对象，访问 http://127.0.0.1:8090/api/v1/token?val=test-token 获取 token 信息，最终返回 validate 结果，如果传入的 k8s 对象的名字是 ut-cue-success-with-parameter 并且 http 返回结果中的 token 是 test-token 的话，校验成功。


