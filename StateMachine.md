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