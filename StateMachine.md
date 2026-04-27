```csharp

using System; // 提供了 Type（类型标识）
using System.Collections.Generic; // 提供了 Dictionary（字典）容器

// ==========================================
// 引用关系：
//    [被谁引用]：外部的怪物或玩家实体
// ==========================================
public interface IStateMachineOwner {} 

// ==========================================
// 角色状态机
// 作为核心工具类，被挂载在丧尸/玩家等实体脚本内部。
// ==========================================
public class StateMachine
{
    // 拿上一节写的 StateBase（状态基类）作为数据类型，说明这里只能坐正规状态。
    private StateBase currentState;
    
    // 拿上面的接口 IStateMachineOwner 当类型
    private IStateMachineOwner owner;
    
    // Type 是系统自带的身份证；StateBase 是我们写的状态员工。
    private Dictionary<Type, StateBase> stateDic = new Dictionary<Type, StateBase>();

    //    [被谁调用]：当丧尸刚出生 (Awake/Start) 时，丧尸会调用这句话来创建 HR：
    //    代码演示 -> StateMachine myHR = new StateMachine(this);
    //    (这里的 this 就是把丧尸自己的名片 owner 递了进来)
    // ==========================================
    public StateMachine(IStateMachineOwner owner) 
    {  
        this.owner = owner; 
    }

    // 🔗 引用关系：
    //    [被谁调用]：丧尸的 AI 发现玩家时，会向 HR 下令切换状态：   
    //    [它调用谁]：它会去调用外部具体状态（如 RunState）里重写的 Enter() 和 Exit()。
    // ==========================================
    public void EnterState<T>() where T : StateBase, new()
    {
        // 1. 防呆拦截：如果现在就在执行这个状态，直接打回指令，防止动画鬼畜重复播放。
        if (currentState != null && currentState.GetType() == typeof(T)) return;
        
        // 2. 强制调用外部状态脚本里的 Exit()
        if (currentState != null)
            currentState.Exit();
        
        // 3.执行LoadState 找人
        currentState = LoadState<T>();
        
        // 4. 强制调用外部新状态脚本里的 Enter()（开始播动画）
        currentState.Enter();
    }

    // 🔗 引用关系：
    //    [被谁调用]：只有它自己身上的 EnterState() 能调用它（因为是 private 绝密）。
    //    [它调用谁]：调用了外部状态的 Init() 方法
    // ==========================================
    private StateBase LoadState<T>() where T: StateBase, new()
    {
        // 提取要找的那个人的身份证（比如提取 RunState 的 Type）
        Type stateType = typeof(T); 
        
        // TryGetValue + out 语法
        if(!stateDic.TryGetValue(stateType, out StateBase state))
        {
            // 只有当这是游戏开局第一次用到该状态时，才 (new)
            state = new T();
            
            // 调用外部状态的 Init，
            state.Init(owner); 
            
            // 下次再用就不用 new 了
            stateDic.Add(stateType, state); 
        }
        return state;  
    }

    // 🔗 引用关系：
    //    [被谁调用]：当丧尸血量归零死亡，即将被系统删除时，丧尸自己会调用：
    //    代码演示 -> myHR.Stop();
    // ==========================================
    public void Stop()
    {
        // 1. 先让当前正在砍人的员工停手下班
        if (currentState != null) 
            currentState.Exit();
        
        // 2. 遍历整个储物柜，把所有老员工身上的数据彻底销毁
        //（注：此处保持了你原先的 Destory 拼写，建议后续在基类和这里统一修正为 Destroy）
        foreach (var state in stateDic.Values)
            state.Destory(); 
            
        // 3. 彻底清空，把内存还给电脑
        stateDic.Clear();
    }
}


```
