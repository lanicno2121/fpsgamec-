```csharp

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Animations.Rigging; // 🔗 引用：用于控制手部 IK（骨骼反向动力学）
using UnityEngine.Rendering;

// ==========================================
// 🗣️ 大白话：【战术开镜打工人】
// 负责处理开镜时的动作锁定、相机视角强行切换、以及第一/第三人称视角的无缝融合。
// ==========================================
public class PlayerAimingState : PlayerStateBase
{
    private int aimingXHash;
    private int aimingYHash;
    private float aimingX = 0;
    private float aimingY = 0;
    private float transitionSpeed = 5;
    private float bodyRotationSpeed = 15f; // 🗣️ 大白话：准星移动时，身体跟着扭的速度

    private RigBuilder cachedRigBuilder;
    private bool currentIsFPS = false; // 🗣️ 大白话：当前是不是处在第一人称机瞄模式
    private float targetBodyWeight = 0f;
    private float cameraVerticalAngle = 0f;
    private float defaultFOV = 40f;

    [Header("高倍镜设置")]
    public float scopeFOV = 20f; // 🗣️ 大白话：开镜时的视野拉近程度

    // ==========================================
    // 🗣️ 大白话：【入职找工具】
    // ==========================================
    public override void Init(IStateMachineOwner owner)
    {
        base.Init(owner);
        aimingXHash = Animator.StringToHash("AimingX");
        aimingYHash = Animator.StringToHash("AimingY");

        // 🔗 引用：找到肉体身上的 RigBuilder（控制骨骼姿势的核心组件）
        cachedRigBuilder = playerModel.GetComponent<RigBuilder>();
        if (cachedRigBuilder == null)
            cachedRigBuilder = playerModel.GetComponentInChildren<RigBuilder>();

        if (playerModel.bodyAimConstraint == null)
            playerModel.bodyAimConstraint = playerModel.GetComponentInChildren<MultiAimConstraint>();

        if (playerModel.fpsCamera != null)
            defaultFOV = playerModel.fpsCamera.m_Lens.FieldOfView; // 记住没开镜前的正常视野
    }

    public override void Enter()
    {
        base.Enter();
        playerModel.PlayStateAnimation("Aiming"); // 🗣️ 大白话：播举枪待机动画
        cameraVerticalAngle = 0f;

        // 🗣️ 大白话：去问枪的说明书，这把枪是不是用第一人称开镜的（比如带八倍镜的狙击枪）
        bool isInputAiming = playerController.isAiming;
        bool weaponAllowsFPS = true;

        if (playerModel.weapon != null && playerModel.weapon.gunData != null)
            weaponAllowsFPS = playerModel.weapon.gunData.useFirstPersonAim;

        currentIsFPS = isInputAiming && weaponAllowsFPS;
        
        UpdateViewMode(currentIsFPS); // 🔗 引用：调用下面专门负责切镜头的方法
        
        if (IsBeControl()) playerController.EnterAim(); // 🗣️ 大白话：通知大脑：“我开镜了，你把镜头灵敏度调慢点！”
    }

    public override void Update()
    {
        base.Update();
        if (IsBeControl())
        {
            // 🗣️ 大白话：实时检测，因为玩家可能在开火中途松开右键（盲射），这时候要瞬间从第一人称退回第三人称。
            bool isInputAiming = playerController.isAiming;
            bool weaponAllowsFPS = true;
            if (playerModel.weapon != null && playerModel.weapon.gunData != null)
                weaponAllowsFPS = playerModel.weapon.gunData.useFirstPersonAim;

            bool targetIsFPS = isInputAiming && weaponAllowsFPS;

            if (targetIsFPS != currentIsFPS)
            {
                currentIsFPS = targetIsFPS;
                UpdateViewMode(currentIsFPS);
            }

            // 🗣️ 大白话：让身体跟着枪的瞄准方向慢慢扭过去（平滑权重）
            if (playerModel.bodyAimConstraint != null)
                playerModel.bodyAimConstraint.weight = Mathf.Lerp(playerModel.bodyAimConstraint.weight, targetBodyWeight, 10f * Time.deltaTime);

            // ==========================================
            // 🗣️ 大白话：【视角与身体扭动逻辑】
            // 第一人称和第三人称，身体扭动的逻辑是完全不一样的！
            // ==========================================
            if (currentIsFPS)
            {
                // FPS 模式：鼠标往哪甩，人物身体直接跟着转（硬锁）
                float mouseX = Input.GetAxis("Mouse X");
                if (Mathf.Abs(mouseX) > 0.01f)
                    playerModel.transform.Rotate(Vector3.up * mouseX * bodyRotationSpeed * 5f * Time.deltaTime);

                if (playerModel.fpsCamera != null)
                {
                    float mouseY = Input.GetAxis("Mouse Y");
                    cameraVerticalAngle -= mouseY * 2f;
                    cameraVerticalAngle = Mathf.Clamp(cameraVerticalAngle, -70f, 70f); // 限制不准把脖子仰断
                    playerModel.fpsCamera.transform.localRotation = Quaternion.Euler(cameraVerticalAngle, 0f, 0f);
                }
            }
            else
            {
                // TPS 模式：身体不能死锁鼠标。
                Vector3 targetDir = Camera.main.transform.forward;
                targetDir.y = 0;

                // 🗣️ 大白话：【防鬼畜死区】只有当你平视前方时，身体才会缓缓扭向准星。
                // 如果你现在正朝着正下方的脚底板看，身体不会跟着低头，防止模型拧成麻花！
                if (targetDir.sqrMagnitude > 0.3f) 
                {
                    Quaternion targetRot = Quaternion.LookRotation(targetDir);
                    playerModel.transform.rotation = Quaternion.Slerp(playerModel.transform.rotation, targetRot, bodyRotationSpeed * Time.deltaTime);
                }
            }

            UpdateAimingTarget(); // 🗣️ 大白话：重新计算射击落点

            // 🗣️ 大白话：【离职条件】没按右键（瞄准），也没按左键（开火），那就退回待机状态
            if (!playerController.isAiming && !playerController.isFire)
            {
                playerModel.SwitchState(PlayerState.Idle);
                return;
            }

            // 🗣️ 大白话：如果按了左键开枪，呼叫大脑去震动屏幕（后坐力）
            if (playerController.isFire && playerModel.weapon != null)
                playerController.ShakeCamera();

            // 🗣️ 大白话：把此时的 WASD 走路方向，喂给 Animator 里的 2D BlendTree。
            // 让你在开镜时，能播放极其真实的“端着枪往左走/往右走”的交叉步动画！
            aimingX = Mathf.Lerp(aimingX, playerController.moveInput.x, transitionSpeed * Time.deltaTime);
            aimingY = Mathf.Lerp(aimingY, playerController.moveInput.y, transitionSpeed * Time.deltaTime);
            playerModel.animator.SetFloat(aimingXHash, aimingX);
            playerModel.animator.SetFloat(aimingYHash, aimingY);
        }
    }

    // ==========================================
    // 🗣️ 大白话：【退镜离职收尾】
    // 走之前把别人相机的焦距、UI、影子，全部打扫干净恢复原样。
    // ==========================================
    public override void Exit()
    {
        base.Exit();
        if (playerModel.fpsCamera != null)
        {
            playerModel.fpsCamera.Priority = 0; // 把摄像头的优先级还回去
            playerModel.fpsCamera.m_Lens.FieldOfView = defaultFOV;
        }
        if (playerModel.fpsLeftArmModel != null)
            playerModel.fpsLeftArmModel.SetActive(false); // 隐藏 FPS 专用的假手
            
        // 把武器的影子打开（因为在 FPS 模式下为了防遮挡，通常会把武器影子关掉）
        if (playerModel.weapon != null)
            SetShadowMode(playerModel.weapon.gameObject, ShadowCastingMode.On); 
            
        if (playerModel.bodyAimConstraint != null) playerModel.bodyAimConstraint.weight = 0f;
        if (cachedRigBuilder != null) cachedRigBuilder.enabled = true;
        
        if (IsBeControl()) playerController.ExitAim(true);
    }
    
    // ... [后面还有一段 UpdateViewMode 主要是相机权重的参数修改，核心都在这里了] ...
}



```
