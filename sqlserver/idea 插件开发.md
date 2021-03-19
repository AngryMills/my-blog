# idea 插件开发 扫描sqlserver

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;最近在搞sqlserver 升级 mysql/tidb，发现代码里的sql有很多地方需要改，想着能不能开发一个省点力。

官方的迁移指南： https://www.mysql.com/why-mysql/white-papers/sql-server-to-mysql-zh/

## 方案选择

最开始想的是在sql拦截器做个拦截，判断标识是否开启转换，把对应的sqlserver 的sql转换后mysql的。技术上可以实现，开发成本高，需要对sqlserver和mysql的语法都熟悉，更主要是业务的sql错综复杂，很难保证100%。

比如说下面这个sql

```
SELECT [Id] FROM [dbo].[Test] with(nolock) WHERE [UId]=#{uid} ORDER BY BrowseTime DESC OFFSET #{page}*#{size}-#{size} ROW FETCH NEXT #{size} ROW ONLY;
```

转成mysql 

```
SELECT Id FROM dbo.Test WHERE UId = #{uid} ORDER BY BrowseTime DESC limit #{page}*#{size},#{size}
```

这个是报错的，根本没法转

![image-20210318165537996](C:\Users\wangguangmin\AppData\Roaming\Typora\typora-user-images\image-20210318165537996.png)

所以想业务方无感知转换sql，是比较难的。

所以就退一步，不转换，帮忙指出哪些sql 无法适用mysql，必须改变。



## idea 插件

使用idea的还是比较多的，就想着能不能开发一款插件完成扫描的工作，类似alibaba的代码扫描。

效果如图

![3](.\3.png)

![4](.\4.png)

简单分享下代码：

plugin.xml 中配置窗口和按钮

```
    <extensions defaultExtensionNs="com.intellij">
        <!-- Add your extensions here -->
        <!-- 自定义控制台输入 -->
        <!--canCloseContents 允许用户关闭-->
        <toolWindow canCloseContents="true" anchor="bottom"
                    id="Ms2Msl"
                    factoryClass="ycplugin.toolwindow.MslToolWindow">
        </toolWindow>
    </extensions>

    <actions>
        <action id="ycpluginSqlserverScan" class="ycplugin.ScanMain" text="ss2ms" description="ss2ms item" >
<!--            <add-to-group group-id="MainToolBar" anchor="last"/>-->
            <add-to-group group-id="ProjectViewPopupMenu" anchor="last"/>
<!--            <add-to-group group-id="ChangesViewPopupMenu" anchor="last"/>-->
<!--            <add-to-group group-id="EditorPopupMenu" anchor="last"/>-->
        </action>
    </actions>
```

MslToolWindow

```
package ycplugin.toolwindow;

import com.intellij.execution.filters.TextConsoleBuilder;
import com.intellij.execution.filters.TextConsoleBuilderFactory;
import com.intellij.execution.ui.ConsoleView;
import com.intellij.openapi.project.Project;
import com.intellij.openapi.wm.ToolWindow;
import com.intellij.openapi.wm.ToolWindowFactory;
import com.intellij.ui.content.Content;
import org.jetbrains.annotations.NotNull;

import javax.swing.*;
import java.awt.*;

public class MslToolWindow implements ToolWindowFactory {
    
    public static JComponent createConsolePanel(ConsoleView view) {
        JPanel panel = new JPanel();
        panel.setLayout(new BorderLayout());
        panel.add(view.getComponent(), BorderLayout.CENTER);
        return panel;
    }
    
    @Override
    public boolean shouldBeAvailable(@NotNull Project project) {
        return true;
    }
    
    @Override
    public void createToolWindowContent(@NotNull Project project, @NotNull ToolWindow toolWindow) {
        TextConsoleBuilder consoleBuilder = TextConsoleBuilderFactory.getInstance().createBuilder(project);
        ConsoleView console = consoleBuilder.getConsole();
        JComponent consolePanel = createConsolePanel(console);
        Content content = toolWindow.getContentManager().getFactory().createContent(consolePanel, "ss2msl plugin", false);
        toolWindow.getContentManager().addContent(content);
        //console.print("------------after add consoleView to tool------------" + "\n", ConsoleViewContentType.LOG_INFO_OUTPUT);
        new ToolWindowConsole(toolWindow, console, project);
        //PropertiesCenter.init(project);
        //console.setOutputPaused(true);
    }
    
    
}
```


ToolWindowConsole

```
package ycplugin.toolwindow;

import com.intellij.execution.filters.TextConsoleBuilderFactory;
import com.intellij.execution.ui.ConsoleView;
import com.intellij.execution.ui.ConsoleViewContentType;
import com.intellij.openapi.project.Project;
import com.intellij.openapi.wm.ToolWindow;


public class ToolWindowConsole {
    private static Project project;
    private static ConsoleView console;
    public static boolean isShow;
    
    
    public ToolWindowConsole(ToolWindow toolWindow, ConsoleView console, Project project) {
        this.console = console;
        this.project = project;
    }
    
    public static void show() {
        isShow = true;
    }
    
    public static void clear() {
        if (console != null) {
            console.clear();
        }
    }
    
    public static void log(String s) {
        if(project == null && console == null){
            return;
        }
        if (console == null) {
            console = TextConsoleBuilderFactory.getInstance().createBuilder(project).getConsole();
        }
        if (console.isOutputPaused()) {
            console.setOutputPaused(false);
        }
        console.print(s + "\n", ConsoleViewContentType.NORMAL_OUTPUT);
        return;
    }
}
```

