# Unity インクシューター試作用スクリプト

これはスプラトゥーン風の「インクでステージを塗る」ゲームを作るための
プロトタイプ構成です。任天堂のアセット、キャラクター、名称、正確なルールを
再現するものではありません。

## シーン設定

1. 地面、坂、壁をまとめた塗れるメッシュに `PaintableSurface` を付けます。
2. 塗れるメッシュには、実用的で重なりの少ないUVが必要です。
   塗り座標には `RaycastHit.textureCoord` を使います。
3. 塗れるメッシュに `MeshCollider` を追加します。
   Colliderは `PaintableSurface` と同じオブジェクト、またはその子に置いてください。
4. `Ink/Transparent Overlay` シェーダのマテリアルを作り、塗れるメッシュに割り当てます。
   未塗装部分を灰色で見たい場合だけ `Ink/Blend Surface` を使います。
5. `Ink/Stamp` シェーダのマテリアルを作り、
   `PaintableSurface` の `paintStampMaterial` に割り当てます。
6. プレイヤーオブジェクトを作り、`CharacterController`、
   `PlayerInkController`、`InkShooter`、`InkHealth`、`PlayerRespawnController` を付けます。
7. インク弾Prefabを作り、`Rigidbody`、小さめのCollider、
   `InkProjectile` を付けます。
8. 空のオブジェクトに `InkGameManager` を付け、
   `PaintableSurface` を割り当てます。

## 武器設定

武器ごとの性能は `WeaponInkProfile` で管理できます。

作り方:

1. Projectビューで右クリックします。
2. `Create > Ink > Weapon Ink Profile` を選びます。
3. 作成したProfileに、弾Prefab、連射速度、弾速、インク消費、
   塗り半径などを設定します。
4. プレイヤーの `InkShooter > Weapon Profile` に割り当てます。

`WeaponInkProfile` で設定できる主な項目:

- `Projectile Prefab`: 発射する `InkProjectile` 付きPrefab
- `Projectile Speed`: 弾速
- `Projectile Life Seconds`: 弾の寿命
- `Use Gravity`: 弾に重力を使うか
- `Use Shooter Trajectory`: シューター用の直進後落下弾道を使うか
- `Straight Flight Seconds`: 発射後、直進する時間
- `Drop Gravity`: 直進後に下へ落とす強さ
- `Max Fall Speed`: 落下速度の上限
- `Fire Rate`: 1秒あたりの発射数
- `Ink Cost`: 1発ごとのインク消費
- `Ink Max`: 最大インク量
- `Refill Per Second`: インク回復速度
- `Refill Delay`: 射撃後、回復開始までの待ち時間
- `Damage`: 1発命中した時の攻撃力
- `First Shot Spread Degrees`: 撃ち始めのブレ角度
- `Max Spread Degrees`: 最大ブレ角度
- `Spread Increase Per Shot`: 1発ごとに増えるブレ
- `Spread Recovery Delay`: 射撃停止後、精度回復が始まるまでの時間
- `Spread Recovery Per Second`: 1秒あたりの精度回復量
- `Paint Radius`: 塗り半径
- `Paint Hardness`: 塗りの硬さ
- `Splash Radius World`: 着弾時の塗り判定の太さ
- `Paint Trail While Flying`: 弾が飛んでいる途中にもインクを落として塗る
- `Trail Paint Start Delay`: 発射後、飛行中塗りを始めるまでの時間
- `Trail Paint Distance Interval`: 何m進むごとにインクを落とすか
- `Trail Paint Ray Start Height`: 弾位置からどれだけ上を起点に下へ判定するか
- `Trail Paint Ray Distance`: 弾位置から下方向にどれだけ塗れる面を探すか
- `Trail Paint Radius Multiplier`: 飛行中に落ちるインクの塗り半径倍率
- `Trail Paint Hardness Multiplier`: 飛行中に落ちるインクの硬さ倍率
- `Trail Paint Side Random Radius`: 飛行中に落ちるインク位置の横ズレ幅
- `Trail Paint Interval Randomness`: インクを落とす間隔のランダム幅
- `Trail Paint Radius Randomness`: インクの塗り半径のランダム幅
- `Hit Effect Prefab`: 着弾エフェクト
- `Character Hit Effect Prefab`: Player/Enemyにダメージが入った時のヒットエフェクト
- `Auto Destroy Hit Effects`: 生成したヒットエフェクトを自動で消す
- `Hit Effect Fallback Life Seconds`: Particleの長さを読めない時に消すまでの秒数
- `Hit Effect Extra Life Seconds`: Particle再生終了後、少し余裕を持たせる秒数
- `Fire Sound`: 発射時に鳴らすAudioClip

`InkProjectile` の `Paint Nearby Surface On Character Hit` をONにすると、
Player/Enemyなど `InkHealth` の付いた相手に弾が当たった時、
キャラ自身ではなく近くの床や壁を探して塗ります。
キャラに当たるたびに塗り失敗ログが出る場合は、この設定をONにしてください。

敵や味方に弾が当たった場所へ火花のようなエフェクトを出したい場合は、
Particle Systemなどで作ったPrefabを `WeaponInkProfile > Character Hit Effect Prefab` に入れます。
武器Profileを使わない場合は、弾Prefabの `InkProjectile > Character Hit Effect Prefab` に入れてください。
`Character Hit Effect Prefab` が未設定の場合は、通常の `Hit Effect Prefab` が使われます。
エフェクトが残り続ける場合は、弾Prefabの `InkProjectile > Auto Destroy Hit Effects` をONにしてください。
通常はONのままでよく、消えるのが遅い場合は `Hit Effect Fallback Life Seconds` を短くします。

`Weapon Profile` が未設定の場合は、`InkShooter` と `InkProjectile` にある
従来の値が使われます。まずは既存設定のまま動かし、武器を増やしたくなったら
Profileを作る流れでも問題ありません。

### 飛行中のインク落とし

