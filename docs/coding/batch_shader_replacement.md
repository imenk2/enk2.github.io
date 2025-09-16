---
layout: default
---

# Unity 批量替换Shader
```cpp
public class ChangeShader : EditorWindow
{
    [MenuItem("*Tool*/Change Shader")]
    public static void ShowWindow()
    {
        EditorWindow editorWindow = GetWindow(typeof(ChangeShader));
        editorWindow.autoRepaintOnSceneChange = false;
    }

    //当前shader
    public Shader currentShader;
    //目标shader
    public Shader changeShader;
    public HashSet<Material> mat = new HashSet<Material>();

    private void OnGUI()
    {
        currentShader = EditorGUILayout.ObjectField("查找 shader", currentShader, typeof(Shader),false) as Shader;
        changeShader = EditorGUILayout.ObjectField("替换 shader", changeShader, typeof(Shader), false) as Shader;
        if (GUILayout.Button("Change"))
        {
            Change();
        }

        if (mat.Count > 0)
        {
        //界面中显示被替换的材质球
            EditorGUILayout.LabelField("替换掉的材质：");
            foreach (var material in mat)
            {
                if (material != null)
                    EditorGUILayout.ObjectField(material, typeof(Material), false);
            }

            if (GUILayout.Button("Clear"))
                mat.Clear();
        }
    }

    public void Change()
    {
        mat.Clear();
        // 加载的路径需要加后缀名
        if (currentShader == null || changeShader == null)
        {
            return;
        }

        var allfiles = Directory.GetFiles("Assets/", "*.mat", SearchOption.AllDirectories);
        var length = allfiles.Length;
        var count = 0;
        for (var i = 0; i < length; i++)
        {
            var file = allfiles[i];
            //显示替换进度
            if (EditorUtility.DisplayCancelableProgressBar("Changing", file, i / (float) length))
            {
                break;
            }
			//根据地址读取材质球
            Material tempMat = AssetDatabase.LoadAssetAtPath<Material>(file);
            if (tempMat.shader == currentShader)
            {
                tempMat.shader = changeShader;
                mat.Add(tempMat);
                // Debug.Log($"Change {tempMat.name} from {currentShader.name} to {changeShader.name}");
                count++;
            }
        }

        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
        EditorUtility.ClearProgressBar();
        EditorUtility.DisplayDialog("一共修改:" + count + "个材质", "就绪", "OK");
    }
}
```

[back](../../coding-page.html)