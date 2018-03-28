---
layout: post
category: "web"
title:  "DWZ结构学习和源代码研究"
tags: [dwz,javascript, jQuery]
---

### DWZ 是什么

DWZ富客户端框架(jQuery RIA framework)，是中国人自己开发的基于jQuery实现的Ajax RIA开源框架。 DWZ富客户端框架设计目标是简单实用、扩展方便、快速开发、RIA思路、轻量级。

DWZ框架支持用HTML扩展的方式来代替JavaScript代码，只要懂HTML语法，再参考DWZ使用手册就可以做Ajax开发。开发人员不写JavaScript的情况下，也能用Ajax做项目和使用各种UI组件。 基本可以保证程序员不懂JavaScript， 也能使用各种页面组件和Ajax技术。 如果有特定需求也可以扩展DWZ做定制化开发。

做Ajax项目时需要写大量的JavaScript才能达到满意的效果，国内很多程序员javascript不熟， 大大影响了开发速度。使用DWZ框架自动绑定JavaScript效果， 不需要开发人员去关心JavaScript怎么写，只要写标准HTML就可以了。DWZ简单扩展了HTML标准，给HTML定义了一些特别的class和attribute。DWZ框架会找到当前请求结果中的那些特别的class和attribute, 并自动关联上相应的js处理事件和效果。

DWZ基于jQuery，可以非常方便的定制特定需求的UI组件，并以jQuery插件的形式发布出来，如有需要也可做定制化开发。