シューター系の弾が、着弾点だけでなく飛んでいる途中にもインクを落として塗る挙動は、
`WeaponInkProfile` の `Trail Paint` で設定します。

おすすめ初期値:

- `Paint Trail While Flying`: ON
- `Trail Paint Start Delay`: `0.03`
- `Trail Paint Distance Interval`: `1.0` から `1.4`
- `Trail Paint Ray Start Height`: `0.15`
- `Trail Paint Ray Distance`: `3`
- `Trail Paint Radius Multiplier`: `0.55` から `0.75`
- `Trail Paint Hardness Multiplier`: `0.8`
- `Trail Paint Side Random Radius`: `0.15` から `0.35`
- `Trail Paint Interval Randomness`: `0.2` から `0.35`
- `Trail Paint Radius Randomness`: `0.15` から `0.25`

`Trail Paint Distance Interval` を小さくすると、弾道上に細かくインクが落ちます。
小さすぎると塗り処理が増えるので、まずは `1.2` くらいから調整してください。
飛行中の塗りが床に届かない場合は、`Trail Paint Ray Distance` を伸ばします。
弾道上の塗りを自然に散らしたい場合は、`Trail Paint Side Random Radius` を少し上げます。
散らばりすぎる場合は `0.1` 前後まで下げてください。

## サブウェポン設定

サブウェポンは `SubWeaponProfile` と `InkSubWeaponController` で管理します。
今後別のサブウェポンを増やす場合も、基本はProfileのPrefabを差し替える流れです。

### スプリンクラーPrefab

スプリンクラーPrefabの作り方:

1. スプリンクラー用のGameObjectを作ります。
2. 見た目用MeshやParticleを子に置きます。
3. ルートに `Rigidbody` を付けます。
4. ルートに小さめのColliderを付けます。
5. ルートに `SprinklerSubWeapon` を付けます。
6. ProjectビューへドラッグしてPrefab化します。

`SprinklerSubWeapon` の主な設定:

- `Team`: 塗るチーム。Player用なら `Player`
- `Paint Mask`: 塗れるメッシュのLayer
- `Paint Radius`: Projectileを使わない場合の1回の塗り半径
- `Paint Hardness`: Projectileを使わない場合の塗りの硬さ
- `Life Seconds`: スプリンクラーが消えるまでの時間
- `Arm Delay`: くっついてから塗り始めるまでの待ち時間
- `Spray Interval`: インクを飛ばす間隔
- `Rays Per Spray`: 1回に発射する弾の数
- `Max Spray Distance`: 周囲を探す距離
- `Spread Along Surface`: 周囲への散らばり幅
- `Min Spread Scale`: 外周方向へ飛ばす最小倍率。`0` に近いほど足元付近にも飛ぶ
- `Max Spread Scale`: 外周方向へ飛ばす最大倍率
- `Center Spray Chance`: 足元付近へ飛ばす弾の割合
- `Center Spray Max Scale`: 足元付近に飛ばす時の最大散らばり幅
- `Use Projectile Spray`: ONにするとプレイヤーと同じ `InkProjectile` を発射する
- `Projectile Weapon Profile`: スプリンクラーが撃つ弾のProfile
- `Projectile Prefab`: Profileを使わない場合の予備Projectile Prefab
- `Projectile Speed`: Profileを使わない場合の予備弾速
- `Projectile Min Surface Angle`: 発射角度の最小値。`0` が設置面と平行
- `Projectile Max Surface Angle`: 発射角度の最大値。`90` が設置面に垂直
- `Projectile Spawn Offset`: 接着面から少し離して弾を出す距離
- `Projectile Forward Spawn Offset`: 弾の発射方向へ少し前にずらす距離
- `Fallback To Ray Paint`: Projectileが未設定の時だけ旧Ray塗りを使う
- `Stick To Hit Object`: 当たった地面/壁の子にする
- `Align To Surface Normal`: スプリンクラーの指定軸を接地面の法線に合わせる
- `Surface Normal Axis`: 接地面に対して垂直にしたいスプリンクラーのローカル軸。基本は `Up`
- `Preserve World Scale When Attached`: 親のスケールを受けず、スプリンクラーの見た目サイズを保つ
- `Spray Effect`: 噴射中に再生するParticleSystem
- `Spin While Spraying`: インクを発射している間だけ回転する
- `Spin Root`: 回転させたいTransform。空ならスプリンクラー本体が回る
- `Spin Degrees Per Second`: 1秒あたりの回転角度

おすすめ初期値:

- `Life Seconds`: `10`
- `Spray Interval`: `0.18`
- `Rays Per Spray`: `3` から `6`
- `Use Projectile Spray`: ON
- `Projectile Weapon Profile`: スプリンクラー用の `WeaponInkProfile`
- `Min Spread Scale`: `0`
- `Max Spread Scale`: `1`
- `Center Spray Chance`: `0.25` から `0.4`
- `Center Spray Max Scale`: `0.2` から `0.35`
- `Projectile Min Surface Angle`: `5` から `10`
- `Projectile Max Surface Angle`: `75` から `88`
- `Projectile Spawn Offset`: `0.15` から `0.25`
- `Projectile Forward Spawn Offset`: `0.2` から `0.45`
- `Align To Surface Normal`: ON
- `Surface Normal Axis`: 基本は `Up`。横倒しになる場合は `Forward` / `Right` / `NegativeUp` などを試す
- `Preserve World Scale When Attached`: ON
- `Spin While Spraying`: ON
- `Spin Degrees Per Second`: `360` から `720`
- `Max Spray Distance`: `3` から `4`
- `Spread Along Surface`: `1` から `1.5`

スプリンクラー用の `WeaponInkProfile` は、プレイヤーのProfileを複製して作ると早いです。
`Projectile Prefab` には普段使っているインク弾Prefabを入れ、`Projectile Speed`、`Paint Radius`、`Splash Radius World`、`Trail Paint` 系をスプリンクラー用に少し弱めに調整します。
スプリンクラーの弾が多すぎて重い場合は、まず `Rays Per Spray` を下げるか、Profile側の `Paint Trail While Flying` をOFFにしてください。

