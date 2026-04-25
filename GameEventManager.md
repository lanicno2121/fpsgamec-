using System; // 必须引入 System 才能使用 Action
using UnityEngine;

// 注意：这是一个 public static class，不需要继承 MonoBehaviour
// 它相当于一个全游戏通用的对讲机频道列表
public static class GameEventManager
{
    // ==========================================
    // 频道 1：无参数的简单事件 (例如：玩家死亡)
    // ==========================================
    public static Action OnPlayerDeath;

    // ==========================================
    // 频道 2：带参数的事件 (例如：敌人死亡，并告诉大家是哪个敌人死了)
    // <EnemyBase> 代表广播时，会顺便把死掉的敌人本体传过去
    // ==========================================
    public static Action<EnemyBase> OnEnemyDied;

    // ==========================================
    // 频道 3：带参数的事件 (例如：更新击杀目标进度)
    // ==========================================
    public static Action<int> OnKillCountUpdated;
}