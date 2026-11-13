# 编辑器 XML 对齐：已做的事和还没做完的事

对照对象是本机安装的 FairyGUI Editor（Unity IL2CPP，`GameAssembly.dll` + `global-metadata.dat`，metadata version 29）。

IL2CPP 反编译拿不到完整 C# 方法体。这次用 Il2CppDumper 拿到类和方法地址，再反汇编 `FairyEditor.*$$Write` / `Serialize` / `Save`，记下 `XML.SetAttribute` 和 `XML.Create` 的调用顺序。属性顺序以这个为准，并用仓库里的 FairyGUI-Editor fixture 核对能同时出现的属性。

反汇编记录在本机 `D:/Project/fairygui-editor-il2cpp-dump/xml-write-order.txt`，不进仓库。

## 这次已经按编辑器顺序改过的写出

- 显示对象公共属性：`visible` → `alpha` → `rotation`；`blend` / `filter` / `filterData` 在 `tooltips` / `customData` 之前。
- 列表、文本、图片、图形、加载器、动画等类型字段放在几何属性之后，不再插到 `xy` / `size` 前面。
- 列表属性顺序对齐 `FList.Write`（例如 `scrollBarFlags` 在 `scrollBar` 前，`margin` 在 `lineGap` / `colGap` 前）。
- 列表项：`url` → `title` → `icon` → `name` → `selectedIcon` → `selectedTitle` → `controllers` → `level`。
- 文本：`autoSize` 在 `underline` / `bold` / `italic` / `strike` 之前。删除线写成编辑器的 `strike="true"`，仍然读 `strikethrough`。
- 输入文本：`keyboardType` 在 `password` 前。
- 加载器：`useResize` 在 `autoSize` 前，`playing` / `frame` 在 `color` 前。
- 动画：`frame` 在 `playing` 前。
- 图形：`corner` → `sides` → `startAngle` → `points` → `distances`。`distances` 编辑器这次没在 `FGraph.Write` 里出现，先留在 `points` 后面。`type` 写成编辑器的 `eclipse` 和 `regular_polygon`。
- 组件根节点属性按 `FComponent.Write_editMode`：`size` → `idnum` → `restrictSize` → `pivot` → `anchor` → `overflow` → `opaque` → 滚动相关 → `margin` → `clipSoftness` → `extention` → `customExtention` → `mask` → `hitTest` → `initName` → 设计图 → 背景色 → `remark` → `customData` → `pageController` → 声音。
- 组件子节点：控制器 → `displayList` → 扩展元素（`Button` 等）→ `relation` → `transition` → `customProperty`。
- 控制器：`exported` 等可选属性在 `pages` / `selected` 之前。
- 动效：`options` 在 `autoPlay` 之前。帧上的 `label` 在 `tween` 之前，`label2` 在 `duration` 之后、`ease` 之前。
- 齿轮：`condition` 在 `tween` 之前。
- `package.xml` 图片：`scale` → `scale9grid` → `gridTile` → `qualityOption` → `quality` → `smoothing` → `atlas` → `duplicatePadding` → `width` → `height`。
- 动画资源：`smoothing` 在 `atlas` 前。
- 发布设置：`packageCount` 在 `branchPath` 前；图集子节点是 `name` → `compression` → `index`。

## 还没对齐、以后再做

这些要么模型里没有字段，要么控制流分支太多，不能只靠指令顺序判定。

1. **`notes`**。`FObject.Write` 和 `FComponent.Write_editMode` 都会写 `notes`。当前模型没有这个字段，读写都没接。
2. **`locked` 的写入点**。它不在 `FObject.Write` 里，而在 `Document.SaveMeta` 和 `HiObject.Process`。编辑器自带工程里 `locked` 紧跟在 `size` 后面。现在仍然是在 `size` 之后写，但没有按 `HiObject.Process` 把整段保存后处理复刻出来。
3. **`idnum="0"`**。编辑器工程里有一个组件写了 `idnum="0"`，但字符串表里没有 `idnum`。现在仍然只有非 0 才写。
4. **字体资源多出来的属性**。`FontAsset.WriteSettings` 还有 `italicStyle`、`boldWeight`、`atlasPadding`。模型没有对应字段。
5. **图片资源 `disableTrim`**。`ImageAsset.WriteSettings` 会写。模型没有。
6. **扩展定义和实例是两条写出路径**。组件文件用 `Write_editMode`（按钮只有 `mode` / `sound` / `volume` / `downEffect` / `downEffectValue`，下拉框只有 `dropdown`）。放进别的组件里的实例用 `Write`（按钮有 `checked` / `title` / `icon` 等）。现在组件文件上的扩展节点仍把两边的字段写在一起，没有拆开。
7. **控制器动作的互斥分支**。`FControllerAction.Write` 把换页和播动效的属性按指令顺序排在一起：`type` → `fromPage` → `toPage` → `objectId` → `controller` → `targetPage` → `transition` → `repeat` → `delay` → `stopOnExit`。实际每一种动作只走其中一支。现在是按动作类型分开写，同一支里的相对顺序已经一致，没有再改。
8. **齿轮值的细节**。`FGearBase.Write` 还有 `customEase`、`duration`、`delay`、`positionsInPercent`，以及按齿轮类型分支的 `pages` / `values`。`FGearXY.WriteValue` 等值函数本身没有再调 `SetAttribute`，值的拼法在别的函数里。这次只把 `condition` 提到了 `tween` 前面。
9. **关系是否包一层 `relations`**。`FRelations.Write` 的指令里先 `Create("relations")` 再 `Create("relation")`。编辑器自带工程里看到的是直接的 `<relation>`，没有外层。在没有把这个函数的分支跟完之前，不改现在的 `<relation>` 写法。
10. **空白、自闭合标签、属性引号、默认值省略的逐字节**。编辑器 `XML.ToXMLString` 决定缩进和空节点怎么写。现有写出已经是两空格缩进和 `<?xml version="1.0" encoding="utf-8"?>`，但没有逐函数对默认值判断做字节级差异对比。
11. **工程描述和其他设置文件**。`FProject.Save` 只看到 `projectDescription` 的 `id` / `type` / `version`。`AdaptationSettings`、`CommonSettings`、`GlobalPublishSettings`、`I18nSettings`、`WorkspaceSettings` 这些是编辑器私有设置，不是工程 XML，这次没动。
12. **二进制发布**。`PublishHandler.WriteResourceItem` 是发布二进制，不是工程 XML。没有改发布格式。
