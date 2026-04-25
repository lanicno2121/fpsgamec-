using UnityEngine;

public class PlayerLandState : PlayerStateBase
{
    private float startTime;
    // 【调节】落地硬直时间
    private float landDuration = 0.25f;

    public override void Enter()
    {
        base.Enter();
        startTime = Time.time;

        playerModel.animator.speed = 1.0f;
        playerModel.verticalSpeed = 0;

        AnimatorStateInfo info = playerModel.animator.GetCurrentAnimatorStateInfo(0);
        if (!info.IsName("JumpLand"))
        {
            playerModel.PlayStateAnimation("JumpLand", 0.1f);
        }
    }

    public override void Update()
    {
        playerModel.verticalSpeed = -10f;

        // 1. 如果有移动输入 -> 切移动
        if (playerController.moveInput.magnitude > 0.1f)
        {
            playerModel.SwitchState(PlayerState.Move);
            return;
        }

        // 2. 如果按了跳跃键 -> 尝试连跳
        if (playerController.isJumping)
        {
            // 【已修改】删除了空手禁止连跳的限制
            PerformJump();
            return;
        }

        if (playerController.isAiming || playerController.isFire) { playerModel.SwitchState(PlayerState.Aiming); return; }

        if (Time.time - startTime > landDuration)
        {
            playerModel.SwitchState(PlayerState.Idle);
        }
    }

    public override void Exit()
    {
        base.Exit();
        playerModel.animator.speed = 1f;
    }
}