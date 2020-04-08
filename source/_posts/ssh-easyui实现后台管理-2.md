---
title: ssh-easyui实现后台管理(2)
date: 2019-07-10 09:50:46
tags:
 - Java
 - SSH
 - EasyUI
categories: 
 - Java
 - SSH
---
# <center>SSH后台实现</center><!--more-->

## Hibernate和Mybatis的区别

这个课程设计使用的是Spirng+Strust2+Hibernate实现的，因为只用到了简单的增删该查功能，没有什么复杂查询，所以使用的Hibernate。

Hiberate和MyBatis的增、删、查、改.对于业务逻辑层来说大同小异，对于映射层而言Hibemate的配置不需要接口和SQL.相反MyBatis是需要的。对于Hibermate而言，不需要编写大量的SQL,就可以完全映射，同时提供了日志、缓存、级联(级联比MyBatis强大)等特性，此外还提供HQL (Hibemate Query Language)对POIO进行操作，使用十分方便，但是它也有致命的缺陷。

MyBatis 可以自由书写SQL、支持动态SQL、处理列表、动态生成表名，支持存储过程。这样就可以灵活地定义查询语句，满足各类需求和性能优化的需要，这些在互联网系统中是十分重要的。

## 后台的实现

### 实体和映射

Employee.java

```java
package com.entity;

public class Employee {
	private Long emp_id;
	private String emp_name;
	private String emp_sex;
	private Integer emp_age;
	private String emp_phone;
	private String emp_dep;
	//getter and setter
}

```

 Employee.hbm.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-mapping PUBLIC
	"-//Hibernate/Hibernate Mapping DTD 3.0//EN"
	"http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
<hibernate-mapping package="com.entity">
	<class name="Employee" table="employees">
		<id name="emp_id">
			<generator class="native"></generator>
		</id>
		<property name="emp_name"></property>
		<property name="emp_sex"></property>
		<property name="emp_age"></property>
		<property name="emp_phone"></property>
		<property name="emp_dep"></property>
	</class>
</hibernate-mapping>
```

分页实体 PageBean.java

```java
package com.entity;

import java.util.List;

public class PageBean<T> {
	private List<T> list;
	private Integer currPage;
	private Integer totalCount;
	private Integer totalPage;
	private Integer pageSize;
	//getter and setter
	
}

```



### DAO层

EmployeeDao.java

```java
package com.dao;

import java.util.List;

import org.hibernate.criterion.DetachedCriteria;

import com.entity.Employee;

public interface EmployeeDao {
	List<Employee> findByPage(DetachedCriteria detachedCriteria, Integer begin, Integer rows);

	Integer findCount(DetachedCriteria detachedCriteria);

	void save(Employee Employee);

	Employee findById(Long Employee_id);

	void delete(Employee Employee);

	void update(Employee Employee);
}

```

EmployeeDaoImpl.java

```java
package com.dao.impl;

import java.util.List;

import org.hibernate.criterion.DetachedCriteria;
import org.hibernate.criterion.Projections;
import org.springframework.orm.hibernate5.support.HibernateDaoSupport;

import com.dao.EmployeeDao;
import com.entity.Employee;

public class EmployeeDaoImpl extends HibernateDaoSupport implements EmployeeDao {

	@Override
	public List<Employee> findByPage(DetachedCriteria detachedCriteria, Integer begin, Integer rows) {
		// TODO Auto-generated method stub
		detachedCriteria.setProjection(null);
		return (List<Employee>) this.getHibernateTemplate().findByCriteria(detachedCriteria,begin,rows);
	}

	@Override
	public Integer findCount(DetachedCriteria detachedCriteria) {
		// TODO Auto-generated method stub
		detachedCriteria.setProjection(Projections.rowCount());
		List<Long> list = (List<Long>) this.getHibernateTemplate().findByCriteria(detachedCriteria);
		if(list.size() > 0){
			return list.get(0).intValue();
		}
		return null;
	}

	@Override
	public void save(Employee Employee) {
		// TODO Auto-generated method stub
		this.getHibernateTemplate().save(Employee);
	}

	@Override
	public Employee findById(Long Employee_id) {
		// TODO Auto-generated method stub
		return this.getHibernateTemplate().get(Employee.class, Employee_id);
	}

	@Override
	public void delete(Employee Employee) {
		// TODO Auto-generated method stub
		this.getHibernateTemplate().delete(Employee);
	}

	@Override
	public void update(Employee Employee) {
		// TODO Auto-generated method stub
		this.getHibernateTemplate().update(Employee);
	}
}

```

### Service层

EmployeeService.java

```java
package com.service;

import org.hibernate.criterion.DetachedCriteria;

import com.entity.Employee;
import com.entity.PageBean;

public interface EmployeeService {
	PageBean<Employee> findByPage(DetachedCriteria detachedCriteria, Integer page, Integer rows);
	
	void save(Employee e);

	Employee findById(Long Employee_id);

