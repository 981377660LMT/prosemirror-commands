好的，我们来深入、详细地讲解 prosemirror-commands 这个模块。

prosemirror-commands 是 ProseMirror 编辑器中负责**“行为”**的核心模块之一。它提供了一系列预设的、可直接使用的编辑命令 (`Command`)。这些命令封装了常见的用户编辑意图，例如“删除选区”、“将段落变成标题”、“在光标处分割段落”等。

这个模块的设计哲学是提供一系列独立的、可组合的构建块。你可以单独使用它们，也可以使用 `chainCommands` 将它们串联起来，以实现更复杂的、有先后顺序的逻辑（例如，按 `Backspace` 键时，优先尝试删除选区，如果选区为空，再尝试向前合并块）。

---

### 宏观逻辑：从用户意图到 `Transaction`

每个导出的命令都是一个函数，遵循 `(state, dispatch, view) => boolean` 的签名：

- `state: EditorState`: 当前的编辑器状态。命令会根据这个状态来判断自己是否适用。
- `dispatch?: (tr: Transaction) => void`: 一个可选的回调函数。如果提供了 `dispatch`，命令在执行成功后会调用它，传入一个描述了变化的 `Transaction`。如果**没有**提供 `dispatch`，命令只进行“空运行 (dry run)”，即只判断自己是否适用，而不产生任何实际效果。这是实现“动态禁用/启用菜单项”等 UI 功能的关键。
- `view?: EditorView`: 可选的编辑器视图实例，某些命令需要它来获取精确的 DOM 布局信息（例如，判断光标是否真的在文本块的视觉开头）。
- **返回值 `boolean`**: 如果命令成功执行（或在空运行时发现自己适用），则返回 `true`；否则返回 `false`。

---

### 核心组件与代码阅读顺序

我们将按照功能类别来组织和讲解这些命令，从最基础的到最复杂的。

#### 第 1 站：基础命令与核心工具

这些是构建其他命令的基础。

- **`chainCommands(...commands)`**: `commands.ts:686`

  - 这是最重要的工具函数之一。它接收任意数量的命令函数，并返回一个新的命令函数。
  - 当这个新命令被调用时，它会**按顺序**依次尝试执行传入的每一个命令。
  - 一旦其中某个命令返回 `true`，链式调用就会立即停止，并返回 `true`。
  - 如果所有命令都返回 `false`，它最终也返回 `false`。
  - 这完美地实现了“后备行为 (fallback behavior)”模式。

- **`deleteSelection`**: `commands.ts:8`

  - 最基础的命令之一。
  - 它检查 `state.selection.empty`。如果选区不为空，它就创建一个删除选区的 `Transaction` (`tr.deleteSelection()`) 并派发。
  - 这是许多按键（如 `Backspace`, `Delete`）处理链的第一个环节。

- **`selectAll`**: `commands.ts:415`

  - 创建一个 `AllSelection` 来选中整个文档。

- **`selectParentNode`**: `commands.ts:401`
  - 将选区扩展到包裹当前选区的父节点。例如，如果光标在段落中，执行此命令会选中整个段落。

#### 第 2 站：处理块级节点连接与分离的命令 (Joining & Lifting)

这是 prosemirror-commands 中最复杂也最精巧的部分，主要用于处理 `Backspace` 和 `Delete` 在块级节点边界的行为。

- **`joinBackward`**: `commands.ts:24`

  - 模拟在块的开头按 `Backspace` 的行为。
  - **核心逻辑**:
    1.  使用 `atBlockStart` 检查光标是否确实在文本块的开头。
    2.  使用 `findCutBefore` 寻找光标前的“可连接点”。
    3.  **场景 A：可以连接 (`$cut` 存在)**: 调用 `deleteBarrier`，这是一个非常复杂的辅助函数，它会尝试用 `tr.join()` 来合并当前块和前一个块。如果直接 `join` 不行（例如，`p` 和 `ul`），它会尝试更复杂的结构变换，比如将 `p` 的内容移动到 `ul` 的最后一个 `li` 中。
    4.  **场景 B：无法连接 (`$cut` 不存在)**: 这通常意味着当前块是其父节点的第一个子节点。此时，它会尝试执行 `lift` 操作，即把当前块从其父节点中“提升”出来（例如，将列表中的第一项变成一个普通段落）。

- **`joinForward`**: `commands.ts:168`

  - 模拟在块的末尾按 `Delete` 的行为。
  - 逻辑与 `joinBackward` 类似，但方向相反，使用 `atBlockEnd` 和 `findCutAfter`。

- **`lift`**: `commands.ts:275`

  - 将选中的块或其最近的可提升祖先块从其父节点中提升出来。
  - 它使用 prosemirror-transform 中的 `liftTarget` 来计算可以提升到的目标深度，然后调用 `tr.lift()`。

- **`liftEmptyBlock`**: `commands.ts:338`
  - 如果光标在一个空的文本块中，尝试提升这个块。这是处理在空列表项中按回车以“跳出”列表等场景的关键。