### SubWeaponProfile

Projectビューで右クリックし、`Create > Ink > Sub Weapon Profile` を作ります。

主な設定:

- `Prefab`: スプリンクラーPrefab
- `Ink Cost`: 消費インク量
- `Refill Delay`: 使用後、インク回復が始まるまでの時間
- `Cooldown`: 連続使用できない時間
- `Throw Speed`: 投げる速さ
- `Upward Velocity`: 投げる時の上向き成分
- `Spawn Forward Distance`: 手元Transformがない時、前方何mに出すか
- `Head Attach Local Position`: 頭に付ける時のローカル位置
- `Head Attach Local Euler`: 頭に付ける時のローカル回転

### Player側

Player本体に `InkSubWeaponController` を付けます。

設定:

- `Aim Camera`: Player Camera
- `Throw Point`: 手元や武器付近のTransform
- `Head Attach Point`: 頭ボーン、または頭に置いた空Object
- `Sub Weapon Profile`: 作成した `SubWeaponProfile`
- `Ink Shooter`: Player本体の `InkShooter`
- `Player Controller`: Player本体の `PlayerInkController`
- `Team`: `Player`
- `Throw Key`: `R`
- `Attach To Head Key`: `L`
- `Only In Human Form`: ON
- `Replace Existing Sub Weapon`: ON
- `Head Attach Local Position Offset`: L装着位置の追加オフセット
- `Head Attach Local Euler Offset`: L装着回転の追加オフセット

`R` で投げ、地面や壁に当たるとくっついて周囲を塗ります。
`L` で `Head Attach Point` に直接くっつけます。
`SubWeaponProfile` の `Head Attach Local Position/Euler` を基本位置にして、Playerごとの微調整は `InkSubWeaponController` の `Head Attach Local Position Offset/Euler Offset` で行います。
まずは `Head Attach Point` 用に頭ボーンの子へ空Objectを作り、位置を少し上に調整するのがおすすめです。

## スペシャルウェポン設定

アメフラシは `SpecialWeaponProfile`、`RainCloudProfile`、`PlayerSpecialWeaponController` で管理します。
塗りポイントを稼ぐとスペシャルゲージが増え、満タン時だけ発動できます。

### RainCloudProfile

Projectビューで右クリックし、`Create > Ink > Rain Cloud Profile` を作ります。

主な設定:

- `Cloud Prefab`: 雨雲Prefab。`InkRainCloudController` を付けたPrefab、または雲見た目Prefab
- `Life Seconds`: 雲が残る時間
- `Cloud Height`: 着弾地点からどれだけ上に雲を出すか
- `Move Speed`: 雲の移動速度
- `Drift Side Amplitude`: 横揺れの強さ
- `Rain Interval`: 雨判定を出す間隔
- `Rain Drops Per Tick`: 1回に落とす雨判定数
- `Rain Area Radius`: 雨が降る範囲
- `Rain Ray Distance`: 下方向に塗れる面を探す距離
- `Paint Mask`: 塗れるメッシュのLayer
- `Main Drop Paint Radius`: 雨滴中心の塗り半径
- `Splash Stamp Count`: 着弾時に周囲へ出す飛沫塗りの数
- `Splash Paint Radius Min/Max`: 飛沫塗りの大きさ
- `Splash Distance Min/Max`: 飛沫が中心から散る距離
- `Damage Mask`: ダメージ対象のLayer
- `Damage Per Second`: 雨に当たった相手への継続ダメージ
- `Damage Radius`: 各雨滴のダメージ範囲

おすすめ初期値:

- `Life Seconds`: `8`
- `Cloud Height`: `5`
- `Move Speed`: `1.8`
- `Rain Interval`: `0.12`
- `Rain Drops Per Tick`: `12`
- `Rain Area Radius`: `4.5`
- `Rain Ray Distance`: `12`
- `Main Drop Paint Radius`: `0.03`
- `Splash Stamp Count`: `4`
- `Splash Paint Radius Min/Max`: `0.006` / `0.016`
- `Splash Distance Min/Max`: `0.06` / `0.28`
- `Damage Per Second`: `12`

### アメフラシ投擲Prefab

1. アメフラシの缶や装置用GameObjectを作ります。
2. ルートに `Rigidbody` を付けます。
3. ルートにColliderを付けます。
4. ルートに `RainCloudThrowProjectile` を付けます。
5. ProjectビューへドラッグしてPrefab化します。

### SpecialWeaponProfile

Projectビューで右クリックし、`Create > Ink > Special Weapon Profile` を作ります。

主な設定:

- `Required Paint Points`: スペシャルゲージMaxに必要な塗りポイント
- `Throw Projectile Prefab`: アメフラシ投擲Prefab
- `Throw Speed`: 投げる速さ
- `Upward Velocity`: 投げる時の上向き成分
- `Spawn Forward Distance`: 手元Transformがない時の生成距離
- `Cooldown`: 使用後の短い待ち時間
- `Rain Cloud Profile`: 作成した `RainCloudProfile`
- `Activate Sound`: 発動音

### Player側

Player本体に `PlayerSpecialWeaponController` を付けます。

設定:

- `Aim Camera`: Player Camera
- `Throw Point`: 手元や武器付近のTransform
- `Special Profile`: 作成した `SpecialWeaponProfile`
- `Player Controller`: Player本体の `PlayerInkController`
- `Team`: `Player`
- `Special Key`: まずは `Q`
- `Only In Human Form`: ON

`Required Paint Points` 分だけ塗りポイントを稼ぐと `Gauge01` が1になり、`Special Key` で発動できます。
使用後、スペシャルゲージは0に戻ります。

### 体力と攻撃力

PlayerやEnemyに `InkHealth` を付けると、体力を持てます。
弾が `InkHealth` の付いた相手に当たると、`WeaponInkProfile > Damage` の値だけ体力が減ります。
同じチーム同士にはダメージが入りません。

