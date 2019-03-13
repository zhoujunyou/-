### 在sublime上使用CTags

#####安装

1. Sublime  上安装CTags插件

2. `brew install ctags`

3. sublime pereference->Packages Settins->CTags->Seting-user command下配置ctags 路径

   ` "command": "/usr/local/Cellar/ctags/5.8_1/bin/ctags",`


##### 使用

1. 右键点击文件夹 -》Rebuild Tags  文件夹下会生成.tag 文件

   | 功能                             | 快捷键                                                  |
   | -------------------------------- | ------------------------------------------------------- |
   | 导航函数(navigate_to_definition) | ["ctrl+t", "ctrl+t"],["ctrl+shift+period"]              |
   | 搜索函数(search_for_definition)  | ["ctrl+t", "ctrl+y"]感觉还是自带的["command+r"]比较好用 |
   | 跳转到前面一步(jump_prev)        | ["ctrl+t", "ctrl+b"]                                    |
   |                                  |                                                         |
   |                                  |                                                         |
   |                                  |                                                         |

