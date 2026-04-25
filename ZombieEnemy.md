using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ZombieEnemy : EnemyBase
{
    // 您可以把攻击伤害定义在这里，方便调整
    public int attackDamage = 15;

    public override void SwitchState(EnemyState state)
    {
        switch (state)
        {
            case EnemyState.Idle:
                stateMachine.EnterState<ZombieIdleState>();
                break;
            case EnemyState.Move:
                stateMachine.EnterState<ZombieMoveState>();
                break;
            case EnemyState.Attack:
                stateMachine.EnterState<ZombieAttackState>();
                break;
            case EnemyState.Dead:
                stateMachine.EnterState<ZombieDeadState>();
                break;
            // ==========================================
            // 【新增】巡逻状态分支
            // ==========================================
            case EnemyState.Patrol:
                stateMachine.EnterState<ZombiePatrolState>();
                break;
        }
    }

    // ==========================================
    //  【核心修复】接收动画事件的方法 
    //   Animation Event 必须填写: OnAttackHit
    // ==========================================
    public void OnAttackHit()
    {
        // 【调试 1】证明动画事件确实触发了
        // *重要*：如果攻击时控制台连这句话都没有，说明脚本和 Animator 没挂在同一个物体上！
        Debug.Log($"[ZombieEnemy] 接收到攻击指令... (目标: {attackTarget?.name})");

        // 1. 确保目标存在
        if (attackTarget == null)
        {
            Debug.LogError("[ZombieEnemy] 攻击失败：丢失目标 (attackTarget is null)");
            return;
        }

        // 2. 计算距离（手动计算，并增加 0.8 米的宽容度）
        // 原来的 IsAttackTargetInAttackRange 太严格了，玩家稍微退一点就打不到
        float currentDist = Vector3.Distance(transform.position, attackTarget.transform.position);

        // 我们允许比“开始攻击的距离”稍微远一点点也能打中，提升手感
        float effectiveRange = minAttackDistance + 0.8f;

        if (currentDist <= effectiveRange)
        {
            // 3. 尝试获取血量脚本 (双重保险)

            // 先找目标身上
            PlayerHealth playerHealth = attackTarget.GetComponent<PlayerHealth>();

            // 【关键修复】如果找不到，去父物体找 
            // (防止 CharacterController 挡住，或者脚本挂在 Root 上)
            if (playerHealth == null)
            {
                playerHealth = attackTarget.GetComponentInParent<PlayerHealth>();
            }

            // 4. 执行扣血
            if (playerHealth != null)
            {
                playerHealth.TakeDamage(attackDamage);
                Debug.Log($"<color=red>【有效命中】</color> 击中玩家！造成 {attackDamage} 点伤害");
            }
            else
            {
                // 如果看到这个报错，说明由于层级问题，没找到 PlayerHealth
                Debug.LogError($"[ZombieEnemy] 严重错误：在 {attackTarget.name} 及其父物体上都找不到 PlayerHealth 脚本！");
            }
        }
        else
        {
            // 如果你看到这句话，说明动画触发了，但是距离确实稍微远了一点点
            Debug.LogWarning($"[ZombieEnemy] 攻击挥空！当前距离: {currentDist:F2}, 有效距离: {effectiveRange:F2}");
        }
    }
}