<template>
    <div
        v-if="onlyTags"
        ref="scroller"
        class="b-tree"
        :style="{ height: option.height + 'px', width: '300px' }"
        @scroll="handleScroll"
    >
        <div class="b-tree__phantom" :style="{ height: contentHeight }"></div>
        <div
            class="b-tree__content"
            :style="{ transform: `translateY(${offset}px)` }"
        >
            <div
                v-for="(item, index) in visibleData"
                :key="item.id"
                class="b-tree__list-view"
                :style="{
                    paddingLeft: 18 * (item.level - 1) + 'px',
                    height: option.itemHeight + 'px',
                }"
            >
                <div
                    v-if="item.children && item.children.length"
                    class="icon-expand-wrap"
                    @click="toggleExpand(item)"
                >
                    <i :class="item.expand ? 'b-tree__expand' : 'b-tree__close'" />
                </div>
                <span v-else style="margin-right: 5px"></span>
                <div :key="index">
                    <el-checkbox
                        v-model="item.checked"
                        :disabled="disabled||onlyTags"
                        :indeterminate="!item.checked && item.semiChecked"
                        @change="handleCheck($event, item)"
                    />
                    {{ item.label }}
                </div>
            </div>
        </div>
    </div>
    <el-select
        v-else
        :value="currentValue"
        :placeholder="disabled ? '下拉查看' : '请选择'"
        popper-class="inner-business-multiple"
        class="tree-search-wrap"
        multiple
        size="small"
        :style="{width: this.width + 'px'}"
        :collapse-tags="isCollapseTags"
        @visible-change="visibleChange"
    >
        <template>
            <el-option
                value="__root"
                style="height: auto; padding: 0; margin: 2px 0; background: #fff"
            >
                <div
                    ref="scroller"
                    class="b-tree"
                    :style="{ height: option.height + 'px' }"
                    @scroll="handleScroll"
                >
                    <div class="b-tree__phantom" :style="{ height: contentHeight }"></div>
                    <div
                        class="b-tree__content"
                        :style="{ transform: `translateY(${offset}px)` }"
                    >
                        <div
                            v-for="(item, index) in visibleData"
                            :key="item.id"
                            class="b-tree__list-view"
                            :style="{
                                paddingLeft: 18 * (item.level - 1) + 'px',
                                height: option.itemHeight + 'px',
                            }"
                        >
                            <div
                                v-if="item.children && item.children.length"
                                class="icon-expand-wrap"
                                @click="toggleExpand(item)"
                            >
                                <i :class="item.expand ? 'b-tree__expand' : 'b-tree__close'" />
                            </div>
                            <span v-else style="margin-right: 5px"></span>
                            <div :key="index">
                                <el-checkbox
                                    v-if="!onlyTags"
                                    v-model="item.checked"
                                    :disabled="disabled"
                                    :indeterminate="!item.checked && item.semiChecked"
                                    @change="handleCheck($event, item, true)"
                                />
                                {{ item.label }}
                            </div>
                        </div>
                    </div>
                </div>
            </el-option>
        </template>
    </el-select>
</template>
<script>
import 'element-ui/lib/theme-chalk/checkbox.css';
import {
    Checkbox as ElCheckbox,
    Select as ElSelect,
    Option as ElOption
} from 'element-ui';
import 'element-ui/lib/theme-chalk/select.css';

