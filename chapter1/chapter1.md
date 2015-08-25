# First steps

In this book we will learn the principal techniques involved in developing 3D games. We will develop our samples in Java and we will use the Java Lightweight Game Library (LWJGL, http://www.lwjgl.org/. This library enables the access to low-level APIs (Application Programming Interface) such as OpenGL.

As said in the previous paragraph we will be using Java for this book. We will be using Java 8, so you need to download Java SDK from Oracle’s pages. Just choose the installer that suits your Operative System and install it.
The source code that accompanies this book has been developed using the Netbeans IDE. You can download the latest version of that IDE from https://netbeans.org/. You only need the Java SE version but remember to download the version that suits with your JDK version (32 bits or 64 bits).

![Netbeans download](netbeans_download.png)
 
For building our samples we will be using Maven (https://maven.apache.org/). Maven is already integrated in Netbeans and you can directly open the different samples from Netbeans, just open the folder that contains the chapter sample and Netbeans will detect that it is a maven project.

![Maven projects](maven_projecs.png)
 
Maven builds projects based on an XML file named pom.xml (Project Object Model) which manages project dependencies (the libraries you need to use) and the steps to performed in build process. Maven follows the principle convention over configuration, that is if you stick to the standard project structure and naming conventions the configuration file does not need to explicitly say where source files are or where compiled classes should be left.

This book does not intend to be a maven tutorial, so please find the information about it in the web in case you need it.  The source code folder defines a parent project which defines the plugins to be used and collects the versions of the libraries to be used. 
We use a special plugin named ```mavennatives``` which unpacks the native libraries provided by LWJGL for your platform.

```xml
            <plugin>
                <groupId>com.googlecode.mavennatives</groupId>
                <artifactId>maven-nativedependencies-plugin</artifactId>
                <version>${natives.version}</version>
                <executions>
                    <execution>
                        <id>unpacknatives</id>
                        <phase>generate-resources</phase>
                        <goals>
                            <goal>copy</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

Those libraries are placed under target/natives directory. When you execute the samples from Netbeans you need to specify the directory where the Java Virtual Machine will look for native libraries. This is done with the command line property: “-Djava.library.path” which sould be set to: “-Djava.library.path="target\natives”. This is done automatically for you in the file ```nbactions.xml```. In case you want to change or learn how to do it manually, right click in your project and select “Properties”. In the dialog that is shown select “Run” category and set the correct value for VM Options.

![VM Settings](vm_settings.png) 

Chapter 1 source code is taken directly from the getting started sample in the LWJGL site (http://www.lwjgl.org/guide). You will see that we are not using Swing or JavaFX as our GUI library. Instead of that we are using GLFW. (www.glfw.org) is  library to handle GUI components (Windows, etc.) and events (key presses, mouse movements, etc.) with an Open GL Context attached in a straight forward way. Previous versions of LWJGL provided a custom GUI API. For LWJGL 3 GLFW is the preferred windowing API.

The samples source code is very well documented and straight forward so we won’t repeat the comments here. If you have your environment correctly set up you should be execute it and see a window with red background.

![Hello World](hello_world.png)


