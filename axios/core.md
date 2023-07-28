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

1. 根据config.headers创建[AxiosHeaders](#AxiosHeadersjs)
2. 使用[transformData](#transformDatajs)转换请求数据
3. 给 'POST'、'PUT' 或 'PATCH'请求，设置内容类型 'application/x-www-form-urlencoded'
4. 根据配置获取相应的 [adapter](./adapters.md/#adaptersjs)， 浏览器环境中使用 XHR adapter，而node.js 中使用 http adapter
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

```js

// 将字符串去头尾空格并小写化
function normalizeHeader(header)

// 将value化为字符串，除了数组、bool和null
function normalizeValue(value)

// 解析字符串为键值对，形如这种 str = "key1=value1; key2=value2; key3"
function parseTokens(str)

// 检测是否为合法的header
const isValidHeaderName = (str) => /^[-_a-zA-Z0-9^`|~,!#$%&'*+.]+$/.test(str.trim());

// 传入的参数 value 匹配 header 中的值
function matchHeaderValue(context, value, header, filter, isHeaderNameFilter)

// 格式化header名, 例如 "Content-Type"
function formatHeader(header)

// 给指定对象 obj 添加一些特殊的访问器 get+Header、set+Header 和 has+Header
function buildAccessors(obj, header)

class AxiosHeaders {
  constructor(headers) {
    headers && this.set(headers);
  }

  // 设置 header, 直接设置到this上
  set(header, valueOrRewrite, rewrite)

  // 获取对应header的值
  get(header, parser)

  // 判断是否存在对应的header
  has(header, matcher)

  // 删除对应的header
  delete(header, matcher)

  // 清除header
  clear(matcher)

  // 格式化header
  normalize(format)

  // 调用类的静态方法static concat
  concat(...targets) {
    return this.constructor.concat(this, ...targets);
  }

  // 将类的实例转化为一个 JSON 对象
  toJSON(asStrings)

  // 实现迭代器，让实例可以使用for of、解构赋值和展开运算等
  [Symbol.iterator]() {
    return Object.entries(this.toJSON())[Symbol.iterator]();
  }

  // 将实例转化为字符串
  toString() {
    return Object.entries(this.toJSON()).map(([header, value]) => header + ': ' + value).join('\n');
  }

  // Symbol.toStringTag 是一个内置的 Symbol，可以通过该 Symbol 来定制 [object AxiosHeaders] 默认标签
  get [Symbol.toStringTag]() {
    return 'AxiosHeaders';
  }

  // 创建实例
  static from(thing) {
    return thing instanceof this ? thing : new this(thing);
  }

  // 将属性拼接到实例上，并返回新的实例
  static concat(first, ...targets) {
    const computed = new this(first);
    targets.forEach((target) => computed.set(target));
    return computed;
  }

  // 给header设置访问器 使用上面的buildAccessors函数
  static accessor(header)
}
```

## transformData.js

主要是格式化响应数据，可以定制格式化函数transformResponse => fns

```js
export default function transformData(fns, response) {
  const config = this || defaults;
  const context = response || config;
  const headers = AxiosHeaders.from(context.headers);
  let data = context.data;

  utils.forEach(fns, function transform(fn) {
    data = fn.call(config, data, headers.normalize(), response ? response.status : undefined);
  });

  headers.normalize();

  return data;
}
```
