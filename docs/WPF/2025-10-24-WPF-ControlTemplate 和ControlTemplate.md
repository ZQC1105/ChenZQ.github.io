
  ## DataTemplate和ControlTemplate

WPF 中 ControlTemplate 与 DataTemplate 的区别。

下面为您详细解释这两者的核心区别：

特性 ControlTemplate DataTemplate
:--- :--- :---
作用对象 控件（Control）本身 数据对象（Data Object）
核心目的 定义控件的外观和视觉结构 定义数据对象的显示方式
应用场景 改变按钮的形状、重定义文本框的边框、自定义滚动条样式等 显示 ListBox、ComboBox 中的列表项内容、在 ContentPresenter 中展示数据等
使用方式 通常通过 Template 属性或 ControlTemplate 资源来设置 通过 ItemTemplate、ContentTemplate 或 DataTemplate 资源来设置
关注点 “这个控件看起来应该是什么样？” “这段数据应该如何被呈现？”

简单来说：

ControlTemplate 是关于控件的皮肤。它决定了一个 Button 是圆角还是方角，TextBox 的边框是什么颜色，或者 ComboBox 的下拉箭头如何绘制。
DataTemplate 是关于数据的展示。它决定了一个包含“姓名”和“年龄”的 Person 对象在 ListBox 里是显示为“张三 (25 岁)”还是用一个包含头像和信息的复杂布局。

总结：

如果您想改变一个 WPF 控件的外观和行为结构，请使用 ControlTemplate。
如果您想定义一段数据如何被可视化，请使用 DataTemplate。
