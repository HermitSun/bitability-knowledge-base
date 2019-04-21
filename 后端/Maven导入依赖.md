### Maven导入依赖

今天遇到了一个很有意思的bug。有人（还不止一个人）来问我，为什么示例项目启动不了，我说不可能啊，我这不跑得好好的吗，而且成功启动的也不止我一个啊。然后他就说，不信你来看，我过去一看，果然如此：

![](D:\BitEnergyProject\项目组材料\孙文撰写的材料\mybatis-bug.jpg)

这就很有意思了。是不是配置文件的问题？我检查了一下，似乎并没有什么问题，而且这代码正在我本地跑着呢：

```xml
<!--mybatis-->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>
```

焦头烂额了半天，我突然想到，是不是Maven版本的问题？因为之前看书的时候，里面提到过，不要使用IDE内嵌的Maven，因为IDE内嵌的版本很容易不一致，而版本不一致很容易导致构建行为的不一致。一查，他用的是2017年的IDEA。而这个包的发布时间呢？2018年3月14日。

![](D:\BitEnergyProject\项目组材料\孙文撰写的材料\maven-repo.png)

于是，我让他更新一下版本，问题解决。话说写Maven配置的时候不在注释里写版本真的没问题吗……

------

（以下是正文）

在用Maven之前，为了往本地项目里导入外部依赖，我一直是从各个依赖的官网直接下载jar包，然后手动添加进项目的lib文件夹里。现在用了Maven，但我为了省事（虽然最后事与愿违），就想着能不能还像之前一样直接从本地的jar包添加依赖，让这些jar包能跟着项目走，就踩了这次的坑。

我应该都知道怎么从本地导入jar包，就不再赘述了。Eclipse和IDEA的操作方式稍微有点区别，不过问题不大。

Maven的原理，大概是有一个远程仓库和本地仓库，以及一个配置文件pom.xml，用的时候把pom.xml 中定义的jar包从远程仓库下载到本地仓库，然后只需要简单配置一下，就可以使用项目需要的依赖，不用再一个一个手动导入，而且比手动导入优雅很多。

导入外部依赖最简单的办法是从远程仓库直接导入，语法也很简单：

```xml
<dependencies>
    <dependency>
        <groupId>example</groupId>
        <artifactId>example</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

但这样做的问题也很明显：万一远程仓库里没有怎么办？即使有，也面临着下载慢（外网），和下载不稳定的问题。一旦本地仓库出现偏差，或者Maven升级就需要重新配置。再者，在多人协作的时候，每到一个人手里就得往本地仓库里下载一遍，而且每个人都会面临，并且重复之前所说的问题。

不过，用远程仓库虽然有各种各样可能存在的问题，但是一般远程仓库有还是用远程仓库，最主要的原因就是配置方便，而这也是Maven出现的初衷吧；不然手动配置会很麻烦，得不偿失。

用这种本地依赖的一个例子是配置SqlServer的依赖，因为Microsoft官方不提供Maven支持，所以只能下载jar包然后添加本地依赖。至于导入本地的jar包，这次主要参考了

[这篇文章]: http://roufid.com/3-ways-to-add-local-jar-to-Maven-project/

，是老外写的：里面介绍了三种办法，我自己是用方法一（install插件+dependency插件）和方法二（System Scope）的。方法三是把项目里的lib文件夹建成一个额外的Maven本地仓库，但是相对费事一点（而且对于这种少量的jar包来说，可以用，但没必要），主要还是介绍方法一和二。先上配置文件（方法一），有点长：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>example</groupId>
    <artifactId>example</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>javax.mail</groupId>
            <artifactId>mail</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.activation</groupId>
            <artifactId>activation</artifactId>
            <version>1.1.1</version>
        </dependency>
        <dependency>
            <groupId>com.microsoft.sqlserver</groupId>
            <artifactId>sqljdbc6</artifactId>
            <version>6.0.8112.200</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-install-plugin</artifactId>
                <version>2.5</version>
                <executions>
                    <execution>
                        <id>install-javax-mail-jar</id>
                        <phase>clean</phase>
                        <configuration>
                            <repositoryLayout>default</repositoryLayout>
                            <groupId>javax.mail</groupId>
                            <artifactId>mail</artifactId>
                            <version>1.6.2</version>
                            <file>${basedir}/src/lib/javax.mail.jar</file>
                            <packaging>jar</packaging>
                            <generatePom>true</generatePom>
                        </configuration>
                        <goals>
                            <goal>install-file</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>install-activation-jar</id>
                        <phase>clean</phase>
                        <configuration>
                            <repositoryLayout>default</repositoryLayout>
                            <groupId>javax.activation</groupId>
                            <artifactId>activation</artifactId>
                            <version>1.1.1</version>
                            <file>${basedir}/src/lib/activation.jar</file>
                            <packaging>jar</packaging>
                            <generatePom>true</generatePom>
                        </configuration>
                        <goals>
                            <goal>install-file</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>install-jdbc-jar</id>
                        <phase>clean</phase>
                        <configuration>
                            <repositoryLayout>default</repositoryLayout>
                            <groupId>com.microsoft.sqlserver</groupId>
                            <artifactId>sqljdbc6</artifactId>
                            <version>6.0.8112.200</version>
                            <file>${basedir}/src/lib/sqljdbc42.jar</file>
                            <packaging>jar</packaging>
                            <generatePom>true</generatePom>
                        </configuration>
                        <goals>
                            <goal>install-file</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer
implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>ui.Main</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

方法一是很正统的做法。具体过程是，通过install插件，把项目里lib文件夹下的jar包安装到本地仓库里，然后再用dependency插件从仓库里“搬”到生成的target文件夹的指定目录里（一般是target/lib）。

方法二有点取巧，就是直接把`<dependency>`的来源换成本地指定目录（原来是远程仓库找，现在直接在本地找）。按照rtw的说法，这个方法明显比方法一三都方便，如果还比另外两个方法稳定，也就没有另外两个方法了。这个方法的主要缺陷，是只能应对jar包；如果要打包成war包，就无能为力了。（而且据我自己测试，target里生成的jar包也会出现在其他位置，可能是根目录，不像方法一一样出现在指定的地方，所以我又用了dependency插件去移位）

值得一提的是，从本地导入的jar包，如果没有手动添加到项目依赖里，在编译期可能会找不到依赖导致无法编译；所以为了确保能够编译，还需要一个compile插件。

另外，这里用的jar插件，是为了指定manifest.mf里面的classpath，让jar文件在运行的时候能够找到依赖来执行。Classpath是相对于jar包所在的位置而言的，所以是直接是lib/，而不是target/lib/。

如果要直接生成可执行的jar包（而不是需要外部依赖的那种），可以直接用shade插件，简单方便。