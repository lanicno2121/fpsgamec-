using UnityEngine;

public class PlayerHoverState : PlayerStateBase
{
    private bool hasTriggeredPreLand = false;
    private float preLandDistance = 2.5f;

    public override void Enter()
    {
        base.Enter();
        playerModel.animator.speed = 1.0f;
        hasTriggeredPreLand = false;
        playerModel.PlayStateAnimation("Hover", 0.2f);
    }

    public override void Update()
    {
        base.Update();

        // =========================================================
        // 【核心修复】下落时也要检测攀爬！
        // =========================================================
        if (playerController.CheckClimbInput())
        {
            playerModel.SwitchState(PlayerState.Climb);
            return;
        }

        // 1. 物理落地
        if (playerModel.cc.isGrounded)
        {
            playerModel.SwitchState(PlayerState.Land);
            return;
        }

        // 2. 视觉落地
        if (playerModel.verticalSpeed < -0.1f && !hasTriggeredPreLand)
        {
            if (Physics.Raycast(playerModel.transform.position, Vector3.down, out RaycastHit hit, preLandDistance))
            {
                if (!hit.collider.isTrigger)
                {
                    playerModel.PlayStateAnimation("JumpLand", 0.1f);
                    hasTriggeredPreLand = true;
                }
            }
        }
    }

    public override void Exit()
    {
        base.Exit();
        playerModel.animator.speed = 1f;
    }
}