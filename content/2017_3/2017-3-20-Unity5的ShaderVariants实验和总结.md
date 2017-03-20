ShaderVariants（下文用shader变种描述）是unity中用来合并组织shader的一个方式之一，在shader中的使用类似宏定义。最近项目使用shader变种的时候发现了一些坑，所以做了如下实验和总结。其中前两节是基础部分，看官方文档也可以了解，只是通过实验来加强理解。第三节shader变种的打包是重点描述的，比较重要。


###一、生成shader变种机制：

为了做实验，制作shader如下：
```c
Shader "Custom/Color"
{
        SubShader 
        {
            Pass 
            {
            Cull Off
            ZWrite Off
                        Lighting Off
 
                        CGPROGRAM
                        #pragma vertex vert
                        #pragma fragment frag
                        #include "UnityCG.cginc"
 
                        #pragma shader_feature RED GREEN BLUE
                        //#pragma shader_feature GREEN
                        //#pragma shader_feature BLUE
                        //#pragma multi_compile RED GREEN BLUE
                        //#pragma multi_compile __ GREEN
 
                        struct v2f 
                        {
                                fixed4 pos : SV_POSITION;
                        };
                         
                        v2f vert (appdata_base v)
                        {
                                v2f o;
                                o.pos = mul (UNITY_MATRIX_MVP, v.vertex);
                                return o;
                        }
                         
                        half4 frag(v2f i) : COLOR
                        {
                                fixed4 c = fixed4(0,0,0,1);
                                #ifdef RED
                                c += fixed4(1,0,0,1);
                                #endif
                                #ifdef GREEN
                                c += fixed4(0,1,0,1);
                                #endif
                                #ifdef BLUE
                                c += fixed4(0,0,1,1);
                                #endif
                                return c;
                        }
                        ENDCG
                }
        }
        CustomEditor "ColorsGUI"
}
```

shader的变种数量可以通过shader面板上面查看到，点击Show按钮可以看到都有哪些变种

