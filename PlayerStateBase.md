```csharp
using System.Collections; // [cite: 1]
using System.Collections.Generic; // [cite: 1]
using UnityEngine; // [cite: 1]

// ==========================================
// 🗣️ 大白话：【玩家专属老父亲基类】
// 🔗 引用：继承自最底层的 StateBase。以后所有的 PlayerIdleState, PlayerRunState 都要认它当爹。
// ==========================================
public class PlayerStateBase : StateBase
{
    // 🗣️ 大白话：每个员工心里都清楚大老板（大脑）和二老板（肉体）是谁。
    protected PlayerController playerController;
    protected PlayerModel playerModel;

    // ==========================================
    // 🗣️ 大白话：【入职培训】
    // 🔗 引用：这个方法是在 HR（StateMachine）执行 state.Init(owner) 时被调用的。
    // ==========================================
    [cite_start]public override void Init(IStateMachineOwner owner) // [cite: 2]
    {
        // 🗣️ 大白话：因为大脑是全服唯一的单例，直接去拿它的实例。
        playerController = PlayerController.INSTANCE; 
        
        // 🗣️ 大白话：HR 递过来的名片（owner）其实就是肉体。这里用 (PlayerModel) 强行把它扒回原形，记在心里。
        playerModel = (PlayerModel)owner; // [cite: 3]
    }


// ==========================================
    // 🗣️ 大白话：【打卡上班】
    // 🔗 引用：使用了外部的 MonoManager（统一更新管理器）。
    // ⭐️ 核心揭秘：为什么不直接写 Update()？
    // 因为这只是个普通的 C# 类（没继承 MonoBehaviour），Unity 不会自动每帧喊它干活。
    // 所以员工在上班时，主动把自己的 Update 名字写进全公司的“打卡机（MonoManager）”里，让打卡机每秒喊它 60 次！
    // ==========================================
    public override void Enter()
    {
        MonoManager.INSTANCE.AddUpdateAction(Update);
    [cite_start]} // [cite: 4]
    
    public override void Destory() { }

    // ==========================================
    // 🗣️ 大白话：【打卡下班 / 离职】
    // 既然下班了，赶紧去打卡机那里把自己的名字划掉，不然系统还会一直喊你，白白浪费电脑性能！
    // ==========================================
    public override void Exit()
    {
        MonoManager.INSTANCE.RemoveUpdateAction(Update);
    [cite_start]} // [cite: 5]


// 🗣️ 大白话：只要员工在上着班，这个方法每秒就会被调用 60 次。
    public override void Update()
    {
        #region 重力计算
        // 🗣️ 大白话：去问肉体的碰撞体（cc），你现在脚踩在地上了吗？
        if (!playerModel.cc.isGrounded)
        {
            // 🗣️ 大白话：人在空中！受地心引力影响，往下掉的速度越来越快。
            // 【修改】还原为最纯粹的重力计算
            playerModel.verticalSpeed += playerModel.gravity * Time.deltaTime; // [cite: 5]
            
            // 🗣️ 大白话：【自动悬空检测】如果脚底下打射线没碰到地（Hover），且当前不是正儿八经的按了空格键的跳跃（Jump）
            [cite_start]// [cite: 6]
            if (playerModel.IsHover() && playerModel.GetCurrentState() != PlayerState.Jump)
            {
                // 🗣️ 大白话：强行剥夺控制权，让肉体进入“悬空掉落(Hover)”状态！防止走在悬崖边走出去还在播走路动画。
                if (playerModel.GetCurrentState() != PlayerState.Hover)
                    playerModel.SwitchState(PlayerState.Hover); // [cite: 7]
            }
        }
        else
        {
            // 🗣️ 大白话：人踩在地上！给一个极其微小的往下踩的死力 (-5f)。
            // 为什么是 -5 而不是 0？因为下斜坡的时候，如果重力是 0，人会像弹片一样飞出去，-5 能让人死死吸在斜坡上！
            // 地面吸附力
            playerModel.verticalSpeed = -5f; // [cite: 8]
        }
        #endregion

// ============================================================
        // 🗣️ 大白话：【瞄准状态监听】不管你在走还是跑，老父亲时刻盯着你有没有按开火/开镜键！
        // ============================================================
        #region 瞄准状态监听

        // 🗣️ 大白话：去问大脑，现在按没按右键？按没按左键？
        bool isInputAiming = playerController.isAiming; // 右键 // [cite: 9]
        bool isInputFire = playerController.isFire; // 左键 // [cite: 10]

        // 1. 基础原则：只要按了右键，无论拿什么枪，必须进瞄准状态
        bool shouldEnterAim = isInputAiming; // [cite: 11]
        
        // 2. 特殊情况：如果没按右键，只按了左键 （也就是玩家想“盲射”）
        [cite_start]if (isInputFire && !isInputAiming) // [cite: 11]
        {
            // 检查当前手里有没有枪，枪里有没有配置数据
            if (playerModel.weapon != null && playerModel.weapon.gunData != null)
            {
                // 🗣️ 大白话：去查这把枪的说明书。有些重型狙击枪不允许盲射，只要你按左键开火，系统强行把你拉进开镜状态！
                [cite_start]if (playerModel.weapon.gunData.aimOnFire) // [cite: 12, 13]
                {
                    shouldEnterAim = true; // [cite: 13]
                }
            }
        }

        // 3. 执行切换
        if (shouldEnterAim)
        {
            // 🗣️ 大白话：【防报错护盾】如果你手里啥都没拿（比如刚复活没捡枪），就算你把鼠标按烂了，老父亲也绝对不允许你进入开镜状态！
            // 【关键修复】只有手里有枪 (weapon != null)，才允许进入瞄准状态！
            [cite_start]if (playerModel.weapon != null) // [cite: 14]
            {
                playerModel.SwitchState(PlayerState.Aiming); // [cite: 15]
            }
        }
        #endregion
    }
    
    // ==========================================
    // 🗣️ 大白话：【防夺舍工具】
    // 问：我现在控制的这个肉体，还是不是大脑关心的那个“主角”？（防联机错乱或者过场动画时控制权被剥夺）
    // ==========================================
    public bool IsBeControl()
    {
        return playerModel == playerController.currentPlayerModel; // [cite: 16]
    }

    // ==========================================
    // 🗣️ 大白话：【智能跳跃拦截器】
    // 为什么把跳跃写在老父亲这里？因为跑、走、待机时按下空格，都能触发跳跃。
    // ==========================================
    public void PerformJump()
    {
        // 🗣️ 大白话：按下空格的一瞬间，先别急着往天上飞！
        // 去问问大脑的激光眼：老板，我现在前面是不是有墙可以爬？
        // 【核心修复】先询问控制器：我现在是不是想爬墙？
        if (playerController.CheckClimbInput())
        {
            Debug.Log("<color=green>满足攀爬条件，进入 Climb 状态！</color>");
            // 🗣️ 大白话：截胡！立刻强制命令肉体进入“爬墙”状态，而不是跳跃。
            playerModel.SwitchState(PlayerState.Climb); // [cite: 17]
            return; // 拦截成功，不再执行下面的普通跳跃
        }

        // 如果不爬墙，再正常正常向天上蹦，切换到跳跃状态
        playerModel.SwitchState(PlayerState.Jump); // [cite: 18]
    }

    // ==========================================
    // 🗣️ 大白话：【动画完工质检员】
    // 动作游戏最头疼的就是“怎么知道挥剑动画播完了”。
    // 这个工具就是专门去问动画师（animator）：“兄弟，你这个动画的进度条是不是走完 90% 了？并且没有在混合切换？”
    // 如果是，就可以切回待机状态了！
    // ==========================================
    protected bool IsAnimationBreak(int layer, float progressThreshold = 0.9f)
    {
        AnimatorStateInfo info = playerModel.animator.GetCurrentAnimatorStateInfo(layer); // [cite: 19]
        // 增加容错：如果当前不在过渡中，且播放进度大于阈值
        return info.normalizedTime >= progressThreshold && !playerModel.animator.IsInTransition(layer); // [cite: 19]
    }
}
    
```
