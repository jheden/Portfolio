# Board Game Utils
## Mesh generation from image
<table><tr>
  <td>Input</td>
  <td>Output</td>
</tr><tr>
  <td><img src="/Images/BoardGameUtils/MeshInput.gif" /></td>
  <td><img src="/Images/BoardGameUtils/MeshOutput.gif" /></td>
</tr></table>

<details><summary>MeshGenerator.cs</summary>
    
```C#
public static class MeshGenerator {
    public static GameObject Generate(Sprite sprite, Sprite backSprite, float thickness, Material material, bool addCollider = false, Transform parent = null) {
        var gameObject = new GameObject();
        gameObject.name = "Mesh";

        var tmpRend = gameObject.AddComponent<SpriteRenderer>();
        tmpRend.sprite = sprite;
        var tmpColl = gameObject.AddComponent<PolygonCollider2D>();
        var mesh = ExtrudeMesh(tmpColl.CreateMesh(false, false), thickness);
        Object.DestroyImmediate(tmpRend);
        Object.DestroyImmediate(tmpColl);

        gameObject.AddComponent<MeshFilter>().mesh = mesh;
        gameObject.AddComponent<MeshRenderer>().material = material;

        var front = new GameObject();
        front.name = "Front";
        front.transform.parent = gameObject.transform;
        front.transform.localPosition = Vector3.back * 0.0001f;
        front.AddComponent<SpriteRenderer>().sprite = sprite;

        var back = new GameObject();
        back.name = "Back";
        back.transform.parent = gameObject.transform;
        back.transform.localPosition = Vector3.forward * (thickness + 0.0001f);
        back.AddComponent<SpriteRenderer>().sprite = backSprite != null ? backSprite : sprite;

        if (addCollider)
            gameObject.AddComponent<MeshCollider>().convex = true;

        if (parent) {
            gameObject.transform.parent = parent;
            gameObject.transform.localPosition = Vector3.zero;
            gameObject.transform.localRotation = Quaternion.identity;
        }

        return gameObject;
    }

    private static Mesh ExtrudeMesh(Mesh mesh, float thickness) {
        var vertexCount = mesh.vertices.Length;
        var vertices = mesh.vertices.ToList();
        var extrudedVertices = vertices.Select(vertex => vertex + Vector3.forward * thickness).ToList();
        vertices.AddRange(extrudedVertices);

        var triangles = mesh.triangles.Select(i => i).ToList();
        var extrudedTriangles = triangles.Select(i => i + vertexCount).Reverse().ToList();

        var boundaries = EdgeHelper.GetEdges(triangles.ToArray()).FindBoundary().SortEdges();
        var extrudedBoundaries = EdgeHelper.GetEdges(extrudedTriangles.Select(i => i).Reverse().ToArray())
            .FindBoundary().SortEdges();

        triangles.AddRange(extrudedTriangles);

        List<int> boundaryTriangles = new();
        for (var i = 0; i < boundaries.Count; i++) {
            boundaryTriangles.Add(boundaries[i].v1);
            boundaryTriangles.Add(extrudedBoundaries[i].v1);
            boundaryTriangles.Add(extrudedBoundaries[i].v2);

            boundaryTriangles.Add(boundaries[i].v1);
            boundaryTriangles.Add(extrudedBoundaries[i].v2);
            boundaryTriangles.Add(boundaries[i].v2);
        }

        triangles.AddRange(extrudedTriangles);
        triangles.AddRange(boundaryTriangles);

        mesh.vertices = vertices.ToArray();
        mesh.triangles = triangles.ToArray();
        mesh.uv = vertices.Select(vertex => ((Vector2)vertex + Vector2.one) / 2).ToArray();

        mesh.RecalculateBounds();
        mesh.Optimize();

        return mesh;
    }
}
```

</details>

## Board Generation
<img src="/Images/BoardGameUtils/BoardGeneration.gif" />

<details><summary>BoardController.cs</summary>
    
```C#
public void New()
{
    ClearTiles();

    List<TileType> types = new();

    foreach ((TileType type, int count) in tileCounts)
        for (int i = 0; i < count; i++)
            types.Add(type);

    types.Shuffle();

    int currDir = 0;
    for (int i = 0; i < types.Count - 1; i++)
    {
        var dir = currDir switch
        {
            -1 => (0, 2),
            1 => (-1, 1),
            _ => (-1, 2),
        };
        var rand = UnityEngine.Random.Range(dir.Item1, dir.Item2);
        currDir += rand;

        CreateTile(types[i], (TileDirection)(rand + 1));
    }
    CreateTile(types.Last(), TileDirection.Straight);
}

public Tile CreateTile(TileType type, TileDirection dir)
{
    var tile = Instantiate(tilePrefab, transform).GetComponent<Tile>();
    tile.TileSO = Resources.Load($"Tiles/{type}") as TileSO;
    tile.name = $"{type}{dir}";
    tile.Direction = dir;
    tile.transform.position = _currPos;
    tile.transform.rotation = Quaternion.Euler(90, _currRot, 0);
    tiles.Add(tile);

    _currRot += dir switch
    {
        TileDirection.Left => -90,
        TileDirection.Right => 90,
        _ => 0,
    };

    _currPos += Quaternion.Euler(0, _currRot, 0) * dir switch
    {
        TileDirection.Left => new Vector3(3.5f, 0, -1.5f),
        TileDirection.Right => new Vector3(3.5f, 0, 1.5f),
        _ => new Vector3(4, 0, 0),
    };

    return tile;
}
```

</details>

## Moving pieces
<img src="/Images/BoardGameUtils/MovingPieces.gif" />

<details><summary>BoardController.cs</summary>
    
```C#
public IEnumerator StartMoving()
{
    Moving = true;

    foreach (var figure in _animals)
        yield return MoveAnimal(figure);

    Moving = false;
}

public IEnumerator MoveAnimal(AnimalFigure figure)
{
    var newPos = positions[figure.Animal] + figure.Moves();
    for (int i = 0; i < figure.Moves(); i++)
        yield return PushAnimal(figure);
}

public IEnumerator PushAnimal(AnimalFigure figure)
{
    var startPos = figure.transform.position;
    var endPos = tiles[positions[figure.Animal] + 1].transform.position;
    endPos.y = startPos.y;
    var dirStep = (endPos - startPos) / 25;

    var startRot = figure.transform.rotation.eulerAngles;
    var endRot = tiles[positions[figure.Animal] + 1].transform.rotation.eulerAngles;
    var rotStep = ((endRot.y - startRot.y + 540) % 360 - 180) / 25;

    for (int j = 0; j < 25; j++)
    {
        figure.transform.position = startPos + dirStep * j + Vector3.up * Mathf.Sin(Mathf.PI * j / 25);
        figure.transform.rotation = Quaternion.Euler(startRot.x, startRot.y + rotStep * j, startRot.z);
        yield return new WaitForFixedUpdate();
    }

    positions[figure.Animal]++;
}
```

</details>
