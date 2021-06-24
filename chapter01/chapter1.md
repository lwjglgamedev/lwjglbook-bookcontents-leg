# 第一步

我们将会在这本书中学会一些在开发 3D 游戏当中涉及到的一些最主要的技术。我们将会使用 Java 与 Lightweight Java Game Library \([LWJGL](http://www.lwjgl.org/)\) 来开发我们的示例项目。LWJGL 将允许您访问一些类似于 OpenGL 的底层 API。

LWJGL 是一个像 OpenGL 的包装器的底层 API。如果您想在短时间内开始制作 3D 游戏的话，或许您应当考虑其他的 API，比如 \[JmonkeyEngine\]。要使用这些底层API，您将需要理解许多概念并写下许多代码来获得您心意的结果，这样做的好处就是，您将会更好地理解与掌握 3D 图形系统。

我们在先前的段落里已经提到了这本书将会使用 Java 进行开发。我们将会使用 Java 10，所以您应该从 Oracle 公司的网页下载 Java SDK（软件开发工具包）。选择适合您系统的版本并安装即可。这本书假定您已经对 Java 语言拥有一定程度的了解。

你需要使用支持 Java 的 IDE（集成开发环境）You may use the Java IDE you want in order to run the samples. You can download IntelliJ IDEA which has good support for Java 10. Since Java 10 is only available, by now, for 64 bits platforms, remember to download the 64 bits version of IntelliJ. IntelliJ provides a free open source version, the Community version, which you can download from here: [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/ "Intellij").

![](/chapter01/intellij.png)

For building our samples we will be using [Maven](https://maven.apache.org/). Maven is already integrated in most IDEs and you can directly open the different samples inside them. Just open the folder that contains the chapter sample and IntelliJ will detect that it is a maven project.

![](/chapter01/maven_project.png)

Maven builds projects based on an XML file named `pom.xml` \(Project Object Model\) which manages project dependencies \(the libraries you need to use\) and the steps to be performed during the build process. Maven follows the principle of convention over configuration, that is, if you stick to the standard project structure and naming conventions the configuration file does not need to explicitly say where source files are or where compiled classes should be located.

This book does not intend to be a maven tutorial, so please find the information about it in the web in case you need it.  The source code folder defines a parent project which defines the plugins to be used and collects the versions of the libraries employed.

LWJGL 3.1 introduced some changes in the way that the project is built. Now the base code is much more modular, and we can be more selective in the packages that we want to use instead of using a giant monolithic jar file. This comes at a cost: You now need to carefully specify the dependencies one by one. But the [download](https://www.lwjgl.org/download) page includes a fancy script that generates the pom file for you. In our case, we will just be using GLFW and OpenGL bindings. You can check what the pom file looks like in the source code.

The LWJGL platform dependency already takes care of unpacking native libraries for your platform, so there's no need to use other plugins \(such as `mavennatives`\). We just need to set up three profiles to set a property that will configure the LWJGL platform. The profiles will set up the correct values of that property for Windows, Linux and Mac OS families.

```xml
    <profiles>
        <profile>
            <id>windows-profile</id>
            <activation>
                <os>
                    <family>Windows</family>
                </os>
            </activation>
            <properties>
                <native.target>natives-windows</native.target>
            </properties>                
        </profile>
        <profile>
            <id>linux-profile</id>
            <activation>
                <os>
                    <family>Linux</family>
                </os>
            </activation>
            <properties>
                <native.target>natives-linux</native.target>
            </properties>                
        </profile>
        <profile>
            <id>OSX-profile</id>
            <activation>
                <os>
                    <family>mac</family>
                </os>
            </activation>
            <properties>
                <native.target>natives-osx</native.target>
            </properties>
        </profile>
    </profiles>
```

在每个项目中，LWJGL 的平台依赖将会使用正确的属性为当前平台的配置文件进行正确的设置。

```xml
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-platform</artifactId>
            <version>${lwjgl.version}</version>
            <classifier>${native.target}</classifier>
        </dependency>
```

Besides that, every project generates a runnable jar \(one that can be executed by typing java -jar name\_of\_the\_jar.jar\). This is achieved by using the maven-jar-plugin which creates a jar with a `MANIFEST.MF` file with the correct values. The most important attribute for that file is `Main-Class`, which sets the entry point for the program. In addition, all the dependencies are set as entries in the `Class-Path` attribute for that file. In order to execute it on another computer, you just need to copy the main jar file and the lib directory \(with all the jars included there\) which are located under the target directory.

The jars that contain LWJGL classes, also contain the native libraries. LWJGL will also take care of extracting them and adding them to the path where the JVM will look for libraries.

第一章的源代码是直接从 LWGPL 的官方站点 \([http://www.lwjgl.org/guide](http://www.lwjgl.org/guide)\) 上得到的。您可能会注意到，我们并未使用 Swing 或是 JavaFX 当做我们的 GUI 库。取而代之的是 [GLFW](www.glfw.org) ：Instead of that we are using  which is a library to handle GUI components \(Windows, etc.\) and events \(key presses, mouse movements, etc.\) with an OpenGL context attached in a straightforward way. Previous versions of LWJGL provided a custom GUI API but, for LWJGL 3, GLFW is the preferred windowing API.

这个开发示例已经经过了十分直白而又简明的排版，所以我们不会再次重复注释了。

如果您成功地设置好了您的开发环境，您应该可以运行这个程序并看到一个带有红色背景的窗口。

![Hello World](hello_world.png)

**这本书的源代码已经发布到了 **[**GitHub**](https://github.com/lwjglgamedev/lwjglbook)**。**

