# 第一步

我们将会在这本书中学会一些在开发 3D 游戏当中涉及到的一些最主要的技术. 我们将会使用 Java 与 Lightweight Java Game Library \([LWJGL](http://www.lwjgl.org/)\) 来开发我们的示例项目. LWJGL 将允许您访问一些类似于 OpenGL 的底层 API.

LWJGL 是一个像 OpenGL 的包装器的底层 API.如果您想在短时间内开始制作 3D 游戏的话, 或许您应当考虑其他的 API,比如 \[JmonkeyEngine\]. 要使用这些底层API, 您将需要理解许多概念并写下许多代码来获得您心意的结果, 这样做的好处就是, 您将会更好地理解与掌握 3D 图形系统.

我们在先前的段落里已经提到了这本书将会使用 Java 进行开发. 我们将会使用 Java 10, 所以您应该从 Oracle 公司的网页下载 Java SDK (软件开发工具包). 选择适合您系统的版本并安装即可. 这本书假定您已经对 Java 语言拥有一定程度的了解.

你需要使用支持 Java 的 IDE (集成开发环境) 来运行示例项目. 您可以下载对 Java 10 拥有超棒支持的 IntelliJ. 到目前为止, Java 10 仅支持 64 位平台, 所以请一定要下载 IntelliJ 的 64 位版本. IntelliJ提供了一个开源版本, 即社区版, 您可从此处下载: [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/ "Intellij").

![](/chapter01/intellij.png)

我们将使用 [Maven](https://maven.apache.org/) 来构建我们的示例项目. Maven 已经被集成进了大多数 IDE, 所以您可以直接打开不同的示例文件. 只需要打开包含该章节示例的文件夹, IntelliJ 就会检测到这是一个 Maven 项目.

![](/chapter01/maven_project.png)

通过 Maven 构建的项目是基于一个叫做 `pom.xml` \(Project Object Model\) 的 XML 文件上的. 这个文件用来管理项目的依赖关系与构建过程中所要进行的步骤. Maven 遵循约定大于配置的原则, 这就是说, 如果您执意使用标准的项目结构与命名规范, 那么您将没有必要在配置文件中加入源文件和编译类的位置.

这本书并未打算变成一个 Maven 的教程, 所以您可以在您需要更多信息的情况下在网上自行查找. 包含源代码的文件夹定义了父级项目, 即用来定义插件的使用与收集被使用的库的版本的项目.

LWJGL 3.1 第一次尝试了一些在构建项目时的变化. 现在核心代码已经变得更加结构化, 从而使得我们在包管理上可以更加有选择性地选择我们需要的包, 而不是取而代之地使用一个庞大单一臃肿不堪的 jar 文件. 但这也需要一个代价: 你需要更仔细地一个个指定依赖关系. 但是不用担心, [下载](https://www.lwjgl.org/download) 页面包含了一个可以帮您生成 pom 文件的脚本. 对于我们而言, 这就意味着我们只需要使用 GLFW 与 OpenGl 的绑定 API 就可以了. 您可以在源代码中看看 pom 文件是什么样的.

LWJGL 的平台依赖关系已经被您平台的原生解包库所掌管, 所以您没有必要去使用其他的插件 \(就例如 `mavennatives`\). 我们只需要准备三个配置文件, 这样就可以设置能够管理 LWJGL 平台的属性了. 这些配置文件将会为 Windows, Linux 和 Mac OS 系列准备合适的属性值.

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

在每个项目中, LWJGL 的平台依赖将会使用正确的属性为当前平台的配置文件进行正确的设置.

```xml
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-platform</artifactId>
            <version>${lwjgl.version}</version>
            <classifier>${native.target}</classifier>
        </dependency>
```

除此之外, 每个项目都会生成一个可运行的 jar 文件 \(可通过输入 java -jar name\_of\_the\_jar.jar\ 来运行).其是通过使用可以生成带有正确属性值与 `MANIFEST.MF` 文件的 jar 文件的插件来打包的. 该文件最重要的一点就是 `Main-Class` (主类), 其设置了程序的进入点. 另外, 所有依赖关系都被设置成了在 `Class-Path` (类路径) 中的进入点. 为了能够在其他计算机中运行这一文件, 只需要复制这个主要的 jar 文件和在目标目录下的 lib 目录 \(包含了所有 jar 文件\).

这些 jar 文件包含了 LWJGL 的类, 也包含了一些原生库. LWJGL 也会压缩他们, 并将他们加入 JVM 寻找库的环境变量中去.T

第一章的源代码是直接从 LWGPL 的官方站点 \([http://www.lwjgl.org/guide](http://www.lwjgl.org/guide)\) 上得到的. 您可能会注意到,我们并未使用 Swing 或是 JavaFX 当做我们的 GUI 库. 取而代之的是 [GLFW](www.glfw.org): 一个可以通过 OpenGL 上下文管理 GUI 组件 \(Windows等\) 与 事件 \(按键, 鼠标的移动等\) 的库. 先前的 LWJGL 版本提供自定义 GUI API, 但是在 LWJGL 3 中, 我们更倾向于使用视窗化的 API.

这个开发示例已经经过了十分直白而又简明的排版, 所以我们不会再次重复注释了.

如果您成功地设置好了您的开发环境, 您应该可以运行这个程序并看到一个带有红色背景的窗口.

![Hello World](hello_world.png)

**这本书的源代码已经发布到了 **[**GitHub**](https://github.com/lwjglgamedev/lwjglbook)**.**

