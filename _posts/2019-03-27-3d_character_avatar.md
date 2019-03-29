---
layout: post
title: "3D人物换装"
description: "Unity动态合并网格和贴图, 实现3D Avatar功能."
tags: [Unity]
---

3D游戏中换装是一个常见的功能. 从实现上来说主要分为两类: 部件换装 和 身体换装.

部件换装常用于更换武器、挂件这类情况. 这种实现起来比较简单，只要将模型挂在指定挂点下即可。  
身体换装是更换身体的不同部位, 也是本篇主要讨论的换装. 身体换装的原理是将角色模型拆分成多个部分,然后通过设置加载不同的部位模型组合起来显示.

在接下来的演示中,我们将模型拆分成头部,身体,手部和脚部. 为了让人物动画可以在这些部位上播放, 要保证所有部位都使用一套骨骼.

## 0x01 最简单的方式

最简单的方式是一个人物包含所有部位的物体, 每个部位上有一个独立的Animator控制播放动画, 所有部位同时播放动画看起来就像是一个整体.

![sep_part](img/character_avatar/sep_part.gif)

这种方式实现起来最简单, 但是显然不能满足我们的需求, 主要有几方面的问题:

1. 使用了多个Animator导致性能上有额外开销.
2. 每个部件会占用多个DrawCall

## 0x02 合并网格

为了解决上述问题, 我们可以将所有部件的mesh, texture合并, 这样就只保留了一个最终的"大"部件, 也只需要使用1个Animator和1个DrawCall.

### 0x02-1 理解SubMesh

一个金属桌子上铺了一块毯子, 金属的材质和毯子的肯定不同, 如何在一个mesh上渲染出不同的材质效果呢?  
Unity提供了SubMesh来实现这样的功能. 众所周知, Mesh的主要数据有Vertex,Triangle等. 我们可以使用submesh实现正方形一个角是红色,一个角是蓝色:

```c#
private void Awake()
{
    Mesh mesh = new Mesh();
    mesh.subMeshCount = 2;
    mesh.vertices = new Vector3[]
    {
        new Vector3(-1,-1),
        new Vector3(-1,1),
        new Vector3(1,1),
        new Vector3(1,-1),
    };

    int[] triangles = new int[] { 0, 1, 3, };
    mesh.SetTriangles(triangles, 0);    //submesh1

    triangles = new int[] { 1, 2, 3 };
    mesh.SetTriangles(triangles, 1);    //submesh2

    gameObject.AddComponent<MeshFilter>().mesh = mesh;
    MeshRenderer mr = gameObject.AddComponent<MeshRenderer>();
    mr.materials = new Material[] { mat1, mat2 };   //每个submesh对应一个material
}
```
![submesh](img/character_avatar/submesh.png)

### 0x02-2 开始合并

我们使用官方提供的`Meth.CombineMeshes`来合并网格.

```c#
private void CombineMesh()
{
    //收集所有部件的CombineInstance
    List<CombineInstance> cis = new List<CombineInstance>();
    cis.AddRange(GetCombineInstance(head));
    cis.AddRange(GetCombineInstance(body));
    cis.AddRange(GetCombineInstance(foot));
    cis.AddRange(GetCombineInstance(hand));

    SkinnedMeshRenderer ssr = skeleton.AddComponent<SkinnedMeshRenderer>();
    ssr.sharedMesh = new Mesh();
    ssr.sharedMesh.CombineMeshes(cis.ToArray(), false, false); // 这里不能使用Mesh.CombineMeshes(CombineInstance[]) 这个重载版本, 因为它默认的会使用matrix, 而下面我们没设置CombineInstance的matrix, 会导致合并完的mesh什么都看不见(matrix默认缩放为0).
}

private List<CombineInstance> GetCombineInstance(GameObject obj)
{
    List<CombineInstance> cis = new List<CombineInstance>();
    SkinnedMeshRenderer ssr = obj.GetComponentInChildren<SkinnedMeshRenderer>();
    for (int i = 0; i < ssr.sharedMesh.subMeshCount; i++)
    {
        //将每个submesh记录下来
        CombineInstance ci = new CombineInstance();
        ci.mesh = ssr.sharedMesh;
        ci.subMeshIndex = i;
        cis.Add(ci);
    }
    return cis;
}
```