export default {
    name: 'VueBigTree',
    components: {
        ElCheckbox,
        ElSelect,
        ElOption
    },
    props: {
        onlyTags: {
            type: Boolean,
            default: false
        },
        width: {
            type: String,
            default: '444'
        },
        disabled: {
            type: Boolean,
            default: false
        },
        data: {
            type: Array,
            default() {
                return [];
            }
        },
        tree: {
            type: Array,
            required: true,
            default() {
                return [];
            }
        },
        defaultExpand: {
            type: Boolean,
            required: false,
            default: false
        },
        option: {
            // 配置对象
            type: Object,
            required: true,
            default() {
                return {
                    height: 500, // 滚动容器的高度
                    itemHeight: 25 // 单个item的高度
                };
            }
        }
    },
    data() {
        return {
            currentValue: [],
            filterText: '',
            treeData: [],
            offset: 0, // translateY偏移量
            contentHeight: '0px',
            visibleData: []
        };
    },
    computed: {
        // 已选tag是否收起
        isCollapseTags() {
            return this.currentValue.length > 10;
        },
        flattenTree() {
            const flatten = function(
                list,
                childKey = 'children',
                level = 1,
                parent = null,
                defaultExpand = true
            ) {
                let arr = [];
                if (list && list.length) {
                    list.forEach((item) => {
                        item.level = level;
                        if (item.expand === undefined) {
                            item.expand = defaultExpand;
                        }
                        if (item.visible === undefined) {
                            item.visible = true;
                        }
                        if (!parent.visible || !parent.expand) {
                            item.visible = false;
                        }
                        /* if (level === 1) {
                            item.expand = false;
                        } */
                        item.parent = parent;
                        arr.push(item);
                        if (item[childKey]) {
                            arr.push(
                                ...flatten(
                                    item[childKey],
                                    childKey,
                                    level + 1,
                                    item,
                                    defaultExpand
                                )
                            );
                        }
                    });
                }
                return arr;
            };
            return flatten(
                this.treeData,
                'children',
                1,
                {
                    level: 0,
                    visible: true,
                    expand: true,
                    children: this.treeData
                },
                this.defaultExpand
            );
        },
        visibleCount() {
            return Math.floor(this.option.height / this.option.itemHeight);
        }
    },
    watch: {
        data: {
            deep: true,
            handler(checked) {
                this.setChecked(checked);
            },
            immediate: true
        }
    },
    mounted() {
        this.updateView();
    },
    methods: {
        visibleChange(val) {
            this.updateVisibleData();
        },
        setChecked(checked = []) {
            const tree = this.$clonedeep(this.tree);
            const setValue = (item, checkedArray, level = 0) => {
                const beingChecked = level === 0 ? item.id.split('|')[1] : item.id;
                if (checkedArray.includes(parseInt(beingChecked))) {
                    this.$nextTick(() => {
                        this.handleCheck(true, item);
                    });
                } else {
                    this.$nextTick(() => {
                        this.handleCheck(false, item);
                        this.collapse(item);
                    });
                }
                if (item.children && item.children.length) {
                    item.children.forEach((i) => {
                        setValue(i, checkedArray, level + 1);
                    });
                }
            };
            if (checked.length) {
                const treeData = tree.map((item) => {
                    const separator = item.id.split('|');
                    if (separator.length) {
                        const foundIndex = checked.findIndex(
                            (item) => String(item.id) === separator[0]
                        );
                        if (foundIndex !== -1) {
                            const checkedNodes = checked[foundIndex].nodes;
                            setValue(item, checkedNodes, 0);
                        }
                    }
                    return item;
                });
                this.treeData = treeData;
            } else {
                this.treeData = [];
                this.$nextTick(() => {
                    this.treeData = this.$clonedeep(this.tree);
                });
            }
            this.$nextTick(() => {
                this.updateView();
                this.currentValue = this.getAllChecked();
            });
        },
        getAllChecked() {
            let tree = this.$clonedeep(this.treeData);
            const checked = [];
            setSimplifiedData(tree, checked);
            function setSimplifiedData(tree, checked) {
                if (tree && tree.length) {
                    tree.forEach((item) => {
                        if (item.path || item.label) {
                            item.checked && checked.push(item.path || item.label);
                        }
                        if (item.children && item.children.length) {
                            setSimplifiedData(item.children, checked);
                        }
                    });
                }
                return [];
            }
            return checked;
        },
        getAllNodes(isStrict) {
            let tree = this.$clonedeep(this.treeData);
            tree = tree.map((item) => {
                if (typeof item.id === 'string' && item.id.indexOf('|') > 0) {
                    let arr = item.id.split('|');
                    return {
                        id: Number(arr[0]),
                        node: {
                            id: Number(arr[1]),
                            checked: isStrict ? item.checked : (item.checked || item.semiChecked),
                            children: setSimplifiedData(item.children)
                        }
                    };
                }
                return item;
            });
            function setSimplifiedData(tree) {
                if (tree && tree.length) {
                    return tree.map((item) => {
                        const simplifyData = {
                            id: item.id,
                            checked: item.checked || item.semiChecked
                        };
                        if (item.children && item.children.length) {
                            simplifyData.children = setSimplifiedData(item.children);
                        }
                        return simplifyData;
                    });
                }
                return [];
            }
            // tree.node.children = setSimplifiedData(tree.node.children);
            return tree;
        },
        handleCheck(checked, node, flag = false) {
            this.$set(node, 'checked', checked);
            this.$set(node, 'semiChecked', false);
            this.recursionChecked(node.children, checked);
            if (checked) {
                this.recursionParentChecked(node.parent, true);
            } else {
                this.recursionParentUnchecked(node.parent, false, node);
            }
            if (flag) {
                this.$nextTick(() => {
                    this.currentValue = this.getAllChecked();
                });
            }
        },
        recursionChecked(children, checked) {
            children.forEach((node) => {
                this.$set(node, 'checked', checked);
                this.$set(node, 'semiChecked', false);
                if (node.children) {
                    this.recursionChecked(node.children, checked);
                }
            });
        },
        recursionParentChecked(parent, checked) {
            if (parent && parent.children && parent.children.length) {
                if (parent.children.every((child) => child.checked)) {
                    parent.checked = true;
                    this.$set(parent, 'semiChecked', false);
                    this.recursionParentChecked(parent.parent, true);
                } else {
                    this.$set(parent, 'checked', false);
                    this.$set(parent, 'semiChecked', true);
                    this.recursionParentUnchecked(parent.parent, false);
                }
            }
        },
        recursionParentUnchecked(parent, checked, node) {
            if (!parent) return;
            parent.checked = checked;
            const setUnchecked = (tree) => {
                tree.checked = false;
                if (tree.children && tree.children.length) {
                    tree.children.forEach((iTree) => {
                        iTree.checked = false;
                        this.$set(iTree, 'semiChecked', false);
                    });
                    setUnchecked(tree.children);
                }
            };
            node && setUnchecked(node);
            if (parent.children.some((child) => child.checked || child.semiChecked)) {
                // 半选
                this.$set(parent, 'semiChecked', true);
                this.$set(parent, 'checked', false);
            } else {
                // 不选
                this.$set(parent, 'checked', false);
                this.$set(parent, 'semiChecked', false);
            }
            if (parent.parent) {
                this.recursionParentUnchecked(parent.parent, checked);
            }
        },
        updateView() {
            this.getContentHeight();
            this.$emit('update', this.tree);
            this.handleScroll();
        },
        handleScroll() {
            this.updateVisibleData(this.$refs.scroller.scrollTop);
        },
        updateVisibleData(scrollTop = 0) {
            let start = Math.floor(scrollTop / this.option.itemHeight) - Math.floor(this.visibleCount / 2);
            start = start < 0 ? 0 : start;
            const end = start + this.visibleCount * 2;
            const allVisibleData = (this.flattenTree || []).filter(
                (item) => item.visible
            );
            this.visibleData = allVisibleData.slice(start, end);
            this.offset = start * this.option.itemHeight;
        },

        getContentHeight() {
            this.contentHeight = (this.flattenTree || [])
                .filter((item) => item.visible).length * this.option.itemHeight + 'px';
        },

        toggleExpand(item) {
            const isExpand = item.expand;
            if (isExpand) {
                this.collapse(item, true); // 折叠
            } else {
                this.expand(item, true); // 展开
            }
            this.updateView();
        },

        // 展开节点
        expand(item) {
            item.expand = true;
            item.children.forEach((node) => {
                this.$set(node, 'expand', false);
                this.$set(node, 'visible', true);
            });
            // this.recursionVisible(item.children, true);
        },
        // 折叠节点
        collapse(item) {
            item.expand = false;
            this.recursionVisible(item.children, false);
        },

        // 折叠所有
        collapseAll(level = 1) {
            this.flattenTree.forEach((item) => {
                item.expand = false;
                if (item.level !== level) {
                    item.visible = false;
                }
            });
            this.updateView();
        },

        // 展开所有
        expandAll() {
            this.flattenTree.forEach((item) => {
                item.expand = true;
                item.visible = true;
            });

            this.updateView();
        },

        // 递归节点
        recursionVisible(children, status) {
            children.forEach((node) => {
                node.visible = status;
                if (node.children) {
                    this.recursionVisible(node.children, status);
                }
            });
        }
    }
};
</script>
<style lang="scss">
.inner-business-multiple {
    .el-scrollbar {
        padding: 0 12px;
        min-width: 400px;
    }
    .el-select-dropdown__item.selected {
        font-weight: normal;
        color: #606266;
    }
    .el-select-dropdown__item {
        font-size: 13px;
    }
}
.tree-search-wrap {
    .el-tag .el-icon-close{
        display: none;
    }
}
.b-tree {
  position: relative;
  overflow-y: scroll;
}
.b-tree__phantom {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}
.b-tree__content {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  min-height: 100px;
}
.b-tree__list-view {
  display: flex;
  align-items: center;
  cursor: pointer;
  .el-checkbox {
    margin-right: 2px;
  }
}
.b-tree__content__item {
  padding: 5px;
  box-sizing: border-box;

  display: flex;
  justify-content: space-between;
  position: relative;
  align-items: center;
  cursor: pointer;
}
.b-tree__content__item:hover,
.b-tree__content__item__selected {
  background-color: #d7d7d7;
}
.b-tree__content__item__icon {
  position: absolute;
  left: 0;
  color: #c0c4cc;
  z-index: 10;
}
.icon-expand-wrap {
  // height: 33px;
  display: inline-block;
  height: 17px;
  line-height: 17px;
}
.b-tree__close {
  display: inline-block;
  width: 0;
  height: 0;
  overflow: hidden;
  font-size: 0;
  margin-right: 4px;
  border-width: 4px;
  border-color: transparent transparent transparent #c0c4cc;
  border-style: dashed dashed dashed solid;
}
.b-tree__expand {
  display: inline-block;
  width: 0;
  height: 0;
  overflow: hidden;
  font-size: 0;
  margin-right: 4px;
  margin-top: 8px;
  border-width: 4px;
  border-color: #c0c4cc transparent transparent transparent;
  border-style: solid dashed dashed dashed;
}
</style>