`InkHealth` の主な設定:

- `Team`: 所属チーム。Playerなら `Player`、敵なら `Enemy`
- `Max Health`: 最大体力
- `Reset Health On Enable`: 有効化された時に体力を全回復する
- `Disable On Defeat`: 体力0でGameObjectを非表示にする
- `Hit Audio Source`: 被弾音を鳴らすAudioSource
- `Hit Sound`: 被弾した時に鳴らすAudioClip
- `Hit Sound Volume`: 被弾音の音量
- `Defeated Sound`: 体力0になった時に鳴らすAudioClip
- `Defeated Sound Volume`: 撃破された時の音量
- `On Health Changed`: 体力が変わった時のイベント
- `On Damaged`: ダメージを受けた時のイベント
- `On Defeated`: 体力0になった時のイベント

まずは以下がおすすめです:

- Player: `Team=Player`, `Max Health=100`, `Disable On Defeat=OFF`
- Enemy: `Team=Enemy`, `Max Health=100`, `Disable On Defeat=OFF`
- Shooter武器: `Damage=25`

敵・味方で音を変えたい場合は、それぞれの `InkHealth > Hit Sound` と `Defeated Sound` に別のAudioClipを入れます。
`Hit Audio Source` は被弾音と撃破音で共通です。未設定の場合は、同じGameObjectの `AudioSource` を自動で探します。
倒された瞬間にGameObjectを非表示にする場合でも、撃破音が途中で切れにくいように再生されます。

### プレイヤーのリスポーン

Playerの体力が0になった時にリスポーンさせるには、Playerに `PlayerRespawnController` を付けます。

基本設定:

- `Respawn Point`: 復活地点。未設定ならゲーム開始時の位置に戻る
- `Respawn Delay`: 倒されてから復活するまでの秒数
- `Reset Health On Respawn`: 復活時に体力を全回復する
- `Defeat Explosion Prefab`: 倒された瞬間に出すインク爆発Prefab
- `Defeat Explosion Offset`: インク爆発を出す位置のオフセット
- `Auto Disable Player Controls`: 倒れている間、`PlayerInkController` と `InkShooter` を止める
- `Auto Hide Child Renderers`: 倒れている間、Player配下のRendererを自動で非表示にする
- `Disable Renderers While Defeated`: 倒れている間だけ非表示にするRenderer
- `Disable Objects While Defeated`: 倒れている間だけ非表示にするモデルなど

おすすめ:

- Playerの `InkHealth > Disable On Defeat`: OFF
- `PlayerRespawnController > Respawn Delay`: `2`
- `Auto Disable Player Controls`: ON

`Respawn Point` 用に空オブジェクトを作り、プレイヤー開始地点に置いて割り当ててください。
倒された時にプレイヤーを消したい場合は、人型/イカ型モデルのRenderer、またはモデル親GameObjectを
`Disable Renderers While Defeated` / `Disable Objects While Defeated` に入れます。
基本は `Auto Hide Child Renderers` をONにしておけば、子にあるモデルRendererは自動で非表示になります。
インク爆発PrefabはParticle Systemで作り、Particleの `Stop Action` を `Destroy` にしておくと自動で消えます。

### 落下によるリスポーン

ステージ下に大きな落下判定を置く場合は `FallDeathZone` を使います。

設定手順:

1. ステージの下に大きなPlane、またはCubeを置きます。
2. Colliderを付けます。Planeなら `MeshCollider`、Cubeなら `BoxCollider` でOKです。
3. Colliderの `Is Trigger` をONにします。
4. そのObjectに `FallDeathZone` を付けます。
5. `Target Mask` に `Player` や `Enemy` のLayerを含めます。

`FallDeathZone` に触れたObjectの親から `InkHealth` を探し、見つかったら即座に倒します。
Playerなら `PlayerRespawnController`、Enemyなら `ScriptedEnemyController` のリスポーン処理につながります。
平面を見せたくない場合は、`Mesh Renderer` をOFFにしてください。

### 発射音

球を発射するたびに音を鳴らすには、プレイヤーに `AudioSource` を追加し、
`InkShooter > Fire Audio Source` にその `AudioSource` を割り当てます。
武器ごとに音を変える場合は、`WeaponInkProfile > Fire Sound` にAudioClipを入れます。
Profileを使わない場合は、`InkShooter > Fire Sound` にAudioClipを入れてください。

おすすめ設定:

- `AudioSource > Play On Awake`: OFF
- `AudioSource > Spatial Blend`: 2D寄りなら `0`、位置で音量を変えたいなら `1`
- `InkShooter > Fire Sound Volume`: `0.6` から `1`

### シューター弾道

シューター系の弾は、`Use Shooter Trajectory` をONにすると、
発射後しばらく直進してから強く落下します。

おすすめ初期値:

- `Use Shooter Trajectory`: ON
- `Use Gravity`: OFF
- `Straight Flight Seconds`: `0.10` から `0.14`
- `Drop Gravity`: `80` から `110`
- `Max Fall Speed`: `35`
- `Projectile Speed`: `24` から `30`

`Straight Flight Seconds` を短くすると早く落ち、長くすると直線的に飛びます。
`Drop Gravity` を大きくすると、直進後にガクッと落ちる感じが強くなります。

### 弾の向き

`InkProjectile > Visual > Rotate To Velocity` をONにすると、
発射された弾が進行方向に向くようになります。
弾モデルの正面がUnityのZ+方向ではない場合は、
`Rotation Offset Euler` で見た目の向きを補正してください。
例えば横向きになる場合は、`Y` や `X` に `90` / `-90` を入れて調整します。

弾の当たり判定は飛んでいるのに見た目だけその場に落ちる場合は、
弾Prefabを以下の構成にしてください。

- ルート: `Rigidbody`、Collider、`InkProjectile`
- 子: 見た目用Mesh/Particle
- 子の見た目用オブジェクトには `Rigidbody` を付けない
- 見た目用Particleの `Simulation Space` は基本 `Local`

