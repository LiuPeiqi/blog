---
layout: post
title: "子弹、手雷、投掷物系统"
data: 2021-02-07
tags: GamePlay Unity Physics
---

## 背景与需求

作为射击游戏里的核心功能，子弹要承担很多重要表现和逻辑判定。这里的子弹是游戏里广义上所有具有飞行和物理碰撞判定的功能，同时也把激光当成一个速度无限大的连续多发的特殊子弹。

当前设计里除了对子弹提出了多种运行轨迹的要求，还要尽可能模拟相对符合直觉的物理表现以及碰撞反弹效果。尤其是引入空气阻力的概念，给玩家制造一种子弹飞行远处下坠感强烈而近处射击相对平直易于瞄准的手感。

最后多玩家多怪物在游戏过程中随时会发射大量子弹，因此子弹系统的性能也是一项必要的考虑内容。

## 方案与选型

当前项目使用Unity的原生物理引擎，检测碰撞一般来说OnCollision，OnTrigger和RayCast都可以实现。

另一方面，逻辑的实现可以放到各个子弹上，也可以把所有子弹统一更新，光复杂度看起来都区别不大。

而具体怎么选定，不如先就一些逻辑和性能上的限定条件进行讨论，再选出最合适的方法。

### 阻挡与穿透

最朴素的实现就是子弹遇到Collider就停止，但是这样不能满足具体的业务需求：友军不阻挡、子弹可以多次穿透伤害等。因此首先就排除了OnCollision方法。

穿透逻辑比较多样，所以这块设计时先确认拆分逻辑：主要的运动和检测这样性能消耗比较大、逻辑比较固定的部分放到C#里，穿透、命中等筛选逻辑和特效播放放置到脚本里方便日后同业务调整而热更新。

### 高速运动的物体物理检测失效

