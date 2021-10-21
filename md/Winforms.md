WinForms: 基于Win32 API的C#封装



C# 

windows窗体应用

### 一:第一个窗口

#### 1可视化操作

可视化设计:

1. 双击Form1.cs打开界面设计器
2. 打开显示工具箱
3. 拖一个按钮到界面上

#### 2项目结构

App.config 应用配置

Form1.cs源码文件(窗口)

​	Form1.Designer.cs 源码文件(界面设计)

​	Form1.resx资源文件

Program.cs源码文件(Main)方法

Form1.cs两种打开方式

1. 双击则打开界面设计器

2. 右键,查看代码,打开的是源码编辑器

   两者结合使用

#### 3手工创建窗口

Form,窗口

可以手工创建一个窗口类

```csharp
class MyForm : Form
{
    
}
```

运行程序并以MyForm为主窗口

```csharp
[STAThread]
static void Main()
{
    MyForm form = new MyForm();
    Application.Run(form);qt
}
```

要点与细节

1. [STAThread]

   相当于Python里的装饰器

2. Application.Run()

   启动消息循环



### 二.添加控件

1. 打开显示工具箱
2. 从工具箱中拖一个Button到窗口中
3. 运行程序

界面设计器操作的时候,比如拖一个button到界面设计器。这样就会自动生成一段代码

Form1.cs 业务代码

Form1.Designer.cs 界面代码自动生成

Form1.Designer.cs一般不手工修改。

要点与细节:

重点是它代码的调用关系

Form1()->InitializeComponent()

