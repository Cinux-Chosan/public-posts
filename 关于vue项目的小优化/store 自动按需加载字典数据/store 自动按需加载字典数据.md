
## store 自动按需加载字典数据（推荐程度 ☆☆☆☆）

*本文属于个人在项目中完成的小优化点记录，其它小优化导航连接：[关于个人在 Vue 项目上的一些小优化记录](https://juejin.cn/post/7209192211975340088)*

在以往我经历过的项目中，经常会出现某些页面需要用到字典数据的场景，但大多数都是要么一开始就加载所有字典数据，在这种场景下又由于我所在的项目组大多数后端接口并没有进行优化，通常都不支持同时传入多个字典键名进行查询，从而导致同时发起多个请求进行数据查询。要么就是在使用的地方编写代码单独发起请求去获取，导致重复劳动力。

但字典数据大多数都通用并且不会频繁发生改变的，我们完全可以将它放到 store 中进行存储，并做成自动化按需加载数据。

但何时加载数据并且存放到 store 中呢？

当然是在 getter 中，页面直接使用 getter 获取数据，如果没有加载过，那就返回默认值并发请求加载数据，当加载完成之后赋值给 state 对应的字段，由于 vue 本身是响应式的，因此页面会自动应最新的 store 数据从而完成数据更新，如果数据已经加载过了就直接返回已经存在的值即可。

下面是一个基于 vuex 的可用示例：

```ts
// 引入获取字典数据的 api 方法
import { getDataDictionary } from "@/api/common";

/**
 * 自动按需加载字典配置数据
 */

// 数据加载的状态
enum LoadingStatus {
  LOADING = "loading",
  LOADED = "loaded",
  NOT_LOAD = "notLoad",
}

// 会根据 config 生成 state 和 getter
const config = {
  "bill.data.product.code": {
    default: [],
    parser: (data) => data[0]?.dataValue?.split(",").map(Number),
  } as ConfigItem<number[]>,
  "some.other.dict.key": null,
} as const;

const state = {} as State;

const getters = {};

Object.keys(config).forEach((keyName: string) => {
  let confItem = config[keyName];

  // 如果 config 中只有键名没有对应的配置，则使用默认配置
  if (!confItem) {
    confItem = config[keyName] = quickCreateConf([]);
  }

  state[keyName] = confItem.default;

  getters[keyName] = function () {
    // 如果未加载过数据则进入加载数据的逻辑
    if (
      !confItem.loadStatus ||
      confItem.loadStatus === LoadingStatus.NOT_LOAD
    ) {
      // 加载数据的函数
      const loadData = async () => {
        confItem.loadStatus = LoadingStatus.LOADING;
        try {
          const resp = await getDataDictionary({ keyName });
          // 将返回值赋值给 state，页面会自动应用最新的数据
          state[keyName] = confItem.parser?.(resp.result) ?? resp.result;
          confItem.loadStatus = LoadingStatus.LOADED;
        } catch (error) {
          confItem.loadStatus = LoadingStatus.NOT_LOAD;
        }
      };

      loadData();
    }
    return state[keyName];
  };
});

export default {
  namespaced: true,
  state,
  getters,
};

function quickCreateConf<T>(defaultValue: T): ConfigItem<T> {
  return {
    default: defaultValue,
  };
}

type ConfigItem<T> = {
  default: T;
  loadStatus?: LoadingStatus;
  parser?: (data: any) => T;
};

type Config = typeof config;

type State = {
  [key in keyof Config]: Config[key]["default"];
};
```

在使用的地方直接通过 vuex 的 `mapGetters` 来引用对应的数据即可，不用再关心这个数据是否加载过，也不用考虑是否要手动发起请求加载等情况，一切都是那么的平静。

以上只是一个自动按需获取数据的思路，并且给出了 vuex 实现的示例，但这种方式在其它框架或者数据管理中同样适用。

另外还可以加入一些控制频率或者失败次数的逻辑，这样能有效避免因请求失败导致频繁重复发起数据请求的情况发生。