Unity物理有个特性，默认的Rigidbody的CollisionDetection不能检测到高速的运动碰撞，而[选择CCD](https://docs.unity3d.com/Manual/ContinuousCollisionDetection.html)的两个选项又有细微差别：

1. ContinuousDynamic可以很好的检测运动方向的碰撞，但是对旋转造成的碰撞则无法检测。
1. ContinuousSpeculative可以很好的检测高速旋转过程中的碰撞，但是运动方向会受到算法导致的碰撞角度偏差。

对于子弹逻辑上的述求，使用ContinuousDynamic就可以满足高速检测了。但是这里又有一个业务上的难点：业务上想实现相对多样（无重力的、有重力的、有阻力的、自动锁敌的）和规则相对自由的飞行轨迹，因此使用位置计算的算法比使用速度计算的算法相对简单，所以不太想用Rigidbody的速度来驱动子弹运动。

比如自动锁敌的追踪子弹，使用位置计算方法如下：

```
var pos = Vector3.MoveTowards(transform.position, target.position, velocity * dt);
transform.position = pos;
```

而使用速度计算的话还要多一些速度计算的步骤，虽然麻烦，但还没有到舍弃RigidBody的程度：

```
var pos = Vector3.MoveTowards(transform.position, target.position, velocity * dt);
rigid_body = transform.GetComponent<RigidBody>();
rigid_body.velocity = (pos - transform.position) / dt
```

### 子弹一定需要GameObject么

通常做物理表现时都是模型GameObject上挂个RigidBody和Collider来做，就算有些业务不想表现子弹飞行过程的话，也会用一个空的看不见的模型来实现。

但是如果限定子弹一定要有GameObject的话，那么加载问题就不得不也考虑进来：发射一颗子弹后，逻辑上要不要等待异步加载子弹的过程的时间。

1. 如果等待，那么相对于第一发子弹会发生效果滞后。
1. 如果不等待，那么就要为加载过程中再写一个特殊逻辑。

再加上一小节的讨论内容，如果子弹的碰撞检测舍弃OnTrigger，使用RayCast来检测，那么逻辑反而相对简单了一些：

1. 因为高速运动的子弹变成每帧静态的线段检测，所以解决了高速穿墙的问题。速度快的子弹只比速度慢的子弹线段长而已。
1. 因为使用RayCast的线段检测，所以子弹有没有GameObject变的不重要了。
    1. 可以找个地方存放上一帧子弹的位置和速度，直接通过逻辑算出下一帧位置拉射线就行。
    1. 看不见弹道的子弹直接不加载模型，更省资源。
    1. 看的见的子弹就正常异步加载，没加载上来的时候只是看不见，不影响正常的物理检测逻辑。
1. 可以分别实现有碰撞半径的子弹（SphereCast）和无半径的子弹（RayCast）。
1. 激光和光剑可以有条件的抽象成距离有限、速度无限大的子弹。
    * 光剑挥舞的过程，如果光剑半径足够小或者转动足够快，那么就会遗漏检测。
    * 如果使用collider和ContinuousSpeculative的方式则可以解决这个问题。
    * 尽管如此，这里考虑了一下业务场景，通过约束转动速度，激光半径和命中距离来解决问题，不再引入新的检测模式。

![激光夹角示意]({{ site.url }}/images/bullet_system/laster_missing.png)

最后，子弹的逻辑不做在各自GameObject上的另一个原因是为了增加子弹系统的运算性能：

* 一个Update循环n次的计算性能表现要远远好于n个Update执行一次。
* 如果子弹数据使用结构体数组，那么这样还能利用内存访问的局部性。

### FixedUpdate还是Update

常规的物理功能都会放到FixedUpdate里，简单来说Update的帧率和间隔不稳定的，因为多数物理过程都是对运动速度积分来求运动轨迹，所以必须使用FixedUpdate来提供一个稳定的dt来求解稳定的积分结果。

![两种Update的积分示意图]({{ site.url }}/images/bullet_system/fixedupdate_vs_update.png)

但是，不稳定的dt只对非线性的曲线积分有影响（变加速运动），而自由落体运动是线性的匀加速运动，其实不稳定dt不会产生误差。

一般大家写路程计算的时候习惯性的：

```
var velocity = v0 + gravity * dt;
var dis = velocity * dt;
v0 = velocity;
```

这样计算其实是错误的，使用FixedUpdate能表现正常只是凑巧错的均匀。正确的方法是使用平均速度来做计算：

```
var velocity = v0 + gravity * dt;
var average_v = (v0 + velocity) * 0.5f;
var dis = average_v * dt;
v0 = velocity;
```

![两种Update的积分示意图]({{ site.url }}/images/bullet_system/average_v.png)

因此对于匀速运动和匀加速运动来说，使用Update并没有什么问题。从而避免使用FixedUpdate以节省计算性能：当发生卡顿时，两帧Update之间会插入更多的FixedUpdate，如果FixedUpdate的计算负担繁重，那么这会进一步增加卡顿表现。

但是子弹运行轨迹里也并不是没有变加速运动，比如要引入子弹飞行阻力的话，那么子弹就会成为变加速运动。

* 引入子弹飞行阻力的原因是想表达高速子弹的前段飞行平直，后段飞行下坠明显的更进一步的拟真。

这里分两步处理这个问题：

* 现行的逻辑还是通过平均速度作为计算路径的参考，并合理估计误差，差距不大就忽略不计了。

如果未来不能接受这个误差，那么仅把有阻力影响的子弹单独处理，有两个改进方案：

1. 把有阻力影响的子弹单独放到FixedUpdate里计算。
1. 还是使用Update来计算，只是子弹额外记录一个remain_time字段，然后在Update里用固定的间隔执行一次或多次路径计算来模拟FixedUpdate的行为。

```
void Update() {
    var dt = GetTimeNow() - last_time;
    foreach(var bullet in bullets) {
        bullet.remain_time += dt;
        var n = (int)(bullet.remain_time / INTERVAL);
        bullet.remain_time -= n * INTERVAL;
        for(var i = 0; i < n; ++i) {
            SimulatePath(bullet);
        }
    }
}
```

### Layer不够用了怎么办

写跟碰撞相关的逻辑经常会碰到Layer不够用的问题：

1. 主角和其他角色不碰撞，子弹又要跟其他角色碰撞
1. 有些空气墙跟角色碰撞，子弹又想穿过去。
1. 为了优化性能和表现，和角色碰撞的collider是顶点少的粗模，而子弹命中的Collider又想是顶点多的高模。
1. 还有为了其他各种业务使用的特殊层级。

为了节省Layer资源，这里子弹其实理论上不需要额外的层来处理阻挡或镂空逻辑，而是给具体的单位上挂个新的mask标记组件来表征业务逻辑，拓展Layer以外的数量。

### MonoBehavior还是static class

这里Mono表达的是Mono单例的意思，这俩种实现其实没有多大差别，但是还是推荐使用静态类来实现子弹系统的主循环。主要的原因有两个：

1. 如果要用MonoBehavior，那么还要再指定Script Execution Order。直接在代码逻辑不容易判定于其他系统的执行的先后次序。
1. MonoBehavior单例在关闭游戏时总会发生先后交错释放导致再激活单例实例而报错的情况。

## 运动

### 直线和抛物线运动

除了匀速直线运动和理想抛物线运动以外，带阻力的直线运动和带阻力的抛物线运动都参考[知乎-物理学导论-Projectiles](https://zhuanlan.zhihu.com/p/124970286)的文章。为简化计算量，这里采用的是线性阻力计算。

有阻力影响的水平速度分量：

```
var v1 = v0 * Mathf.Exp(-damping * dt);
var distance = (v1 + v0) * 0.5f * dt;
v0 = v1;
```

*如果一个子弹有阻力而且还可以动态的被额外的逻辑影响的话，那么可以实现黑客帝国中子弹悬停的效果。* :)

有阻力和重力影响的垂直速度分量：

```
var vter = -g / damping;
var v1 = vter + (v0 - vter) * Mathf.Exp(-damping * dt);
var distance = (v1 + v0) * 0.5f * dt;
v0 = v1;
```

![阻力抛物线]({{ site.url }}/images/bullet_system/damping_vs_gravity.png)

由上图可以看到带阻力子弹的轨迹在飞行的前段其实差别不大，但是在飞行后段会很快下坠。阻力特性用来给玩家感受子弹在远处更强烈的下坠，和近处射击手感平直更容易瞄准。

### 拟合的曲线运动

怪物通常也会发射一个曲线子弹，这个曲线子弹如果也使用抛物线的话，那么发射角度、命中点比较难以计算（求解弹道方程，解还不唯一）。这里使用了两个正弦曲线来拼接一个曲线弹道。这么做的原因除了方便计算外，另一个是怪物可能是在室内发射的曲线子弹，使用两个正弦来拼接曲线容易得到一个在接缝处导数相等的曲线，又方便配置弹道的最大高度，避免打到房顶上。

```
var time_factor = (last_update_time - start_time) / (life_time - start_time);
var ratio = Mathf.Clamp((0.5f - time_factor) * 2, -2, 1);

var caster_pos = caster.transform.position;
var target_pos = target.transform.position;
var height = Mathf.Max(caster_pos.y, target_pos.y) + cfg_delta_height;
var dis = (target_pos - caster_pos).normalized * velocity * dt;
var y = 0f;
if (ratio > 0) {
    y = (height - caster_pos.y) * Mathf.Sin(Mathf.Deg2Rad * 90 * (1 - ratio)) + caster_pos.y;
} else {
    y = (height - target_pos.y) * Mathf.Sin(Mathf.Deg2Rad * 90 * (1 + ratio)) + target_pos.y;
}
var point = new Vector3(dis.x, y - pos.y, dis.z);
```

![正弦拼接示意]({{ site.url }}/images/bullet_system/sin_sin.png)

### 碰撞与反弹

如果子弹碰撞以后要反弹，而不是直接销毁，那么逻辑上有一些不得不注意的细节：

* 如果是有半径的子弹碰到Collider，那么返回的`raycastHit.point`并不是子弹要停留的点，而是撞击点。真正子弹停留的点是：`point = raycastHit.point + raycastHit.normal * bullet.radius`。即，向着法线移动一个半径的距离的点才是子弹发生碰撞时的位置。

![碰撞点和半径关系]({{ site.url }}/images/bullet_system/collision.png)

* 如果子弹不是匀速运动的，那么碰撞了以后，在撞击点计算出射速度时不能直接使用轨迹计算得出的终速度，而是要用真实路程反向矫正撞击点的终速度。匀加匀减速运动的撞击拟合速度计算如下：

```
// p0: 当帧初始位置; p1:当帧结束位置; hit_pos:撞击位置
var ratio = (hit_pos - p0).magnitude / (p1 - p0).magnitude;
var velocity = Vector3.Lerp(v0, v1, ratio);
```

* 同样，当撞击时，子弹使用的时间也不是dt了，而是需要使用撞击时的路程除以速度以矫正：

```
var real_time = (hit_pos - p0).magnitude / velocity.magnitude;
```

* 由上一步计算得出，没有消耗完的时间也不能丢弃，要叠加到下次运动模拟中。

`remain_time = dt - real_time;`

*这里有个表现特性，如果当帧通过再迭代执行一次路径模拟把remain_time消耗完，那么子弹在碰撞时的速度表现是相对稳定的，但是玩家有可能看不到碰撞点只能看到子弹反弹的结果。如果把remain_time积攒到下一帧再运算时，玩家会清晰的看到子弹在哪里碰撞（因为子弹在这个位置多停留了），但是下一帧的出射速度会因时间压缩而增加：`v = (dt + remain_time) / dt * v`*

* 如果子弹在斜波连续碰撞（特别快速的连续碰撞的表现是贴地翻滚），那么要额外在碰撞当帧给子弹增加一个重力影响在斜坡上的速度分量，用以增强斜坡滚动加速能力：

```
bullet.velocity += Vector3.Reflect(Vector3.down, raycastHit.normal) * gravity * dt;
```

* 如果子弹在碰撞时还要表现翻滚和旋转，那么需要用两次实际撞击点的距离和子弹半径算出翻滚角度。

```
var s = Vector3.Distance(p0, p1);
var alpha = s / (Mathf.PI * 2 * bullet.radius) * 360;
var rot = Quaternion.AngleAxis(alpha, Vector3.Cross(normal, bullet.forward));
bullet.localRotation = rot * bullet.localRotation;
```

![翻滚示意]({{ site.url }}/images/bullet_system/roll.gif)

* 碰撞以后的动能损失

因为模拟的过程中其实没有处理质量影响，所以这里的动能损失其实速损失。由一个-1到1的参数控制：`velocity *= (1 - factor);`。当配置为1时，碰撞完全损失速度，表现是粘到墙上；当配置是0时，是完全弹性碰撞；反弹的出射速度等于入射速度。当配置小于0时是增强弹射速度，表现是越弹越快。搭配使用增强弹射和命中次数可以实现某些游戏里的弹射子弹效果。

* 碰撞位置的同步。

当前没有同步碰撞的位置，只是同步了子弹的出射位置和方向。考虑是因为大部分碰撞是跟静态场景碰撞，所以根据上述章节的讨论，各个端分别模拟的结果其实差别不是很大。未来如果强制同步碰撞位置，那么需要增加一个等待/记录的逻辑，当子弹碰撞以后原地等待或使用已经收到的新的同步速度和方向(这样的表现等同于PUBG端游，网络卡时，手雷停到碰撞点等同步)。

### 激光

如前文介绍，激光是一个特化版子弹，主要需要配置的参数是激光半径、最大出射距离、生命时间和最小伤害时间。在实现激光时需要额外记录的上一次的发射位置和方向（Ray）和当前位置和方向（Ray），最后用`var n = Mathf.Ceil(dt / interval);`得出需要发射的子弹数插值依次计算。

```
if(bullet.life_time > now) {
    for(var i = 0; i < n; ++i) {
        var factor = ((float)i) / ray_count;
        var pos = Vector3.Lerp(old_ray.origin, cur_ray.origin, factor);
        var dir = Vector3.Slerp(old_ray.direction, cur_ray.direction, factor);
        BulletHitLogic(pos, dir, raduis, length)
    }
} else {
    bullet.Destory();
}
```

![激光]({{ site.url }}/images/bullet_system/laser.gif)

### 还是超大地图范围的影响

因为地块移动，所以子弹在计算位置时要么直接使用GlobalPos来计算，要么就使用WorldPos来计算，然后当地块发生移动的时候统一重新校准所有还在飞行中的子弹。但是这样只能保证子弹的逻辑位置是正确的，如果子弹的模型上由world类型的拖尾特效，那么这个特效会因为地块的突然移动而出现表现异常。

另一个是，如果是直接使用GlobalPos来计算，那么子弹位置中使用UnityEngine.Vector3记录的精度问题又不得不考虑进来。如果地图边界小于8km的话，那么可以认为GlobalPos最少有0.01的精度，在位置设置中额外矫正一下其实问题也不大。

### 参考系影响

这块内容其实在[惯性参考系]()中有讨论，不做额外说明。

## 命中表现

命中表现纯粹是一个堆工作量的内容，堆的越多，表现就越丰富。设计一个合理的表现配置逻辑有助于花少量工作来拓展表现能力。

这里对命中表现做两种分类，一种是固定表现特效，一种是依据材质的差异表现特效。固定表现的特效很简单，条件到了播放就行。材质特效在配置上有过争议，为了表现不同枪支打到不同材质上播放不同的特效，早先配置的行索引是枪械类型，列索引是当前游戏里的物理材质，逻辑按照当前枪械类型和当前命中的材质索引最终表现。后面考虑到材质拓展和地形层次（TerrainLayer）拓展的方便性，把材质配置为行索引，具体的子弹ID作为列索引而舍弃枪支类型（武器关联技能，技能关联子弹，子弹关联表现。用单线条的配置关联逻辑取代之前交叉的关配置关联逻辑）。调整后的配置逻辑可以非常简单的来丰富或裁剪材质命中表现，比如想要或关闭手雷打地上弹起尘土，打水面上溅起水花这样的表现，就修改对应材质两行数据中手雷子弹列的下材质特效就成。

特效播放的逻辑里有一个需要在制作流程上就要明确的约定：美术制作前就要明确当前特效的参考轴向是什么。比如一个贴在地上的弹孔特效就要是参考具体地面**法线方向**；一个空中爆炸榴弹的烟火特效就要是参考榴弹**前进方向**；未来可能还有有一些水面特效只能参考空间坐标下的**Y方向**。资源制作明确轴向以后，配置时也要给逻辑指明使用哪种方式来播放，不然就会出现奇奇怪怪的特效偏转不正确现象。

除了上述的空间绝对位置，一些命中特效要跟随命中目标移动。这里有三种跟随方式：

1. 跟随并使用骨骼位置。（目标的特定位置播放特效）
1. 跟随骨骼运动，并保持命中位置和骨骼之间的相对位置。（表现打到某具体位置，并跟着它动，怪物命中后的创伤、箭羽等）
1. 跟随命中Collider运动，并保持命中Collider的相对位置。（跟上一条一样，只是表现更精细，比如打到门上后，门开了并带着弹痕移动）

未来还可以扩展一些水面、玻璃这样的可击碎表现。其中水面的实现相对简单，检测到碰撞直接播放材质特效就行；但是玻璃需要服务器（可同步的）或本地（离线种的表演单位）计时刷新击碎状态，不能连续不断的打碎。

## 其他相关联的特性

### 怪物的瞄准IK

AimIK给主角使用相对简单，表现也正常，但是给怪物（炮塔、机器人）使用时，就会放大问题：IK启用和关闭时没有缓动；IK解算器在一些边界角度无法求解。考虑到怪物瞄准时需要转动的骨骼是非常有限的，所以这里造了一个分别限定水平和垂直旋转规则的简易IK的轮子：

![IK改进]({{ site.url }}/images/bullet_system/aimik_vs_aimctrl.gif)

### 射线检查与子弹出射点

TPS射击模式有个问题是当角色顶着墙或者怪物射击时，枪口是跟对应Collider是相交的，这样导致：

1. 双射线（摄像机到目标点，目标点到子弹出射点）中第二条射线已经在被检测的Collider内部，因此检测失效无法击中目标。
1. 子弹特效的出射方向大幅偏离枪口方向，跟直觉理解有差别。

另一个现象是，如果有能量防护罩玩法，角色跟防护罩没有碰撞，但是阻挡子弹，玩家穿过或半穿过防护罩，但是摄像机没有，那么双射线仍然会选错目标点（可能会选到防护罩上，而不是角色前方）。

完全解决这个问题需要细节调整：

1. 双射线的筛选规则中需要指定当第一条射线距离目标点多近时第二条射线失效的容错距离。当目标已经贴脸时就不启用枪口检查了。
1. 双射线的筛选规则，两个射线的筛选规则相互独立。这样可以配置出两个不一样的目标选中规则。比如摄像机不筛选防护罩，但是枪口射线选中防护罩。如果角色和摄像机都没有穿过防护罩，那么相机射线选中的目标在第二次枪口射线选中时被阻挡；如果角色穿过但是相机没有穿过，那么枪口射线能正常选中目标，符合直觉体验。
1. 子弹的出射挂点不使用枪口位置，而是使用靠近枪托的位置。这样可以最大限度保证贴墙射击时，子弹的出射方向不会偏差太大。（PUBG和全境封锁2是这么做的）

## 结

现在的设计里其实还存在单位运动导致物理检测失效的情况，只不过不是子弹高速运动导致的失效，而是单位运动速度大于或接近子弹运动速度时发生（当前没有观测到发生，通过逻辑推断出会存在这样的Bug）。原因也同一开始CCD里讨论的一样。这里暂时先放过这个问题，要具体看游戏里出现的频率和情况，如果部分条件下不能接受，那么还可以选择性的给有限的子弹上加上RigidBody和CCD检测来解决问题。

最后，子弹系统主体上是个不太麻烦的内容，但是设计一个相对容易扩展，能在后期不断投入工作量就能丰富细节的系统也是一个有挑战的内容。
