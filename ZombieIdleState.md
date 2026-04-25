using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ZombieIdleState : EnemyStateBase
{
    public override void Enter()
    {
        base.Enter();
        enemyModel.PlayStateAnimation("Idle");

        // 安全锁：确保在网格上才刹车
        if (enemyModel.navMeshAgent != null && enemyModel.navMeshAgent.isOnNavMesh)
        {
            enemyModel.navMeshAgent.velocity = Vector3.zero;
            enemyModel.navMeshAgent.isStopped = true;
        }

        // ==========================================
        // 【新增】每次进入发呆状态时，把休息计时器清零
        // ==========================================
        enemyModel.idleTimer = 0f;
    }

    public override void Exit()
    {
        base.Exit();
        if (enemyModel.navMeshAgent != null && enemyModel.navMeshAgent.isOnNavMesh)
        {
            enemyModel.navMeshAgent.isStopped = false;
        }
    }

    public override void Update()
    {
        base.Update();

        // 1. 【最高优先级】如果有目标，且玩家进入了警戒范围 -> 强制打断休息，开始追击！
        if (enemyModel.HasAttackTarget() && enemyModel.IsTargetInDetectionRange())
        {
            if (!enemyModel.IsAttackTargetInAttackRange())
            {
                enemyModel.SwitchState(EnemyState.Move); // 追击
            }
            else if (Time.time > enemyModel.lastAttackTime + enemyModel.attackInterval)
            {
                enemyModel.SwitchState(EnemyState.Attack); // 贴脸直接打
            }
            return; // 发现玩家了，直接 return，不走下面的巡逻逻辑
        }

        // ==========================================
        // 【新增】2. 【巡逻逻辑】如果没有发现玩家，且设置了巡逻点 -> 休息一会后开始巡逻
        // ==========================================
        if (enemyModel.waypoints != null && enemyModel.waypoints.Length > 0)
        {
            enemyModel.idleTimer += Time.deltaTime;
            if (enemyModel.idleTimer >= enemyModel.idleWaitTime)
            {
                // 休息够了，切换到巡逻状态
                enemyModel.SwitchState(EnemyState.Patrol);
            }
        }
    }
}