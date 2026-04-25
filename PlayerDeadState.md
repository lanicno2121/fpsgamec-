using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Animations.Rigging;
using Cinemachine; // 必须引用，我们需要关掉 CinemachineBrain

public class PlayerDeadState : PlayerStateBase
{
    // 缓存主相机，方便 Update 里移动
    private Transform mainCamTrans;

    // 参数设置
    private float riseSpeed = 0.5f;   // 上升速度 (极其缓慢)
    private float rotateSpeed = 5f; // 旋转速度 (极其缓慢)
    private float uiDelayTimer = 0f;  // 计时器
    private bool uiShown = false;     // 防止重复显示

    public override void Enter()
    {
        base.Enter();

        // 1. 基础清理 (动画、物理、输入禁用) - 保持之前的逻辑
        playerModel.PlayStateAnimation("Die");
        playerModel.verticalSpeed = 0;
        playerController.ExitAim();

        if (playerModel.fpsLeftArmModel != null)
            playerModel.fpsLeftArmModel.SetActive(false);

        // =======================================================
        // 【核心修复】：增加判空！只有手里有枪，才去处理枪的影子！
        // 这样空手被打死就不会报错崩溃了。
        // =======================================================
        if (playerModel.weapon != null)
        {
            SetShadowMode(playerModel.weapon.gameObject, ShadowCastingMode.On);
        }

        if (playerController.windEffect != null)
        {
            playerController.windEffect.Stop();
            playerController.windEffect.Clear();
        }
        DisableComponent<DynamicCrosshair>();
        playerController.enabled = false; // 切断输入

        // 归零 IK
        if (playerModel.rightHandConstraint != null) playerModel.rightHandConstraint.weight = 0f;
        if (playerModel.leftHandConstraint != null) playerModel.leftHandConstraint.weight = 0f;
        if (playerModel.bodyAimConstraint != null) playerModel.bodyAimConstraint.weight = 0f;
        if (playerModel.rightHandAimConstraint != null) playerModel.rightHandAimConstraint.weight = 0f;

        DisableComponent<GunAmmoSystem>();
        DisableComponent<PlayerWeapon>();
        DisableComponent<WeaponRecoil>();
        DisableComponent<ShootingPoseAdjuster>();

        // ==========================================
        // 相机接管
        // ==========================================
        if (Camera.main != null)
        {
            mainCamTrans = Camera.main.transform;

            // 必须关掉 CinemachineBrain，否则我们的代码动不了相机，它会被 Cinemachine 强行拉回去
            var brain = Camera.main.GetComponent<CinemachineBrain>();
            if (brain != null) brain.enabled = false;
        }

        // ==========================================
        // 解锁鼠标光标 (非常重要！否则点不了按钮)
        // ==========================================
        Cursor.lockState = CursorLockMode.None;
        Cursor.visible = true;

        // 重置计时器
        uiDelayTimer = 0f;
        uiShown = false;
    }

    public override void Update()
    {
        // 这里的 Update 现在有用了！我们要用来控制相机和 UI 延时

        // 1. 相机特效：极其缓慢的旋转上升
        if (mainCamTrans != null)
        {
            // 上升
            mainCamTrans.Translate(Vector3.up * riseSpeed * Time.deltaTime, Space.World);

            // 围绕角色旋转 (LookAt 保证一直盯着尸体)
            mainCamTrans.LookAt(playerModel.transform.position);
            mainCamTrans.RotateAround(playerModel.transform.position, Vector3.up, rotateSpeed * Time.deltaTime);
        }

        // 2. 延时显示 UI (比如死后 1.5 秒再出弹窗，更有电影感)
        if (!uiShown)
        {
            uiDelayTimer += Time.deltaTime;
            if (uiDelayTimer > 1.5f)
            {
                if (playerController.deathPanel != null)
                {
                    playerController.deathPanel.SetActive(true);
                }
                uiShown = true;
            }
        }
    }

    // --- 辅助函数 ---
    private void DisableComponent<T>() where T : MonoBehaviour
    {
        T component = playerModel.GetComponentInChildren<T>();
        if (component == null && playerController != null)
            component = playerController.GetComponentInChildren<T>();
        if (component != null) component.enabled = false;
    }

    private void SetShadowMode(GameObject target, ShadowCastingMode mode)
    {
        if (target == null) return;
        Renderer[] renderers = target.GetComponentsInChildren<Renderer>();
        foreach (var r in renderers) r.shadowCastingMode = mode;
    }
}