using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ZombiePatrolState : EnemyStateBase
{
    public override void Enter()
    {
        base.Enter();
        // 播放移动动画 (如果你有专门的"Walk"缓慢行走动画，可以把这里换成"Walk")
        enemyModel.PlayStateAnimation("Move");

        if (enemyModel.navMeshAgent != null && enemyModel.navMeshAgent.isOnNavMesh)
        {
            enemyModel.navMeshAgent.isStopped = false;
            // 把导航目标设置为当前的巡逻点
            enemyModel.navMeshAgent.SetDestination(enemyModel.waypoints[enemyModel.currentWaypointIndex].position);
        }
    }

    public override void Update()
    {
        base.Update();

        // 1. 【打断机制】巡逻途中只要发现玩家，立刻切成追击！
        if (enemyModel.HasAttackTarget() && enemyModel.IsTargetInDetectionRange())
        {
            enemyModel.SwitchState(EnemyState.Move);
            return;
        }

        // 2. 【寻路机制】判断是否走到了巡逻点
        if (enemyModel.navMeshAgent != null && enemyModel.navMeshAgent.isOnNavMesh)
        {
            // 如果还没开始算路径，或者距离小于等于刹车距离 (增加 0.2f 宽容度防止卡死)
            if (!enemyModel.navMeshAgent.pathPending &&
                enemyModel.navMeshAgent.remainingDistance <= enemyModel.navMeshAgent.stoppingDistance + 0.2f)
            {
                // 走到当前点了，把目标索引切到下一个点 (使用 % 取余来实现首尾循环)
                enemyModel.currentWaypointIndex = (enemyModel.currentWaypointIndex + 1) % enemyModel.waypoints.Length;

                // 走到目标后，切回 Idle 状态发呆休息一会
                enemyModel.SwitchState(EnemyState.Idle);
            }
        }
    }
}