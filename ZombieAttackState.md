using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ZombieAttackState : EnemyStateBase
{
    public override void Enter()
    {
        base.Enter();
        enemyModel.PlayStateAnimation("Attack");

        // 攻击瞬间强制让敌人面向玩家
        if (enemyModel.attackTarget != null)
        {
            Vector3 lookPos = enemyModel.attackTarget.transform.position;
            lookPos.y = enemyModel.transform.position.y;
            enemyModel.transform.LookAt(lookPos);
        }
    }

    //新增：退出攻击状态时记录时间 
    public override void Exit()
    {
        base.Exit();
        // 记录打完这一拳的时间，作为冷却的开始
        enemyModel.lastAttackTime = Time.time;
    }
    // 结束 
    public override void Update()
    {
        base.Update();

        // 0 代表 Base Layer
        if (IsAnimationBreak(0))
        {
            enemyModel.SwitchState(EnemyState.Idle);
        }
    }
}