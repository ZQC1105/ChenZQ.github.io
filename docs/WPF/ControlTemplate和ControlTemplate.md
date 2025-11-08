# ControlTemplate 与 DataTemplate 核心区别

在 WPF 中，这两类模板服务于不同的抽象层级：

| 特性         | **ControlTemplate**                              | **DataTemplate**                                              |
| :----------- | :----------------------------------------------- | :------------------------------------------------------------ |
| **目标对象** | **控件**（Button, TextBox 等）                   | **数据对象**（业务实体、ViewModel）                           |
| **核心职责** | 定义控件的视觉结构和交互行为                     | 定义数据的视觉呈现方式                                        |
| **典型应用** | 自定义按钮形状、重制滚动条样式、创建全新控件皮肤 | 定制列表项布局、内容呈现形式、集合数据显示                    |
| **使用属性** | `Control.Template`, `Style.Setter`               | `ItemsControl.ItemTemplate`, `ContentControl.ContentTemplate` |
| **设计思维** | **无外观的控件** - 控件逻辑与视觉分离            | **无外观的数据** - 数据与呈现分离                             |

## 本质差异

**ControlTemplate** 回答：**"控件应该长什么样？"**  
它重塑控件的整个视觉树（Visual Tree），包括背景、边框、状态效果等，但保留控件的核心行为逻辑。例如，将方形按钮变为圆形，不影响其点击命令。

**DataTemplate** 回答： **"数据应该如何可视化？"**  
它仅为数据对象生成呈现内容，通常作为控件内容的一部分。例如，将 `Person` 对象显示为"姓名+头像"的组合，而不改变承载它的 ListBoxItem 控件本身。

## 决策指南

- 需要**改变控件外观或交互结构** → 使用 **ControlTemplate**
- 需要**定义数据的展示方式** → 使用 **DataTemplate**
- 两者常配合使用：DataTemplate 定义数据呈现，ControlTemplate 定义承载数据的容器样式

**最佳实践**：优先使用 DataTemplate 呈现数据；仅在需要深度定制控件交互或视觉结构时才使用 ControlTemplate。