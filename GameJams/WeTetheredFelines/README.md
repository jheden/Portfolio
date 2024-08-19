# We Tethered Felines

A co-op platformer starring two cats with their tails in a knot.

* The tails and tether mechanic between players.
## Player state machine
<details>
<summary>PlayerController.cs</summary>```C#
private void ChangeState(PlayerState newState)
{
    if (newState == _state) return;

    #region Exit states
    switch (_state)
    {
        case PlayerState.Idle:
            break;
        case PlayerState.Walking:
            break;
        case PlayerState.Running:
            break;
        case PlayerState.Jumping:
            break;
        case PlayerState.Falling:
            break;
        case PlayerState.Dangling:
        case PlayerState.Swinging:
            MaxDistanceOnly = true;
            Orientation = Vector2.up;
            break;
        case PlayerState.GroundClinging:
        case PlayerState.WallClinging:
            body.constraints = RigidbodyConstraints2D.FreezeRotation;
            _collider.enabled = true;
            break;
        case PlayerState.WallSliding:
            body.constraints = RigidbodyConstraints2D.FreezeRotation;
            _collider.enabled = true;
            body.gravityScale = 1;
            break;
    }

    if (Clinging && otherPlayer.State == PlayerState.Swinging)
    {
        var force = FastMath.Normalize(otherPlayer.TailPos - TailPos) * FastMath.Magnitude(otherPlayer.body.velocity);
        Yeet(force);
        otherPlayer.Yeet(force);
    }
    #endregion

    #region Enter states
    switch (newState)
    {
        case PlayerState.Idle:
            break;
        case PlayerState.Walking:
            break;
        case PlayerState.Running:
            break;
        case PlayerState.Jumping:
            CanJump = false;
            break;
        case PlayerState.Falling:
            break;
        case PlayerState.Dangling:
            _audio.OneShot("Meow");
            break;
        case PlayerState.Swinging:
            MaxDistanceOnly = false;
            break;
        case PlayerState.GroundClinging:
        case PlayerState.WallClinging:
            body.constraints = RigidbodyConstraints2D.FreezeAll;
            break;
        case PlayerState.WallSliding:
            body.constraints = RigidbodyConstraints2D.FreezePositionX | RigidbodyConstraints2D.FreezeRotation;
            body.gravityScale = 0.5f;
            break;
    }

    if (newState == PlayerState.WallClinging || newState == PlayerState.WallSliding)
        _collider.enabled = false;
    #endregion

    _state = newState;
}
```</details>
## Level generation from images
```C#
using UnityEngine;
using UnityEngine.Tilemaps;

[RequireComponent(typeof(Grid))]
public class LevelController : MonoBehaviour {
    #region Singleton
    public static LevelController Instance { get; private set; }

    private void Awake() {
        if (Instance == null) Instance = this;
        else Destroy(this);
    }
    #endregion

    [SerializeField] private Texture2D[] levels;
    [SerializeField] private TileBase[] tileSets;

    private Color[] _pixels;
    private Vector3Int[] _positions;
    private GameObject _tilemapObject;
    private Tilemap _tilemap;
    private TileBase[] _tiles;
    private int _yPos;

    private void Start() {
        CreateTilemap();

        _yPos = -(int)Camera.main.orthographicSize;

        for (int i = 0; i < levels.Length; i++)
            ProcessLevel(levels[i], tileSets[i % tileSets.Length]);
    }

    private void CreateTilemap() {
        _tilemapObject = new GameObject();
        _tilemapObject.transform.parent = transform;
        _tilemapObject.layer = gameObject.layer;
        _tilemap = _tilemapObject.AddComponent<Tilemap>();
        _tilemapObject.AddComponent<TilemapRenderer>();
        _tilemapObject.AddComponent<TilemapCollider2D>();
    }
    
    private void ProcessLevel(Texture2D level, TileBase tile) {
        _pixels = level.GetPixels();
        _positions = new Vector3Int[_pixels.Length];
        _tiles = new TileBase[_pixels.Length];

        for (int i = 0; i < _pixels.Length; i++) {
            _positions[i] = new Vector3Int(i % level.width - level.width / 2, i / level.width + _yPos, 0);
            _tiles[i] = _pixels[i].r > 0.5f ? tile : null;
        }

        _tilemap.SetTiles(_positions, _tiles);
        _yPos += level.height;
    }
}
```