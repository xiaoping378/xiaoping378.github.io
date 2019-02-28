### vscode打开intelli idea java项目的设置

个人经验记录：

1. install deps

    ![](/assets/2019-02-27-21-21-44.png)

2. clean workspace 

    代开设置 Ctrl+Shift+P, 输入 ``clean java workspace`` and restart

3. support lombok

    java代码出了名的冗长，lombok可以优雅解决此类问题，如果项目依赖了lombok， vsocde打开java项目就会显示各种``cannot be resloved``错误,

    下面是我个人的配置,其中``java.jdt.ls.vmargs``的配置（看个人项目maven依赖和安装路径了），会消除错误，并可以支持跳转：
    ```json
    {
        "window.menuBarVisibility": "toggle",
        "window.zoomLevel": 1,
        "explorer.confirmDelete": false,
        "workbench.colorTheme": "Solarized Dark",
        "files.associations": {
            "default": "toml"
        },
        "editor.fontLigatures": true,
        "java.configuration.checkProjectSettingsExclusions": false,
        "go.autocompleteUnimportedPackages": true,
        "java.jdt.ls.vmargs":"-javaagent:/home/xxp/.m2/repository/./org/projectlombok/lombok/1.16.20/lombok-1.16.20.jar -Xbootclasspath/a:/home/xxp/.m2/repository/./org/projectlombok/lombok/1.16.20/lombok-1.16.20.jar"
    }
    ```

参考链接

- https://github.com/redhat-developer/vscode-java/wiki/Lombok-support
- https://code.visualstudio.com/docs/languages/java
- https://github.com/redhat-developer/vscode-java/wiki/Troubleshooting