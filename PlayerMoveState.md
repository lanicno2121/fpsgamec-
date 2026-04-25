```csharp

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

// ==========================================
// 🗣️ 大白话：【移动/跑酷打工人】
// 🔗 引用关系：继承自老父亲 PlayerStateBase，享受自动计算重力的福利。
// ==========================================
public class PlayerMoveState : PlayerStateBase
{
    #region 动画器相关
    // 🗣️ 大白话：这几个变量是专门用来配合 Unity 动画机里的“混合树 (BlendTree)”的。
    // 比如 0 是走路，1 是冲刺，中间的 0.5 就会自动混合出“快走”的动画。
    private int moveBlendHash; 
    private float moveBlend;
    private float runThreshold = 0;
    private float sprintThreshold = 1;
    private float transitionSpeed = 5; // 动画过渡的丝滑程度
    #endregion

    private bool isStopping = false; // 🗣️ 大白话：是否正在“踩刹车”
    private float stopAnimSpeed = 1.5f;

    // ==========================================
    // 🗣️ 大白话：【入职培训】
    // 整个游戏开局只执行一次，提前把密码本准备好，提高运行效率。
    // ==========================================
    public override void Init(IStateMachineOwner owner)
    {
        base.Init(owner);
        // 🔗 引用：把 Unity Animator 里名叫 "MoveBlend" 的参数转成数字 ID (Hash)，以后调用的速度极快。
        moveBlendHash = Animator.StringToHash("MoveBlend");
    }

    // ==========================================
    // 🗣️ 大白话：【开始上班】
    // 每次从待机或跳跃切入移动状态时，第一秒要做的事。
    // ==========================================
    public override void Enter()
    {
        base.Enter();
        isStopping = false; // 刚起步，肯定没踩刹车
        playerModel.animator.speed = 1f;
        playerModel.PlayStateAnimation("Move"); // 🗣️ 大白话：让肉体开始播跑路动画

        // 🗣️ 大白话：问问大脑，我现在是按了 Shift 键在冲刺，还是普通跑？
        if (playerController.isSprint) moveBlend = sprintThreshold;
        else moveBlend = runThreshold;

        // 强行把动画树的进度条拉到对应位置
        playerModel.animator.SetFloat(moveBlendHash, moveBlend);
    }

    // ==========================================
    // 🗣️ 大白话：【上班干活（每秒执行 60 次）】
    // ==========================================
    public override void Update()
    {
        base.Update(); // 🔗 引用：呼叫老父亲，老父亲会在这里自动帮你算重力！
        if (IsBeControl())
        {
            #region 急停打断逻辑
            // 🗣️ 大白话：如果你现在正在“脚底打滑踩刹车”的硬直动画里
            if (isStopping)
            {
                // 如果你突然又按了 WASD 键（想继续走）
                if (playerController.moveInput.magnitude > 0.1f)
                {
                    isStopping = false; // 🗣️ 大白话：立刻打断刹车！
                    playerModel.animator.speed = 1f;
                    playerModel.PlayStateAnimation("Move", 0.15f); // 强行切回跑步动画
                }
                else
                {
                    // 🗣️ 大白话：如果没按方向键，那就老老实实等“刹车”动画播完
                    AnimatorStateInfo info = playerModel.animator.GetCurrentAnimatorStateInfo(0);
                    // 刹车动画播到 80% 了
                    if (info.IsName("RunStop") && info.normalizedTime >= 0.8f) 
                    {
                        playerModel.animator.speed = 1f;
                        playerModel.SwitchState(PlayerState.Idle); // 🗣️ 大白话：车停稳了，向 HR 申请下班，切回“待机”状态！
                    }

                    playerModel.animator.SetFloat(moveBlendHash, 0); // 刹车时速度归零
                    return; // 🗣️ 大白话：正在刹车，后面的正常跑路逻辑就别执行了
                }
            }
            #endregion

            // 🗣️ 大白话：【随时准备开溜】跑到一半，如果按了滑铲或跳跃，立刻向 HR 辞职并切状态！
            #region 滑铲状态监听
            if (playerController.isSprint && playerController.isSlide)
            {
                playerModel.SwitchState(PlayerState.Slide);
                return;
            }
            #endregion

            #region 跳跃监听
            if (playerController.isJumping)
            {
                PerformJump(); // 🔗 引用：调用老父亲里的跳跃方法（老父亲会帮你检测有没有撞墙）
                return;
            }
            #endregion

            #region 触发急停监听
            // 🗣️ 大白话：如果你正在跑，突然松开了 WASD（输入几乎为0），且还没开始刹车
            if (playerController.moveInput.magnitude < 0.1f && !isStopping)
            {
                isStopping = true; // 🗣️ 大白话：踩刹车！
                playerModel.PlayStateAnimation("RunStop", 0.1f); // 播鞋底摩擦地面的动画
                playerModel.animator.speed = stopAnimSpeed;
                return;
            }
            #endregion

            #region 移动混合与旋转逻辑
            // 🗣️ 大白话：用 Lerp (插值) 让起步和减速像真车一样有线性加速感，而不是瞬间满速。
            if (playerController.isSprint)
                moveBlend = Mathf.Lerp(moveBlend, sprintThreshold, transitionSpeed * Time.deltaTime);
            else
                moveBlend = Mathf.Lerp(moveBlend, runThreshold, transitionSpeed * Time.deltaTime);

            playerModel.animator.SetFloat(moveBlendHash, moveBlend);

            // 🗣️ 大白话：角色转身。根据大脑算出的相对方向（localMovement），让角色的身体转过去。
            if (playerController.moveInput.magnitude > 0.1f)
            {
                float rad = Mathf.Atan2(playerController.localMovement.x, playerController.localMovement.z);
                playerModel.transform.Rotate(0, rad * playerController.rotationSpeed * Time.deltaTime, 0);
            }
            #endregion
        }
    }
}
```
