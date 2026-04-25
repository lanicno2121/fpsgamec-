using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ZombieMoveState : EnemyStateBase
{
    public override void Enter()
    {
        base.Enter();
        enemyModel.PlayStateAnimation("Move");
    }

    public override void Update()
    {
        base.Update();

        // ======================================================
        // 【核心修改】脱战逻辑：如果玩家跑出了警戒距离的 1.5倍 远，放弃追击
        // ======================================================
        if (!enemyModel.IsTargetInDetectionRange(1.5f))
        {
            enemyModel.SwitchState(EnemyState.Idle);
            return;
        }

        // 正常追击逻辑
        if (!enemyModel.IsAttackTargetInAttackRange())
        {
            enemyModel.chaseTarget();
        }
        else
        {
            enemyModel.SwitchState(EnemyState.Idle);
        }
    }
}