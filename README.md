    #ProceduralGrass
    [Range(0, 1000)]
    public int terrainSize = 250;
    [Range(0, 100f)]
    public float terrainHeight = 10f;
    private float xOffset;
    private float zOffset;

    [Range(1f, 100f)]
    public float scaleFatter = 10f;
    [Range(1f, 100f)]
    public float offsetFatter = 10f;

    public Material terrainMat;

    List<Vector3> vertexs = new List<Vector3>();
    List<int> triangles = new List<int>();
    float[,] perlinNoise;
    //存储地形的顶点法线
    Vector3[] terrainNormals;

    /// <summary>
    /// Grass Data
    /// </summary>
    //草根集,定义草的广度
    [Range(0, 100)]
    public int grassRowCount = 50;
    //定义每一堆 草根集 的草密度
    [Range(1, 1000)]
    public int grassCountPerPatch = 100;
    //草的材质
    public Material grassMat;
    //存储草的顶点
    List<Vector3> grassVerts = new List<Vector3>();
    //存储草的顶点法线
    Vector3[] grassNormals;
    //存储草的顶点法线列表
    List<Vector3> grassNormalList = new List<Vector3>();

    /// <summary>
    /// GrassDrawInstance
    /// </summary>
    //定义草的网格
    public Mesh grassMesh;
    List<Matrix4x4[]> grassMatrixList = new List<Matrix4x4[]>();
    Vector4[] grassColors = new Vector4[1023];
    Vector4[] grassPos = new Vector4[1023];
    //DrawMeshInstanced 方法
    //将每一个草顶点的 Transform 信息传入 matrixs 列表里
    //且数目不能超过 1023
    Matrix4x4[] grassMatrix = new Matrix4x4[1023];
    Quaternion rotation = Quaternion.Euler(Vector3.zero);
    Vector3 scale = Vector3.one;

    void Start()
    {
        terrainNormals = new Vector3[terrainSize * terrainSize];
        perlinNoise = new float[terrainSize, terrainSize];
        GenerateTerrain();
        GenerateGrassArea(grassRowCount, grassCountPerPatch);
    }

    void Update()
    {
        //DrawGrassMeshInstance();
    }

    //生成地形网格数据
    void CreateVertsAndTris()
    {
        //从左往右遍历每一个顶点，并用 vertexs 存储
        for (int i = 0; i < terrainSize; i++)
        {
            for (int j = 0; j < terrainSize; j++)
            {
                //float noiseHeight = Mathf.PerlinNoise(xCoord, zCoord);

                float noiseHeight = GeneratePerlinNoise(i, j);

                perlinNoise[i, j] = noiseHeight;

                //print(noiseHeight * terrainHeight);
                vertexs.Add(new Vector3(i, noiseHeight * terrainHeight, j));

                //不算上坐标轴的顶点
                if (i == 0 || j == 0)
                    continue;

                //每三个顶点作为一个索引建立三角形
                triangles.Add(terrainSize * i + j);
                triangles.Add(terrainSize * i + j - 1);
                triangles.Add(terrainSize * (i - 1) + j - 1);
                triangles.Add(terrainSize * (i - 1) + j - 1);
                triangles.Add(terrainSize * (i - 1) + j);
                triangles.Add(terrainSize * i + j);

            }
        }
    }

    //生成地形的随机 Perlin 值
    float GeneratePerlinNoise(int i, int j)
    {
        float xCoord = (float)i / terrainSize * scaleFatter + offsetFatter;
        float zCoord = (float)j / terrainSize * scaleFatter + offsetFatter;

        return Mathf.PerlinNoise(xCoord, zCoord);
    }

    //生成地形
    void GenerateTerrain()
    {
        CreateVertsAndTris();

        //计算UV
        Vector2[] uvs = new Vector2[vertexs.Count];
        for (int i = 0; i < vertexs.Count; i++)
        {
            uvs[i] = new Vector2(vertexs[i].x, vertexs[i].z);
        }

        //添加一个 MyTerrain 的物体，并添加 MeshFilter / MeshRenderer 两个属性
        //MeshFilter 存储物体的网格信息
        //MeshRenderer 负责接收这些信息并把这些信息渲染出来
        GameObject Myterrain = new GameObject("Terrain0");
        Myterrain.AddComponent<MeshFilter>();
        MeshRenderer renderer = Myterrain.AddComponent<MeshRenderer>();
        //开启地面的阴影投射和接受
        renderer.receiveShadows = true;
        renderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.On;
        MeshCollider collider = Myterrain.AddComponent<MeshCollider>();
        renderer.sharedMaterial = terrainMat;

        //创建一个 Mesh 网格
        Mesh groundMesh = new Mesh();
        groundMesh.vertices = vertexs.ToArray();
        groundMesh.triangles = triangles.ToArray();
        groundMesh.uv = uvs;
        groundMesh.RecalculateNormals();
        terrainNormals = groundMesh.normals;
        Myterrain.GetComponent<MeshFilter>().mesh = groundMesh;
        collider.sharedMesh = groundMesh;

        grassVerts.Clear();
    }

    //生成草的网格数据
    void GenerateGrassArea(int rowCount, int perPatchSize)
    {
        //Unity 一个网格能包含的最大顶点数为 65535
        List<int> indices = new List<int>();
        for (int i = 0; i < 65000; i++)
        {
            indices.Add(i);
        }

        //初始位置
        Vector3 currentPos = Vector3.zero;
        //草根集 每一次循环偏移的距离
        Vector3 patchSize = new Vector3(terrainSize / rowCount, 0, terrainSize / rowCount);

        //每一堆 草根集 进行循环
        for (int i = 0; i < rowCount; i++)
        {
            for (int j = 0; j < rowCount; j++)
            {
                GenerateGrass(currentPos, patchSize, grassCountPerPatch);
                currentPos.x += patchSize.x;

                //mm++;
                ////当存储的顶点量达到 1022 个
                ////重新增加一个可以装载 1023 个顶点的 grassMatrix 数组
                //if (mm % 1022 == 0)
                //{
                //    grassMatrixList.Add(new Matrix4x4[1023]);
                //    grassMatrixList[grassMatrixList.Count - 1] = grassMatrix;
                //    grassMatrix = new Matrix4x4[1023];
                //    mm = 0;
                //}
            }
            currentPos.x = 0;
            currentPos.z += patchSize.z;
        }

        //for (int i = 0; i < 1023; i++)
        //{
        //    float r = Random.Range(0.160f, 0.165f);
        //    float b = Random.Range(0.290f, 0.360f);
        //    grassColors[i] = new Vector4(r, .945f, b, 1);
        //}

        //生成 GrassLayerGruop 来成为父级管理物理
        GameObject grassLayerGroup0 = new GameObject("GrassLayerGroup0");
        //生成 GrassLayer 物体来存储草数据
        GameObject grassLayer;
        MeshFilter grassMeshFilter;
        //Mesh grassMesh;
        MeshRenderer grassMeshRenderer;
        int a = 0;

        ////DrawMeshInstanced 方法
        ////将每一个草顶点的 Transform 信息传入 matrixs 列表里
        ////且数目不能超过 1023
        //Matrix4x4[] grassMatrix = new Matrix4x4[1023];
        //Quaternion rotation = Quaternion.Euler(Vector3.zero);
        //Vector3 scale = Vector3.one;
        //int mm = 0;

        ////print(grassVerts[0]);
        ////print(grassVerts[1]);

        //for (int i = 0; i < grassVerts.Count; i++)
        //{
        //    //grassMatrix 数组存储每一个顶点的 Transform 信息
        //    grassMatrix[mm] = Matrix4x4.TRS(grassVerts[mm], rotation, scale);
        //    mm++;

        //    //当存储的顶点量达到 1022 个
        //    //重新增加一个可以装载 1023 个顶点的 grassMatrix 数组
        //    if (mm % 1022 == 0)
        //    {
        //        grassMatrixList.Add(new Matrix4x4[1023]);
        //        grassMatrixList[grassMatrixList.Count - 1] = grassMatrix;
        //        grassMatrix = new Matrix4x4[1023];
        //        mm = 0;
        //    }
        //}
        //for (int i = 0; i < 1023; i++)
        //{
        //    float r = Random.Range(0.160f, 0.165f);
        //    float b = Random.Range(0.290f, 0.360f);
        //    grassColors[i] = new Vector4(r, .945f, b, 1);
        //}

        /*Matrix4x4[] one = grassMatrixList[0];
        for (int i = 0; i < one.Length; i++)
        {
            Matrix4x4 oneNum;
            oneNum = one[i];
            print(oneNum);
        }

        grassMesh = new Mesh();
        grassLayer = new GameObject("grassLayer");
        grassMeshFilter = grassLayer.AddComponent<MeshFilter>();
        grassMeshRenderer = grassLayer.AddComponent<MeshRenderer>();
        //开启草地的阴影投射和接受
        grassMeshRenderer.receiveShadows = true;
        grassMeshRenderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.On;
        grassMeshFilter.mesh = grassMesh;
        grassMeshRenderer.sharedMaterial = grassMat;*/

        //------------------------------------------------------
        //当 grassVerts.Count 的数量即草的全部顶点数超过 65000 个时
        //创立多个网格处理
        while (grassVerts.Count > 65000)
        {
            grassMesh = new Mesh();
            grassMesh.vertices = grassVerts.GetRange(0, 65000).ToArray();

            //存储每个草顶点的法线（当顶点超过 65000个）
            grassNormals = new Vector3[65000];
            grassNormalList.GetRange(0, 65000);
            for (int i = 0; i < 65000; i++)
            {
                grassNormals[i] = grassNormalList[i];
            }

            //设置子网格的索引缓冲区,相关官方文档：https://docs.unity3d.com/ScriptReference/Mesh.SetIndices.html
            //每一个创建的网格的顶点数目不会超过 65000 个
            grassMesh.SetIndices(indices.ToArray(), MeshTopology.Points, 0);

            //创建一个新的 GameObject 来承载这些点
            grassLayer = new GameObject("GrassLayer " + a++);
            grassLayer.transform.SetParent(grassLayerGroup0.transform);
            grassMeshFilter = grassLayer.AddComponent<MeshFilter>();
            grassMeshRenderer = grassLayer.AddComponent<MeshRenderer>();
            //开启草地的阴影投射和接受
            grassMeshRenderer.receiveShadows = true;
            grassMeshRenderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.On;
            grassMeshRenderer.sharedMaterial = grassMat;
            //grassMesh 网格是由点组成（线也一样）不能自动生成法线 !!!!!
            //grassMesh.RecalculateNormals();
            grassMesh.normals = grassNormals;
            grassMeshFilter.mesh = grassMesh;
            //移除前 65000 个顶点
            grassVerts.RemoveRange(0, 65000);

            //移除前 65000 个法线信息
            grassNormalList.RemoveRange(0, 65000);
        }

        //当 grassVerts.Count 的数量即草的全部顶点数没有超过 65000 个时
        grassLayer = new GameObject("GrassLayer" + a);
        grassLayer.transform.SetParent(grassLayerGroup0.transform);
        grassMeshFilter = grassLayer.AddComponent<MeshFilter>();
        grassMeshRenderer = grassLayer.AddComponent<MeshRenderer>();
        //开启草地的阴影投射和接受
        grassMeshRenderer.receiveShadows = true;
        grassMeshRenderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.On;
        grassMesh = new Mesh();
        grassMesh.vertices = grassVerts.ToArray();
        //设立子网格数据
        grassMesh.SetIndices(indices.GetRange(0, grassVerts.Count).ToArray(), MeshTopology.Points, 0);
        //grassMesh 网格是由点组成（线也一样）不能自动生成法线 !!!!!
        //grassMesh.RecalculateNormals();
        //存储每个草顶点的法线（当顶点没有超过 65000个）
        grassNormals = new Vector3[grassMesh.vertexCount];
        grassNormalList.GetRange(0, grassMesh.vertexCount);
        for (int i = 0; i < grassMesh.vertexCount; i++)
        {
            grassNormals[i] = grassNormalList[i];
        }
        grassMesh.normals = grassNormals;
        grassMeshFilter.mesh = grassMesh;
        grassMeshRenderer.sharedMaterial = grassMat;
    }

    //生成草
    void GenerateGrass(Vector3 vertPos, Vector3 patchSize, int grassCountPerPatch)
    {
        //每一堆 草根集 里的草进行循环
        for (int i = 0; i < grassCountPerPatch; i++)
        {
            //Random.value 返回 0~1 之间的随机值
            //得到在两个 草根集 之间的草的随机位置并用索引值
            float randomX = Random.value * patchSize.x;
            float randomZ = Random.value * patchSize.z;

            int indexX = (int)(vertPos.x + randomX);
            int indexZ = (int)(vertPos.z + randomZ);

            //防止草种出地形
            if(indexX >= terrainSize)
            {
                indexX = (int)terrainSize - 1;
            }

            if (indexZ >= terrainSize)
            {
                indexZ = (int)terrainSize - 1;
            }

            //添加每一个草的顶点位置到 grassVert 列表里
            grassVerts.Add(new Vector3(vertPos.x + randomX, perlinNoise[indexX, indexZ] * terrainHeight , vertPos.z + randomZ));
            //添加每一个草的顶点法线到 grassNormalList 列表里
            grassNormalList.Add(terrainNormals[indexX]);

            // DrawInstance 方法
            //grassMatrix 数组存储每一个顶点的 Transform 信息
            //grassMatrix[mm] = Matrix4x4.TRS(new Vector3(vertPos.x + randomX, perlinNoise[indexX, indexZ] * terrainHeight, vertPos.z + randomZ)
            //    , rotation, scale);
        }
    }

    //利用 DrawMeshInstanced 方法绘制草的网格
    void DrawGrassMeshInstance()
    {
        MaterialPropertyBlock grassMatBlock = new MaterialPropertyBlock();

        for (int i = 0; i < 1023; i++)
        {
            grassPos[i] = new Vector4(1, 1, 1, 1);
        }

        grassMatBlock.SetVectorArray("_Color", grassColors);

        foreach (Matrix4x4[] matrix in grassMatrixList)
        {
            //Graphics.DrawMeshInstanced(grassMesh, 0, grassMat, matrix);
            Graphics.DrawMeshInstanced(grassMesh, 0, grassMat, matrix, 1023, grassMatBlock, UnityEngine.Rendering.ShadowCastingMode.On, true);
            //print("Yes");
        }
    }

    //生成高度图
    Texture2D GenerateHeightMap()
    {
        float sample;
        float xCoord, zCoord;
        Color noiseColor;
        Texture2D heightMap = new Texture2D(terrainSize, terrainSize);

        for (int i = 0; i < terrainSize; i++)
        {
            for (int j = 0; j < terrainSize; j++)
            {
                // scaleFatter，offsetFatter 分别控制贴图的缩放和偏移
                xCoord = (float)i / terrainSize * scaleFatter + offsetFatter;
                zCoord = (float)j / terrainSize * scaleFatter + offsetFatter;

                sample = Mathf.PerlinNoise(xCoord, zCoord);
                noiseColor = new Color(sample, sample, sample);

                heightMap.SetPixel(i, j, noiseColor);
            }
        }

        heightMap.Apply();
        return heightMap;
    }

    //画Gizmos
    //private void OnDrawGizmos()
    //{
    ////    //if (vertexs.Count == 0)
    ////    //    return;

    ////    //for (int i = 0; i < vertexs.Count; i++)
    ////    //{
    ////    //    Gizmos.DrawSphere(vertexs[i], 0.2f);
    ////    //}

    //    if (grassVerts.Count == 0)
    //        return;

    //    for (int i = 0; i<grassVerts.Count; i++)
    //    {
    //        Gizmos.DrawSphere(grassVerts[i], 0.01f);
    //    }
    //}
