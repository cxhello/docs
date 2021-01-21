# Maven集成CheckStyle

最近在项目组里开发项目的时候，经常遇到项目里面代码格式不统一，merge 的时候经常冲突一大片，在网上搜索，查到了 Maven 可以集成 checkstyle 进行代码格式化审查。现将我的经验做以分享。

### 配置CheckStyle插件

在项目根目录新建一个 `config` 文件夹，将代码规约配置文件放到此路径下，当然你也可以根据自己的需求去自行定义

常见的语法配置如下：

> http://checkstyle.sourceforge.net/config.html

我这里用的是 `sun` 公司的代码规约，配置文件地址如下：

> https://github.com/checkstyle/checkstyle/tree/master/src/main/resources

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>3.1.1</version>
	<configuration>
		<configLocation>config/checkstyle.xml</configLocation>
	</configuration>
	<executions>
		<execution>
			<id>checkstyle</id>
			<phase>validate</phase>
			<goals>
				<goal>check</goal>
			</goals>
			<configuration>
				<failOnViolation>true</failOnViolation>
			</configuration>
		</execution>
	</executions>
</plugin>
```

我们定义了在 Maven Lifecycle 的 validate 阶段执行 check task，并且如果发现有违反标准的情况就会 fail 当前的 build。

### 运行CheckStyle检查

```bash
mvn checkstyle:check
```

maven-checkstyle-plugin 内置了4种规范

- config/sun_checks.xml
- config/maven_checks.xml
- config/turbine_checks.xml
- config/avalon_checks.xml

其中sun_checks.xml为默认值。如果想要使用其他三种规范，则只需配置configuration。

### 参考文章

> https://github.com/checkstyle/checkstyle

> https://blog.csdn.net/frank823/article/details/80787142

> https://1991421.cn/2020/03/15/11de5b7f/

> https://blog.csdn.net/qq_26440803/article/details/89344479