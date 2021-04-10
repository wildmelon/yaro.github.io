## 1. 字体
1. FiraCode, https://github.com/tonsky/FiraCode
1. source-code-pro, https://github.com/adobe-fonts/source-code-pro

## 2. settings.json

```
{
    "editor.fontSize": 17,
    "workbench.iconTheme": "vs-seti", //插件
    "workbench.colorTheme": "Solarized Light", //自带主题
    "editor.fontFamily": "'Fira Code', '微软雅黑', Consolas, 'Courier New', monospace",
    "editor.wordWrap": "bounded",
    "files.associations": {
        "*.uxml": "xml"
    },
    "xmlTools.enforcePrettySelfClosingTagOnFormat": true,
    "xmlTools.splitAttributesOnFormat": true,
    "csharp.referencesCodeLens.enabled": false,
}
```
## 3. 插件
1. Chinese，中文包，https://marketplace.visualstudio.com/items?itemName=MS-CEINTL.vscode-language-pack-zh-hans
2. C#，https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp
3. Material Icon Theme，文件图标，https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme
4. TabOut，简单跳出右括号，https://marketplace.visualstudio.com/items?itemName=albert.TabOut
5. Unity Tools，https://marketplace.visualstudio.com/items?itemName=Tobiah.unity-tools
6. Debugger for Unity，https://marketplace.visualstudio.com/items?itemName=Unity.unity-debug 
7. Unity Code Snippets，MonoBehaviours 等方法提示补全，https://marketplace.visualstudio.com/items?itemName=kleber-swf.unity-code-snippets
8. XML Tools，https://marketplace.visualstudio.com/items?itemName=DotJoshJohnson.xml

## 4. Vscode 中设置C#右大括号不换行
在根目录创建 omnisharp.json 文件，添加以下代码：
```
// 更多属性和全局配置参考 https://github.com/OmniSharp/omnisharp-roslyn/wiki/Configuration-Options
{
    "FormattingOptions": {
        "NewLinesForBracesInLambdaExpressionBody": false,
        "NewLinesForBracesInAnonymousMethods": false,
        "NewLinesForBracesInAnonymousTypes": false,
        "NewLinesForBracesInControlBlocks": false,
        "NewLinesForBracesInTypes": false,
        "NewLinesForBracesInMethods": false,
        "NewLinesForBracesInProperties": false,
        "NewLinesForBracesInObjectCollectionArrayInitializers": false,
        "NewLinesForBracesInAccessors": false,
        "NewLineForElse": false,
        "NewLineForCatch": false,
        "NewLineForFinally": false
    }
}
```
