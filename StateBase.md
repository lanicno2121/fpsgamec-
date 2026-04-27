```csharp
//状态基类
public abstract class StateBase
{
    //初始化
    public abstract void Init(IStateMachineOwner owner);

    //进入状态
    public abstract void Enter();

    //退出状态
    public abstract void Exit();

    //销毁状态
    public abstract void Destory();

    public abstract void Update();

}
```







