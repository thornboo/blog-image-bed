---
title: Vue3 组件封装：让 UI 组件库“乖乖听话”的正确姿势
date: '2025-05-14 16:16:21'
updated: '2025-05-16 16:11:53'
permalink: >-
  /post/vue3-component-packaging-the-correct-way-to-make-the-ui-component-library-obedient-1wxjv5.html
comments: true
toc: true
---



# Vue3 组件封装：让 UI 组件库“乖乖听话”的正确姿势

　　在日常开发中，UI 组件库（比如 Ant Design Vue、Element Plus）就像外卖快餐，方便快捷，但总有点“不合胃口”。于是，为了让这些组件更符合我们的需求，我们常常需要对它们进行二次封装。本文将带你体验如何用 Vue3 的 `$attrs` 和 `$slots`，优雅地“调教”这些组件，让它们更灵活、更贴合项目需求。

　　当然，封装组件这件事儿，说起来简单，做起来就像修水管——一开始觉得能搞定，结果越修越漏。别急，我们会逐步分析封装过程中可能遇到的问题，并给出解决方案。

---

## 1. 需求背景：为什么要封装？

　　假设我们在项目中频繁使用 Ant Design Vue 的 `a-modal` 组件，但总有一些“个性化需求”需要实现，比如：

- 修改弹窗样式
- 自定义按钮文字
- 每次关闭后销毁内容
- 将弹窗渲染到指定元素上

　　于是，我们封装了一个自定义弹窗组件，代码如下：

```vue
<template>
  <div ref="myModal" class="custom-modal"></div>
  <a-modal
    v-model:visible="visible"
    centered
    destroyOnClose
    :getContainer="() => $refs.myModal"
    @ok="handleOk"
    @cancel="handleCancel"
    :style="{ width: '560px', ...style }"
    :cancelText="cancelText"
    :okText="okText"
  >
    <slot></slot>
  </a-modal>
</template>

<script setup>
const props = defineProps({
  title: { type: String, default: "" },
  style: { type: Object, default: () => ({}) },
  cancelText: { type: String, default: "取消" },
  okText: { type: String, default: "确定" },
});
const emits = defineEmits(["handleOk", "handleCancel"]);
const visible = ref(false);

const handleOk = () => emits("handleOk");
const handleCancel = () => emits("handleCancel");

defineExpose({ visible });
</script>

<style lang="less" scoped>
.custom-modal {
  :deep(.ant-modal) {
    // 样式代码省略
  }
}
</style>
```

　　看起来还不错，对吧？我们可以这样使用它：

```vue
<CustomModal ref="xxxModal" title="提示" @ok="onOk" @cancel="onCancel">
  内容区域
</CustomModal>
```

　　但别高兴太早，问题很快就来了。

---

## 2. 问题来了：组件封装中的“烦恼”

### 问题 1：属性扩展的“无底洞”

　　同事 A 说：“我想去掉右上角的关闭按钮，能不能加个 `closable` 属性？”

　　同事 B 说：“我想让弹窗不居中显示，能不能加个 `centered` 属性？”

　　同事 C 说：“我的需求比较特别，我想让弹窗的 z-index 是 9999。”

　　于是，代码开始膨胀：

```vue
<a-modal :closable="closable" :centered="centered" :zIndex="zIndex"></a-modal>

<script setup>
const props = defineProps({
  closable: { type: Boolean, default: true },
  centered: { type: Boolean, default: true },
  zIndex: { type: Number, default: 1000 },
});
</script>
```

　　每次有新需求，就得改组件代码，最终变成一个“巨无霸”组件。有没有更简单的方式？当然有，`$attrs` 闪亮登场！

#### 解决方案：用 `$attrs` 动态绑定属性

　　`$attrs` 是 Vue3 提供的一个神奇属性，它会自动收集父组件传递的所有非 `props` 参数。我们可以用 `v-bind="$attrs"` 将这些参数一股脑绑定到子组件上。

　　改造后的代码：

```vue
<a-modal
  v-model:visible="visible"
  :getContainer="() => $refs.myModal"
  :style="{ width: '560px', ...style }"
  destroyOnClose
  v-bind="$attrs"
>
  <slot></slot>
</a-modal>
```

　　父组件想加啥属性都随意，`$attrs` 会帮你兜底：