ScanMain

```
package ycplugin;

import com.intellij.openapi.actionSystem.AnAction;
import com.intellij.openapi.actionSystem.AnActionEvent;
import com.intellij.openapi.actionSystem.CommonDataKeys;
import com.intellij.openapi.actionSystem.PlatformDataKeys;
import com.intellij.openapi.project.Project;
import com.intellij.openapi.util.IconLoader;
import com.intellij.openapi.vfs.VirtualFile;
import ycplugin.analysis.ScanAnalyzer;
import ycplugin.toolwindow.ToolWindowConsole;

public class ScanMain extends AnAction {
    // 如果通过Java代码来注册，这个构造函数会被调用，传给父类的字符串会被作为菜单项的名称
    // 如果你通过plugin.xml来注册，可以忽略这个构造函数
    public ScanMain() {
        // 设置菜单项名称
        // 还可以设置菜单项名称，描述，图标
        super("Ss2msl Scan", "sqlserver-to-mysql plugin", IconLoader.getIcon("/icon/open.png"));
    }
    
    @Override
    public void actionPerformed(AnActionEvent event) {
        ToolWindowConsole.show();
        // 每次触发清空 console
        ToolWindowConsole.clear();
        Project project = event.getData(PlatformDataKeys.PROJECT);
        VirtualFile[] virtualFiles = event.getData(CommonDataKeys.VIRTUAL_FILE_ARRAY);
        // 扫描开始
        new ScanAnalyzer(virtualFiles[0], project).start();
    }
}
```

ScanAnalyzer

```
package ycplugin.analysis;

import com.intellij.openapi.project.Project;
import com.intellij.openapi.vfs.VirtualFile;
import org.jetbrains.annotations.NotNull;
import ycplugin.analysis.filter.ScanFilter;
import ycplugin.analysis.filter.ScanXmlFilter;
import ycplugin.check.Ss2mslChecker;
import ycplugin.check.Ss2mslCheckerKeyword;
import ycplugin.convert.Ss2mslConverter;
import ycplugin.convert.Ss2mslConverterMysql;
import ycplugin.toolwindow.ToolWindowConsole;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.Charset;

public class ScanAnalyzer {
    private VirtualFile virtualFile;
    private Project project;
    private ScanFilter scanFilter;
    private Ss2mslChecker ss2mslChecker;
    private Ss2mslConverter ss2mslConverter;
    private int total;
    
    public ScanAnalyzer() {
    }
    
    public ScanAnalyzer(VirtualFile virtualFile, Project project) {
        this.virtualFile = virtualFile;
        this.project = project;
        scanFilter = new ScanXmlFilter();
        ss2mslChecker = new Ss2mslCheckerKeyword();
        ss2mslConverter = new Ss2mslConverterMysql();
    }
    
    public void start() {
        // 目录下
        if (virtualFile.isDirectory()) {
            findChildrenFile();
        } else {
            // 单个文件 
            String byLine = getSqlByLine(virtualFile);
            if (byLine != null) {
                // 说明有异常sql,先打印出文件夹
                ToolWindowConsole.log(byLine);
            }
            total = 1;
        }
        ToolWindowConsole.log("共扫描文件:【" + total + "】");
    }
    
    private void findChildrenFile() {
        @NotNull VirtualFile[] elements = virtualFile.getCanonicalFile().getChildren();
        for (int i = 0; i < elements.length; i++) {
            VirtualFile element = elements[i];
            if (!element.isDirectory()) {
                total++;
                if (scanFilter.filter(element)) {
                    String byLine = getSqlByLine(element);
                    if (byLine != null) {
                        // 说明有异常sql,先打印出文件夹
                        ToolWindowConsole.log(byLine);
                    }
                }
            } else if (element.isDirectory()) {
                virtualFile = element;
                findChildrenFile();
            }
        }
    }
    
    private String getSqlByLine(VirtualFile virtualFileElement) {
        StringBuilder builder = new StringBuilder();
        int lineNum = 1;
        try {
            BufferedReader br = new BufferedReader(new InputStreamReader(virtualFileElement.getInputStream(), Charset.forName("utf8")));
            String line;
            while ((line = br.readLine()) != null) {
                // 返回false,校验失败,
                String check = ss2mslChecker.check(line);
                if (check != null) {
                    if (builder.length() == 0) {
                        builder.append("当前扫描文件为:【" + virtualFileElement.getName() + "】" + "\r\n");
                    }
                    builder.append("第【" + lineNum + "】行 " + "\t" + ss2mslConverter.convert(line).replaceAll(" ", "") + "\t" + "异常字符:【" + check + "】" + "\r\n");
                    lineNum++;
                }
            }
            return builder.toString();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

至于里边的filter和checker 自己写下就行，逻辑也没复杂的，目前就是关键字识别匹配，也没什么功能。

打包遇到乱码问题的话，build.gradle 中添加

```
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}
```



## 总结

轮子都这么多了，总有一款适合你的，如果没有，就自己开发。