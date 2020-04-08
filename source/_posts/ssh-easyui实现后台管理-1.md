---
title: ssh-easyui实现后台管理(1)
date: 2019-07-10 08:53:24
tags:
 - Java
 - SSH
 - EasyUI
categories: 
 - Java
 - SSH
---
# <center>easy-ui+ssh</center>

使用easy-ui作为前台界面，后台用的是ssh框架，做的一个课程设计，实现一个后台管理系统。使用easy-ui的datagrid，通过ajax实现CURD操作。<!--more-->

## 前台页面 reception.jsp

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
    <%@taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>前台</title>
<link rel="stylesheet" type="text/css" href="${pageContext.request.contextPath}/themes/bootstrap/easyui.css">
	<link rel="stylesheet" type="text/css" href="${pageContext.request.contextPath}/themes/icon.css">
	<script type="text/javascript" src="${pageContext.request.contextPath}/js/jquery.min.js"></script>
	<script type="text/javascript" src="${pageContext.request.contextPath}/js/jquery.easyui.min.js"></script>
		<style type="text/css">
		.menuA{
			text-decoration :none;
		}
	</style>
	
	<script type="text/javascript">
		$(function(){
			$(".menuA").click(function(){
				var contentText = this.innerHTML;
				var url = this.href;
				//alert(url);
				createTab(contentText,url);
				// 超链接就不跳转
				return false;
			});
		});
		
		function createTab(contentText,url){
			// 判断选项卡是否存在:
			var flag = $("#tt").tabs("exists",contentText);
			if(flag){
				// 如果已经存在，让其被选中
				$("#tt").tabs("select",contentText);
			}else{
				// 如果不存在,创建新的选项卡
				$('#tt').tabs('add',{    
				    title:contentText,    
				    content:createUrl(url),    
				    closable:true   
				}); 
			} 
		}
		
		function createUrl(url){
			return "<iframe src='"+url+"' style='width:100%;height:95%;border:none;'></iframe>";
		}
	</script>
</head>
<body>
<div id="cc" class="easyui-layout" data-options="fit:true">   
    <div data-options="region:'north',title:'商场后台管理',split:true" style="height:100px;"></div>   
    <div data-options="region:'west',title:'系统菜单',split:true" style="width:200px;">
    	<div id="aa" class="easyui-accordion" data-options="fit:true">   		   
		    <div title="订单管理" data-options="iconCls:'icon-man'" style="overflow:auto;padding:10px;">   
		    	<a href="${pageContext.request.contextPath}/order.html" class="menuA">订单列表</a>
		    </div>   
		    <div title="员工管理" data-options="iconCls:'icon-man'" style="overflow:auto;padding:10px;">   
		    	<a href="${pageContext.request.contextPath}/emp.html" class="menuA">员工列表</a>
		    </div>    
		    <div title="商品管理" data-options="iconCls:'icon-man'" style="overflow:auto;padding:10px;">   
		    	<a href="${pageContext.request.contextPath}/good.html" class="menuA">商品列表</a>
		    </div>  
		</div>  
    </div>   
    <div data-options="region:'center',title:''" style="padding:5px;background:#eee;">
    	<div id="tt" class="easyui-tabs" data-options="fit:true">   
		    <div title="数据显示" style="padding:20px;display:none;">   
		        <h1>欢迎光临</h1>
		    </div>   
		</div> 
    </div>   
