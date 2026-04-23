```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System;
public interface IStateMachineOwner {} //状态机宿主
//角色状态机
public class StateMachine
{
    private StateBase currentState;
    private IStateMachineOwner owner;
    private Dictionary<Type, StateBase> stateDic = new Dictionary<Type, StateBase>();

    public StateMachine(IStateMachineOwner owner) {  this.owner = owner; }
    //进入动画状态
    public void EnterState<T>() where T : StateBase, new()
    {
        if (currentState != null && currentState.GetType() == typeof(T)) return;
        if (currentState != null)
        currentState.Exit();
        currentState = LoadState<T>();
        currentState.Enter();
    }

    private StateBase LoadState<T>() where T: StateBase, new()
    {
        Type stateType = typeof(T);
        if(!stateDic.TryGetValue(stateType, out StateBase state))
        {
            state = new T();
            state.Init(owner);
            stateDic.Add(stateType, state);
        }
        return state;  
    }

    public void Stop()
    {
        if (currentState != null) 
        currentState.Exit() ;
        foreach (var state in stateDic.Values)
            state.Destory();
        stateDic.Clear();
    }
}
```

> [!info]- 💡 点击展开查看：第一部分 - HR的办公室（变量与构造）
> 
> **`public interface IStateMachineOwner {}`**
> 
> - **大白话**：这是一个“老板徽章”。任何想使用状态机的人（比如怪物本体、玩家本体），只要胸前挂上这个徽章，HR 就承认你是我的直属老板。
>     
> 
> **三个核心变量：**
> 
> 1. `private StateBase currentState;`
>     
>     - **大白话**：**“正在打工的那个倒霉蛋”**。HR 必须时刻盯着现在是谁在干活（比如现在是奔跑状态）。
>         
> 2. `private IStateMachineOwner owner;`
>     
>     - **大白话**：**“直属大老板是谁”**。HR 需要知道自己是在为丧尸A服务，还是丧尸B服务，方便以后告诉员工。
>         
> 3. `private Dictionary<Type, StateBase> stateDic ...`
>     
>     - **大白话**：**“员工休息室（人才库）”**。这是一个高级的字典（Dictionary）。前面是工种名字（比如 `Type` 为跑、跳），后面是具体的员工本人。
>         

> [!info]- 💡 点击展开查看：第二部分 - `EnterState<T>()` 换班仪式
> 
> 怪物遇到玩家，大脑下令：“给我切换到攻击状态！”这时候 HR 就会执行这个方法。
> 
> 4. `if (currentState != null && currentState.GetType() == typeof(T)) return;`
>     
>     - **防瞎搞机制**：如果现在正在打工的人，和你要切换的人是同一个（比如丧尸已经在跑了，你还一直下令让它跑），那就直接 `return`（别烦我，打回指令），防止动画鬼畜。
>         
> 5. `if (currentState != null) currentState.Exit();`
>     
>     - **交接班**：如果当前有人在干活，立刻让他停下手里的活，执行“打卡下班” (`Exit`) 擦屁股。
>         
> 6. `currentState = LoadState<T>();`
>     
>     - **喊人**：去休息室把下一个要干活的员工叫过来（下文详细讲）。
>         
> 7. `currentState.Enter();`
>     
>     - **上岗**：新员工，打卡上班 (`Enter`)！
>         

> [!info]- 💡 点击展开查看：第三部分 - `LoadState<T>()` 抠门老板的招人逻辑
> 
> 这段代码极其精妙，专业名词叫**“懒加载 (Lazy Load)”或“对象池思维”**。它的核心思想是：能不花钱招新员工，就绝对不招！
> 
> 8. `if(!stateDic.TryGetValue(stateType, out StateBase state))`
>     
>     - HR 先去 **“员工休息室 (`stateDic`)”** 里翻一翻。
>         
>     - 比如大脑下令切到“奔跑状态”，HR 先看休息室里有没有叫“奔跑”的员工。如果有，直接拽出来干活（极其省电脑性能）。
>         
> 9. `state = new T();` 到 `stateDic.Add...`
>     
>     - **如果休息室里没有呢？（说明游戏刚开始，这个状态是第一次被用到）**
>         
>     - HR 只能咬牙花钱招一个新员工：`new T()`。
>         
>     - 赶紧给他办入职手续，告诉他老板是谁：`state.Init(owner)`。
>         
>     - 最后，把它登记到休息室的花名册上：`stateDic.Add`。这样下次再跑的时候，就不用花钱重新 `new` 啦！
>         

> [!info]- 💡 点击展开查看：第四部分 - `Stop()` 关门大吉
> 
> 丧尸被打死了，尸体要被删除了，HR 部门也要跟着解散。
> 
> 10. 先让正在干活的人下班：`currentState.Exit()`。
>     
> 11. 打开休息室的大门：`foreach (var state in stateDic.Values)`。
>     
> 12. 把所有人统统开除、销毁：`state.Destory()`。
>     
> 13. 彻底清空花名册：`stateDic.Clear()`。
>     

```

### 🏆 核心亮点梳理

这段代码最牛的地方，就在于那个 `Dictionary` (字典) 和 `LoadState` (抠门招人法)。

新手写状态机，经常是每次切换状态都 `new RunState()`，切回待机又 `new IdleState()`。这就像现实中，开一次会就招一个人，开完会直接把人杀掉，下次开会再重新招人。这种写法会让游戏疯狂产生垃圾（GC），导致游戏严重卡顿。

而你这份代码里的 HR 非常聪明，他准备了一个 **员工休息室 (`stateDic`)**。用过的人就放在休息室里，下次随叫随到，这属于极其标准和高级的商业级游戏写法！

现在的状态机拼图已经完整了：
1. **状态基类 (`StateBase`)**：负责定规矩。
2. **状态大管家 (`StateMachine`)**：负责按规矩换班和招人。

太帅了！你现在不仅看懂了代码，甚至连它为什么要这么优化的底层逻辑都拿下了！这个 HR 大管家的运作逻辑，哪一步
```