![dock](https://raw.githubusercontent.com/liuxq/blog/master/images/ShaderVariants/SV_1.png)

做如下实验对比shader_feature：
单一行命令
 #pragma shader_feature RED
两个变种： __， RED
 #pragma shader_feature RED GREEN
两个变种：RED，GREEN
 #pragma shader_feature RED GREEN BLUE
三个变种：RED，GREEN，BLUE

 #pragma multi_compile RED
一个变种：RED
 #pragma multi_compile RED GREEN
两个变种：RED，GREEN
 #pragma multi_compile RED GREEN BLUE
三个变种：RED，GREEN，BLUE

分析：shader_feature 和multi_compile 在Keyword 数量大于1时，生成变种的机制是一样的，都是一个keyword一个变种；当keyword只有1个时，shader_feature 会增加一个none变种。再来做个实验：


 #pragma shader_feature \_ RED 
两个变种： none， RED
可见，当shader_feature 的keyword数量是1时不论是否有__符号，都会增加一个空keyword（__），除了这个在生成变种的机制上和multi_compile都是一致的。

多行命令
 #pragma multi_compile __ RED
 #pragma multi_compile __ GREEN
四个变种：__，RED，GREEN，RED GREEN
分析：多行命令就是单行命令的乘法组合，shader_feature和multi_compile除了单一keyword时是否补__之外，在多行命令中也是一致的。

###二、匹配shader变种机制
为了实验shader变种的匹配，做一个方便定义keyword的shader界面，代码如下：

```c
public class ColorsGUI: ShaderGUI {
 
    private static bool bRed = false;
    private static bool bGreen = false;
    private static bool bBlue = false;
 
        public override void OnGUI(MaterialEditor materialEditor, MaterialProperty[] properties)
        {
        // render the default gui
        base.OnGUI(materialEditor, properties);
 
        Material targetMat = materialEditor.target as Material;
 
        bRed = Array.IndexOf(targetMat.shaderKeywords, "RED") != -1;
        bGreen = Array.IndexOf(targetMat.shaderKeywords, "GREEN") != -1;
        bBlue = Array.IndexOf(targetMat.shaderKeywords, "BLUE") != -1;
 
        EditorGUI.BeginChangeCheck();
 
        bRed = EditorGUILayout.Toggle("红", bRed);
        bGreen = EditorGUILayout.Toggle("绿", bGreen);
        bBlue = EditorGUILayout.Toggle("蓝", bBlue);
 
        if (EditorGUI.EndChangeCheck())
        {
            if (bRed) 
                targetMat.EnableKeyword("RED");
            else
                targetMat.DisableKeyword("RED");
            if (bGreen)
                targetMat.EnableKeyword("GREEN");
            else
                targetMat.DisableKeyword("GREEN");
            if (bBlue)
                targetMat.EnableKeyword("BLUE");
            else
                targetMat.DisableKeyword("BLUE");
        }
        }
}
```

shader界面如下：
![dock](https://raw.githubusercontent.com/liuxq/blog/master/images/ShaderVariants/SV_2.png)

这样，勾选一个颜色，就会enable一个keyword，通过查看结果颜色就能知道匹配到了哪个shader变种，实验如下：
 #pragma multi_compile RED GREEN（两个变种：RED， GREEN）
材质keyword为RED :  显示红色（匹配RED）
材质keyword为GREEN :  显示绿色（匹配GREEN）
材质keyword为__ :  显示红色（匹配RED）
材质keyword为RED GREEN:  显示红色（匹配RED）

分析：当keyword存在正好匹配的变种时直接匹配、当keyword不存在匹配变种时取第一个变种

###三、shader变种打包
打包的代码如下：

```c
[MenuItem("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles()
    {
        List<AssetBundleBuild> maps = new List<AssetBundleBuild>();
        maps.Clear();
        //资源打包
 
        string[] files = {
                            "Assets/ShaderVariants/Resources/red.prefab",
                         };
 
        AssetBundleBuild build = new AssetBundleBuild();
        build.assetBundleName = "ShaderVariantsPrefab";
        build.assetNames = files;
        maps.Add(build);
 
        string[] file2s = {
                            "Assets/ShaderVariants/Resources/Shader/Colors.shader",
                          };
 
        AssetBundleBuild build2 = new AssetBundleBuild();
        build2.assetBundleName = "ShaderVariantsShader";
        build2.assetNames = file2s;
        maps.Add(build2);
 
        BuildAssetBundleOptions options = BuildAssetBundleOptions.DeterministicAssetBundle;
        BuildPipeline.BuildAssetBundles("Assets/StreamingAssets", maps.ToArray(), options, BuildTarget.StandaloneWindows);
 
        AssetDatabase.Refresh();
    }
```

读包并实例化的代码如下：

```c
public class BundleLoader : MonoBehaviour
{
        void Start () {
        load();
        }
 
    public void load()
    {
        StartCoroutine(LoadMainGameObject("file://" + Application.dataPath + "/StreamingAssets/" + "ShaderVariantsShader"));
        StartCoroutine(LoadMainGameObject("file://" + Application.dataPath + "/StreamingAssets/" + "ShaderVariantsPrefab"));
    }
 
    private IEnumerator LoadMainGameObject(string path)
    {
        WWW bundle = new WWW(path);
 
        yield return bundle;
 
        if (bundle.url.Contains("ShaderVariantsShader"))
        {
            //依赖shader包
            bundle.assetBundle.LoadAllAssets();
        }
        else
        {
            UnityEngine.Object obj = bundle.assetBundle.LoadAsset("assets/ShaderVariants/resources/red.prefab");
            Instantiate(obj);
            bundle.assetBundle.Unload(false);
        } 
    }
}
```

为了对比shader_feature 和multi_compile 以及shader依赖和非依赖打包，做如下实验：


 #pragma shader_feature RED GREEN BLUE
将选中RED关键字的prefab打包，加载bundle和其中的prefab，显示了红色，此时改变此材质的keyword为GREEN或者BLUE，没有效果
 #pragma multi_compile RED GREEN BLUE
将选中RED关键字的prefab打包，加载bundle和其中的prefab，显示了红色，此时改变此材质的keyword为GREEN或者BLUE，可以显示绿色和蓝色
分析：shader_feature声明变种时，打包只会打包被资源引用的keyword变种，multi_compile声明变种时，打包会把所有变种都打进去

 #pragma shader_feature RED GREEN BLUE
将选中RED关键字的prefab和shader依赖打包，加载bundle和其中的prefab，显示了异常粉红，任何变种都没有生效
分析：shader_feature标记的shader单独依赖打包时，任何变种都不会打进去，分析原因估计是unity认为单包中shader没有被引用过

###总结：
unity5中新出的shader_feature可以只将引用过的shader变种打进包里面，听起来很有用，可是大部分项目中为了节省冗余shader的内存，shader都是作为依赖包单独成一包的，此时没有任何shader变种被打进包中；更何况即使shader没有依赖打包，如果计划代码中动态修改shader的变种而不是记录在材质里面，此时也不能用shader_feature。基本上我们的项目中shader_feature可以废弃了。。。