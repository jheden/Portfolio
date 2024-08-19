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