Playerの弾は正常でEnemyの弾だけ見た目がおかしい場合は、
Enemy用 `WeaponProfile > Projectile Prefab` にPlayerと同じ弾Prefabを一度入れて確認してください。
それで直る場合は、Enemy用弾Prefabの子オブジェクトにRigidbodyがある、
または見た目用オブジェクトが弾Prefabルートの子になっていない可能性が高いです。

見た目用オブジェクトを `InkProjectile > Visual Root` に入れると、
`Lock Visual To Projectile` がONの間、見た目を弾ルートのローカル位置に固定します。
通常は `Visual Local Position` を `(0, 0, 0)` にしてください。
`Disable Child Rigidbodies` をONにすると、弾Prefab配下の余計なRigidbodyを自動でKinematicにします。

### たまブレと照準

`WeaponInkProfile` の `Accuracy` で、武器ごとのブレを設定できます。
撃ち始めは `First Shot Spread Degrees` の小さいブレで飛び、撃ち続けると
`Spread Increase Per Shot` ずつ広がって `Max Spread Degrees` まで悪化します。
撃たない時間が `Spread Recovery Delay` を超えると、
`Spread Recovery Per Second` の速度で初弾精度まで戻ります。

おすすめ初期値:

- `First Shot Spread Degrees`: `0.2`
- `Max Spread Degrees`: `5`
- `Spread Increase Per Shot`: `0.7`
- `Spread Recovery Delay`: `0.25`
- `Spread Recovery Per Second`: `12`
照準UIを作る場合は、Canvas上に中央配置した照準用オブジェクトを作り、
`InkHudUI > Crosshair Root` にその `RectTransform` を割り当てます。
照準サイズは、`WeaponInkProfile` の現在ブレ角度をもとに、
カメラのFOVと画面高さから自動計算されます。

`InkHudUI` 側の調整項目:

- `Crosshair Base Size`: ブレがほぼない時の最低サイズ
- `Crosshair Pixels Per Degree`: 0ならカメラFOVから自動計算。手動調整したい時だけ設定
- `Crosshair Size Multiplier`: 自動計算された広がりの倍率

基本的には `Crosshair Pixels Per Degree` は `0` のままでOKです。
照準が小さすぎる/大きすぎる場合は、`Crosshair Size Multiplier` を調整してください。
連射中の照準変化が見えにくい場合は、`WeaponInkProfile` の `Spread Increase Per Shot` を少し上げるか、
`InkHudUI` の `Crosshair Size Multiplier` を上げてください。
`InkShooter` の `Crosshair Follow Per Second` は、実際の弾ブレに照準表示が追いつく速さです。
値を下げるとぬるっと遅れて動き、上げると実際のブレに近い速度で広がります。

## Blenderモーションの再生

Blenderで作ったモーションは、FBXとしてUnityに読み込み、Animator Controllerで再生します。
このプロトタイプでは `InkCharacterAnimator` を使うと、移動やイカ状態をAnimatorへ渡せます。

FBXのImport設定:

- `Rig > Animation Type`: 人型なら `Humanoid`、イカなど独自骨格なら `Generic`
- `Rig > Avatar Definition`: 同じ骨格なら `Copy From Other Avatar`、初回なら `Create From This Model`
- `Animation > Import Animation`: ON
- 歩き/待機などループする動き: AnimationClipの `Loop Time` をON
- その場歩きで移動させたい場合: 基本はRoot Motionを使わず、Animatorの `Apply Root Motion` はOFF

Animator Controllerに作るおすすめパラメータ:

- `Speed` Float: 横移動速度
- `VerticalSpeed` Float: 上下速度
- `IsGrounded` Bool: 接地中
- `IsSquid` Bool: イカ状態
- `IsSubmerged` Bool: インクに潜っている
- `IsWallSwimming` Bool: 壁泳ぎ中
- `IsDefeated` Bool: 倒されている
- `Damaged` Trigger: 被弾した瞬間
- `Defeated` Trigger: 倒された瞬間
- `Taunt` Trigger: 煽りモーション再生

設定手順:

1. ヒトモデル、イカモデルそれぞれに `Animator` を付けます。
2. Blenderから読み込んだAnimationClipをAnimator Controllerに配置します。
3. Player、またはEnemyの親オブジェクトに `InkCharacterAnimator` を付けます。
4. `Animators` にヒト用Animatorとイカ用Animatorを入れます。
5. `Position Source` にはPlayer本体、または移動している親Transformを入れます。
6. Animator Controller側のパラメータ名を、`InkCharacterAnimator` のParameter名と合わせます。

ヒトとイカでボーン構造やAnimator Controllerが別でも大丈夫です。
`Animators` に両方入れておけば、同じ状態パラメータが両方へ送られます。
使わないパラメータはAnimator側に作らなくても動きます。
`Speed` と `VerticalSpeed` は `Position Source` の位置差分から計算されます。

### 煽りモーション入力

キー入力で煽りモーションを再生するには、Player本体に `PlayerTauntInput` を付けます。

基本設定:

- `Character Animator`: Player本体の `InkCharacterAnimator`
- `Player Controller`: Player本体の `PlayerInkController`
- `Health`: Player本体の `InkHealth`
- `Taunt Key`: 煽りを出すキー。初期値は `T`
- `Taunt Trigger`: Animator Controller側のTrigger名。初期値は `Taunt`
- `Only In Human Form`: ヒト状態の時だけ煽れる
- `Ignore While Defeated`: 倒されている間は煽れない

Animator Controller側では、`Taunt` をTriggerで作り、
`Any State -> 煽りモーション` のConditionに `Taunt` を入れます。
煽りモーションから待機へ戻るTransitionは、`Has Exit Time` をON、Conditionなしにします。

## 三人称カメラ設定

本家に近い、プレイヤーを画面中央より少し下に残すカメラにしたい場合は、
`ThirdPersonInkCamera` を使います。
`CameraRoot` を中心にCameraを回す方式ではなく、Cameraの位置と注視点を別々に計算します。

