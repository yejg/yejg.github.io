---
layout: post
title: Eclipse插件改造——自定义代码检查规则
categories: [eclipse plug-in, p3c]
description: 添加自定义代码检查规则
keywords: eclipse, plugin, plug-in, smartfox, p3c
---
## 前言
前面已经写了2篇介绍了源码编译了
- [EGit源码编译](https://yejg.top/2018/07/23/build-egit-source/)
- [Alibaba smartfox-eclipse插件源码编译](https://yejg.top/2018/07/26/build-ali-p3c-smartfox-eclipse-source/)

编译成功了，就该朝着目标去改造了。  
根据公司现有代码的情况，希望能增加一条校验规则
> 在SpringMVC的Controller中禁止直接使用Mapper/DAO

## 工具准备
要做代码检查，需要使用一款PMD的工具。前面介绍的阿里巴巴的smartfox-eclipse插件，其实也是基于PMD做的。   

[PMD](https://pmd.github.io/)  
一款采用BSD协议发布的Java程序代码检查工具。该工具可以做到检查Java代码中是否含有未使用的变量、是否含有空的抓取块、是否含有不必要的对象等。该软件功能强大，扫描效率高，是Java程序员debug的好帮手。   


## 技术知识准备
正式改造代码前，先了解下pmd的用法。

### PMD工具使用
把官网的pmd-bin下载下来之后，有一个[designer.bat]，这是一个图形界面工具，能将java源代码转化为AST（抽象语法树），这个工具非常有用，后面写代码找xpath全靠它。

还有一个[bgastviewer.bat]，功能和[designer.bat]差不多。

### designer.bat使用
先来个网上的示例热热身。
> 自定义的规则：While循环必须使用括号

先对比看下while语句块有括号和没括号的时候，抽象语法树的区别
- while语句块带括号

![image](/images/posts/eclipse-plugin/AST-while-with-block.png)

- while语句块不写括号

![image](/images/posts/eclipse-plugin/AST-while-without-block.png)

对比上面2张图，可以看出有括号比没括号的 AST抽象语法树 多一个Block节点。   
我们代码检查就可以根据这个[有无Block节点]来判断while语句块是否写了大括号了。

### 代码实现
1. 新建一个Java类继承net.sourceforge.pmd.lang.java.rule.AbstractJavaRule；   
2. 覆写public Object visit(ASTWhileStatement node, Object data)方法。
代码如下：

	```java
	import net.sourceforge.pmd.lang.ast.Node;
	import net.sourceforge.pmd.lang.java.ast.ASTBlock;
	import net.sourceforge.pmd.lang.java.ast.ASTWhileStatement;
	import net.sourceforge.pmd.lang.java.rule.AbstractJavaRule;

	public class WhileLoopsMustUseBracesRule extends AbstractJavaRule {
		public Object visit(ASTWhileStatement node, Object data) {
			Node firstStmt = node.jjtGetChild(1);
			if (!hasBlockAsFirstChild(firstStmt)) {
				addViolation(data, node);
			}
			return super.visit(node, data);
		}

		private boolean hasBlockAsFirstChild(Node node) {
			return (node.jjtGetNumChildren() != 0 && (node.jjtGetChild(0) instanceof ASTBlock));
		}
	}

	```	
3. 参考pmd源码中的xml文件，建一个自己的规则集，例如：
	
	```
	<?xml version="1.0"?>
	<ruleset name="My custom rules"
		xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 http://pmd.sourceforge.net/ruleset_2_0_0.xsd">
		<rule name="WhileLoopsMustUseBracesRule"
			  message="Avoid using 'while' statements without curly braces"
			  class="net.sourceforge.pmd.lang.java.test.WhileLoopsMustUseBracesRule">
		  <description>
		  Avoid using 'while' statements without using curly braces
		  </description>
			<priority>3</priority>
	 
		  <example>
	<![CDATA[
		public void doSomething() {
		  while (true)
			  x++;
		}
	]]>
		  </example>
		</rule>
	```
4. 运行验证   
可以打包然后使用pmd.bat命令
> pmd.bat -d c:\path\to\my\src -f xml -R c:\path\to\mycustomrules.xml

也可以直接debugger/run修改后的pmd源码


## 修改smartfox插件
前面花了较大篇幅写技术知识准备，因为没有这些技术基础，想改smartfox是无从改起的。   
再回过头来看开篇的目标：
> 在SpringMVC的Controller中不能直接使用Mapper/DAO

### 整理思路：

> [SpringMVC的Controller类]
>>  如何判断一个类是MVC中的C呢？
>>> 类头上有@Controller或者@RestController注解标记

> [不能直接使用Mapper或DAO]
>> C类中一般是通过@Autowired或者@Resource注入Mapper的

综上，我们可以这么处理：
> 当前类标记了 @Controller 或 @RestController，且有通过Autowired或Resource来注入 XXXMapper 或 XXXDao

### smartfox代码修改

1. 新建一个规则类，继承AbstractAliRule。
    ```
    注意，这里不是直接继承AbstractJavaRule，
    因为阿里做了国际化支持，在AbstractAliRule中做了资源翻译，
    所以我们也遵照ali的方式来改
    ```
2. 覆写public Object visit(ASTTypeDeclaration node, Object data)方法
3. 新建一个xml规则集
4. 在p3c-pmd\src\main\resources\messages.xml中增加国际化资源
5. 修改smartfox-plugin中的如下xml，加一行使之生效
    ```
    eclipse-plugin\com.alibaba.smartfox.eclipse.plugin\src\main\resources\rulesets\java\ali-pmd.xml
    在上面的xml中增加一行：<rule ref="rulesets/java/crh-springmvc.xml"/>
    ```

代码如下：   
■ControllerShouldNotAutowiredMapperRule.java

```
package com.alibaba.p3c.pmd.lang.java.rule.crh;

/**
 * 检查Controller类中是否直接注入了xxxMapper或者XXXDao
 * @author yejg
 */
public class ControllerShouldNotAutowiredMapperRule extends AbstractAliRule {

	private static final String ANNOTATION_CONTROLLER = "Controller";
	private static final String ANNOTATION_REST_CONTROLLER = "RestController";

	private static final String ANNOTATION_AUTOWIRED = "Autowired";
	private static final String ANNOTATION_RESOURCE = "Resource";

	private static final String MAPPER = "mapper";
	private static final String DAO = "dao";

	@Override
	public Object visit(ASTTypeDeclaration node, Object data) {
		if (this.hasControllerAnnotation(node)) {
			try {
				List<Node> nodes = node.findChildNodesWithXPath("ClassOrInterfaceDeclaration/ClassOrInterfaceBody/ClassOrInterfaceBodyDeclaration");
				if (nodes != null && nodes.size() > 0) {
					for (int i = 0; i < nodes.size(); i++) {
						ASTClassOrInterfaceBodyDeclaration tmpNode = (ASTClassOrInterfaceBodyDeclaration) (nodes.get(i));
						if (this.hasAutowiredAnnotion(tmpNode)) {
							List<ASTFieldDeclaration> astFieldDeclarations = tmpNode.findChildrenOfType(ASTFieldDeclaration.class);
							for (ASTFieldDeclaration astFieldDeclaration : astFieldDeclarations) {
								List<ASTType> childrenOfType = astFieldDeclaration.findChildrenOfType(ASTType.class);
								for (ASTType astType : childrenOfType) {
									String typeImage = astType.getTypeImage();
									if (typeImage != null && (typeImage.toLowerCase().endsWith(MAPPER) || typeImage.toLowerCase().endsWith(DAO))) {
										// super.addViolation(data, astType);
										ViolationUtils.addViolationWithPrecisePosition(this, astType, data,
								                I18nResources.getMessage("java.crh.ControllerShouldNotAutowiredMapperRule.rule.msg"));
									}
								}
							}
						}
					}
				}
			} catch (JaxenException e) {
				e.printStackTrace();
			}
		}
		return super.visit((JavaNode) node, data);
	}

	private boolean hasControllerAnnotation(ASTTypeDeclaration node) {
		List<ASTAnnotation> childrenOfASTAnnotation = node.findChildrenOfType(ASTAnnotation.class);
		for (ASTAnnotation astAnnotation : childrenOfASTAnnotation) {
			String annotationName = astAnnotation.jjtGetChild(0).jjtGetChild(0).getImage();
			if (ANNOTATION_CONTROLLER.equals(annotationName) || ANNOTATION_REST_CONTROLLER.equals(annotationName)) {
				return true;
			}
		}
		return false;
	}

	private boolean hasAutowiredAnnotion(ASTClassOrInterfaceBodyDeclaration bodyDeclaration) {
		List<ASTAnnotation> childrenOfType = bodyDeclaration.findChildrenOfType(ASTAnnotation.class);
		for (ASTAnnotation astAnnotation : childrenOfType) {
			// String annotationName = astAnnotation.getAnnotationName();// 高版本才有这个getAnnotationName方法
			String annotationName = astAnnotation.jjtGetChild(0).jjtGetChild(0).getImage();
			if (ANNOTATION_AUTOWIRED.equals(annotationName) || ANNOTATION_RESOURCE.equals(annotationName)) {
				return true;
			}
		}
		return false;
	}
}

```
■ 规则集crh-springmvc.xml

```
<?xml version="1.0"?>

<ruleset name="AlibabaJavaOthers" xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 http://pmd.sourceforge.net/ruleset_2_0_0.xsd">

    <rule name="ControllerShouldNotAutowiredMapperRule" language="java" since="crh.extremely.serious"
        message="java.crh.ControllerShouldNotAutowiredMapperRule.rule.msg"
        class="com.alibaba.p3c.pmd.lang.java.rule.crh.ControllerShouldNotAutowiredMapperRule">
        <description>java.crh.ControllerShouldNotAutowiredMapperRule.rule.desc</description>
        <priority>1</priority>
        <example>
			<![CDATA[
反例:
  @Controller
  public class ExampleAction {
    @Autowired
    private UserMapper usermapper;
    
    ....
  }
			]]>
      </example>
    </rule>
</ruleset>
```