这段代码非常简单, 只做了两件事:
1. 根据所有部件的mesh生成CombineInstance, 每个submesh生成一个.
2. 创建一个新的mesh,调用CombineMeshes.

下图演示了部件合并以后的效果.

![combine_mesh](img/character_avatar/combine_mesh.png)

### 0x02-3 设置骨骼

现在虽然mesh合并完了, 但是在unity中看不见任何效果, 原因是并没有设置骨骼.  

```c#
List<Transform> newBones = new List<Transform>();
newBones.AddRange(hand.GetComponentInChildren<SkinnedMeshRenderer>().bones);
newBones.AddRange(head.GetComponentInChildren<SkinnedMeshRenderer>().bones);
newBones.AddRange(foot.GetComponentInChildren<SkinnedMeshRenderer>().bones);
newBones.AddRange(body.GetComponentInChildren<SkinnedMeshRenderer>().bones);
ssr.bones = newBones.ToArray();
```
![set_bones](img/character_avatar/set_bones.png)

### 0x02-4 设置材质

现在能看到东西了, 但是有一点比较奇怪, 只有手部,其他部位都看不见.  
还记得之前提到的SubMesh吗? 每个SubMesh对应一个Material. 我们合并网格的时候,每个部位都有一个(也有可能不止)SubMesh, 所以也需要对应数量的Material.

```c#
List<Material> materials = new List<Material>();
materials.AddRange(hand.GetComponentInChildren<SkinnedMeshRenderer>().sharedMaterials); //这里要用sharedMaterials避免创建新的材质.
materials.AddRange(head.GetComponentInChildren<SkinnedMeshRenderer>().sharedMaterials);
materials.AddRange(foot.GetComponentInChildren<SkinnedMeshRenderer>().sharedMaterials);
materials.AddRange(body.GetComponentInChildren<SkinnedMeshRenderer>().sharedMaterials);
ssr.materials = materials.ToArray();
```

![set_material](img/character_avatar/set_material.png)

## 0x03 优化

目前已经实现了换装功能, 但是使用了多个材质使得DrawCall比较高, 现在我们将材质合并一下.

## 0x03-1 合并材质

```c#
//收集Texture
List<Texture2D> textures = new List<Texture2D>();
foreach (var mat in materials)
{
    textures.Add(mat.GetTexture("_MainTex") as Texture2D);
}
//合并Texture
newTex = new Texture2D(COMBINE_TEX_SIZE, COMBINE_TEX_SIZE, TextureFormat.ARGB32, true);
Rect[] uvs = newTex.PackTextures(textures.ToArray(), 0);
//重新计算UV
List<Vector2[]> oldUV = new List<Vector2[]>();
Vector2[] uva, uvb;
for (int ciIndex = 0; ciIndex < cis.Count; ciIndex++)
{
    CombineInstance ci = cis[ciIndex];
    uva = ci.mesh.uv;
    uvb = new Vector2[uva.Length];
    for (int i = 0; i < uva.Length; i++)
    {
        //UV映射
        uvb[i] = new Vector2(uva[i].x * uvs[ciIndex].width + uvs[ciIndex].x, uva[i].y * uvs[ciIndex].height + uvs[ciIndex].y);
    }
    oldUV.Add(ci.mesh.uv);
    ci.mesh.uv = uvb;
}

if (ssr != null)
{
    DestroyImmediate(ssr);
}

ssr = skeleton.AddComponent<SkinnedMeshRenderer>();
ssr.sharedMesh = new Mesh();
ssr.sharedMesh.CombineMeshes(cis.ToArray(), true, false);
ssr.bones = newBones.ToArray();
//设置材质
ssr.material = new Material(Shader.Find("Diffuse"));
ssr.material.mainTexture = newTex;
for (int ciIndex = 0; ciIndex < cis.Count; ciIndex++)
{
    CombineInstance ci = cis[ciIndex];
    ci.mesh.uv = oldUV[ciIndex];    //恢复原始部件mesh中的uv.
}
```

