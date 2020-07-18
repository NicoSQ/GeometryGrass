    #Procedural Grass
    #region Terrain Data
    [Range(0, 1000)]
    public int terrainSize = 250;          //地形大小          
    [Range(0, 100f)]
    public float terrainHeight = 10f;          //地形高度              
    private float xOffset;
    private float zOffset;

    [Range(1f, 100f)]
    public float scaleFatter = 10f;          //网格缩放
    [Range(1f, 100f)]
    public float offsetFatter = 10f;          //网格偏移                              

    public Material terrainMat;          //地形材质
    List<Vector3> vertexs = new List<Vector3>();          //网格顶点信息
    List<int> triangles = new List<int>();          //网格三角形信息
    float[,] perlinNoise;          //存储每个顶点的高度值
    Vector3[] terrainNormals;          //存储地形的顶点法线
    #endregion

    #region Grass Data
    [Range(0, 100)]
    public int grassRowCount = 50;          //草根集,定义草的广度
    [Range(1, 1000)]
    public int grassCountPerPatch = 100;          //定义每一堆 草根集 的草密度
    public Material grassMat;           //草的材质
    public Mesh grassMesh;          //草的网格
    List<Vector3> grassVerts = new List<Vector3>();          //存储草的顶点
    Vector3[] grassNormals;          //存储草的顶点法线
    List<Vector3> grassNormalList = new List<Vector3>();          //存储草的顶点法线列表
    #endregion

    void Start()
    {
        xOffset = transform.position.x;
        zOffset = transform.position.z;

        terrainNormals = new Vector3[terrainSize * terrainSize];
        perlinNoise = new float[terrainSize, terrainSize];
        GenerateTerrain();
        GenerateGrassArea(grassRowCount, grassCountPerPatch);
    }

    //生成地形网格数据
    void CreateVertsAndTris()
    {
        //从左往右遍历每一个顶点，并用 vertexs 存储
        for (int i = 0; i < terrainSize; i++)
        {
            for (int j = 0; j < terrainSize; j++)
            {
                float noiseHeight = GeneratePerlinNoise(i, j);

                perlinNoise[i, j] = noiseHeight;

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
        float xCoord = (float)(i + xOffset) / terrainSize * scaleFatter + offsetFatter;
        float zCoord = (float)(j + zOffset) / terrainSize * scaleFatter + offsetFatter;

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
        Myterrain.transform.position = this.gameObject.transform.position;
        Myterrain.AddComponent<MeshFilter>();
        MeshRenderer renderer = Myterrain.AddComponent<MeshRenderer>();
        //开启地面的阴影投射和接受
        renderer.receiveShadows = true;
        renderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.On;
        MeshCollider collider = Myterrain.AddComponent<MeshCollider>();
        renderer.sharedMaterial = terrainMat;

        //创建一个 Mesh 网格
        Mesh groundMesh = new Mesh();
        //输入网格顶点数据
        groundMesh.vertices = vertexs.ToArray();
        //输入网格三角面数据
        groundMesh.triangles = triangles.ToArray();
        groundMesh.uv = uvs;
        //为了得到正确的光照需要重新计算得到正确的法线信息
        groundMesh.RecalculateNormals();
        terrainNormals = groundMesh.normals;
        Myterrain.GetComponent<MeshFilter>().mesh = groundMesh;
        collider.sharedMesh = groundMesh;

        grassVerts.Clear();
    }

    //生成草的网格数据
    void GenerateGrassArea(int rowCount, int perPatchSize)
    {
        //最大顶点数为 65535
        List<int> indices = new List<int>();
        for (int i = 0; i < 65000; i++)
        {
            indices.Add(i);
        }

        //初始位置
        Vector3 currentPos = transform.position;
        //草根集 每一次循环偏移的距离
        Vector3 patchSize = new Vector3(terrainSize / rowCount, 0, terrainSize / rowCount);

        //每一堆 草根集 进行循环
        for (int i = 0; i < rowCount; i++)
        {
            for (int j = 0; j < rowCount; j++)
            {
                GenerateGrass(currentPos, patchSize, grassCountPerPatch);
                currentPos.x += patchSize.x;
            }
            currentPos.x = transform.position.x;
            currentPos.z += patchSize.z;
        }

        //生成 GrassLayerGruop 来成为父级管理物理
        GameObject grassLayerGroup1 = new GameObject("GrassLayerGroup1");
        //生成 GrassLayer 物体来存储草数据
        GameObject grassLayer;
        MeshFilter grassMeshFilter;
        MeshRenderer grassMeshRenderer;
        int a = 0;

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
            grassLayer.transform.SetParent(grassLayerGroup1.transform);
            grassMeshFilter = grassLayer.AddComponent<MeshFilter>();
            grassMeshRenderer = grassLayer.AddComponent<MeshRenderer>();
            //关闭草地的阴影投射和接受
            grassMeshRenderer.receiveShadows = false;
            grassMeshRenderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.Off;
            grassMeshRenderer.sharedMaterial = grassMat;
            grassMeshFilter.mesh = grassMesh;
            //移除前 65000 个顶点
            grassVerts.RemoveRange(0, 65000);

            //移除前 65000 个法线信息
            grassNormalList.RemoveRange(0, 65000);
        }

        //当 grassVerts.Count 的数量即草的全部顶点数没有超过 65000 个时
        grassLayer = new GameObject("GrassLayer" + a);
        grassLayer.transform.SetParent(grassLayerGroup1.transform);
        grassMeshFilter = grassLayer.AddComponent<MeshFilter>();
        grassMeshRenderer = grassLayer.AddComponent<MeshRenderer>();
        //关闭草地的阴影投射和接受
        grassMeshRenderer.receiveShadows = false;
        grassMeshRenderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.Off;
        grassMesh = new Mesh();
        grassMesh.vertices = grassVerts.ToArray();
        //设立子网格数据
        grassMesh.SetIndices(indices.GetRange(0, grassVerts.Count).ToArray(), MeshTopology.Points, 0);
        
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

            int indexX = (int)((vertPos.x - transform.position.x) + randomX);
            int indexZ = (int)((vertPos.z - transform.position.z) + randomZ);

            //防止草种出地形
            if (indexX >= terrainSize)
            {
                indexX = (int)terrainSize - 1;
            }

            if (indexZ >= terrainSize)
            {
                indexZ = (int)terrainSize - 1;
            }

            //添加每一个草的顶点位置到 grassVert 列表里
            grassVerts.Add(new Vector3(vertPos.x + randomX, perlinNoise[indexX, indexZ] * terrainHeight, vertPos.z + randomZ));
            //添加每一个草的顶点法线到 grassNormalList 列表里
            grassNormalList.Add(terrainNormals[indexX]);
        }
    }
