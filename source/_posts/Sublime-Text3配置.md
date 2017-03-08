---
title: Sublime Text3配置
date: 2016-09-21 11:09:54
categories: 开发环境配置
tags:
    - 开发环境配置
    - Sublime Text
---
# 快捷键（Eclipse风格）
Sublime Text -> Preferences -> Key Bindings - User
```
[
    /**
     * 代码格式化
     */
    { "keys": ["command+r"], "command": "markdown_preview", "args": { "target": "browser"} },
    {
        "keys": ["alt+shift+f"],
        "command": "reindent"
    },
    /**
     * json格式化
     */
    { "keys": ["command+shift+x"], "command": "tidy_xml" },
    { "keys": ["command+shift+j"], "command": "pretty_json" }, 
    { "keys": ["command+shift+m"], "command": "un_pretty_json" },
    /**
     * 适配eclipse快捷键
     */
    { "keys": ["alt+/"], "command": "auto_complete" },
    { "keys": ["command+i"], "command": "reindent" },
    // 当前行和下面一行交互位置
    { "keys": ["alt+up"], "command": "swap_line_up" },
    { "keys": ["alt+down"], "command": "swap_line_down" },
    // 复制当前行到上一行
    { "keys": ["command+alt+up"], "command": "duplicate_line" },
    // 复制当前行到下一行
    { "keys": ["command+alt+down"], "command": "duplicate_line" },
    // 删除整行
    { "keys": ["command+d"], "command": "run_macro_file", "args": {"file": "Packages/Default/Delete Line.sublime-macro"} },
    // 光标移动到指定行
    { "keys": ["command+l"], "command": "show_overlay", "args": {"overlay": "goto", "text": ":"} },
    // 快速定位到选中的文字
    { "keys": ["command+k"], "command": "find_under_expand_skip" },
    // { "keys": ["command+shift+x"], "command": "swap_case" },
    { "keys": ["command+shift+x"], "command": "upper_case" },
    { "keys": ["command+shift+y"], "command": "lower_case" },
    // 在当前行的下一行插入空行(这时鼠标可以在当前行的任一位置, 不一定是最后)
    { "keys": ["shift+enter"], "command": "run_macro_file", "args": {"file": "Packages/Default/Add Line.sublime-macro"} },
    // 定位到对于的匹配符(譬如{})(从前面定位后面时,光标要在匹配符里面,后面到前面,则反之)
    { "keys": ["command+p"], "command": "move_to", "args": {"to": "brackets"} },
    // outline
    { "keys": ["command+o"], "command": "show_overlay", "args": {"overlay": "goto", "text": "@"} },
    // 当前文件中的关键字(方便快速查找内容)
    { "keys": ["command+alt+o"], "command": "show_overlay", "args": {"overlay": "goto", "text": "#"} },
    // open resource
    { "keys": ["command+shift+r"], "command": "show_overlay", "args": {"overlay": "goto", "show_files": true} },
    // 文件内查找/替换
    { "keys": ["command+f"], "command": "show_panel", "args": {"panel": "replace"} },
    // 全局查找/替换, 在查询结果中双击跳转到匹配位置
    {"keys": ["command+h"], "command": "show_panel", "args": {"panel": "find_in_files"} },
 
    // plugin配置
    { "keys": ["alt+a"], "command": "alignment" },
    {"keys": ["command+shift+f"], "command": "js_format"}
]
```
<!-- more -->

# Color Scheme
Monokai

# Package Control
安装方法各版本不同，以最新搜索结果为准。
## 常用插件
Markdown Preview
Markdown Editing
[Pretty Json](https://github.com/dzhibas/SublimePrettyJson)
(也可以在chrome浏览器中安装JSON Formatter插件)

# 激活
原则上支持正版XD。