[***官方网站***](http://jui.org/)
 
我总结的：<font color="red">一个JavaScript框架，不像一个类库，不是用于扩展javascript的功能(如jQuery)， 而像扩展HTML功能的。</font>

### DWZ 特点与优点

第一次打开页面时载入界面到客户端, 之后和服务器的交互只是数据交互, 不占用界面相关的网络流量.

[***特点演示***](http://www.5admo.cn)

支持HTML扩展方式来调用DWZ组件. 
标准化Ajax开发, 降低Ajax开发成本.

* 完全开源，源码没有做任何混淆处理，方便扩展。
* CSS和JS代码彻底分离，修改样式方便。
* 简单实用，扩展方便，轻量级框架，快速开发。
* 仍然保留了HTML的页面布局方式。
* 支持HTML扩展方式调用UI组件，开发人员不需写JS。
* 只要懂HTML语法不需精通JS，就可以使用Ajax开发后台。
* 基于jQuery，UI组件以jQuery插件的形式发布，扩展方便。

### 扩展的"HTML标签"

[***使用手册***](https://gitee.com/dwzteam/dwz_jui/raw/master/doc/dwz-user-guide.pdf)


### 扩展的标签是如何实现的

DWZ的初始化

```
DWZ.init("dwz.frag.xml", {})
initEnv() [async call]
{
	initLayout()
	initUI()
} [timer call] (我不喜欢定时器)
```

[***源代码参考***](http://www.5admo.cn/public/dwz/js/dwz.core.js)

initUI分析(扩展标签的实现)及其完成的主要任务

1. 处理各种控件，包括 tabs, jTree, checkbox, xheditor, uploadify等。
2. 增加控件数据
3. 绑定控件风格, 与css连接
4. 添加控件事件处理函数
5. 调用控件的初化函数

下面摘出绑定控件风格部分
```
	// init styles
	$("input[type=text], input[type=password], textarea", $p).addClass("textInput").focusClass("focus");

	$("input[readonly], textarea[readonly]", $p).addClass("readonly");
	$("input[disabled=true], textarea[disabled=true]", $p).addClass("disabled");

	$("input[type=text]", $p).not("div.tabs input[type=text]", $p).filter("[alt]").inputAlert();

	//Grid ToolBar
	$("div.panelBar li, div.panelBar", $p).hoverClass("hover");

	//Button
	$("div.button", $p).hoverClass("buttonHover");
	$("div.buttonActive", $p).hoverClass("buttonActiveHover");
	
	//tabsPageHeader
	$("div.tabsHeader li, div.tabsPageHeader li, div.accordionHeader, div.accordion", $p).hoverClass("hover");
```

[***源代码参考***](http://www.5admo.cn/public/dwz/js/dwz.ui.js)

### dwz.frag.xml 配置文件

从上面可以看到DWZ.init到initEnv是异步调用，原因时需要向服务器请求dwz.frag.xml文件， 此文件的内容主要是公共部分的html代码, 比如dialog, loading, pagintion等。

[***dwz.frag.xml***](http://www.5admo.cn/dwz.frag.xml)

### 表单的验证与提交（以增加角色为例演示）

[***效果演示***](http://www.5admo.cn)

```

<div class="pageContent">
	<form method="post" action="<?php echo Yii::app()->createUrl("admin/role/Add", array('navTabId'=>Yii::app()->request->getQuery('navTabId')))?>" class="pageForm required-validate" onsubmit="return validateCallback(this, dialogAjaxDone)">
		<div class="pageFormContent" layoutH="56">
			<p>
				<label>角色名：</label>
                                <input name="role[role_name]" type="text" class="required" size="30" value="" title="必须填写角色名"/>
			</p>
			<p>
				<label>描述：</label>
				<input name="role[role_desc]" type="text" size="30" value="" alt=""/>
			</p>
		</div>
		<div class="formBar">
			<ul>
				<li><div class="buttonActive"><div class="buttonContent"><button type="submit">保存</button></div></div></li>
				<li>
					<div class="button"><div class="buttonContent"><button type="button" class="close">取消</button></div></div>
				</li>
			</ul>
		</div>
	</form>
</div>

```

说明：
1. form 增加 class="pageForm required-validate" onsubmit="return validateCallback(this, dialogAjaxDone)"
2. 必填的文本输入框增加 class="required" title="错误提示" 即可。
3. validateCallback是实现对设定规则的验证，验证失败不提交表单。
4. dialogAjaxDone 是表单提交后响应器响应处理函数。
5. 一般 onsubmit的是固定的，除非dwz实现的规则无法满足您的业务验证。

源代码参考：

[***validateCallback***](http://www.5admo.cn/public/dwz/js/dwz.ajax.js)

[***form.valid***](http://www.5admo.cn/public/jquery/jquery.validate.js)

[***dialogAjaxDone***](http://www.5admo.cn/public/dwz/js/dwz.ajax.js)


### a标签ajaxTodo扩展(删除角色为例)

[***效果演示***](http://www.5admo.cn)

```
<a href="<?php echo Yii::app()->createUrl("admin/role/delete", array('roleid'=>$role->role_id))?>" target="ajaxTodo" title="确定要删除角色吗?">删除角色</a>
```

说明：
1. ajaxTodo扩展将弹出一个选择框，用户选择“是”后进行操作。
2. a标签增加 target="ajaxTodo" title="对话框提示内容"

[***源代码参考***](http://www.5admo.cn/public/dwz/js/dwz.ajax.js)


### layoutH, 容器高度自适应

[***效果演示***](http://www.5admo.cn)

容器高度自适应, 只要增加扩展属性layoutH=”xx”, 单位是像素.
LayoutH表示容器内工具栏高度.  浏览器窗口大小改变时容器高度自适应, 但容器内工具栏高度是固定的, 需要告诉js工具栏高度来计算出内容的高度.

示例:

```
<div class=”layoutBox”>
	<div layoutH=“150”>内容</div>
</div>
```
注意： layoutH=“150”的高度是根据含有class=”layoutBox”的父容器div动态更新的

源代码参考：

[***initLayout***](http://www.5admo.cn/public/dwz/js/dwz.ui.js)

[***layoutH***](http://www.5admo.cn/public/dwz/js/dwz.core.js)













