<!-- TOC -->

- [vue组件传值方式整理](#vue组件传值方式整理)
- [父子组件传值](#父子组件传值)
    - [props/$emit](#propsemit)
    - [ref与$parent/$children](#ref与parentchildren)
- [隔代组件传值（爷孙组件参数互传）](#隔代组件传值爷孙组件参数互传)
    - [$attrs/$listeners](#attrslisteners)
    - [provide/inject](#provideinject)
- [全局传值](#全局传值)
    - [EventBus](#eventbus)
    - [VUEX](#vuex)

<!-- /TOC -->
# vue组件传值方式整理
# 父子组件传值
## props/$emit
+ 父组件

```
<template>
  <div>
    {{ msg }}
    <Children :msg="msg" @changeMsg="changeMsg"/>
  </div>
</template>

<script>
import Children from "@/views/assembly/children";

export default {
  components: {
    Children
  },
  name: "parent",
  data() {
    return {
      msg: "父组件的值"
    }
  },
  methods: {
    changeMsg(data) {
      this.msg = data
    }
  }
}
</script>

<style scoped>

</style>
```
+ 子组件

```
<template>
  <div>
    {{ msg }}
    <button @click="clickbtn">点我改变父组件</button>
  </div>
</template>

<script>
export default {
  name: "children",
  props: {
    msg: {
      type: String,
      default: "default"
    }
  },
  data() {
    return {}
  },
  methods: {
    clickbtn() {
      let msg = "改变后的值" + Math.ceil(Math.random()*10)
      this.$emit("changeMsg", msg)
    }
  }
}
</script>

<style scoped>

</style>
```
## ref与$parent/$children
+ 父组件

```
<template>
  <div>
    {{ msg }}
    <button @click="changeChildren">点击改变子元素的值</button>
    <Children ref="children"/>
  </div>
</template>

<script>
import Children from "@/views/assembly/children";

export default {
  components: {
    Children
  },
  name: "parent",
  data() {
    return {
      msg: "父组件的值"
    }
  },
  methods: {
    changeChildren() {
      let msg="父组件改变子组件"
      this.$refs.children.setMsg(msg);
    },
    childrenValue(nsg){
      this.msg=nsg
    }
  }
}
</script>

<style scoped>

</style>
```
+ 子组件

```
<template>
  <div>
    {{ childrenMsg }}
    <button @click="clickbtn">点我改变父组件的值</button>
  </div>
</template>

<script>
export default {
  name: "children",
  data() {
    return {
      childrenMsg: "子元素的值"
    }
  },
  methods: {
    setMsg(msg) {
      this.childrenMsg = msg
    },
    clickbtn() {
      let msg="子组件改变父组件"
      this.$parent.childrenValue(msg);
    }
  }
}
</script>

<style scoped>

</style>
```
# 隔代组件传值（爷孙组件参数互传）
## $attrs/$listeners

+ 爷组件

```
<template>
  <div>
    {{ msg }}
    {{ msg2 }}
    <Parent :msg="msg" @setVal="setVal"/>
  </div>
</template>

<script>
import Parent from "./parent"

export default {
  components: {
    Parent
  },
  name: "grandparent",
  data() {
    return {
      msg: "我是爷组件的值",
      msg2: "接收孙组件的值"
    }
  },
  methods: {
    setVal(msg) {
      this.msg2 = msg
    }
  }
}
</script>

<style scoped>

</style>
```
+ 父组件

```
<template>
  <div>
    <Children v-bind="$attrs" v-on="$listeners"/>
    <!-- $attrs爷传孙，$listeners孙传爷 -->
  </div>
</template>

<script>
import Children from "./children";

export default {
  components: {
    Children
  },
  name: "parent",
  data() {
    return {}
  },
  methods: {}
}
</script>

<style scoped>

</style>
```
+ 孙组件

```
<template>
  <div>
    {{ msg }}
    {{ msg2 }}
    <button @click="toValue">点击传值给爷组件</button>
  </div>
</template>

<script>
export default {
  name: "children",
  props: {
    msg: String,
  },
  data() {
    return {
      msg2: "我是孙组件的值"
    }
  },
  methods: {
    toValue() {
      let msg = "孙组建的传值" + Math.ceil(Math.random() * 10)
      this.$emit("setVal", msg)
    }
  }
}
</script>

<style scoped>

</style>
```

## provide/inject
> 提示：provide 和 inject 绑定并不是可响应的。这是刻意为之的。然而，如果你传入了一个可监听的对象，那么其对象的 property 还是可响应的。如果传入的值是字符串，数字，布尔值等基本类型则会无响应！！！

+ 祖先组件（所有后代组件都能拿到该值，但传动态值必须是个对象！！！）

```
<template>
  <div>
    {{dataObj.msg}}
    <Parent />
  </div>
</template>

<script>
import Parent from "./parent"

export default {
  components: {
    Parent
  },
  name: "grandparent",
  provide() {
    return {
      data: this.dataObj
    }
  },
  data() {
    return {
      dataObj: {
        msg: '这是爷组件的值',
        num: 10
      }
    }
  },
  methods: {}
}
</script>

<style scoped>

</style>
```
+ 后代组件（任意一个后代元素通过该写法都能拿到值）

```
<template>
  <div>
    {{ txt }}
  </div>
</template>

<script>
export default {
  name: "parent",
  inject: ['data'],
  computed: {
    txt() {
      return `${this.data.msg}+${this.data.num}+${this.parentMsg}`;
    }
  },
  data() {
    return {
      parentMsg: "后代组件"
    }
  },
  methods: {}
}
</script>

<style scoped>

</style>
```
# 全局传值
## EventBus
> 调用完之后必须销毁，否则会出现bug！！

+ src/tools/event-bus.js(新建文件)

```
import Vue from 'vue'
export const Bus = new Vue()
```
+ 事件总线

```
<template>
  <div id="app">
    <button @click="setVal">点击</button>
    <Home></Home>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import {Bus} from '@/tools/event-bus'
import Home from '@/views/Home';

export default Vue.extend({
  name: 'App',
  components: {
    Home
  },
  data() {
    return {
      msg: 0
    }
  },
  methods: {
    setVal() {
      this.msg++;
      Bus.$emit("share", this.msg);
    }
  }
});
</script>
```
+ 事件接收

```
<template>
  <div class="sun">
    <p>{{msg}}</p>
  </div>
</template>

<script>
import { Bus } from '@/tools/event-bus'
export default {
  name: 'Sun',
  data(){
    return {
      msg:''
    }
  },
  methods:{
    setMsg(){
      Bus.$on("share",(data)=>{
        this.msg = data;
      })
    }
  },
  created() {
    this.setMsg();
  },
  destroyed(){
    Bus.$off("share");
  }
}
</script>
```
## VUEX
不在这展开说了，具体使用方法很多博客，官网也有示例