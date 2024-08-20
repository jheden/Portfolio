# We Tethered Felines

A co-op platformer starring two cats with their tails in a knot.

* The tails and tether mechanic between players.
## Player state machine
<details><summary>PlayerController.cs</summary>

```C#
private void ChangeState(PlayerState newState)
{
    if (newState == _state) return;

    #region Exit states
    switch (_state) {
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

    if (Clinging && otherPlayer.State == PlayerState.Swinging) {
        var force = FastMath.Normalize(otherPlayer.TailPos - TailPos) * FastMath.Magnitude(otherPlayer.body.velocity);
        Yeet(force);
        otherPlayer.Yeet(force);
    }
    #endregion

    #region Enter states
    switch (newState) {
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
```

</details>

## Level generation from images
<details><summary>LevelController.cs</summary>

```C#
private void Start() {
    CreateTilemap();

    _yPos = -(int)Camera.main.orthographicSize;

    for (var i = 0; i < levels.Length; i++)
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

    for (var i = 0; i < _pixels.Length; i++) {
        _positions[i] = new Vector3Int(i % level.width - level.width / 2, i / level.width + _yPos, 0);
        _tiles[i] = _pixels[i].r > 0.5f ? tile : null;
    }

    _tilemap.SetTiles(_positions, _tiles);
    _yPos += level.height;
}
```

</details>

## Knotted tails simulation
<details><summary>TailController.cs</summary>

```C#
void Update() {
    SimulateRope();

    _renderer.SetPositions(_pos.Select(section => (Vector3)section).ToArray());

    knot.transform.position = _pos[_pos.Count / 2];
    knot.transform.right = _players[1].TailPos - _players[0].TailPos;
}

private void SimulateRope() {
    _pos[0] = _players[0].TailPos;
    _pos[^1] = _players[1].TailPos;

    _pos2 = new List<Vector2>(_pos);

    for (var j = 2; j > 0; j--)
        for (int i = j; i < _pos.Count - j; i++) {
            _vecL = _pos[i - j] - _pos2[i];
            _normL = FastMath.Normalize(_vecL);
            _stretchL = FastMath.Magnitude(_vecL) - _sectionLength;

            _vecR = _pos[i + j] - _pos2[i];
            _normR = FastMath.Normalize(_vecR);
            _stretchR = FastMath.Magnitude(_vecR) - _sectionLength;

            var modifier = Mathf.Max(1, _players[0].CurrentDistance) / _maxDistance;

            _vel[i] = _vel[i] * damping + (_normL * _stretchL + _normR * _stretchR) * modifier / 10;
        }

    for (var i = 1; i < _pos.Count - 1; i++) {
        _vel[i] += Time.fixedDeltaTime * ropeSectionMass * Physics2D.gravity;
        _pos2[i] += _vel[i];
    }

    _pos = new List<Vector2>(_pos2);
}
```

</details>
