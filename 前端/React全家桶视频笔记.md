{% raw %}

# React 全家桶视频笔记

## UmiJS

package.json 里以@开头的，都是 umijs 的插件

点官网插件找怎么用

### 常用配置

```js
//定义主题颜色
threme:{
    `@primary-color`:   `#1DA57A`
}
hash:{ } //给build出的结果，每个文件加个后缀
```

### 路由

WebStorm 快捷键 - RCC/RSC 生成 class/函数 组件

@符号表示从项目的 src 路径寻找

routes 配置，可以用于配置 layouts

wrapper 用来加入权限路由

title 给页面加标题

路由参数添加 `:id`

```jsx
routes:[
    path: '/user',
    component: '@/layouts/index',
    wrapper: '@/wrappers/auth'
    routes:[
	    {path: 'aaa', component: 'bbb'},
    ]
]
```

不写 route 的话，就会走约定式路由，识别 page/layouts 下得文件

### 页面跳转

声明式/命令式

```jsx
import {Link} from 'umi';

<Link to="/user/one">One</Link>   //声明式
<NavLink to="/user/one">One</NavLink> //多加了一个active class用来定义样式
////////////////////////////////////////////////////////////////////////
import {history} from 'umi';

history.push('/user/one');  //命令式
//从函数组件中获取history
props.history.push('/user/one');  //命令式2
```

### HTML 模板

node_modules/@umijs/core/document.ejs 默认的 id=root 的 div 在这里

#### 修改默认模板

新建 `src/pages/document.ejs`，umi 约定如果这个文件存在，会作为默认模板

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Your App</title>
</head>
<body>
<div id="root"></div>
</body>
</html>
```

#### 配置模板

模板里可通过 context 来获取到 umi 提供的变量，context 包含：

- `route`，路由信息，需要打包出多个静态 HTML 时（即配置了 exportStatic 时）有效
- `config`，用户配置信息

### Mock 数据

#### 约定式 Mock 文件

Umi 约定 `/mock` 文件夹下所有文件为 mock 文件。

#### 编写 Mock 文件

如果 `/mock/api.ts` 的内容如下，

```js
export default {
    // 支持值为 Object 和 Array
    "GET /api/users": {users: [1, 2]},
    // GET 可忽略
    "/api/users/1": {id: 1},
    // 支持自定义函数，API 参考 express@4
    "POST /api/users/create": (req, res) => {
        // 添加跨域请求头
        res.setHeader("Access-Control-Allow-Origin", "*");
        res.end("ok");
    },
};
```

#### 配置 Mock

可以通过配置关闭，

```js
export default {
  mock: false,
};
```

也可以通过环境变量临时关闭，

```.bash
$ MOCK=none umi dev
```

#### 引入 Mock.js 生成大量模拟数据

[Mock.js](http://mockjs.com/) 是常用的辅助生成模拟数据的三方库，借助他可以提升我们的 mock 数据能力。

比如：

```js
import mockjs from "mockjs";

export default {
    // 使用 mockjs 等三方库
    "GET /api/tags": mockjs.mock({
        "list|100": [{name: "@city", "value|1-100": 50, "type|0-2": 1}],
    }),
};
```

### 整合 Dva

文档在这里 https://umijs.org/zh-CN/plugins/plugin-dva

1. 创建 UI 组件
2. 创建 model
3. 连接 UI 和 model

有没有有效的 dva model，可通过执行 `umi dva list model` 检查，或者执行 `umi g tmp` 后查看 `src/.umi/plugin-dva/dva.ts` 中检查 model 注册情况

#### 约定式的 model 组织方式

符合以下规则的文件会被认为是 model 文件，

- `src/models` 下的文件
- `src/pages` 下，子目录中 models 目录下的文件
- `src/pages` 下，所有 model.ts 文件(不区分任何字母大小写)

### 运行时配置

运行时配置和配置的区别是他跑在浏览器端，基于此，我们可以在这里写函数、jsx、import 浏览器端依赖等等，注意不要引入 node 依赖。

#### 配置方式

约定 `src/app.tsx` 为运行时配置。

#### patchRoutes({ routes })

修改路由。

比如在最前面添加一个 `/foo` 路由，

```js
export function patchRoutes({routes}) {
    routes.unshift({
        path: "/foo",
        exact: true, //严格匹配模式
        component: require("@/extraRoutes/foo").default,
    });
}
```

`render` 配置配合使用，请求服务端根据响应动态更新路由，**根据权限返回路由**

#### render(oldRender: Function) - 权限路由校验

覆写 render。

比如用于渲染之前做权限校验，

```js
let extraRoutes;

export function patchRoutes({routes}) {
    merge(routes, extraRoutes);
}

//获取路由
export function render(oldRender) {
    //模拟后端
    extraRoutes = [
        {path: "/server",component=“/user2" }
    ]
    //真得问后端要
    fetch('/api/routes').then(res => res.json()).then((res) => {
        extraRoutes = res.routes;
        oldRender();  //负责渲染
    })
}

//渲染之前做权限校验
    export function render(oldRender) {
        fetch('/api/auth').then(auth => {
            if (auth.isLogin) {
                oldRender()
            } else {
                history.push('/login');
                oldRender()
            }
        });
    }
```

#### onRouteChange({ routes, matchedRoutes, location, action }) - 埋点统计

在初始加载和路由切换时做一些事情。

比如用于做**埋点统计**，

```js
export function onRouteChange({ location, routes, action }) {
  bacon(location.pathname);
}
```

比如用于设置标题，

```js
export function onRouteChange({ matchedRoutes }) {
    if (matchedRoutes.length) {
        document.title = matchedRoutes[matchedRoutes.length - 1].route.title || "";
    }
}
```

### 使用 Umi UI

添加区块，只能在 src/下的 page

dva 添加需要配置

{% endraw %}