導入手順:

1. `Main Camera` を選びます。
2. `ThirdPersonInkCamera` を追加します。
3. `Follow Target` に `Player` を入れます。
4. `Player Controller` にPlayerの `PlayerInkController` を入れます。
5. `InkShooter > Aim Camera` に同じ `Main Camera` を入れます。
6. `PlayerSpecialWeaponController > Aim Camera` に同じ `Main Camera` を入れます。
7. `Main Camera` はできれば `Player/CameraRoot` の子から外し、Scene直下に置きます。

おすすめ初期値:

- `Distance`: `4.2`
- `Target Height`: `1.25`
- `Look At Height`: `1.55`
- `Shoulder Offset`: `X 0.45`, `Y 0.05`, `Z 0`
- `Look Down Target Lift`: `0.8`
- `Look Up Target Drop`: `0.15`
- `Screen Vertical Offset`: `0.18`
- `Look Ahead Distance`: `7`
- `Avoid Obstacles`: ON
- `Collision Radius`: `0.22`
- `Min Distance`: `0.8`

キャラクターが画面中央に寄りすぎる場合は、`Screen Vertical Offset` と
`Look Down Target Lift` を上げます。
キャラクターが下に寄りすぎる場合は、この2つを下げます。
壁際でCameraがめり込む場合は、`Obstacle Mask` にステージのLayerを入れてください。

## カメラ切り替え

複数カメラの表示をキーで切り替えるには、空のGameObjectに `CameraDisplaySwitcher` を付けます。

基本設定:

- `Cameras`: 切り替えたいCameraを順番に入れる
- `Start Index`: 最初に表示するCamera番号
- `Disable Inactive Cameras`: 表示していないCameraを無効化する
- `Switch Audio Listeners`: 表示中CameraのAudioListenerだけ有効にする
- `Next Camera Key`: 次のCameraへ切り替えるキー。初期値は `Tab`
- `Previous Camera Key`: 前のCameraへ切り替えるキー。使わないなら `None`
- `Use Number Keys`: `1` から `9` キーで直接Cameraを選ぶ
- `Shooters To Update`: 切り替えたCameraを照準に使わせたい `InkShooter`

プレイヤーの射撃方向も表示中Cameraに合わせたい場合は、
Playerの `InkShooter` を `Shooters To Update` に入れてください。
これを入れないと、画面は切り替わっても弾は古い `Aim Camera` 基準で飛びます。

## 頭と武器の上下エイム

カメラの上下方向に合わせて頭や武器を少し傾けたい場合は、
Player、または人型モデルの親に `PlayerAimPoseController` を付けます。
この処理はAnimator再生後の `LateUpdate` で、指定したTransformへ回転を足します。

基本設定:

- `Player Controller`: Player本体の `PlayerInkController`
- `Head`: 頭ボーン、または首/頭の親Transform
- `Weapon`: 武器の親Transform
- `Head Pitch Axis`: 頭を上下に曲げるローカル軸
- `Head Pitch Multiplier`: 頭がカメラ角度に追従する強さ
- `Head Max Pitch`: 頭の最大傾き
- `Weapon Pitch Axis`: 武器を上下に傾けるローカル軸
- `Weapon Pitch Multiplier`: 武器がカメラ角度に追従する強さ
- `Weapon Max Pitch`: 武器の最大傾き
- `Only In Human Form`: ヒト状態の時だけ適用する

おすすめ初期値:

- `Head Pitch Axis`: まずは `X`
- `Head Pitch Multiplier`: `0.4` から `0.6`
- `Head Max Pitch`: `25` から `35`
- `Weapon Pitch Axis`: まずは `X`
- `Weapon Pitch Multiplier`: `1`
- `Weapon Max Pitch`: `60` から `75`

上下が逆に動く場合は、`X` を `NegativeX` に変えてください。
横や斜めに曲がる場合は、`Y` / `Z` / `NegativeY` / `NegativeZ` を試してください。
頭ボーンを直接強く回すと見た目が崩れやすいので、最初は武器だけ設定してから頭を少し足すのがおすすめです。

## 操作

- WASD / 左スティック軸: 移動
- マウス: 視点操作
- Space / Jumpボタン: ジャンプ
- 左クリック / Fire1: インク弾を発射
- R: サブウェポンを投げる
- L: サブウェポンを頭に装着
- Q: スペシャルウェポンを使う
- Left Shift: イカ状態
- T: 煽りモーション
- Tab: 次のCameraへ切り替え
- 1から9: 指定番号のCameraへ切り替え

射撃はヒト状態のときだけ可能です。
イカ状態で `Fire1` を押しても弾は出ません。

## イカ状態設定

`PlayerInkController` に人型/イカ型の切り替え設定があります。

必要な参照:

- `Humanoid Model`: 人型モデルの親GameObject
- `Squid Model`: イカ型モデルの親GameObject
- `Swim Splash Effect`: 味方インクに潜って移動中に出すParticleSystem
- `Humanoid Model Yaw Offset`: 人型モデルの初期回転に足す前方向補正
- `Squid Model Yaw Offset`: イカ型モデルの初期回転に足す前方向補正

おすすめ初期値:

- `Controlled Team`: プレイヤーなら `Player`、敵なら `Enemy`
- `Squid Key`: `LeftShift`
- `Squid Own Ink Speed`: `11`
- `Squid Unpainted Speed`: `3.2`
- `Squid Enemy Ink Speed`: `2.2`
- `Enemy Ink Force Human Seconds`: `1`
- `Show Squid On Unpainted And Enemy Ink`: ON
- 人型だけ後ろ向きの場合: `Humanoid Model Yaw Offset=180`, `Squid Model Yaw Offset=0`
- 両方そのままで正しい場合: `Humanoid Model Yaw Offset=0`, `Squid Model Yaw Offset=0`
- 坂でインク判定が抜ける場合:
  - `Surface Sample Radius`: `0.18` から `0.25`
  - `Surface Sample Start Height`: `0.8` から `1.0`
  - `Ground Sample Distance`: `1.3` から `1.8`

