### google hover

[仓库地址](https://github.com/google/hover)

看这个仓库的主要目的呢是为了完成美易点体检悬浮窗。这个是弹窗是系统层级的所以只关注了Window相关的 

![image-20180915122645127](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915122645127.png)UI稿



1. 系统悬浮窗权限申请

      1. 在AndroidManifest文件中加入权限声明

          ```
            <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
          ```

      2. Android6.0以上动态申请悬浮窗权限

         ```
          if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                     if (!Settings.canDrawOverlays(this)) {
                         Intent drawOverlaysSettingsIntent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
                         drawOverlaysSettingsIntent.setData(Uri.parse("package:" + getPackageName()));
                         startActivity(drawOverlaysSettingsIntent);
                     }
                 }
         ```

![hover](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/hover.png)



***

1. HoverMenuService 作为WindowManager的载体 与悬浮窗交互

2. HoverMenu 悬浮窗接口 控制悬浮窗显示隐藏展开折叠

3. WindowHoverMenu为实现类控制系统悬浮窗显示隐藏展开折叠

4. WindowViewController封装了WindowsManger的方法(系统弹窗对view的add remove      updatelayou都需要经过WindowsManager)

5. HoverMenuView 是系统弹窗的根布局 控制子view显示隐藏拖动逻辑

   1.  CollapsedMenuViewController 控制浮动菜单点击拖动释放的一些逻辑(前面有一层mDragListener先处理一些 比如是否拖到了删除区域)
     

6. HoverMenuAdapter 可实现该接口 控制布局中的tab 及content 的内容吧 

7. InWindowDragger 是Dragger的实现类 用于定义及监听能拖动区域 里面的OnTouchListener根据不同滑动逻辑定义滑动事件并通过Dragger.DragListener回调给HoverMenuView的CollapsedMenuViewController处理

8. CollapseMenuAnchor 用于记录折叠菜单的位置。

9. Navigator 和DefaultNavigator 是用于控制内容的显示(这部分感觉操作不是很方便 没看 自己直接放在adapter中去控制内容了)



#### 建议

1. 建议增加一个拖到指定位置退出悬浮窗功能 开关的话在其他界面就看不到 想关还得先进
2. 展开的话建议在固定位置展开 关闭后悬浮球回到原来位置（因为展开后是不能拖动的,还有原因就是不单左右两边 在各个角落也要不同展示效果比较麻烦 个人觉的固定展开位置体验还是OK的）

1. 选

