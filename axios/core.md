## Axios.js

主要有Axios类
其中interceptors使用[InterceptorManager](#InterceptorManager.js)来实例化的
发送真实请求使用[dispatchRequest](#dispatchRequest.js)

```js
class Axios {
  /**
   * 初始化两个属性 defaults 与 interceptors
   */
  construct (instanceConfig)

  /** 主要流程：
   * 1. 处理参数，允许将第一个参数合并只写config
   * 2. 合并配置，将defaults和config合并
   * 3. 平铺headers，根据config.method paramsSerializer headers[config.method]
   * 4. 构建拦截器链 分别添加到 requestInterceptorChain 和 responseInterceptorChain 链中
   * 5. 异步请求-同步请求-响应流程，主要是用dispatchRequest发送真正的请求，前后分别为 requestInterceptorChain 和 responseInterceptorChain
   * 6. 返回Promise
   */
  request (configOrUrl, config)

  getUri (config)
}
```

## InterceptorManager.js

```js
class InterceptorManager {

  // 初始化一个handlers的空数组，存储所有拦截器
  constructor()

  /**
   * 往handlers中添加拦截器，拦截器主要结构为:
   * {
   *  fulfilled,
   *  rejected,
   *  synchronous: boolean,
   *  runWhen
   * }
   */
  use(fulfilled, rejected, options)

  // 从handlers中移除一个拦截器，id主要是拦截器的序列号
  eject(id)

  // 清空handlers中的所有拦截器
  clear()

  // 遍历拦截器
  forEach(fn)
}

```

## dispatchRequest.js

1. 根据config.headers创建[AxiosHeaders](#AxiosHeaders.js)
2. 使用[transformData](#transformData.js)转换请求数据
3. 给 'POST'、'PUT' 或 'PATCH'请求，设置内容类型 'application/x-www-form-urlencoded'
4. 根据配置获取相应的 [adapter](./adapters.md/#adapters.js)， 浏览器环境中使用 XHR adapter，而node.js 中使用 http adapter
5. 使用adapter发送请求，并返回
6. 请求成功，对返回的响应数据进行转换，并处理响应的 headers，最后返回处理后的响应对象
7. 请求失败，同理

```js
export default function dispatchRequest(config) {
  config.headers = AxiosHeaders.from(config.headers);

  config.data = transformData.call(
    config,
    config.transformRequest
  );

  if (['post', 'put', 'patch'].indexOf(config.method) !== -1) {
    config.headers.setContentType('application/x-www-form-urlencoded', false);
  }

  const adapter = adapters.getAdapter(config.adapter || defaults.adapter);

  return adapter(config).then(function onAdapterResolution(response) {
    response.data = transformData.call(
      config,
      config.transformResponse,
      response
    );

    response.headers = AxiosHeaders.from(response.headers);

    return response;
  }, function onAdapterRejection(reason) {
    if (!isCancel(reason)) {
      if (reason && reason.response) {
        reason.response.data = transformData.call(
          config,
          config.transformResponse,
          reason.response
        );
        reason.response.headers = AxiosHeaders.from(reason.response.headers);
      }
    }

    return Promise.reject(reason);
  });
}

```

## AxiosHeaders.js
