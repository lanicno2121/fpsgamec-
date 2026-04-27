```csharp
public static class GameEventManager
{
    public static Action OnPlayerDeath;
    // 频道 2：带参数的事件 (例如：敌人死亡，并告诉大家是哪个敌人死了)
    // <EnemyBase> 代表广播时，会顺便把死掉的敌人本体传过去
    public static Action<EnemyBase> OnEnemyDied;


    // 频道 3：带参数的事件 (例如：更新击杀目标进度)
    public static Action<int> OnKillCountUpdated;
}
```
