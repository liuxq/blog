* 背景：很多自定义编辑器可能带有好多的窗口，比如一个自定义的模型编辑器，可能有模型资源选择窗口、动作编辑窗口、动作播放设置窗口等。如果这些互相有联系的窗口都各自分散的显示。会给相关的策划和美术带来很多困扰。于是最近有一个实现编辑器EditorWindow窗口布局的需求，具体要求是把一些自定义的编辑器窗口按照策划喜欢的方式停靠到一个窗口中去（如下图）。

![dock](http://liuxq.github.io/blog/images/dockDemo.png)

* 实现方案一：编辑器的停靠效果并没有现成的api可用，反编译unityEditor.dll找到了用反射实现的方法，主要思路就是模拟鼠标拖拽一个窗口停靠到另一个窗口的过程。代码如下：

```c
public static void DockEditorWindow(EditorWindow parent, EditorWindow child)
{
    Vector2 screenPoint = parent.position.min + new Vector2(parent.position.width * .9f, 100f);

    Assembly assembly = typeof(UnityEditor.EditorWindow).Assembly;
    Type ew = assembly.GetType("UnityEditor.EditorWindow");
    Type da = assembly.GetType("UnityEditor.DockArea");
    Type sv = assembly.GetType("UnityEditor.SplitView");

    var tp = ew.GetField("m_Parent", BindingFlags.NonPublic | BindingFlags.Instance);
    var opArea = tp.GetValue(parent);
    var ocArea = tp.GetValue(child);
    var tview = da.GetProperty("parent", BindingFlags.Public | BindingFlags.Instance);
    var oview = tview.GetValue(opArea, null);
    var tDragOver = sv.GetMethod("DragOver", BindingFlags.Public | BindingFlags.Instance);
    var oDropInfo = tDragOver.Invoke(oview, new object[] { child, screenPoint });
    var tDockArea_ = da.GetField("s_OriginalDragSource", BindingFlags.NonPublic | BindingFlags.Static);
    tDockArea_.SetValue(null, ocArea);
    var tPerformDrop = sv.GetMethod("PerformDrop", BindingFlags.Public | BindingFlags.Instance);
    tPerformDrop.Invoke(oview, new object[] { child, oDropInfo, null });

}
```
效果如下：
![dock](http://liuxq.github.io/blog/images/editorWindowDock.png)

由于使用的拖拽模拟，有一些限制，比如窗口比例如果想1:1的话，需要先把第一个窗口的大小调到最小（我用的100,100），然后再停靠一个窗口时，两个窗口就会一样大，停靠之后再把窗口放大到想要的尺寸可以保持1:1的比例。

* 实现方案二：方案一是使用纯代码的方式实现一个窗口停靠到另一个窗口。有很多缺陷，比如停靠的样式完全靠代码，导致如果策划想要窗口是另一个停靠样式，就需要改代码；而且一些复杂的停靠样式并不能实现，比如本文第一张图的效果。其实unity已经为我们提供了一个靠谱的自己设计停靠样式并保存的方式，如下图
![dock](http://liuxq.github.io/blog/images/layouts.png)
可以先把各个窗口拖动停靠到想要的位置，然后savelayout，下一次直接点击保存的布局就可以啦，那么保存的布局存放在哪里了呢，放在C:\Users\Administrator\AppData\Roaming\Unity\Editor-5.x\Preferences\Layouts这里啦，是一个.wlt文件，存在这里是有问题的，布局文件没有存储在unity工程下面就不会被项目组的所有人共享，而懒惰的策划是希望一次update就可以使用别人调好的布局的。所以要实现一个保存布局文件在自定义位置的功能，这个功能也是需要用反射的，unity大大并没有把现成的函数暴露出来。。。，下面是实现保存和加载自定义布局的方法：

```c
using UnityEngine;
using UnityEditor;
using System.IO;
using System.Reflection;

using Type = System.Type;

public static class LayoutUtility
{
    private static MethodInfo _miLoadWindowLayout;
    private static MethodInfo _miSaveWindowLayout;
    private static MethodInfo _miReloadWindowLayoutMenu;

    private static bool _available;

    static LayoutUtility()
    {
        Type tyWindowLayout = Type.GetType("UnityEditor.WindowLayout,UnityEditor");
        Type tyEditorUtility = Type.GetType("UnityEditor.EditorUtility,UnityEditor");
        Type tyInternalEditorUtility = Type.GetType("UnityEditorInternal.InternalEditorUtility,UnityEditor");

        if (tyWindowLayout != null && tyEditorUtility != null && tyInternalEditorUtility != null)
        {
            _miLoadWindowLayout = tyWindowLayout.GetMethod("LoadWindowLayout", BindingFlags.NonPublic | BindingFlags.Public | BindingFlags.Static, null, new Type[] { typeof(string), typeof(bool) }, null);
            _miSaveWindowLayout = tyWindowLayout.GetMethod("SaveWindowLayout", BindingFlags.NonPublic | BindingFlags.Public | BindingFlags.Static, null, new Type[] { typeof(string) }, null);
            _miReloadWindowLayoutMenu = tyInternalEditorUtility.GetMethod("ReloadWindowLayoutMenu", BindingFlags.Public | BindingFlags.Static);

            if (_miLoadWindowLayout == null || _miSaveWindowLayout == null || _miReloadWindowLayoutMenu == null)
                return;

            _available = true;
        }
    }

    public static bool IsAvailable
    {
        get { return _available; }
    }

    public static void SaveLayoutToAsset(string assetPath)
    {
        SaveLayout(Path.Combine(Directory.GetCurrentDirectory(), assetPath));
    }

    public static void LoadLayoutFromAsset(string assetPath)
    {
        if (_miLoadWindowLayout != null)
        {
            string path = Path.Combine(Directory.GetCurrentDirectory(), assetPath);
            _miLoadWindowLayout.Invoke(null, new object[] { path , false });
        }
    }

    public static void SaveLayout(string path)
    {
        if (_miSaveWindowLayout != null)
            _miSaveWindowLayout.Invoke(null, new object[] { path });
    }

}
```

我们可以拖出来一个设计好的布局然后使用SaveLayoutToAsset保存布局到自定义路径，然后把布局文件上传到svn，策划update之后就可以直接通过菜单调用LoadLayoutFromAsset来加载布局啦