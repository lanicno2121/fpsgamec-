```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Animations.Rigging;
using Cinemachine;

// ==========================================
// 🗣️ 大白话：【状态清单】游戏里所有的状态名字，全记在这个枚举本子上。
// ==========================================
[cite_start]public enum PlayerState { Idle, Move, Hover, Aiming, Slide, Jump, Land, Dead, Cover, Climb, Fall, Vault } [cite: 2]

public class PlayerModel : MonoBehaviour, IStateMachineOwner
{
    [cite_start][Tooltip("角色武器")] public PlayerWeapon weapon; [cite: 2]
    
    // 🗣️ 大白话：肉体的两大核心肌肉块！
    // 🔗 引用：Animator 负责播放动画，CharacterController (cc) 负责物理碰撞和移动。
    [cite_start][HideInInspector] public Animator animator; [cite: 3]
    [cite_start]public CharacterController cc; [cite: 3]
    
    // 🗣️ 大白话：肉体雇佣的私人 HR 大管家。
    // 🔗 引用：严重依赖我们之前写的 StateMachine 脚本。
    [cite_start]private StateMachine stateMachine; [cite: 3]
    
    // 🗣️ 大白话：当前正在执行的状态标签。
    [cite_start]private PlayerState currentState; [cite: 3]

    // ==========================================
    // 🗣️ 大白话：【骨骼约束】IK 组件，让手能稳稳抓在枪上，身体能跟着准星扭动。
    // ==========================================
    [Header("IK 约束组件")]
    [cite_start]public TwoBoneIKConstraint rightHandConstraint; [cite: 4]
    [cite_start]public TwoBoneIKConstraint leftHandConstraint; [cite: 4]
    [cite_start]public MultiAimConstraint rightHandAimConstraint; [cite: 4]
    [cite_start]public MultiAimConstraint bodyAimConstraint; [cite: 4]

    [cite_start][Header("FPS 专用模型")] public GameObject fpsLeftArmModel; [cite: 5]

    [Header("相机与渲染设置")]
    [cite_start]public CinemachineVirtualCameraBase tpsCamera; [cite: 5]
    [cite_start]public CinemachineVirtualCamera fpsCamera; [cite: 5]

    // ==========================================
    // 🗣️ 大白话：【物理参数】
    // ==========================================
    [Header("移动参数")]
    [cite_start]public float gravity = -15; [cite: 6]
    [cite_start]public float jumpHeight = 1.5f; [cite: 6]
    [cite_start][HideInInspector] public float verticalSpeed; [cite: 6] // 🗣️ 大白话：往下掉的速度
    [cite_start]public float fallHeight = 0.2f; [cite: 7]
    [cite_start]public float slideMoveSpeedMultiplier = 2.5f; [cite: 7]
    [cite_start]public float stopDistMultiplier = 0.2f; [cite: 7]

    // ==========================================
    // 🗣️ 大白话：【物理惯性缓存池】（极度高级的防卡顿设计！）
    // 为什么要有这个？因为起跳在空中的时候，动画是不提供位移的。
    // 这里存下起跳前最后 3 帧的速度，算个平均值，当做在空中的惯性推力。
    // ==========================================
    [cite_start]private static readonly int CACHE_SIZE = 3; [cite: 8]
    [cite_start]Vector3[] speedCache = new Vector3[CACHE_SIZE]; [cite: 8]
    [cite_start]private int speedCache_index = 0; [cite: 8]
    [cite_start]private Vector3 averageDeltaMovement; [cite: 8]

    [cite_start]private int hitHash; [cite: 9]
    [cite_start]private int unarmedLayerIndex = -1; [cite: 9]



private void Awake()
    {
        // 🗣️ 大白话：【历史性的一刻！】肉体花钱把 HR 造出来了，并且把自己的名片 (this) 递给了他。
        // 从这一秒起，HR 就可以在这具肉体上安排工作了！
        [cite_start]stateMachine = new StateMachine(this); [cite: 9]

        // 🗣️ 大白话：找齐自己身上的骨骼和物理外壳。
        [cite_start]animator = GetComponent<Animator>(); [cite: 10]
        [cite_start]cc = GetComponent<CharacterController>(); [cite: 10]
        [cite_start]hitHash = Animator.StringToHash("Hit"); [cite: 10] // 把字符串转成数字密码，播动画时速度更快
    }

    void Start()
    {
        [cite_start]if (bodyAimConstraint != null) bodyAimConstraint.weight = 0f; [cite: 10]
        [cite_start]if (fpsLeftArmModel != null) fpsLeftArmModel.SetActive(false); [cite: 11]
        [cite_start]if (animator != null) unarmedLayerIndex = animator.GetLayerIndex("Unarmed Layer"); [cite: 11]
        
        // 🗣️ 大白话：刚出生，默认命令 HR 切换到“待机状态”。
        [cite_start]SwitchState(PlayerState.Idle); [cite: 11]
    }

// ==========================================
    // 🗣️ 大白话：【指令翻译机】
    // 🔗 引用关系：[被大脑调用] -> 大脑喊 "SwitchState(Cover)"。
    // [调用 HR] -> 这里把它翻译成 "stateMachine.EnterState<PlayerCoverState>()"，正式下发给 HR 换班。
    // ==========================================
    public void SwitchState(PlayerState state)
    {
        switch (state)
        {
            [cite_start]case PlayerState.Idle: stateMachine.EnterState<PlayerIdleState>(); break; [cite: 12, 13]
            [cite_start]case PlayerState.Move: stateMachine.EnterState<PlayerMoveState>(); break; [cite: 13]
            [cite_start]case PlayerState.Hover: stateMachine.EnterState<PlayerHoverState>(); break; [cite: 13]
            [cite_start]case PlayerState.Aiming: stateMachine.EnterState<PlayerAimingState>(); break; [cite: 13]
            [cite_start]case PlayerState.Slide: stateMachine.EnterState<PlayerSlidingState>(); break; [cite: 13]
            [cite_start]case PlayerState.Jump: stateMachine.EnterState<PlayerJumpState>(); break; [cite: 13, 14]
            [cite_start]case PlayerState.Land: stateMachine.EnterState<PlayerLandState>(); break; [cite: 14]
            [cite_start]case PlayerState.Dead: stateMachine.EnterState<PlayerDeadState>(); break; [cite: 14]
            [cite_start]case PlayerState.Cover: stateMachine.EnterState<PlayerCoverState>(); break; [cite: 14]
            [cite_start]case PlayerState.Climb: stateMachine.EnterState<PlayerClimbState>(); break; [cite: 14]
            [cite_start]case PlayerState.Vault: stateMachine.EnterState<PlayerVaultState>(); break; [cite: 14, 15]
        }
        [cite_start]currentState = state; [cite: 15] // 记下当前状态，方便大脑随时查岗
    }
    
    // ==========================================
    // 🗣️ 大白话：这个特殊的方法每一帧都会在动画播放完之后触发。
    // 我们在这里接管所有的移动权力！
    // ==========================================
    private void OnAnimatorMove()
    {
        [cite_start]Vector3 finalMovement = Vector3.zero; [cite: 16]

        // 🛑 特殊情况 1：正在爬墙
        [cite_start]if (currentState == PlayerState.Climb) [cite: 17]
        {
            // 🗣️ 大白话：【防穿模神级修复】爬墙时绝对不给往前的力，只给上下的力。
            // 往前的距离全部交给攀爬的具体状态脚本去算，防止角色卡进墙里出不来。
            [cite_start]finalMovement = Vector3.zero; [cite: 18]
            [cite_start]finalMovement.y = verticalSpeed * Time.deltaTime; [cite: 19]
            [cite_start]cc.Move(finalMovement); [cite: 19]
        }
        // 🛑 特殊情况 2：正在翻越障碍
        [cite_start]else if (currentState == PlayerState.Vault) [cite: 20]
        {
            return; // 🗣️ 大白话：翻越动画自己带有极其精确的位移（Root Motion），代码绝对不插手干预，直接 return。
        }
        // 🟢 常规情况：跑、跳、滑铲
        else
        {
            // 去问大脑：老板，现在是不是按着冲刺键呢？
            [cite_start]bool isSprinting = (currentState == PlayerState.Move && PlayerController.INSTANCE != null && PlayerController.INSTANCE.isSprint); [cite: 21]

            // 【动力源 A：代码强推】如果是冲刺
            [cite_start]if (isSprinting) [cite: 22]
            {
                [cite_start]float speed = PlayerController.INSTANCE.sprintSpeed; [cite: 22]
                [cite_start]finalMovement = transform.forward * speed * Time.deltaTime; [cite: 23]
                UpdateAverageCacheSpeed(transform.forward * speed); // 存下当前速度当惯性 [cite: 23]
            }
            // 【动力源 B：代码强推】如果是滑铲
            [cite_start]else if (currentState == PlayerState.Slide) [cite: 24]
            {
                [cite_start]float speed = PlayerController.INSTANCE.sprintSpeed * slideMoveSpeedMultiplier; [cite: 24]
                [cite_start]finalMovement = transform.forward * speed * Time.deltaTime; [cite: 25]
                [cite_start]UpdateAverageCacheSpeed(transform.forward * speed); [cite: 25]
            }
            // 【动力源 C：动画牵引 或 惯性飞行】
            else
            {
                // 如果脚踩在地上（不是跳跃也不是悬空）
                [cite_start]if (currentState != PlayerState.Hover && currentState != PlayerState.Jump) [cite: 26]
                {
                    // 🗣️ 大白话：完全交给动画本身自带的位移去移动！（非常真实自然）
                    [cite_start]finalMovement = animator.deltaPosition; [cite: 26]
                    [cite_start]UpdateAverageCacheSpeed(animator.velocity); [cite: 27]
                }
                else 
                {
                    // 🗣️ 大白话：人在空中！动画没法提供位移了。
                    // 调出刚才起跳前存下的【惯性速度】，用惯性推着人在天上飞！
                    [cite_start]finalMovement = averageDeltaMovement * Time.deltaTime; [cite: 27]
                }
            }

            // 统一加上地心引力的往下掉的速度
            [cite_start]finalMovement.y = verticalSpeed * Time.deltaTime; [cite: 28]
            
            // 最终执行：用 CharacterController 把肉体推过去！
            [cite_start]cc.Move(finalMovement); [cite: 29]
        }
    }
    
    // 🗣️ 大白话：让别人随时能查到肉体当前在干嘛。
    [cite_start]public PlayerState GetCurrentState() => currentState; [cite: 29]
    
    public void TriggerHit() { if (animator != null) animator.SetTrigger(hitHash); } [cite: 30]
    
    // 🗣️ 大白话：往脚底打一根射线，判断肉体是不是悬空了。
    [cite_start]public bool IsHover() => !Physics.Raycast(transform.position, Vector3.down, fallHeight); [cite: 30]

    // ==========================================
    // 🗣️ 大白话：【一键播放动画神器】
    // 员工（比如待机状态脚本）只需要说一句 PlayStateAnimation("Idle")，
    // 这里就会帮你用 CrossFadeInFixedTime 极其丝滑地过渡动画，还能自动判断拿没拿枪。
    // ==========================================
    [cite_start]public void PlayStateAnimation(string name, float t = 0.25f, int l = 0) [cite: 31]
    {
        [cite_start]if (animator != null) animator.CrossFadeInFixedTime(name, t, 0); [cite: 31]
        // 如果手里没枪，还需要在没有武器的那个动画层（Unarmed Layer）也同步播放动画
        [cite_start]if (weapon == null && unarmedLayerIndex != -1) animator.CrossFadeInFixedTime(name, t, unarmedLayerIndex); [cite: 32]
    }

    // ==========================================
    // 🗣️ 大白话：【物理惯性录音机】
    // 就像行车记录仪一样，永远只录下最近 3 帧的速度，挤掉最老的数据。
    // ==========================================
    [cite_start]private void UpdateAverageCacheSpeed(Vector3 s) [cite: 33]
    {
        [cite_start]speedCache[speedCache_index++] = s; [cite: 33]
        [cite_start]speedCache_index %= CACHE_SIZE; [cite: 34] // 让索引在 0, 1, 2 之间循环
        [cite_start]Vector3 sum = Vector3.zero; [cite: 34]
        [cite_start]foreach (Vector3 c in speedCache) sum += c; [cite: 34]
        averageDeltaMovement = sum / CACHE_SIZE; // 算出平均值备用 [cite: 34]
    }

    // 🗣️ 大白话：如果你想强行给人一个往前的惯性推力，就调用这个。
    [cite_start]public void SetForwardMomentum(float speed) [cite: 35]
    {
        [cite_start]Vector3 forwardVel = transform.forward * speed; [cite: 35]
        [cite_start]for (int i = 0; i < CACHE_SIZE; i++) [cite: 36]
        {
            [cite_start]speedCache[i] = forwardVel; [cite: 36]
        }
        [cite_start]averageDeltaMovement = forwardVel; [cite: 37]
    }
}
```