#### 第 3 站：创建与分割块级节点的命令

这些命令主要与 `Enter` 键的行为相关。

- **`splitBlock`**: `commands.ts:362`

  - 在光标位置分割当前的块。
  - 如果选区不为空，它会先删除选区内容，然后再分割。
  - 它内部调用了 `tr.split()`。

- **`splitBlockKeepMarks`**: `commands.ts:365`

  - `splitBlock` 的一个变体。默认情况下，`splitBlock` 后新块中的光标会清除所有激活的标记（stored marks）。这个命令会保留这些标记，使得在新行中输入的文本继续保持之前的样式（如加粗）。

- **`createParagraphNear`**: `commands.ts:320`

  - 当一个节点（如 `horizontal_rule` 或图片）被选中时，按 `Enter` 键应该做什么？这个命令解决了这个问题：它会在被选中的节点之前或之后创建一个新的空段落，并将光标移入其中。

- **`exitCode`**: `commands.ts:300`
  - 如果光标在代码块的末尾，执行此命令会创建一个新的默认块（通常是段落）并跳出代码块。

#### 第 4 站：参数化命令 (Parameterized Commands)

这些是高阶函数，它们接收一些参数（如节点类型或标记类型）并返回一个具体的命令函数。这使得它们非常灵活，可以用于构建自定义的菜单和工具栏。

- **`toggleMark(markType, attrs)`**: `commands.ts:575`

  - 这是最常用的标记命令，用于切换加粗、斜体等。
  - **核心逻辑**:
    1.  如果选区为空，它会切换“存储标记”(`storedMarks`)，影响下一次的输入。
    2.  如果选区不为空，它会检查选区内的文本是否**已经**应用了该标记。
    3.  如果选区内**部分或全部**文本已有该标记，它会移除该标记 (`tr.removeMark`)。
    4.  如果选区内**完全没有**该标记，它会添加该标记 (`tr.addMark`)。
    5.  它很智能，会忽略选区两端的空白字符，避免给纯空格添加标记。

- **`wrapIn(nodeType, attrs)`**: `commands.ts:519`

  - 将当前选中的块包裹在一个新的父节点中。例如，将一个段落包裹成一个 `blockquote`。
  - 它使用 prosemirror-transform 的 `findWrapping` 来检查这个包裹操作是否合法，如果合法，则调用 `tr.wrap()`。

- **`setBlockType(nodeType, attrs)`**: `commands.ts:532`
  - 将选中的所有文本块的类型更改为指定的 `nodeType`。例如，将一个段落变成一个 `h1` 标题。

#### 第 5 站：预设的按键映射 (`baseKeymap`)

模块的最后部分定义了几个预设的快捷键映射表，这是开箱即用功能的核心。

- **`pcBaseKeymap`** 和 **`macBaseKeymap`**: `commands.ts:703`, `commands.ts:720`

  - 这两个对象定义了在 PC 和 Mac 平台上的基础快捷键绑定。
  - 它们大量使用了 `chainCommands`。例如，`Backspace` 键被绑定到 `chainCommands(deleteSelection, joinBackward, selectNodeBackward)`。这意味着当用户按 `Backspace` 时，ProseMirror 会：
    1.  首先尝试 `deleteSelection`。如果选区不为空，删除它并停止。
    2.  如果上一步失败（选区为空），则尝试 `joinBackward`。如果光标在块的开头，执行合并或提升操作并停止。
    3.  如果上一步也失败，则尝试 `selectNodeBackward`，即选中光标前的原子节点（如图片）。

- **`baseKeymap`**: `commands.ts:736`
  - 这是一个非常简单的逻辑：它检测当前运行的平台，然后导出 `pcBaseKeymap` 或 `macBaseKeymap` 作为 `baseKeymap`。
  - 你可以直接将这个 `baseKeymap` 对象传入 prosemirror-keymap 的 `keymap()` 函数中，以获得一套完整的、符合用户习惯的基础编辑体验。

### 总结

prosemirror-commands 是一个展示 ProseMirror 设计思想的绝佳范例：

- **原子化与可组合性**: 将复杂的编辑行为分解成独立的、可测试的命令，然后通过 `chainCommands` 灵活地组合它们。
- **状态驱动**: 所有命令都严格地基于当前的 `EditorState` 来判断自身行为，并产生一个描述变化的 `Transaction`，而不是直接操作视图。
- **分层与抽象**: 提供了从低级（如 `joinBackward`）到高级（如 `toggleMark`）不同抽象层次的命令，满足不同场景的需求。
- **开箱即用**: 通过 `baseKeymap` 提供了一套合理的默认配置，大大降低了上手门槛。

通过深入理解这个模块，你不仅能学会如何使用这些命令，更能掌握如何按照 ProseMirror 的模式去思考和构建你自己的、更复杂的编辑行为。
