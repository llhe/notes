MVC, MVP, MVVM
===============

MVC
------
Model - View - Controller  
* User `Sees` View
* User `Uses` Controller
* Controller `Manipulates` Model
* Model `Updates` View

一般情况下，对于web框架，MV通过模版语言实现(如struts模版)，VC通过配置文件实现(如struts配置文件)，CM之间一般是代码调用关系

MVP
------
介于MVC和MVVM之间的一种设计模式：MV之间的交互完全放到P中，从而M实现与V的完全解耦

MVVM
------
更适合一个Model对应多种View的情况，如Web，Mobile，API等不同表现形式
1. Model
  biz model，不同于MVC，可以实现与UI完全解耦
2. View
  UI representation, e.g. xaml
3. ViewModel
  跟UI显示直接关联的对象，包含数据的映射和UI命令，底层可调用Model