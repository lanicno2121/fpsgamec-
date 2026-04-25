using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerIdleState : PlayerStateBase
{
    public override void Enter()
    {
        base.Enter();
        playerModel.PlayStateAnimation("Idle");
    }

    public override void Update()
    {
        base.Update();

        if (IsBeControl())
        {
            #region 移动状态监听
            if (playerController.moveInput.magnitude != 0)
                playerModel.SwitchState(PlayerState.Move);
            #endregion

            #region 跳跃监听
            if (playerController.isJumping)
            {
                // 【已修改】删除了之前的“空手禁止起跳”判断
                // 现在无论有没有枪，只要按空格，都能跳！
                PerformJump();
            }
            #endregion
        }
    }
}