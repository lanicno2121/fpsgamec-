using UnityEngine;

public class PlayerJumpState : PlayerStateBase
{
    public override void Enter()
    {
        base.Enter();

        float targetAnimSpeed = 1.2f;

        // 1. 判断是否奔跑跳
        if (playerController.moveInput.y > 0.1f)
        {
            playerModel.PlayStateAnimation("RunJump", 0.1f);
            targetAnimSpeed = 1.2f;
        }
        else
        {
            playerModel.PlayStateAnimation("JumpStart", 0.1f);
            // 空手原地跳加速
            if (playerModel.weapon == null) targetAnimSpeed = 3.5f;
            else targetAnimSpeed = 1.2f;
        }

        playerModel.animator.speed = targetAnimSpeed;
        playerModel.verticalSpeed = Mathf.Sqrt(-2 * playerModel.gravity * playerModel.jumpHeight);
    }

    public override void Update()
    {
        base.Update();

        // =========================================================
        // 【核心修复】必须在这里检测攀爬！
        // 否则跳起来撞到墙，系统只会觉得你在撞墙，不会切状态
        // =========================================================
        if (playerController.CheckClimbInput())
        {
            playerModel.SwitchState(PlayerState.Climb);
            return;
        }

        // 切换到悬空状态
        if (playerModel.verticalSpeed < 2.0f)
        {
            playerModel.SwitchState(PlayerState.Hover);
        }
    }

    public override void Exit()
    {
        base.Exit();
        playerModel.animator.speed = 1f;
    }
}