	void delete(Employee e);

	void update(Employee e);
}

```

EmployeeServiceImpl.java

```java
package com.service.impl;

import java.util.List;

import org.hibernate.criterion.DetachedCriteria;

import com.dao.EmployeeDao;
import com.entity.Employee;
import com.entity.Good;
import com.entity.PageBean;
import com.service.EmployeeService;

public class EmployeeServiceImpl implements EmployeeService {
	private EmployeeDao employeeDao;
	
	public void setEmployeeDao(EmployeeDao employeeDao) {
		this.employeeDao = employeeDao;
	}

	@Override
	public PageBean<Employee> findByPage(DetachedCriteria detachedCriteria, Integer page, Integer rows) {
		// TODO Auto-generated method stub
		PageBean<Employee> pageBean = new PageBean<Employee>();
		// 设置当前页数:
		pageBean.setCurrPage(page);		
		pageBean.setPageSize(rows);		
		Integer totalCount = employeeDao.findCount(detachedCriteria);
		pageBean.setTotalCount(totalCount);
		// 设置总页数：
		double tc = totalCount;
		Double num = Math.ceil(tc /rows);
		pageBean.setTotalPage(num.intValue());
		// 设置每页显示数据的集合:
		Integer begin = (page - 1) * rows;
		List<Employee> list = employeeDao.findByPage(detachedCriteria,begin,rows);
		pageBean.setList(list);
		return pageBean;
	}
	@Override
	public void save(Employee e) {
		employeeDao.save(e);
	}
	@Override
	public Employee findById(Long Employee_id) {
		return employeeDao.findById(Employee_id);
	}
	@Override
	public void delete(Employee e) {	
		employeeDao.delete(e);
	}
	@Override
	public void update(Employee e) {
		employeeDao.update(e);
	}
}

```

### Spring配置文件

applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
		xmlns="http://www.springframework.org/schema/beans" 
		xmlns:context="http://www.springframework.org/schema/context" 
		xmlns:aop="http://www.springframework.org/schema/aop" 
		xmlns:tx="http://www.springframework.org/schema/tx" 
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.2.xsd 
							http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.2.xsd 
							http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.2.xsd 
							http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.2.xsd ">

	<bean name="employeeAction" class="com.web.action.EmployeeAction" scope="prototype">
		<property name="employeeService" ref="employeeService"></property>
	</bean>
	<bean name="employeeDao" class="com.dao.impl.EmployeeDaoImpl">
		<property name="sessionFactory" ref="sessionFactory"></property>
	</bean>
	<bean name="employeeService" class="com.service.impl.EmployeeServiceImpl">
		<property name="employeeDao" ref="employeeDao"></property>
	</bean>
	
	<!-- 事务核心管理器，封装了所有事物操作，使用HibernateTransactionManager -->
	<bean name="transactionManager" class="org.springframework.orm.hibernate5.HibernateTransactionManager">
		<property name="sessionFactory" ref="sessionFactory"></property>
	</bean>
	<!-- 配置事务通知 -->
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="save*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="false" />
			<tx:method name="persist*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="false" />
			<tx:method name="update*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="false" />
			<tx:method name="modify*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="false" />
			<tx:method name="delete*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="false" />
			<tx:method name="remove*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="false" />
			<tx:method name="get*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="true" />
			<tx:method name="find*" isolation="REPEATABLE_READ" propagation="REQUIRED" read-only="true" />
		</tx:attributes>
	</tx:advice>
	<!-- 配置织入 -->
	<aop:config>
		<aop:pointcut expression="execution(* com.service.impl.*ServiceImpl.*(..))" id="txPc"/>
		<aop:advisor advice-ref="txAdvice" pointcut-ref="txPc"/>
	</aop:config>
	
	<!-- 将sessionFactory配置到spring容器 -->
	<bean name="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
		<property name="dataSource" ref="dataSource"></property>
		<property name="hibernateProperties">
			<props>				
				<prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>				
				<prop key="hibernate.show_sql">true</prop>
				<prop key="hibernate.format_sql">true</prop>
				<prop key="hibernate.hbm2ddl.auto">update</prop>
			</props>		
		</property>
		<!-- 读取hibernate mapping 文件夹 -->
		<property name="mappingDirectoryLocations" value="classpath:com/entity"></property>
	</bean>	
	<!-- 自动读取c3p0配置文件 -->
	<bean name="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"></bean>
</beans>
```

### Struts2

EmployeeAction.java

