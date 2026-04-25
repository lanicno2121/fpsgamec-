using UnityEngine;

public class PlayerVaultState : PlayerStateBase
{
    private float timer;
    private Vector3 startPos;
    private Vector3 targetPos;
    private Quaternion startRot;
    private Quaternion targetRot;
    private bool isReadyToVault = false;

    // 翻越动画的时间 (可以根据你的动画长度微调)
    private float vaultDuration = 0.6f;

    public override void Enter()
    {
        base.Enter();
        timer = 0f;
        playerModel.verticalSpeed = 0;
        isReadyToVault = false;

        // 关碰撞，防止翻越过程被引擎强行阻挡
        if (playerModel.cc != null) playerModel.cc.detectCollisions = false;

        if (playerModel.animator.HasState(0, Animator.StringToHash("Vault")))
        {
            playerModel.animator.Play("Vault", 0, 0f);
        }

        // 屏蔽上半身和IK干扰
        for (int i = 1; i <= 4; i++) playerModel.animator.SetLayerWeight(i, 0f);
        if (playerModel.leftHandConstraint != null) playerModel.leftHandConstraint.weight = 0f;
        if (playerModel.rightHandConstraint != null) playerModel.rightHandConstraint.weight = 0f;

        CalculateVaultPoints();
    }

    private void CalculateVaultPoints()
    {
        startPos = playerModel.transform.position;
        startRot = playerModel.transform.rotation;

        // 从角色头顶前方往下打射线，寻找墙顶
        Vector3 rayOrigin = startPos + Vector3.up * 2.5f + playerModel.transform.forward * 0.6f;

        if (Physics.Raycast(rayOrigin, Vector3.down, out RaycastHit topHit, 3.0f, playerController.climbableLayer))
        {
            // 目标点：墙面最高点，再往墙内深入 0.4 米（确保双脚站稳）
            targetPos = topHit.point + (playerModel.transform.forward * 0.4f);
            targetRot = playerModel.transform.rotation;
            isReadyToVault = true;
        }
        else
        {
            // 兜底：如果没有检测到墙顶，强制给一个向斜上方的位移
            targetPos = startPos + playerModel.transform.forward * 1.0f + Vector3.up * 1.5f;
            targetRot = startRot;
            isReadyToVault = true;
        }
    }

    public override void Update()
    {
        timer += Time.deltaTime;

        if (isReadyToVault)
        {
            float progress = Mathf.Clamp01(timer / vaultDuration);

            // =========================================================
            // 【核心数学魔法】：分离 Y 轴和 XZ 轴的运动曲线
            // =========================================================

            // 1. 高度 (Y轴) 优先：使用 Sin 曲线 (Ease-Out)。起步极快，快速越过墙头高度。
            float yProgress = Mathf.Sin(progress * Mathf.PI * 0.5f);
            float currentY = Mathf.Lerp(startPos.y, targetPos.y, yProgress);

            // 2. 水平 (XZ轴) 滞后：使用平方曲线 (Ease-In)。起步慢，等身体拔高后，再快速前推。
            float xzProgress = progress * progress;
            Vector3 startXZ = new Vector3(startPos.x, 0, startPos.z);
            Vector3 targetXZ = new Vector3(targetPos.x, 0, targetPos.z);
            Vector3 currentXZ = Vector3.Lerp(startXZ, targetXZ, xzProgress);

            // 3. 组合最终坐标
            Vector3 curvedPosition = new Vector3(currentXZ.x, currentY, currentXZ.z);

            // 应用位置和旋转
            playerModel.transform.position = curvedPosition;
            playerModel.transform.rotation = Quaternion.Slerp(startRot, targetRot, progress);

            // =========================================================
            // 【无痛微创修复 1】：偷运权重法 (解决动画生硬卡顿)
            // 在动画快结束的最后 30% 时间里，提前悄悄把层级权重拉起来。
            // 这样状态机切回 Idle 时，层级已经是 1 了，完全不会发生瞬间抽搐！
            // =========================================================
            if (progress > 0.7f)
            {
                // 计算后 30% 的局部进度 (从 0 到 1)
                float layerProgress = (progress - 0.7f) / 0.3f;

                // 判断当前是空手还是持枪，分别恢复不同的动画层
                int targetLayer = (playerModel.weapon == null) ?
                                  playerModel.animator.GetLayerIndex("Unarmed Layer") :
                                  playerModel.animator.GetLayerIndex("UpperBody");

                if (targetLayer != -1)
                {
                    // 悄悄地平滑增加权重
                    playerModel.animator.SetLayerWeight(targetLayer, layerProgress);
                }
            }
        }

        if (timer >= vaultDuration)
        {
            // 动作结束，强制对齐最终位置
            playerModel.transform.position = targetPos;
            playerModel.SwitchState(PlayerState.Idle);
        }
    }

    public override void Exit()
    {
        base.Exit();

        // 恢复碰撞体
        if (playerModel.cc != null)
        {
            playerModel.cc.detectCollisions = true;

            // =========================================================
            // 【无痛微创修复 2】：减轻物理冲击 (解决镜头顿挫)
            // 原本是 0.1f，会导致镜头猛地砸地。现在改成极其微小的 0.02f
            // =========================================================
            playerModel.cc.Move(Vector3.down * 0.02f);
        }
    }
}