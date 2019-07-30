# Java

本章节主要对 Java 基础知识点进行梳理,请须知.

---

## 1. key map

在 mac 里面的 eclipse 的快捷键,一言难尽.这里记录一下常用的快捷键.

mac 和 win 对应关系如下:

- win -> command
- alt -> option
- ctrl -> ctrl

| 功能           | 快捷键      | 备注                                        |
| -------------- | ----------- | ------------------------------------------- |
| 查看快捷方式   | win+shift+L |                                             |
| 生成变量名称   | win+alt+L   |                                             |
| 格式化代码     | win+ctrl+F  |                                             |
| 自动补全代码   | alt+/       | ^+Option 被输入法切换占用了所以修改该快捷键 |
| 注释当前行代码 | win+/       |                                             |
| 整段代码注释   | win+ctrl+/  |                                             |
| 删除当前行代码 | win+d       |                                             |

修改自动补全代码快捷键: `eclipse->preference->general->keys->content assist`修改 binding 的快捷键即可.

---

## 2. 注释模板

注释模板很重要,写好了一次就不用自己写了.

在`preference -> java -> code style -> comments -> types`里面进行调节.

```java
/**
 * 
 * </p>
 * @author cs12110 create at: ${date} ${time}
 *
 * ${tags}
 *
 * since version1.0.0
 */
```
