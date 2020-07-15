    #ProcedualGrass
    [Range(0, 1000)]
    public int terrainSize = 250;          //地形大小
    [Range(0, 100f)]
    public float terrainHeight = 10f;          //地形高度 
    public Material terrainMat;          //地形材质

    private float xOffset;
    private float zOffset;

    [Range(1f, 100f)]
    public float scaleFatter = 10f;          //地形缩放
    [Range(1f, 100f)]
    public float offsetFatter = 10f;          //地形偏移

    List<Vector3> vertexs = new List<Vector3>();          //顶点列表，存储网格顶点信息
    List<int> triangles = new List<int>();          //三角形面列表，存储网格三角形信息
    float[,] perlinNoise;          //存储每个顶点的高度值

    void Start()
    {
        xOffset = transform.position.x;
        zOffset = transform.position.z;

        perlinNoise = new float[terrainSize, terrainSize];
        GenerateTerrain();
    }

    void CreateVertsAndTris()
    {
        //遍历每一个顶点，并用列表 vertexs 和 triangles 存储
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

    //生成地形高度的随机 Perlin 值
    float GeneratePerlinNoise(int i, int j)
    {
        float xCoord = (float)(i + xOffset) / terrainSize * scaleFatter + offsetFatter;
        float zCoord = (float)(j + zOffset) / terrainSize * scaleFatter + offsetFatter;

        return Mathf.PerlinNoise(xCoord, zCoord);
    }

    //生成地形网格数据
    void GenerateTerrain()
    {
        CreateVertsAndTris();

        //计算UV
        Vector2[] uvs = new Vector2[vertexs.Count];
        for (int i = 0; i < vertexs.Count; i++)
        {
            uvs[i] = new Vector2(vertexs[i].x, vertexs[i].z);
        }

        //添加一个 MyTerrain 的物体，并添加 MeshFilter / MeshRenderer 两个组件
        //MeshFilter 存储物体的网格信息
        //MeshRenderer 负责接收这些信息并把这些信息渲染出来
        GameObject Myterrain = new GameObject("Terrain0");
        Myterrain.transform.position = this.gameObject.transform.position;
        Myterrain.AddComponent<MeshFilter>();
        MeshRenderer renderer = Myterrain.AddComponent<MeshRenderer>();
        //开启地面的阴影投射和接受
        renderer.receiveShadows = true;
        renderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.On;
        //添加Mesh Collider
        MeshCollider collider = Myterrain.AddComponent<MeshCollider>();
        //添加材质
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
        Myterrain.GetComponent<MeshFilter>().mesh = groundMesh;
        collider.sharedMesh = groundMesh;
    }
