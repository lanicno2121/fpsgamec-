using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerSlidingState : PlayerStateBase
{
    private float originalHeight;
    private Vector3 originalCenter;

    // 滑铲动画播放倍速
    private float slideAnimSpeed = 1.5f;

    // 【新增】记录当前是在哪个层播放滑铲
    private int activeLayerIndex = 0;

    public override void Enter()
    {
        base.Enter();

        // 1. 自动判断该去哪个层检查动画
        // 如果手里没枪(空手)，就去 Unarmed Layer (通常是层级4，为了保险我们动态获取)
        if (playerModel.weapon == null)
        {
            activeLayerIndex = playerModel.animator.GetLayerIndex("Unarmed Layer");
            if (activeLayerIndex == -1) activeLayerIndex = 0; // 如果找不到，这就回滚到0
        }
        else
        {
            activeLayerIndex = 0; // 持枪时查 Base Layer
        }

        // 2. 播放动画
        // (PlayerModel.PlayStateAnimation 已经帮我们把指令发给正确的层了，这里不用操心)
        playerModel.PlayStateAnimation("Slide");

        // 3. 加速播放
        playerModel.animator.speed = slideAnimSpeed;

        // 4. 缩小碰撞体
        originalHeight = playerModel.cc.height;
        originalCenter = playerModel.cc.center;

        playerModel.cc.height = originalHeight * 0.5f;
        playerModel.cc.center = new Vector3(originalCenter.x, originalHeight * 0.25f, originalCenter.z);
    }

    public override void Update()
    {
        base.Update();

        // 移动逻辑由 PlayerModel 接管，这里只负责“什么时候退出”

        // ==============================================================
        // 【核心修复】检查 activeLayerIndex 而不是写死的 0
        // ==============================================================
        AnimatorStateInfo info = playerModel.animator.GetCurrentAnimatorStateInfo(activeLayerIndex);

        // 判断是否正在播放 Slide 动画
        // 注意：请确保 Unarmed Layer 里的滑铲状态名字也叫 "Slide"
        if (info.IsName("Slide"))
        {
            // 进度大于 80% (0.8f) 时退出，稍微晚一点点，防止卡顿
            if (info.normalizedTime >= 0.8f)
            {
                CheckExitState();
            }
        }
        else
        {
            // 【安全保险】
            // 如果代码刚进状态，发现当前层根本没在播 Slide (可能过渡还没开始，或者名字错了)
            // 我们可以做一个超时检测，或者什么都不做等待它开始
            // 这里暂且不做处理，通常是因为过渡需要几帧时间
        }
    }

    public override void Exit()
    {
        base.Exit();

        // 恢复碰撞体
        playerModel.cc.height = originalHeight;
        playerModel.cc.center = originalCenter;

        // 恢复动画速度
        playerModel.animator.speed = 1f;
    }

    private void CheckExitState()
    {
        // 逻辑：按着 Shift+W -> 继续冲刺；只按 W -> 变跑步；没按键 -> 变待机
        if (playerController.isSprint && playerController.moveInput.magnitude > 0.1f)
        {
            playerModel.SwitchState(PlayerState.Move);
        }
        else if (playerController.moveInput.magnitude > 0.1f)
        {
            playerModel.SwitchState(PlayerState.Move);
        }
        else
        {
            playerModel.SwitchState(PlayerState.Idle);
        }
    }
}