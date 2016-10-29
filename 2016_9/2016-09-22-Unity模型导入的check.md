* 介绍：
资源管理是Unity开发者经常遇到的问题，比如在导入模型（fbx）资源时，对于一些不需要材质的纯动作模型，如果美术没有取消勾选Import Materials选项(如下图)，就会增加包中的无用材质和贴图，造成空间浪费。
![dock](http://liuxq.github.io/blog/images/fbxImport1.png)
如果单纯靠人工每个模型检查设置，无疑既增加了成本，也为疏漏创造了条件。因此本文提供一种自动核查的方式处理模型资源导入的管理。

* 过程：
要实现资源导入核查需要获取Unity导入资源时的回调入口，这里需要介绍一下AssetPostprocessor 类。该类是Unity大部分资源（模型、图片、声音）导入时调用的处理类。有导入之前、导入之后等时间节点的处理函数。具体介绍可以查看api（https://docs.unity3d.com/ScriptReference/AssetPostprocessor.html）。

* 今天要介绍的是导入fbx模型时的核查，比如帮助美术检查导入fbx文件，如果是动作模型，就不导入材质。用到的AssetPostprocessor方法是OnPreprocessModel，该方法在导入fbx之前调用，可以用于设置各种导入参数。代码如下:

```c
    private string path = "Assets/Resources/Model";
    private void OnPreprocessModel()
    {
        ModelImporter modelim = assetImporter as ModelImporter;
        if (modelim == null)
            return;
        //model目录下带动作的fbx不要导入材质
        if (assetPath.Contains(path))
            {
            if (modelim.importAnimation && modelim.clipAnimations.Length > 0 && modelim.importMaterials)
            {
                 modelim.importMaterials = false;
            }
        }
    }
```

* 另外有的fbx模型的材质是有问题的，此时可以在导入时给美术提示，此时可以用到OnAssignMaterialModel方法，该方法在导入模型之后设置材质时候调用，可以查看导入的材质信息。代码如下：

```c
private void OnAssignMaterialModel(Material material, Renderer renderer)
    {
        //提示材质有问题的fbx
        if (material.mainTexture == null)
        {
            ModelImporter modelim = assetImporter as ModelImporter;
            if (modelim != null && modelim.importMaterials == true)
            {
                EditorUtility.DisplayDialog("fbx材质问题", "模型:" + assetPath + "的材质mainTexture是空！请检查模型", "ok");
            }
        }
    }
```

* 如果模型材质文理是空就会提示：
![dock](http://liuxq.github.io/blog/images/fbxImport2.png)

导入其他资源也同理。