```vue
<CustomModal :footer="null" :centered="false" :zIndex="9999"></CustomModal>
```

　　再也不用频繁修改组件代码了，舒服！

---

### 问题 2：插槽的“加班地狱”

　　UI 组件通常提供多个插槽，比如 `a-modal` 的 `title`、`footer` 等。如果我们逐一定义这些插槽，代码会变得又臭又长：

```vue
<a-modal>
  <!-- 默认插槽 -->
  <slot></slot>

  <!-- 标题插槽 -->
  <template #title>
    <slot name="title">{{ title }}</slot>
  </template>

  <!-- 页脚插槽 -->
  <template #footer>
    <slot name="footer"></slot>
  </template>
</a-modal>
```

　　一旦组件新增了插槽，就得加班修改代码。有没有更优雅的方式？当然有，`$slots` 来解救你！

#### 解决方案：动态绑定插槽

　　`$slots` 是 Vue 提供的另一个神奇属性，它会自动收集父组件传递的所有插槽。我们可以动态遍历 `$slots`，将它们绑定到子组件上。

　　改造后的代码：

```vue
<a-modal>
  <template v-for="(_val, name) in $slots" #[name]="options">
    <slot :name="name" v-bind="options || {}"></slot>
  </template>
</a-modal>
```

　　父组件传递的插槽会自动绑定到 `a-modal` 上，无需手动定义。

　　使用示例：

```vue
<CustomModal>
  <template #title="{ arg1, arg2 }">
    自定义标题
  </template>
</CustomModal>
```

---

### 问题 3：`v-model` 的“控制权之争”

　　目前，我们需要通过 `ref` 来控制弹窗的显示和隐藏：

```vue
<CustomModal ref="modalRef"></CustomModal>
```

　　但这种方式不够直观。我们可以通过 Vue3 的 `v-model` 语法糖，让父组件直接控制弹窗的显隐。

#### 改造代码：支持 `v-model`

　　我们可以监听 `props.visible`，并通过事件同步状态：

```vue
<template>
  <div ref="myModal" class="custom-modal"></div>
  <a-modal
    :visible="visible"
    v-bind="$attrs"
  >
    <template v-for="(_val, name) in $slots" #[name]="ops">
      <slot :name="name" v-bind="ops || {}"></slot>
    </template>
  </a-modal>
</template>

<script setup>
defineProps(["visible"]);
const emit = defineEmits(); // Vue3 会自动处理 "update:visible"

watch(
  () => props.visible,
  (newVal) => emit("update:visible", newVal)
);
</script>
```

　　使用方式：

```vue
<CustomModal v-model:visible="modalVisible"></CustomModal>
```

---

## 3. 完整代码示例

　　最终封装组件如下：

```vue
<template>
  <div ref="myModal" class="custom-modal"></div>
  <a-modal
    :visible="visible"
    destroyOnClose
    v-bind="$attrs"
  >
    <template v-for="(_val, name) in $slots" #[name]="ops">
      <slot :name="name" v-bind="ops || {}"></slot>
    </template>
  </a-modal>
</template>

<script setup>
defineProps(["visible"]);
const emit = defineEmits();

watch(
  () => props.visible,
  (newVal) => emit("update:visible", newVal)
);
</script>

<style lang="less" scoped>
.custom-modal {
  // 样式代码
}
</style>
```

---

## 4. 总结：封装组件的“深坑”

　　封装组件不是一件轻松的事，尤其是当需求越来越复杂时。以下是一些常见的“深坑”：

1. **属性扩展问题**：频繁新增 `props` 会导致组件代码臃肿。

    - **解决方案**：使用 `$attrs` 动态绑定属性。
2. **插槽管理问题**：手动定义插槽过于繁琐。

    - **解决方案**：使用 `$slots` 动态绑定插槽。
3. **状态同步问题**：显隐控制不够直观。

    - **解决方案**：通过 `v-model` 实现状态同步。

---

## 5. 展望：封装组件的“无限可能”

　　在实际开发中，封装组件还会遇到以下问题：

- **复杂交互逻辑**：如何封装支持多种交互的组件？
- **性能优化**：如何避免组件封装导致的性能问题？
- **通用性扩展**：如何设计更通用的组件，适配更多场景？

　　这些问题等待下次文章继续讨论进阶。如果你有更好的思路，欢迎交流！