注意: 因为每个SubMesh都对应一个Material, 而合并材质后就只有一个Material了, 所以在CombineMesh那一步需要在第二个参数传递`true`, 这样Unity会将所有SubMesh合并成一个,也就只需要一个Material了.

![combine_material](img/character_avatar/combine_material.png)


## 0x04 完整代码

将之前的代码整理一下后,给出完整代码如下:

```c#
public class CharacterAvatar : MonoBehaviour
{
    public const int COMBINE_TEX_SIZE = 1024;

    public GameObject skeleton;
    public GameObject head;
    public GameObject body;
    public GameObject foot;
    public GameObject hand;

    private SkinnedMeshRenderer ssr;

    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.Q))
        {
            CombineMesh(head.GetComponentInChildren<SkinnedMeshRenderer>(),
                        hand.GetComponentInChildren<SkinnedMeshRenderer>(),
                        body.GetComponentInChildren<SkinnedMeshRenderer>(),
                        foot.GetComponentInChildren<SkinnedMeshRenderer>());
        }
    }

    private void CombineMesh(params SkinnedMeshRenderer[] skinnedMeshRenderers)
    {
        Transform[] skeletonBones = skeleton.GetComponentsInChildren<Transform>(true);
        List<CombineInstance> combineInstances = new List<CombineInstance>();
        List<Transform> bones = new List<Transform>();
        List<Texture2D> textures = new List<Texture2D>();
        foreach (var ssr in skinnedMeshRenderers)
        {
            for (int i = 0; i < ssr.sharedMesh.subMeshCount; i++)
            {
                CombineInstance ci = new CombineInstance();
                ci.mesh = ssr.sharedMesh;
                ci.subMeshIndex = i;
                combineInstances.Add(ci);               //收集网格
            }

            // 收集骨骼
            // 这里不能按照上面的直接使用ssr.bones,否则动画将不生效.
            foreach (var b in ssr.bones)
            {
                Transform bone = Array.Find(skeletonBones, (t) => t.name == b.name);
                if (bone != null)
                {
                    bones.Add(bone);
                }
            }

            foreach (var mat in ssr.sharedMaterials)    //收集贴图
            {
                textures.Add(mat.GetTexture("_MainTex") as Texture2D);
            }
        }

        //合并贴图
        Texture2D texture = new Texture2D(COMBINE_TEX_SIZE, COMBINE_TEX_SIZE, TextureFormat.ARGB32, true);
        Rect[] uvs = texture.PackTextures(textures.ToArray(), 0);

        //UV重映射
        List<Vector2[]> oldUVs = new List<Vector2[]>();
        Vector2[] oldUV, newUV;
        for (int ciIndex = 0; ciIndex < combineInstances.Count; ciIndex++)
        {
            CombineInstance ci = combineInstances[ciIndex];
            oldUV = ci.mesh.uv;
            newUV = new Vector2[oldUV.Length];
            for (int uvIndex = 0; uvIndex < oldUV.Length; uvIndex++)
            {
                newUV[uvIndex] = new Vector2(oldUV[uvIndex].x * uvs[ciIndex].width + uvs[ciIndex].x, oldUV[uvIndex].y * uvs[ciIndex].height + uvs[ciIndex].y);
            }
            oldUVs.Add(ci.mesh.uv);     //保存原始uv,用于恢复原mesh
            ci.mesh.uv = newUV;         //设置新uv,用于新mesh
        }

        if (ssr != null)
        {
            DestroyImmediate(ssr);
        }
        ssr = skeleton.AddComponent<SkinnedMeshRenderer>();
        ssr.sharedMesh = new Mesh();
        ssr.sharedMesh.CombineMeshes(combineInstances.ToArray(), true, false);
        ssr.bones = bones.ToArray();
        ssr.material = new Material(Shader.Find("Diffuse"));
        ssr.material.mainTexture = texture;
        for (int i = 0; i < combineInstances.Count; i++)
        {
            combineInstances[i].mesh.uv = oldUVs[i];
        }
    }
}
```

## 0x05 最终效果

![character_avatar](img/character_avatar/character_avatar.gif)

---

**参考**

>[解读unity3d角色换装](https://gameinstitute.qq.com/community/detail/126729)