挙動:

- キーを押している間だけイカ状態になります。
- `Controlled Team` と同じインク上では潜伏し、モデルは非表示になります。
- 潜伏中に移動している時だけ `Swim Splash Effect` が再生されます。
- 未塗装エリアではイカ型モデルを表示し、人型より遅く移動します。
- `Controlled Team` と違うインク上でも遅く移動できますが、1秒以上触れ続けると人型に戻ります。
- 敵インクで強制解除された後は、一度キーを離すまで再変身できません。

### イカ状態の壁移動

`Squid Wall Swim` の設定で、塗られた壁への張り付き移動ができます。

おすすめ初期値:

- `Enable Squid Wall Swim`: ON
- `Wall Sample Distance`: `0.75`
- `Wall Sample Radius`: `0.18`
- `Wall Swim Vertical Speed`: `7.5`
- `Wall Swim Side Speed`: `5.5`
- `Wall Stick Force`: `2`
- `Wall Jump Up Speed`: `6.5`
- `Wall Jump Away Speed`: `4`
- `Wall Reattach Delay`: `0.25`
- `Max Wall Normal Y`: `0.35`

挙動:

- イカ状態で、自チームのインクが塗られた壁を正面に捉えると張り付きます。
- 前入力で上、後ろ入力で下、左右入力で壁沿いに横移動します。
- 壁移動中は重力を抑え、壁に吸い付く力を加えます。
- ジャンプ入力で壁から離れます。
- `Wall Reattach Delay` の間は再び壁に張り付かないため、壁ジャンプ後に押し戻され続けるのを防ぎます。

壁を登れない場合は、壁の塗り専用メッシュにUVと `MeshCollider` があり、
Layerが `Ground Mask` に含まれているか確認してください。

## 敵設定

あらかじめ決めた動きをする敵には `ScriptedEnemyController` を使います。
敵オブジェクトに `InkHealth` と `ScriptedEnemyController` を付けてください。
高低差のあるステージでは、敵に `CharacterController` を付けるのがおすすめです。

基本手順:

1. 敵オブジェクトに `CharacterController` を付けます。
2. 敵の開始位置に空オブジェクトを置き、`EnemySpawn` などの名前にします。
3. 敵が通る場所に空オブジェクトを複数置き、`EnemyRoute_01`, `EnemyRoute_02` のようにします。
4. 敵の `ScriptedEnemyController > Spawn Point` に開始位置を入れます。
5. `Route Points` に通過地点を順番に入れます。
6. 敵の `InkHealth > Team` を `Enemy` にします。
7. リスポーンさせたい場合は、`InkHealth > Disable On Defeat` をOFFにします。

`ScriptedEnemyController` の主な設定:

- `Spawn Point`: 試合開始/リスポーン時に戻る位置
- `Route Points`: 敵が順番に向かう地点
- `Loop Route`: 最後の地点まで行った後、最初に戻る
- `Loop Return Index`: 最後の地点まで行った後に戻る `Route Points` の番号。`0`なら最初に戻る
- `Move Speed`: 移動速度
- `Turn Speed`: 向きを変える速さ
- `Arrive Distance`: 地点に到着したとみなす距離
- `Use Character Controller`: `CharacterController` を使って移動する
- `Gravity`: 落下の強さ
- `Grounded Stick Force`: 接地中に地面へ軽く押し付ける力
- `Respawn Delay`: 倒されてから復活するまでの秒数
- `Restart Route On Respawn`: リスポーン時にルートを最初からやり直す
- `Defeat Explosion Prefab`: 倒された瞬間に出すインク爆発Prefab
- `Defeat Explosion Offset`: インク爆発を出す位置のオフセット
- `Auto Hide Child Renderers`: 倒れている間、Enemy配下のRendererを自動で非表示にする
- `Disable Renderers While Defeated`: 倒れている間だけ非表示にするRenderer
- `Disable Colliders While Defeated`: 倒れている間だけ無効にするCollider
- `Paint While Moving`: 移動中に敵インクを塗る
- `Auto Shoot`: 自動で弾を撃つ
- `Weapon Profile`: 敵が使う `WeaponInkProfile`
- `Fire Point`: 弾を出す位置
- `Shoot Target`: 狙う対象。未設定なら敵の正面に撃つ
- `Shoot Target Offset`: `Shoot Target` のどの高さを狙うか
- `Shoot Range`: `Shoot Target` を狙って撃つ最大距離
- `Min Horizontal Aim Distance`: ほぼ真下/真上を狙ってしまう時に正面撃ちへ戻す距離
- `Fire Rate Multiplier`: `WeaponInkProfile > Fire Rate` への倍率
- `Use Weapon Spread`: `WeaponInkProfile` の初弾ブレを使う

試合開始時とリスポーン時に、敵は `Spawn Point` に戻り、
`Route Points` の最初から同じ動きを始めます。
通常ループでは最後の地点の後に `Loop Return Index` の地点へ戻ります。
例えば `Route Points` が `0,1,2,3,4` で `Loop Return Index=2` の場合、
最初は `0→1→2→3→4` と進み、その後は `2→3→4→2...` を繰り返します。

高低差用の `CharacterController` おすすめ設定:

- `Slope Limit`: `45` から `60`
- `Step Offset`: `0.3` から `0.6`
- `Skin Width`: `0.05` から `0.08`
- `Height` / `Radius`: 敵モデルに合わせる

敵に弾を撃たせる場合:

1. 敵用の `WeaponInkProfile` を作ります。
2. `Projectile Prefab` に敵が撃つ弾Prefabを入れます。
3. `Damage`、`Projectile Speed`、`Fire Rate`、`Paint Radius` などを設定します。
4. 敵の `ScriptedEnemyController > Auto Shoot` をONにします。
5. `Weapon Profile` に敵用Profileを入れます。
6. `Fire Point` に銃口位置のTransformを入れます。
7. プレイヤーを狙うなら `Shoot Target` にプレイヤーのTransformを入れます。

