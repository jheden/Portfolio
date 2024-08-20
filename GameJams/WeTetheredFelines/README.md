# We Tethered Felines
A co-op platformer starring two cats with their tails in a knot.

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

## Player Animation
<details><summary>PlayerAnimator.cs</summary>

```C#
    private static readonly int Spawn = Animator.StringToHash("Spawn");
    private static readonly int Idle = Animator.StringToHash("Idle");
    private static readonly int Walk = Animator.StringToHash("Walk");
    private static readonly int Run = Animator.StringToHash("Run");
    private static readonly int GroundToJump = Animator.StringToHash("GroundToJump");
    private static readonly int Midair = Animator.StringToHash("Midair");
    private static readonly int MidairToFall = Animator.StringToHash("MidairToFall");
    private static readonly int Land = Animator.StringToHash("Land");
    private static readonly int GroundImpact = Animator.StringToHash("GroundImpact");
    private static readonly int WallImpact = Animator.StringToHash("WallImpact");
    private static readonly int GroundToCling = Animator.StringToHash("GroundToCling");
    private static readonly int GroundCling = Animator.StringToHash("GroundCling");
    private static readonly int Fall = Animator.StringToHash("Fall");
    private static readonly int FallToDangle = Animator.StringToHash("FallToDangle");
    private static readonly int Dangle = Animator.StringToHash("Dangle");
    private static readonly int WallCling = Animator.StringToHash("WallCling");
    private static readonly int WallClingToSlide = Animator.StringToHash("WallClingToSlide");
    private static readonly int WallSlide = Animator.StringToHash("WallSlide");
    private static readonly int WallJump = Animator.StringToHash("WallJump");

    private Dictionary<KeyValuePair<PlayerState, PlayerState>, int> _transitions = new()
    {
        { new(PlayerState.Idle, PlayerState.GroundClinging), GroundToCling },
        { new(PlayerState.Walking, PlayerState.GroundClinging), GroundToCling },
        { new(PlayerState.Running, PlayerState.GroundClinging), GroundToCling },

        { new(PlayerState.Idle, PlayerState.Jumping),  GroundToJump },
        { new(PlayerState.Walking, PlayerState.Jumping),  GroundToJump },
        { new(PlayerState.Running, PlayerState.Jumping), GroundToJump },

        { new(PlayerState.Jumping, PlayerState.Idle), Land },
        { new(PlayerState.Jumping, PlayerState.Walking), Land },
        { new(PlayerState.Jumping, PlayerState.Running), Land },

        { new(PlayerState.Jumping, PlayerState.Falling), MidairToFall },

        { new(PlayerState.Falling, PlayerState.Idle), GroundImpact },
        { new(PlayerState.Falling, PlayerState.Walking), GroundImpact },
        { new(PlayerState.Falling, PlayerState.Running), GroundImpact },

        { new(PlayerState.Falling, PlayerState.Dangling), FallToDangle },

        { new(PlayerState.WallClinging, PlayerState.WallSliding), WallClingToSlide },

    };                        

    private void Awake() {
        if (!TryGetComponent(out IPlayerController player)) {
            Destroy(this);
            return;
        }

        _player = player;
        _animator = GetComponent<Animator>();
        _body = GetComponent<Rigidbody2D>();
        _renderer = GetComponent<SpriteRenderer>();
        pControl = GetComponent<PlayerController>();

        foreach (AnimationClip clip in _animator.runtimeAnimatorController.animationClips)
            switch (clip.name) {
                case "Spawn":
                    _spawnTime = clip.length;
                    break;
                case "GroundToCling":
                    _groundToClingTime = clip.length;
                    break;
                case "GroundToJump":
                    _groundToJumpTime = clip.length;
                    break;
                case "Land":
                    _landTime = clip.length;
                    break;
                case "MidairToFall":
                    _midairToFallTime = clip.length;
                    break;
                case "GroundImpact":
                    _groundImpactTime = clip.length;
                    break;
                case "FallToDangle":
                    _fallToDangleTime = clip.length;
                    break;
                case "WallClingToSlide":
                    _wallClingToSlideTime = clip.length;
                    break;
            }
    }

    void Update() {
        state = GetState();

        if (state == _currentState) return;
        
        _animator.CrossFade(state, 0, 0);
        _currentState = state;
        _lastPlayerState = pControl.State;
    }

    int GetTransition(PlayerState prev, PlayerState next) {
        _transitions.TryGetValue(new(prev, next), out int result);
        return result;
    }

    int GetState() {
        if (Time.time < _lockedUntil) return _currentState;

        var transition = GetTransition(_lastPlayerState, pControl.State);
        if (transition == GroundToCling) return LockState(GroundToCling, _groundToClingTime);
        if (transition == GroundToJump) return LockState(GroundToCling, _groundToJumpTime);
        if (transition == Land) return LockState(Land, _landTime);
        if (transition == MidairToFall) return LockState(MidairToFall, _midairToFallTime);
        if (transition == GroundImpact) return LockState(GroundImpact, _groundImpactTime);
        if (transition == FallToDangle) return LockState(FallToDangle, _fallToDangleTime);
        if (transition == WallClingToSlide) return LockState(WallClingToSlide, _wallClingToSlideTime);

        switch (pControl.State) {
            case PlayerState.Jumping: return Midair;
            case PlayerState.Walking: return Walk;
            case PlayerState.Running: return Run;
            case PlayerState.Falling: return Fall;
            case PlayerState.Dangling: return Dangle;
            case PlayerState.Swinging: return Dangle;
            case PlayerState.GroundClinging: return GroundCling;
            case PlayerState.WallClinging: return WallCling;
            case PlayerState.WallSliding: return WallSlide;
            case PlayerState.Idle: return Idle;
            default: return _currentState;
        }
    }

    int LockState(int s, float t) {
        _lockedUntil = Time.time + t;
        return s;
    }
```

</details>
