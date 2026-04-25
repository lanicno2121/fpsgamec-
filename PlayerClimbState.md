using UnityEngine;

public class PlayerClimbState : PlayerStateBase
{
    private WeaponManager weaponManager;
    private int climbHash;
    private float currentClimbTimer;
    private int lastWeaponIndex = -1;

    [Header("=== 🛠️ 攀爬绝对参数 ===")]
    [Tooltip("极限高度检测：1.2f (必须爬到胸口过墙才能翻越)")]
    public float topCheckHeight = 1.2f;
    public float backwardOffset = 0.5f;
    [Tooltip("绝对防穿模距离：0.45m")]
    public float idealDistance = 0.45f;

    public override void Init(IStateMachineOwner owner)
    {
        base.Init(owner);
        climbHash = Animator.StringToHash("IsClimbing");
        weaponManager = playerModel.GetComponent<WeaponManager>();
        if (weaponManager == null) weaponManager = playerModel.GetComponentInParent<WeaponManager>();
    }

    public override void Enter()
    {
        base.Enter();
        currentClimbTimer = 0f;
        playerModel.verticalSpeed = 0;

        // 【物理隔离】关闭碰撞，防止被物理引擎强行挤回墙里穿模！
        if (playerModel.cc != null) playerModel.cc.detectCollisions = false;

        if (weaponManager != null)
        {
            lastWeaponIndex = weaponManager.CurrentIndex;
            weaponManager.EnterUnarmedMode(true);
        }

        if (playerModel.animator.HasState(0, Animator.StringToHash("Climb")))
        {
            playerModel.animator.Play("Climb", 0, 0f);
        }
        playerModel.animator.SetBool(climbHash, true);

        for (int i = 1; i <= 4; i++) playerModel.animator.SetLayerWeight(i, 0f);
        if (playerModel.leftHandConstraint != null) playerModel.leftHandConstraint.weight = 0f;
        if (playerModel.rightHandConstraint != null) playerModel.rightHandConstraint.weight = 0f;

        SnapToWall();
    }

    private void SnapToWall()
    {
        Vector3 origin = playerModel.transform.position + Vector3.up * 1.0f - (playerModel.transform.forward * backwardOffset);
        if (Physics.Raycast(origin, playerModel.transform.forward, out RaycastHit hit, 2.0f, playerController.climbableLayer))
        {
            // 抵消 PlayerModel 里每帧 0.05f 的强推力
            Vector3 targetPos = hit.point + (hit.normal * (idealDistance + 0.05f));
            playerModel.transform.position = new Vector3(targetPos.x, playerModel.transform.position.y, targetPos.z);
            playerModel.transform.rotation = Quaternion.LookRotation(-hit.normal);
        }
    }

    public override void Update()
    {
        currentClimbTimer += Time.deltaTime;

        bool hasInput = playerController.moveInput.y > 0.1f;
        bool hasStamina = currentClimbTimer <= playerController.maxClimbDuration;

        if (!hasInput || !hasStamina)
        {
            FallDown();
            return;
        }

        // =========================================================
        // 【绝杀修复】双重射线法 (解决半路掉下来的问题)
        // 1. 高位射线 (胸口) -> 探测是否到顶
        // 2. 低位射线 (腰部) -> 探测是否真离开了墙壁
        // =========================================================
        Vector3 originHigh = playerModel.transform.position + Vector3.up * topCheckHeight - (playerModel.transform.forward * backwardOffset);
        Vector3 originLow = playerModel.transform.position + Vector3.up * 0.4f - (playerModel.transform.forward * backwardOffset);

        bool hitHigh = Physics.Raycast(originHigh, playerModel.transform.forward, out RaycastHit hitHighInfo, playerController.climbCheckDistance + 1.0f, playerController.climbableLayer);
        bool hitLow = Physics.Raycast(originLow, playerModel.transform.forward, out RaycastHit hitLowInfo, playerController.climbCheckDistance + 1.0f, playerController.climbableLayer);

        if (!hitLow && !hitHigh)
        {
            // 真的没墙了（比如爬出了墙的侧面），正常掉落
            FallDown();
            return;
        }

        if (!hitHigh && hitLow)
        {
            // 胸口没墙了，但腰部还在墙上 -> 说明刚好爬到了墙顶！立刻翻越！
            PerformVault();
            return;
        }

        // 正常攀爬逻辑 (锁定坐标)
        RaycastHit alignHit = hitHigh ? hitHighInfo : hitLowInfo;
        playerModel.transform.rotation = Quaternion.LookRotation(-alignHit.normal);

        // 绝对坐标锁定，无视一切力
        Vector3 idealPos = alignHit.point + (alignHit.normal * (idealDistance + 0.05f));
        playerModel.transform.position = new Vector3(idealPos.x, playerModel.transform.position.y, idealPos.z);

        // 向上移动
        playerModel.verticalSpeed = playerController.climbSpeed;
    }

    private void FallDown()
    {
        playerController.StartClimbCooldown(0.5f);
        playerModel.verticalSpeed = 0;
        playerModel.SwitchState(PlayerState.Hover);
    }

    private void PerformVault()
    {
        playerController.StartClimbCooldown(1.0f);
        playerModel.SwitchState(PlayerState.Vault);
    }

    public override void Exit()
    {
        base.Exit();
        playerModel.animator.SetBool(climbHash, false);
        playerModel.verticalSpeed = 0;

        // 恢复碰撞体
        if (playerModel.cc != null) playerModel.cc.detectCollisions = true;

        if (weaponManager != null)
        {
            if (lastWeaponIndex == -1) weaponManager.EnterUnarmedMode(true);
            else weaponManager.EquipWeapon(lastWeaponIndex);
        }
    }
}