```java
package com.web.action;


import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

import org.apache.struts2.ServletActionContext;
import org.hibernate.criterion.DetachedCriteria;

import com.dao.EmployeeDao;
import com.entity.Employee;
import com.entity.Good;
import com.entity.PageBean;
import com.opensymphony.xwork2.ActionSupport;
import com.opensymphony.xwork2.ModelDriven;
import com.service.EmployeeService;
import com.utils.DateJsonValueProcessor;

import net.sf.json.JSONObject;
import net.sf.json.JsonConfig;

public class EmployeeAction extends ActionSupport implements ModelDriven<Employee>{
	private Employee employee = new Employee();
	@Override
	public Employee getModel() {
		
		return employee;
	}
	private EmployeeService employeeService;
	public void setEmployeeService(EmployeeService employeeService) {
		this.employeeService = employeeService;
	}
	//设置分页参数
	private Integer page=1;
	private Integer rows=5;
	public void setPage(Integer page) {
			if (page == null) {
				page = 1;
			}
			this.page = page;
	}
	public void setRows(Integer rows) {
			if (rows == null) {
				rows = 5;
			}
			this.rows = rows;
	}
	public String findByPage() throws IOException {
		DetachedCriteria detachedCriteria = DetachedCriteria.forClass(Employee.class);

		PageBean<Employee> pageBean = employeeService.findByPage(detachedCriteria, page, rows);
		// 使用JSONlib转成JSON。 FastJSON gson
		// 创建一个Map集合:
		System.out.println(pageBean.getList().toString());
		Map<String, Object> map = new HashMap<String, Object>();
		map.put("total", pageBean.getTotalCount());
		map.put("rows", pageBean.getList());
		// 将map转成JSON:
		//转换日期格式
		JsonConfig jsonConfig = new JsonConfig();
		jsonConfig.registerJsonValueProcessor(java.util.Date.class, new DateJsonValueProcessor());
		
		JSONObject jsonObject = JSONObject.fromObject(map,jsonConfig);
		System.out.println(jsonObject.toString());
		ServletActionContext.getResponse().setContentType("text/html;charset=UTF-8");
		ServletActionContext.getResponse().getWriter().println(jsonObject.toString());
		return NONE;
	}
	public String save() throws IOException {
		Map<String,String> map = new HashMap<String,String>();		
		//System.out.println(employee.toString());
		try {
			employeeService.save(employee);
			map.put("msg", "保存成功");
		} catch (Exception e) {
	
			e.printStackTrace();
			map.put("msg", "保存失败");
		}
		JSONObject jsonObject = JSONObject.fromObject(map);
		ServletActionContext.getResponse().setContentType("text/html;charset=UTF-8");
		ServletActionContext.getResponse().getWriter().println(jsonObject.toString());
		return NONE;
	}
	public String findById() throws IOException {
		
		employee = employeeService.findById(employee.getEmp_id());
		JSONObject jsonObject = JSONObject.fromObject(employee);
		ServletActionContext.getResponse().setContentType("text/html;charset=UTF-8");
		ServletActionContext.getResponse().getWriter().println(jsonObject.toString());
		
		return NONE;
	}
	public String update() throws IOException {
		Map<String,String> map = new HashMap<String,String>();
		System.out.println(employee.toString());
		try {
			employeeService.update(employee);
			map.put("msg", "保存成功");
		} catch (Exception e) {
	
			e.printStackTrace();
			map.put("msg", "保存失败");
		}
		
		JSONObject jsonObject = JSONObject.fromObject(map);
		ServletActionContext.getResponse().setContentType("text/html;charset=UTF-8");
		ServletActionContext.getResponse().getWriter().println(jsonObject.toString());
		return NONE;
	}
	public String delete() throws IOException {
		Map<String,String> map = new HashMap<String,String>();	
		try {
			
			employee = employeeService.findById(employee.getEmp_id());
			employeeService.delete(employee);
			map.put("msg", "删除成功");
		} catch (Exception e) {

			e.printStackTrace();
			map.put("msg", "删除失败");
		}
		JSONObject jsonObject = JSONObject.fromObject(map);
		ServletActionContext.getResponse().setContentType("text/html;charset=UTF-8");
		ServletActionContext.getResponse().getWriter().println(jsonObject.toString());
		return NONE;
	}
}

```

struts.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE struts PUBLIC
	"-//Apache Software Foundation//DTD Struts Configuration 2.3//EN"
	"http://struts.apache.org/dtds/struts-2.3.dtd">
<struts>
	<constant name="struts.objectFactory" value="spring"></constant>
	<constant name="struts.action.extension" value="action"></constant>
	
	<package name="hotel" namespace="/" extends="struts-default">
		<global-results>
			<result name="login">/login.jsp</result>
		</global-results>
		<global-exception-mappings>
			<exception-mapping result="login" exception="java.lang.RuntimeException"></exception-mapping>
		</global-exception-mappings>
		<action name="order_*" class="orderAction" method="{1}">
		</action>
		<action name="emp_*" class="employeeAction" method="{1}">
		</action>
		<action name="good_*" class="goodAction" method="{1}">
		</action>
		<action name="user_*" class="userAction" method="{1}">
			<result name="success" type="dispatcher" >/WEB-INF/reception.jsp</result>
			<result name="login">/index.jsp</result>
			<result name="error">/index.jsp</result>
		</action>
	</package>
</struts>  
```

