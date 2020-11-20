# vue-big-tree
- 使用虚拟渲染，支持上万级数据量的tree组件(支持全选与半选, 需要element的checkbox)
- Yet another tree component which supports the rendering of > 10,000 chidlren nodes.

### API说明(待补充)
props:
- option: 
    - height 滚动容器的高度
    - itemHeight: 25 // 单个item的高度
- tree: 传入的初始化tree模板
- data: 默认选中的节点
- onlyTags: 是不是不需要input搜索框
- disabled: 节点是否禁用

method:
 * getAllNodes 获取树层结构所有的节点
 * getAllChecked 获取所有选中的节点