using UnityEngine;
using Cinemachine;

public class PlayerCoverState : PlayerStateBase
{
    private CoverSpot currentSpot;
    private int crouchHash;
    private WeaponManager weaponManager;

    public override void Init(IStateMachineOwner owner)
    {
        base.Init(owner);
        crouchHash = Animator.StringToHash("IsCrouch");

        weaponManager = playerModel.GetComponent<WeaponManager>();
        if (weaponManager == null)
            weaponManager = playerModel.GetComponentInParent<WeaponManager>();
    }

    public override void Enter()
    {
        base.Enter();

        currentSpot = playerController.currentCoverSpot;
        if (currentSpot == null)
        {
            playerModel.SwitchState(PlayerState.Idle);
            return;
        }

        // 1. 切枪
        if (weaponManager != null) weaponManager.EquipWeapon(0);

        // 2. 位移
        playerModel.transform.position = currentSpot.standPoint.position;
        playerModel.transform.rotation = currentSpot.standPoint.rotation;

        // 3. 动画
        playerModel.animator.SetBool(crouchHash, true);

        // 4. 物理
        playerModel.verticalSpeed = 0;
        playerController.SetBodyHeight(true);
    }

    public override void Update()
    {
        // 唯一的输入检测：按 F 退出
        if (Input.GetKeyDown(KeyCode.F))
        {
            playerModel.SwitchState(PlayerState.Idle);
            return;
        }

        // =========================================================
        // 【核心修复】暴力镇压所有层级和 IK
        // =========================================================

        // 1. 镇压动画层 (UpperBody 索引是 3)
        playerModel.animator.SetLayerWeight(3, 0f);
        playerModel.animator.SetLayerWeight(1, 0f);
        playerModel.animator.SetLayerWeight(2, 0f);
        playerModel.animator.SetLayerWeight(4, 0f);

        // 2. 镇压左手 IK (之前你做对了)
        if (playerModel.leftHandConstraint != null)
            playerModel.leftHandConstraint.weight = 0f;

        // 3. 【新增】镇压右手 IK (这次加上这个，右手就老实了)
        if (playerModel.rightHandConstraint != null)
            playerModel.rightHandConstraint.weight = 0f;

        // 4. 【新增】镇压瞄准 IK (防止右手被拉去瞄准)
        if (playerModel.rightHandAimConstraint != null)
            playerModel.rightHandAimConstraint.weight = 0f;

        if (playerModel.bodyAimConstraint != null)
            playerModel.bodyAimConstraint.weight = 0f;
    }

    public override void Exit()
    {
        base.Exit();

        playerController.StartCoverCooldown();

        // 1. 还原动画
        playerModel.animator.SetBool(crouchHash, false);

        // 2. 还原物理
        playerController.SetBodyHeight(false);

        // 3. 还原 Layer (UpperBody 回复到 1)
        playerModel.animator.SetLayerWeight(3, 1f);

        // 4. 还原 IK (虽然 PlayerController 会接管，但手动还原更保险)
        if (playerModel.leftHandConstraint != null) playerModel.leftHandConstraint.weight = 1f;
        if (playerModel.rightHandConstraint != null) playerModel.rightHandConstraint.weight = 1f;
    }
}