</div>  
</body>
</html>
```

## 数据展示页面 emp.html

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
<link rel="stylesheet" type="text/css" href="./themes/bootstrap/easyui.css">
	<link rel="stylesheet" type="text/css" href="./themes/icon.css">
	<script type="text/javascript" src="./js/jquery.min.js"></script>
	<script type="text/javascript" src="./js/jquery.easyui.min.js"></script>
	<script type="text/javascript" src="./locale/easyui-lang-zh_CN.js"></script>
<script type="text/javascript">
	$(function(){
		$('#dg').datagrid({    
		    url:'emp_findByPage.action',    
		    columns:[[    
		        {field:'emp_name',title:'员工姓名',width:200},    
		        {field:'emp_sex',title:'员工性别',width:200},    
		        {field:'emp_age',title:'员工年龄',width:200} ,   
		        {field:'emp_phone',title:'手机号码',width:200} ,   
		        {field:'emp_dep',title:'职位',width:200} , 
		    ]],
		    fitColumns:false,
		 // 显示分页工具条
	        pagination:true,
	        // 初始化的页数
	        pageNumber:1,
	        // 每页显示记录数:
	        pageSize:3,
	        // 分页工具条中下拉列表中的值：
	        pageList:[3,5,10],
	        // 隔行换色
	        striped:true,
	        singleSelect:true,
	        toolbar: [{
	    		iconCls: 'icon-add',
	    		handler: function(){
	    			$("#winAdd").window("open");
	    			}
	    		},
	    		{
	    		iconCls: 'icon-remove',
	    		handler: function(){
	    			deleteById();
	    			}
	    		},
	    		{
		    		iconCls: 'icon-edit',
		    		handler: function(){
		    			editById()
		    	}
	    	}],

		});  
	})
	
	function save(){
		$('#formAdd').form({    
		    url:"emp_save.action",    
		    success:function(data){    
		       var jsonData = eval("("+data+")");
		       $.messager.show({
		    		title:'提示消息',
		    		msg:jsonData.msg,
		    		timeout:3000,
		    		showType:'slide'
		    	});
		    	// 关闭窗口:
		    	$("#winAdd").window("close");
		    	// 重新加载数据:
		    	$("#dg").datagrid("reload");
		    }    
		});    
		// submit the form    
		$('#formAdd').submit();  
	}
	function update(){
		$('#formUpdate').form({    
		    url:"emp_update.action",    
		    success:function(data){    
		       var jsonData = eval("("+data+")");
		       $.messager.show({
		    		title:'提示消息',
		    		msg:jsonData.msg,
		    		timeout:3000,
		    		showType:'slide'
		    	});
		    	// 关闭窗口:
		    	$("#winUpdate").window("close");
		    	// 重新加载数据:
		    	$("#dg").datagrid("reload");
		    }    
		});    
		// submit the form    
		$('#formUpdate').submit();  
	}
	function deleteById(){
		var row = $('#dg').datagrid('getSelected');
		if(row){
			//alert(row.cust_id);
			$.messager.confirm('确认','确认要删除：'+row.emp_name+'吗？',function(flag){
				if(flag){
					$.post("emp_delete.action",{"emp_id":row.emp_id},function(data){
						$.messager.show({
				    		title:'提示消息',
				    		msg:data.msg,
				    		timeout:5000,
				    		showType:'slide'
				    	});
						$("#dg").datagrid("reload");
					},"json");
				}				
			});			
		}
		else{
			alert("dont find");
		}
	}
	function editById(){
		var row = $('#dg').datagrid('getSelected');
		if(row){
			$("#winUpdate").window("open");
			$('#formUpdate').form('load',"emp_findById.action?emp_id="+row.emp_id+"");
		}
		else{
			alert("没有选中列！");
		}
	}
</script>
</head>
<body>
<table id="dg"></table>  

	<!-- 添加客户的表单，默认是隐藏的 -->
	<div id="winAdd" class="easyui-window" title="添加" style="width:440px;height:200px"   
	        data-options="iconCls:'icon-save',modal:true,closed:true">   
		    
	        <form id="formAdd" method="post">
				 <table style="margin:auto">
				 	<tr>
				 		<td>员工姓名：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_name" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>员工性别：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_sex" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>员工年龄：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_age" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>手机号码：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_phone" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>职位：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_dep" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>				 		
				 		<td>
							<button class="easyui-linkbutton" type="button" onclick="save()">保存</button>
						</td>
				 		</td>
				 	</tr>
				 </table>
			</form>
		</div>
	<div id="winUpdate" class="easyui-window" title="修改" style="width:440px;height:200px"   
	        data-options="iconCls:'icon-save',modal:true,closed:true">   		    
	        <form id="formUpdate" method="post">
	        	<input type="hidden" name="emp_id"/>
				<table style="margin:auto">
				 	<tr>
				 		<td>员工姓名：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_name" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>员工性别：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_sex" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>员工年龄：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_age" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>手机号码：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_phone" style="width:200px">
				 		</td>
				 	</tr>
				 	<tr>
				 		<td>职位：</td>
				 		<td>
				 			<input class="easyui-textbox" name="emp_dep" style="width:200px">
				 		</td>
				 	</tr>
					<tr>
						<td rowspan=2>
							<button class="easyui-linkbutton" type="button" onclick="update()">保存</button>
						</td>
					</tr>
				</table>   
			</form>
		</div>
</body>
</html>
```

## 效果图

{% asset_img easy1.png %}

{% asset_img easy2.png %}