`Shoot Target` が未設定の場合、敵は自分の正面方向に撃ちます。
敵弾がその場で真下に落ちる場合は、まず `Weapon Profile > Projectile Speed` が0になっていないか確認してください。
次に、`Shoot Target Offset` の `Y` を `1` から `1.5` くらいにして、プレイヤーの足元ではなく体の中心を狙わせてください。

## UI設定

基本HUDを作る場合は、Canvasに `InkHudUI` を付けます。

おすすめのUI要素:

- `Time Text`: 残り時間表示用の `Text`
- `Player Score Text`: 自分の塗り率表示用の `Text`
- `Enemy Score Text`: 敵の塗り率表示用の `Text`
- `Player Paint Point Text`: 自分の塗りポイント表示用の `Text`
- `Enemy Paint Point Text`: 敵の塗りポイント表示用の `Text`
- `Special Gauge Text`: スペシャルゲージ表示用の `Text`
- `Ink Slider`: インク残量表示用の `Slider`
- `Health Slider`: 体力表示用の `Slider`
- `Special Slider`: スペシャルゲージ表示用の `Slider`
- `Result Text`: 試合終了時の `WIN` / `LOSE` / `DRAW` 表示用の `Text`
- `Player Score Fill`: 自分の塗り率バーに使う `Image`
- `Enemy Score Fill`: 敵の塗り率バーに使う `Image`

`InkHudUI` の `Game Manager` には `InkGameManager`、
`Player Shooter` にはプレイヤーの `InkShooter` を割り当てます。
`Player Health` にはプレイヤーの `InkHealth` を割り当てます。
`Player Special` にはプレイヤーの `PlayerSpecialWeaponController` を割り当てます。
未設定でも自動検索しますが、手動で入れる方が安全です。

塗りポイントは、現在の塗り率ではなく、塗った瞬間に加算される累計ポイントです。
相手に塗り替えられても減らず、自分が塗り返すとさらに増えます。
すでに自分の色で塗られている場所をもう一度塗っても、ポイントは増えません。
ポイントになるのは、その塗りで新しく自分の色になった面積だけです。
ポイント量は `InkGameManager > Paint Point Multiplier` で調整できます。
まずは `10000` くらいがおすすめです。
1発あたりのポイントが少なすぎる場合は、この値を上げてください。
`PaintableSurface > Paint Point Ink Threshold` を下げると、薄い外周インクもポイント面積として数えやすくなります。
表示名を変えたい場合は、`Player Paint Point Prefix` と `Enemy Paint Point Prefix` を変更してください。

## 重要な注意点

- このシェーダはBuilt-in Render Pipeline向けの書き方です。
  URP/HDRPで使う場合は、`Ink/Blend Surface` をShader Graphまたは
  URP/HDRP向けHLSLに変換してください。
- スコア計算はCPU Readbackを使っています。
  プロトタイプでは問題ありませんが、大きなゲームにする場合は
  Async GPU Readbackや低解像度のスコア用テクスチャを使うのがおすすめです。
- インクが意図しない場所に出る場合は、まずメッシュのUVを確認してください。

## マテリアルの使い分け

塗り専用メッシュを透明なインク表示にする場合は、以下の2つを使います。

- `M_InkOverlay`
  - Shader: `Ink/Transparent Overlay`
  - 入れる場所: 塗り専用メッシュの `Mesh Renderer > Materials`
  - 役割: 未塗装部分は透明、塗った部分だけインク色で表示
  - `Painted` ログが出るのに見えない場合は、塗り用メッシュの面が裏向きの可能性があります。このシェーダは両面表示にしてあります。
- `M_InkStamp`
  - Shader: `Ink/Stamp`
  - 入れる場所: `PaintableSurface > Paint Stamp Material`
  - 役割: RenderTextureにインク跡を書き込む内部処理

`Ink/Blend Surface` は、塗り専用メッシュ自体を灰色の地面として表示したい
検証用のシェーダです。見た目用ステージが別にある場合は、
`Ink/Transparent Overlay` を使ってください。

## Debug.Logの使い方

各主要スクリプトには `Debug Logs` というチェック項目があります。
ONにすると、Unity Consoleに以下のような情報が出ます。

- `PlayerInkController`: 初期化、ジャンプ、踏んでいるインク状態の変化
- `InkShooter`: 初期化、発射、発射できない理由
- `InkProjectile`: 発射、衝突、塗り判定の成功/失敗
- `InkSubWeaponController`: サブウェポン使用、使用できない理由
- `SprinklerSubWeapon`: 接着、塗り処理
- `InkHealth`: 回復、ダメージ、撃破
- `ScriptedEnemyController`: リスポーン、ルート移動
- `PaintableSurface`: 初期化、塗り処理、スタンプMaterial未設定
- `InkGameManager`: 試合開始、試合終了、スコア更新

まず動作確認するときは、`InkShooter`、`InkProjectile`、
`PaintableSurface` の `Debug Logs` をONにするのがおすすめです。
`Painted` ログの `maskPixel=(r,g,b,a)` で、Playerインクなら `r`、Enemyインクなら `b` が増えているか確認できます。

- `maskPixel` の `r` / `b` が増えている: マスクへの書き込みは成功。Material/Shader/Renderer表示側の問題
- `maskPixel` がずっと `(0,0,0,...)`: Paint関数は呼ばれているが、スタンプMaterialやUVの問題

`maskPixel=(0,0,0,0)` のままなら、まず `M_InkStamp` を確認してください。

- `M_InkStamp > Shader`: `Ink/Stamp`
- `PaintableSurface > Paint Stamp Material`: `M_InkStamp`
- `M_InkStamp` に `Ink/Transparent Overlay` や `Ink/Blend Surface` を設定しない

`PaintableSurface > Apply Mask To Material Instance` はON推奨です。
ONにすると、`MaterialPropertyBlock` だけでなくRendererのMaterialインスタンスにも `_InkMask` を渡します。
