---
title: 使用MpKit的事件、Mixin、SetData优化、全局拦截等功能增强开发多平台小程序
---

> MpKit与本文章均在不断更新，为保证您获取到的内容是最新的，请关注：[https://imingyu.github.io/2020/mpkit/](https://imingyu.github.io/2020/mpkit/)

# 前言

近年来，多个公司都开发出了小程序这样的“微应用”方案，其在生态扩展、功能增强等方面都发挥着重要的作用；

作为开发者却要同时面对多个小程序平台，实现相似的功能，如果不考虑使用框架的话，在底层的一些基础技术解决方案上，有没有一个轻量级的选择？

我们在开发一个小程序时，往往会有以下技术需求：
1. 全局事件总线
2. 优化小程序的setData函数
3. 希望将App/Page/Component上的共有功能拆分，并可以有效性的重复利用，甚至组建为App/Page/Component基类
4. 可以全局拦截执行某些操作，如：异常报错、网络请求、功能制止、App/Page/Component的生命周期等

基于以上需求，都是我以往实现过的功能，所以我将我的经验总结成了一个开源项目MpKit，里面就包含对以上需求的功能实现，且不止于此；

MpKit的主要功能都经过单元测试，可放心使用，项目主页：[https://github.com/imingyu/mpkit](https://github.com/imingyu/mpkit)；

下面我来介绍下它的具体用法和功能列表。


# 简介

MpKit发布到了NPM平台，以模块化的方式划分为多个包，某些包不止能用在小程序平台，还可以用到h5等平台；某些包却无法在小程序上使用，包列表及相关功能如下：

> 以下包均支持TypeScript语言

| 包  | | 适用平台| | | | | 简介 |
|--|--|--|--|--|--|--|--|
|  | 小程序 | | | | H5 | Node.js | |
|  | 微信 |支付宝|百度|字节跳动| | | |
| [@mpkit/inject](https://www.npmjs.com/package/@mpkit/inject) | ● |● |● |● | | |提供小程序环境适用的多种实用函数或组件，如setData优化、Mixin、事件总线等。[查看文档](https://github.com/imingyu/mpkit/tree/master/packages/inject) |
| [@mpkit/set-data](https://www.npmjs.com/package/@mpkit/set-data) | ● |● |● |● || |小程序setData优化。[查看文档](https://github.com/imingyu/mpkit/tree/master/packages/set-data) |
| [@mpkit/ebus](https://www.npmjs.com/package/@mpkit/ebus) | ● |● |● |● |● | ●|提供事件触发、监听等功能。[查看文档](https://github.com/imingyu/mpkit/tree/master/packages/ebus) |
| [@mpkit/mixin](https://www.npmjs.com/package/@mpkit/mixin) | ● |● |● |● | | |为小程序提供混入功能。[查看文档](https://github.com/imingyu/mpkit/tree/master/packages/mixin) |
| [@mpkit/util](https://www.npmjs.com/package/@mpkit/util) | ● |● |● |● | | |工具函数[查看文档](https://github.com/imingyu/mpkit/tree/master/packages/util) |


# 安装
如果你的小程序项目中支持引入npm包，那么你直接根据自己的需要安装对应包即可，如：
```bash
npm i @mpkit/ebus -S
```

但是如果你是原生开发，不支持引入npm包，你最大的可能需要用到`@mpkit/inject`包，此包中的功能基本包含了其他包的所有功能，且支持按需插件化安装，可节省你的字节空间；使用`@mpkit/inject`：

1. 在磁盘任意位置创建临时目录如`xx/temp`;
2. 在此目录中执行命令：
```bash
npm i @mpkit/inject
```
3. 找到`xx/temp/node_modules/@mpkit/inject`下的`dist`目录，将其下内容拷贝到你的小程序项目目录下；
4. 找到`dist/config.js`文件，将你需要的功能插件引入，如：
```javascript
// 提供事件类功能
import { plugin as EbusPlugin } from './plugins/ebus';
// 提供混入功能
import { plugin as MixinPlugin } from './plugins/mixin';
// 提供setData优化
import { plugin as SetDataPlugin } from './plugins/set-data';
var config = {
    // 是否重写全局变量
    rewrite: {
        App: false, // 重写全局变量App为 plugins/mixin 中的MkApp 即 MpKit.App
        Page: false, // 重写全局变量Page为 plugins/mixin 中的MkPage 即 MpKit.Page
        Component: false, // 重写全局变量Component为 plugins/mixin 中的MkComponent 即 MpKit.Component
        Api: false, // 重写全局api变量wx/my/tt为 plugins/mixin 中的MkApi 即 MpKit.Api
        setData: false // 重写Page/Component中的setData为优化后的setData 即 MpKit.setData
    },
    plugins: [
        EbusPlugin,
        MixinPlugin,
        SetDataPlugin
    ]
};
export default config;
```
5. 如果配置了`config.rewrite`项，请在小程序项目的`app.js`的第一行处引入`@mpkit/inject/dist`中的`index.js`文件，否则无法实现全局变量重写；如果没有配置该项，则需要在使用时引入，如：
```javascript
import MpKit from '@mpkit/inject/dist/index';
MpKit.on(...);
```
6. 安装完成。
7. 小提示：不需要的插件js文件可以直接删掉，插件不直接相互依赖。


# 使用
> 这里着重介绍`@mpkit/inject`包的使用方式和细节，其他包可自行参考对应文档；


`@mpkit/inject`包提供下列功能，

| 方法/变量 | 作用 | 依赖插件 |
| -- | -- | -- |
| `on(eventName:string, handler:Function)` | 为某全局事件添加监听函数 | ebus |
| `off(eventName:string, handler:Function)` | 为某全局事件移除监听函数 | ebus |
| `emit(eventName:string, data:any)` | 触发某全局事件并传递数据 | ebus |
| `App(...mixins:MpAppSpec[]) : MpAppSpec` | 接收多个对象，对象结构与小程序`App`函数接收的对象结构一致，并将这些对象合并，返回一个新对象，包含所有对象的功能和数据，合并策略下文描述。 | mixin |
| `Page(...mixins:MpPageSpec[]) : MpPageSpec` | 接收多个对象，对象结构与小程序`Page`函数接收的对象结构一致，并将这些对象合并，返回一个新对象，包含所有对象的功能和数据，合并策略下文描述。 | mixin |
| `Component(...mixins:MpComponentSpec[]) : MpComponentSpec` | 接收多个对象，对象结构与小程序`Component`函数接收的对象结构一致，并将这些对象合并，返回一个新对象，包含所有对象的功能和数据，合并策略下文描述。 | mixin |
| `Api` | 与小程序上的`wx` `my` `swan` `tt`等对象的属性和方法一致，只不过在其方法上添加了钩子函数，方便拦截处理 | mixin |
| `MixinStore.addHook(type:string, hook:MpMethodHook) ` | `MpKit`中内置了很多全局钩子函数，方可实现了全局拦截，setData重写等功能，而如果你也想在全局添加自己的钩子函数，那么可以调用此函数 | mixin |
| `setData(view:any, data:any, callback:Function) : Promise<DiffDataResult>` | 以优化的方式向某个Page/Component设置数据，仅设置变化的数据，并以Promise的方式返回diff后的数据结果 | set-data |


## 依赖
从上面的功能列表中可以看到，某些方法或变量是依赖插件的，如果没有安装相关插件，则无法使用对用方法；

## App/Page/Component
当使用`MpKit.App/Page/Component`时，可传递多个对象，如：
```javascript
import MpKit from '@mpkit/inject/dist/index';
// 如果在config中配置了rewrite.App=true，则调用App等同于调用了[未重写的App(MpKit.App)]
App(MpKit.App({
    globalData: {
        name: 'Tom',
        age: 10
    },
    onShow() {
        console.log('onShow1')
    },
    add(a, b) {
        return a + b;
    }
}, {
    globalData: {
        age: 20
    },
    onShow() {
        console.log(`onShow2, ${this.add(2, 4)}`);
        console.log(this.globalData);
    }
}));
// 输出：onShow1
// 输出：onShow2, 6
// 输出：{ name: 'Tom', age: 20 }


Component(MpKit.Component({
    data: {
        name: 'Alice',
        products: [
            {
                name: '苹果',
                price: 6
            },
            {
                name: '香蕉',
                price: 5
            }
        ]
    },
    created() {
        console.log('created1')
    },
    methods: {
        sayhi() {
            console.log(`hi ${this.data.name}`);
        }
    }
}, {
    data: {
        products: [
            {
                price: 8
            }
        ]
    },
    created() {
        console.log('created2');
        this.sayhi();
        console.log(this.data.list);
    },
    methods: {
        sayhi() {
            console.log(`你好 ${this.data.name}`);
        }
    }
}));

// 输出：created1
// 输出：created2
// 输出：hi Alice
// 输出：你好 Alice
// 输出：[ { name: '苹果', price: 8 }, { name: '香蕉', price: 5 } ]
```

## 合并策略
从上面的例子可以看出合并策略是：
- 属性进行深度合并
- 方法会保留所有mixin中的方法体，按照顺序全部执行

## 钩子函数
为可以进行全局拦截`App/Page/Component/Api`上的方法，MpKit做了钩子函数的机制，具体为：
- 每个方法执行前会调用`before`钩子
- 如果`before`钩子函数存在，且有返回值（如果有多个`before`钩子则取最后一个不为`undefined|true`的结果）时
    - 对于`App/Page/Component`如果返回值为`false`，则不会继续向下执行
    - 对于`Api`如果返回值为`false`，则不会继续向下执行；同时如果返回值不为`true`和`undefined`时，会直接将结果返回出去，且不会继续向下执行
- 然后执行`方法体`，如果有多个，则依次全部执行，并返回最后一个不为空的结果
- 执行`after`钩子，并传入`方法体`的执行结果
- 如果是`Api`上的异步方法，还会根据结果回调（success|fail）在执行`complete`钩子

MpKit内置了很多钩子，用于全局事件触发、方法重写等，同时你可以添加自己的钩子函数，调用：

`MpKit.MixinStore.addHook(type:MpViewType.App|MpViewType.Page|MpViewType.Component|'Api', hook:MpMethodHook)`可为 App/Page/Component/Api 添加全局钩子函数；`MpMethodHook`的定义如下：

```typescript
interface MpMethodHookLike {
    before?(
        methodName: string,
        methodArgs: any[],
        methodHandler: Function,
        funId?: string
    );
    after?(
        methodName: string,
        methodArgs: any[],
        methodResult: any,
        funId?: string
    );
    catch?(
        methodName: string,
        methodArgs: any[],
        error: Error,
        errType?: string,
        funId?: string
    );
    complete?(
        methodName: string,
        methodArgs: any[],
        res: any,
        success?: boolean,
        funId?: string
    );
}
interface MpMethodHook extends MpMethodHookLike {
    [prop: string]: Function | MpMethodHookLike;
}
```

示例1：
```javascript
import MpKit from "@mpkit/inject/dist/index";
import { MpViewType } from "@mpkit/inject/dist/types";
MpKit.MixinStore.addHook(MpViewType.App, {
    before(methodName, methodArgs) {
        console.log(`before methodName=${methodName}`);
    },
    after(methodName, methodArgs, methodResult) {
        console.log(`after methodName=${methodName}, ${methodResult}`);
    },
    catch(methodName, methodArgs, error) {
        console.log(`catch err=${error.message}`);
    },
});
App(
    MpKit.App({
        onLaunch() {
            this.add(1, 2);
        },
        onShow() {
            throw new Error("test");
        },
        add(a, b) {
            return a + b;
        },
    })
);
// 输出：before methodName=onLaunch
// 输出：before methodName=add
// 输出：after methodName=add, 2
// 输出：after methodName=onLaunch,
// 输出：before methodName=onShow,
// 输出：catch err=test

MpKit.MixinStore.addHook("Api", {
    before(methodName, methodArgs, methodHandler, funId) {
        console.log(`before api=${methodName}`);
    },
    after(methodName, methodArgs, methodResult, funId) {
        console.log(`after api=${methodName}, ${methodResult}`);
    },
    complete(methodName, methodArgs, res, isSuccess, funId) {
        console.log(`complete api=${methodName}, ${isSuccess}, ${res}`);
    },
});
MpKit.Api.request({
    url: "...",
});
// 输出：before api=request
// 输出：after api=request, [RequestTask Object]
// 假设请求成功且返回字符串“1”，则输出：complete api=request, true, 1
// 假设请求失败，则输出：complete api=request, false, { errMsg:'...' }
```

示例2：当在`before`钩子中返回`false`会具体值时：
```javascript
MpKit.MixinStore.addHook(MpViewType.App, {
    onShow: {
        before(methodName, methodArgs) {
            console.log("hook onShow");
            return false;
        },
    },
});
App(
    MpKit.App({
        onLaunch() {},
        onShow() {
            console.log("self onShow");
        },
    })
);
// 仅输出：hook onShow

const store = {};
MpKit.MixinStore.addHook("Api", {
    before(methodName, methodArgs, methodHandler, funId) {
        console.log(`before methodName=${methodName}`);
        if (methodName === "setStorageSync") {
            store[methodArgs[0]] = methodArgs[1];
        }
        if (methodName === "getStorageSync") {
            // 并不会真正执行(wx|my|tt|..).getStorageSync
            return store[methodArgs[0]];
        }
    },
    after(methodName) {
        console.log(`after methodName=${methodName}`);
    },
});
MpKit.Api.setStorageSync("name", "Tom");
const name = MpKit.Api.getStorageSync("name");
console.log(name === store.name);
// 输出：before methodName=setStorageSync
// 输出：after methodName=setStorageSync
// 输出：before methodName=getStorageSync
// 输出：true
```

# 结语
希望MpKit可以为你带来便捷愉快的开发体验，祝大家工作顺心，家庭美满，加油！

> 可关注作者的其他开源项目：
> - [小松鼠](https://github.com/imingyu/squirrel)：适用于JavaScript平台的数据上报工具
> - [小程序调试辅助工具](https://github.com/imingyu/mp-console)：小程序控制台调试辅助工具