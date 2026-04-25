```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Cinemachine; // 🔗 引用：第三人称高级相机插件 [cite: 1]
using UnityEngine.Animations.Rigging; // 🔗 引用：动画约束插件（用于 IK 瞄准） [cite: 1]

// ==========================================
[cite_start]// 🗣️ 大白话：确保全游戏只有一个“主角大脑”，并且保证它比别的脚本先运行（执行顺序为 100） [cite: 2]。
[cite_start]// 🔗 引用关系：继承自咱们之前写的 SingleMonoBase 单例基类 [cite: 2]。
// ==========================================
[DefaultExecutionOrder(100)]
public class PlayerController : SingleMonoBase<PlayerController>
{
    [cite_start]// 🗣️ 大白话：老板的“肉体”。所有具体的动作、状态切换，最后都要靠这个肉体去执行 [cite: 2]。
    [cite_start]// 🔗 引用：严重依赖 PlayerModel 脚本 [cite: 2]。
    public PlayerModel currentPlayerModel;
    private Transform cameraTransform;

    [Header("瞄准灵敏度")]
    public float aimXSensitivity = 15f; // 🗣️ 大白话：开镜时鼠标左右移动的速度 [cite: 3]
    public float aimYSensitivity = 0.002f; // 🗣️ 大白话：开镜时鼠标上下移动的速度 [cite: 3]

    [Header("相机设置")]
    [cite_start]// 🗣️ 大白话：日常跑图用的相机和开镜瞄准用的相机 [cite: 4]。
    public CinemachineFreeLook freeLookCamera;
    public CinemachineFreeLook aimingCamera;
    public Transform fpsCameraTransform; // 🗣️ 大白话：第一人称视角的基准点 [cite: 4]

    [Header("UI 引用")]
    public GameObject deathPanel; // 🗣️ 大白话：死后弹出的黑屏或重新开始界面 [cite: 5]

    [Header("视觉特效")]
    public ParticleSystem windEffect; // 🗣️ 大白话：跑得太快时屏幕边缘的风线特效 [cite: 5]
    public float runWindRate = 5f;
    public float sprintWindRate = 30f; // 🗣️ 大白话：冲刺时风线更密集 [cite: 5, 6]

    [Header("掩体系统")]
    public CinemachineFreeLook coverCamera; // 🗣️ 大白话：进掩体时的专属固定视角相机 [cite: 6]
    [HideInInspector] public CoverSpot currentCoverSpot; // 🗣️ 大白话：当前躲在哪个具体的掩体点 [cite: 6]
    private bool canEnterCover = false;
    private float nextCoverTime = 0f; // 🗣️ 大白话：防止刚出掩体又立刻卡进去的冷却时间 [cite: 7]
    public GameObject coverHintUI; // 🗣️ 大白话：提示你“按F进入掩体”的 UI [cite: 7]

    [Header("攀爬系统 UI")]
    public GameObject climbHintUI; // 🗣️ 大白话：提示你能爬墙的 UI [cite: 8]

    // =========================================================
    // 【音效系统】各种动作的“声音盲盒”，每次随机抽一个播，防机械音
    // =========================================================
    [Header("玩家音效 (支持多个声音随机播放)")]
    [cite_start]public AudioClip[] runSounds; [cite: 8]
    [cite_start]public AudioClip[] sprintSounds; [cite: 9]
    public AudioClip[] slideSounds;
    [cite_start]public AudioClip[] climbSounds; [cite: 9]

    [Header("跳跃/落地音效 (物理驱动)")]
    public AudioClip[] jumpSounds;
    [cite_start]public AudioClip[] landSounds; [cite: 9]

    [Header("通用交互音效 (兜底默认声音)")]
    [cite_start]public AudioClip interactSound; [cite: 10]

    [Header("瞄准音效 (Foley)")]
    public AudioClip aimInSound; // 🗣️ 大白话：举枪瞄准时的衣服摩擦声 [cite: 10, 11]
    public AudioClip aimOutSound; // 🗣️ 大白话：放下枪时的声音 [cite: 11]
    private bool wasAimingLastFrame = false; // 🗣️ 大白话：记忆上一帧有没有瞄准，用来探测瞬间动作 [cite: 11, 12]

    private bool wasGrounded = true; // 🗣️ 大白话：上一帧脚有没有踩在地上 [cite: 12]
    private float airTime = 0f; // 🗣️ 大白话：在空中停留了多久（用来判断落地要不要播重音） [cite: 12]

    private AudioSource audioSource; // 🗣️ 大白话：挂在主角身上真正负责发声的喇叭 [cite: 12]

    [Header("攀爬系统 (Apex Style)")]
    public float climbSpeed = 5f;
    public float maxClimbDuration = 100f; // 🗣️ 大白话：最多能爬多高/多久 [cite: 13]
    public LayerMask climbableLayer; // 🗣️ 大白话：规定哪些图层（比如墙壁）是允许爬的 [cite: 13]
    [cite_start]public float climbCheckDistance = 1.0f; [cite: 14]

    [Header("=== 🧗‍♂️ 攀爬触发限制 ===")]
    public float minClimbDistance = 0.25f; // 🗣️ 大白话：离墙太近不准爬 [cite: 14]
    public float maxClimbDistance = 0.8f; // 🗣️ 大白话：离墙太远不准爬 [cite: 15]
    public float maxClimbAngle = 45f; // 🗣️ 大白话：墙面太斜不准爬 [cite: 15]
    public float backwardOffset = 0.5f; // 🗣️ 大白话：探测射线往后退一点，防穿模 [cite: 15]
    private float climbCooldownTimer = 0f; // 🗣️ 大白话：爬完一次歇一会 [cite: 15]

    [Header("物理高度设置")]
    [cite_start]// 🗣️ 大白话：站立和蹲下时，肉体碰撞体（胶囊体）的高度和中心点 [cite: 16, 17]
    public float standHeight = 1.8f; 
    [cite_start]public float crouchHeight = 1.0f; [cite: 16]
    public Vector3 standCenter = new Vector3(0, 0.9f, 0);
    [cite_start]public Vector3 crouchCenter = new Vector3(0, 0.5f, 0); [cite: 17]
    public float crouchSmoothTime = 0.1f; // 🗣️ 大白话：下蹲时的丝滑过渡时间 [cite: 18]

    private float currentHeightVelocity;
    [cite_start]private Vector3 currentCenterVelocity; [cite: 18]

    #region 玩家输入相关
    // 🗣️ 大白话：这些是大脑的“神经接收器”，专门装键盘鼠标的实时信号。
    [cite_start]// 🔗 引用：使用了 Myinputsystem（Unity 的新版输入系统代码） [cite: 18]。
    [cite_start]private Myinputsystem input; [cite: 18]
    [HideInInspector] public Vector2 moveInput; // 🗣️ 大白话：WASD 摇杆的数值 [cite: 19]
    [HideInInspector] public bool isSprint; // 🗣️ 大白话：是否在冲刺 [cite: 19]
    [HideInInspector] private bool isSprintToggle = false; 
    [cite_start][HideInInspector] public bool isAiming; [cite: 19]
    [HideInInspector] public bool isJumping;
    [HideInInspector] public bool isFire;
    [cite_start][HideInInspector] public bool isSlide; [cite: 20]
    #endregion

    #region 瞄准相关
    public Transform AimTarget; // 🗣️ 大白话：屏幕准星在 3D 世界里实际对准的那个点 [cite: 21]
    public float maxRayDistance = 1000f; // 🗣️ 大白话：瞄准射线的最大距离 [cite: 21]
    public LayerMask aimLayerMask = ~0; // 🗣️ 大白话：射线能打中什么东西（~0代表所有东西） [cite: 22]
    private float targetAimWeight = 0f; // 🗣️ 大白话：IK 骨骼动画的权重（0是正常拿枪，1是标准瞄准姿势） [cite: 22]
    [cite_start]public float aimSmoothSpeed = 10f; [cite: 22]
    #endregion

    private CinemachineImpulseSource impulseSource; // 🗣️ 大白话：用来让屏幕震动（比如开枪后坐力）的组件 [cite: 23]
    public float runSpeed = 4f;
    [cite_start]public float sprintSpeed = 7f; [cite: 23]
    public float rotationSpeed = 300; // 🗣️ 大白话：角色转身的速度 [cite: 24]

    [HideInInspector] public Vector3 localMovement; // 🗣️ 大白话：相对于自身的移动方向 [cite: 24]
    [HideInInspector] public Vector3 worldMovement; // 🗣️ 大白话：相对于世界地图的移动方向 [cite: 24]

    // ==========================================
    [cite_start]// 🗣️ 大白话：开镜时的“锁枪”记忆点。记录正常拿枪的位置，方便退镜时能把枪放回原位 [cite: 24, 25]。
    // ==========================================
    private Transform cachedAdsPoint;
    [cite_start]private Transform cachedWeaponTrans; [cite: 24]
    private Vector3 defaultWeaponPos;
    private Quaternion defaultWeaponRot;
    [cite_start]private bool hasCachedDefaults = false; [cite: 25]
    
    protected override void Awake()
    {
        base.Awake(); // 🗣️ 大白话：调用祖宗类的逻辑，确保单例宝座坐稳 [cite: 26]
        input = new Myinputsystem(); // 🗣️ 大白话：买一块键盘接上 [cite: 26]
    }

    void Start()
    {
        cameraTransform = Camera.main.transform; // 🗣️ 大白话：把主摄相机的眼睛借过来 [cite: 27]
        Cursor.lockState = CursorLockMode.Locked; // 🗣️ 大白话：把玩家的鼠标指针锁死在屏幕正中间，并且隐藏起来 [cite: 28]
        ExitAim(); // 🗣️ 大白话：游戏刚开始，确保处于没开镜的正常状态 [cite: 28]

        impulseSource = GetComponent<CinemachineImpulseSource>(); // 🗣️ 大白话：找到震动组件 [cite: 28]

        [cite_start]// 🗣️ 大白话：像寻宝一样，去相机下面找那个叫 "ADS_Point" 的精确瞄准基准点 [cite: 28, 29]
        Transform searchRoot = fpsCameraTransform != null ?
            [cite_start]fpsCameraTransform : (aimingCamera != null ? aimingCamera.transform : null); [cite: 28, 29]
        if (searchRoot != null)
        {
            cachedAdsPoint = searchRoot.Find("ADS_Point");
            if (cachedAdsPoint == null)
            {
                [cite_start]cachedAdsPoint = searchRoot.GetComponentInChildren<Transform>().Find("ADS_Point"); [cite: 30]
            }
        }

        [cite_start]// 🗣️ 大白话：去问肉体（currentPlayerModel）的碰撞体（cc）要它初始的身高数据 [cite: 31]
        if (currentPlayerModel != null && currentPlayerModel.cc != null)
        {
            standHeight = currentPlayerModel.cc.height;
            [cite_start]standCenter = currentPlayerModel.cc.center; [cite: 31, 32]
        }

        [cite_start]// 🗣️ 大白话：在自己身上找发声喇叭，没有就现场造一个 [cite: 32, 33]
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null)
        {
            [cite_start]audioSource = gameObject.AddComponent<AudioSource>(); [cite: 33]
        }
        audioSource.playOnAwake = false; // 🗣️ 大白话：刚出生别瞎叫 [cite: 34]
        audioSource.spatialBlend = 0f; // 🗣️ 大白话：0代表这是2D环境音，在主角耳朵边直接响，不受距离影响 [cite: 34]
    }
    
    
    
    void Update()
    {
        #region 更新玩家输入
        [cite_start]// 🗣️ 大白话：从键盘实时获取按键信号 [cite: 35, 36]
        Vector2 lookInput = input.Player.Look.ReadValue<Vector2>();
        moveInput = input.Player.Move.ReadValue<Vector2>().normalized; // 🗣️ 大白话：归一化防止斜着走速度翻倍 [cite: 36]
        isSprint = input.Player.IsSprint.IsPressed();
        [cite_start]isAiming = input.Player.IsAming.IsPressed(); [cite: 36]

        // ==========================================
        [cite_start]// 🗣️ 大白话：探测“瞬间”的瞄准动作，并播放衣服摩擦声 [cite: 36, 37]
        // ==========================================
        if (isAiming && !wasAimingLastFrame)
        {
            // 上一帧没瞄准，这一帧瞄准了 = 刚刚举枪
            [cite_start]if (aimInSound != null && audioSource != null) audioSource.PlayOneShot(aimInSound); [cite: 36]
        }
        else if (!isAiming && wasAimingLastFrame)
        {
            // 上一帧瞄准了，这一帧没瞄准 = 刚刚放下枪
            [cite_start]if (aimOutSound != null && audioSource != null) audioSource.PlayOneShot(aimOutSound); [cite: 37]
        }
        [cite_start]wasAimingLastFrame = isAiming; [cite: 38]

        isJumping = input.Player.IsJuming.IsPressed();
        isFire = input.Player.Fire.IsPressed();
        isSlide = input.Player.IsSlide.WasPressedThisFrame(); // 🗣️ 大白话：滑铲必须是瞬间按下，不能按住不放 [cite: 38]
        
        [cite_start]// 🗣️ 大白话：冲刺键的切换逻辑（按一下开，再按一下关，或者松开方向键就关） [cite: 39]
        if (input.Player.IsSprint.WasPressedThisFrame()) isSprintToggle = !isSprintToggle;
        if (moveInput.magnitude < 0.1f) isSprintToggle = false;
        [cite_start]isSprint = isSprintToggle; [cite: 39]

        // ==========================================
        [cite_start]// 🗣️ 大白话：【大脑妥协】如果肉体正在掩体里，大脑强行把所有输入清零，剥夺玩家控制权 [cite: 40, 41]
        [cite_start]// 🔗 引用：调用肉体的 GetCurrentState() 问 HR 当前状态 [cite: 40]。
        // ==========================================
        if (currentPlayerModel != null && currentPlayerModel.GetCurrentState() == PlayerState.Cover)
        {
            isFire = false;
            isAiming = false;
            isSprint = false;
            isJumping = false;
            isSlide = false;
            [cite_start]moveInput = Vector2.zero; [cite: 40, 41]
        }
        #endregion

        [cite_start]// 🗣️ 大白话：如果是刚刚按下跳跃，并且踩在地上，且不是在爬墙或掩体里，就放跳跃音效 [cite: 42, 43]
        if (input.Player.IsJuming.WasPressedThisFrame() && currentPlayerModel != null)
        {
            PlayerState currentState = currentPlayerModel.GetCurrentState();
            if (wasGrounded && currentState != PlayerState.Climb && currentState != PlayerState.Cover && currentState != PlayerState.Dead)
            {
                [cite_start]PlayActionSound(jumpSounds); [cite: 42, 43]
            }
        }

        if (climbCooldownTimer > 0) climbCooldownTimer -= Time.deltaTime; // 🗣️ 大白话：攀爬冷却倒计时 [cite: 44]

        [cite_start]// 🗣️ 大白话：控制屏幕上“按 F 进入掩体”提示 UI 的显示和隐藏 [cite: 45, 46]
        if (coverHintUI != null)
        {
            bool shouldShow = canEnterCover &&
                              currentPlayerModel.GetCurrentState() != PlayerState.Cover &&
                              Time.time >= nextCoverTime;
            [cite_start]if (coverHintUI.activeSelf != shouldShow) coverHintUI.SetActive(shouldShow); [cite: 45, 46]
        }

        [cite_start]// 🗣️ 大白话：控制屏幕上“攀爬”提示 UI 的显示和隐藏，必须要眼睛（射线）看到墙才能显示 [cite: 46, 47, 48, 49]
        if (climbHintUI != null)
        {
            bool showClimbHint = false;
            if (currentPlayerModel != null)
            {
                PlayerState currentState = currentPlayerModel.GetCurrentState();
                if (currentState != PlayerState.Climb && currentState != PlayerState.Vault &&
                    currentState != PlayerState.Cover && currentState != PlayerState.Dead)
                {
                    showClimbHint = CheckWallInFront(); // 🔗 引用：调用自己下方的射线检测方法 [cite: 48]
                }
            }
            [cite_start]if (climbHintUI.activeSelf != showClimbHint) climbHintUI.SetActive(showClimbHint); [cite: 49]
        }

        [cite_start]// 🗣️ 大白话：如果条件满足且按了 F，大脑下达终极指令：喂肉体！切到掩体状态！ [cite: 50]
        if (canEnterCover && Input.GetKeyDown(KeyCode.F) && Time.time >= nextCoverTime)
        {
            if (currentPlayerModel.GetCurrentState() != PlayerState.Cover)
            {
                currentPlayerModel.SwitchState(PlayerState.Cover); // 🔗 引用：跨脚本调用 HR 部门换班 [cite: 50]
                PlayInteractSound(); // 进入掩体时如果你也想算作交互，可以直接在这里播 [cite: 51]
            }
        }

        #region 移动方向计算
        // 🗣️ 大白话：核心算法！把玩家按下的 WASD，转换成基于当前相机朝向的 3D 世界方向。
        [cite_start]// （如果不这么写，你按 W 永远是往地图正北方走，而不是往你看着的方向走） [cite: 51, 52]
        Vector3 cameraForwardProjection = new Vector3(cameraTransform.forward.x, 0, cameraTransform.forward.z).normalized;
        [cite_start]worldMovement = cameraForwardProjection * moveInput.y + cameraTransform.right * moveInput.x; [cite: 51, 52]
        
        [cite_start]// 🗣️ 大白话：把世界方向转换成以玩家肉体为中心的相对方向（方便传给动画树做走路动画） [cite: 52]
        [cite_start]localMovement = currentPlayerModel.transform.InverseTransformVector(worldMovement); [cite: 52]
        #endregion

        HandleAimIKWeight(); // 🔗 引用：调用下方的方法处理手部和身体的 IK 扭动 [cite: 53]

        [cite_start]// 🗣️ 大白话：如果在瞄准或开火，强行用鼠标输入去修改瞄准相机的参数（控制镜头移动） [cite: 53, 54]
        if ((isAiming || isFire) && aimingCamera != null)
        {
            aimingCamera.m_XAxis.Value += lookInput.x * aimXSensitivity * Time.deltaTime;
            [cite_start]aimingCamera.m_YAxis.Value -= lookInput.y * aimYSensitivity * Time.deltaTime; [cite: 53, 54]
            aimingCamera.m_YAxis.Value = Mathf.Clamp(aimingCamera.m_YAxis.Value, 0.01f, 0.85f); // 🗣️ 大白话：限制抬头低头的死角 [cite: 54]
        }

        HandleWindEffect(); // 🔗 引用：根据速度处理屏幕风线 [cite: 55]
    }
    
    void LateUpdate()
    {
        if (currentPlayerModel == null || currentPlayerModel.weapon == null) return;

        [cite_start]// 🗣️ 大白话：防换枪错乱。如果你切枪了，把原来记下的武器位置清空重新记 [cite: 56, 57, 58]
        if (cachedWeaponTrans != null && cachedWeaponTrans != currentPlayerModel.weapon.transform)
        {
            cachedWeaponTrans = null;
            [cite_start]hasCachedDefaults = false; [cite: 56, 57]
        }
        if (cachedWeaponTrans == null)
        {
            cachedWeaponTrans = currentPlayerModel.weapon.transform;
            defaultWeaponPos = cachedWeaponTrans.localPosition;
            defaultWeaponRot = cachedWeaponTrans.localRotation;
            [cite_start]hasCachedDefaults = true; [cite: 57, 58]
        }

        bool useFPS = false;
        [cite_start]// 🗣️ 大白话：去问当前手里的枪，支不支持第一人称机瞄 [cite: 59]
        if (currentPlayerModel.weapon != null && currentPlayerModel.weapon.gunData != null)
            [cite_start]useFPS = currentPlayerModel.weapon.gunData.useFirstPersonAim; [cite: 59]

        [cite_start]// 🗣️ 大白话：如果是第一人称机瞄，把枪的模型强行粘在眼睛（ADS_Point）前面！ [cite: 60, 61]
        if (isAiming && useFPS && cachedAdsPoint != null)
        {
            cachedWeaponTrans.position = cachedAdsPoint.position;
            [cite_start]cachedWeaponTrans.rotation = cachedAdsPoint.rotation; [cite: 60, 61]
        }
        else if (hasCachedDefaults)
        {
            [cite_start]// 🗣️ 大白话：如果退镜了，乖乖把枪放回盲射时候的腰间位置 [cite: 61, 62]
            cachedWeaponTrans.localPosition = defaultWeaponPos;
            [cite_start]cachedWeaponTrans.localRotation = defaultWeaponRot; [cite: 61, 62]
        }

        // ==========================================
        [cite_start]// 🗣️ 大白话：滞空探测与落地砸坑的重音 [cite: 62, 63, 64, 65]
        // ==========================================
        if (currentPlayerModel != null)
        {
            [cite_start]// 🗣️ 大白话：往脚底下打一根极短的射线，看有没有踩在地上 [cite: 62, 63]
            Vector3 rayStart = currentPlayerModel.transform.position + Vector3.up * 0.5f;
            [cite_start]bool isCurrentlyGrounded = Physics.Raycast(rayStart, Vector3.down, 0.6f, ~0, QueryTriggerInteraction.Ignore); [cite: 62, 63]
            PlayerState currentState = currentPlayerModel.GetCurrentState();
            if (currentState != PlayerState.Climb)
            {
                if (!isCurrentlyGrounded) airTime += Time.deltaTime; // 🗣️ 大白话：记录脚离地多久了 [cite: 64]
                if (!wasGrounded && isCurrentlyGrounded)
                {
                    [cite_start]// 🗣️ 大白话：上一秒离地，这一秒踩地了 = 刚落地！只要滞空超过 0.05 秒就播落地音效 [cite: 65]
                    if (airTime > 0.05f) PlayActionSound(landSounds);
                    [cite_start]airTime = 0f; [cite: 65, 66]
                }
            }
            [cite_start]wasGrounded = isCurrentlyGrounded; [cite: 66]
        }
    }
    
    [cite_start]// 🗣️ 大白话：大脑的眼睛：射激光看能不能爬墙 [cite: 67, 68]
    public bool CheckWallInFront()
    {
        if (currentPlayerModel == null) return false;
        Vector3 origin = currentPlayerModel.transform.position + Vector3.up * 1.0f - (currentPlayerModel.transform.forward * backwardOffset);
        [cite_start]float checkRayLength = backwardOffset + maxClimbDistance + 0.2f; [cite: 67, 68]
        if (Physics.Raycast(origin, currentPlayerModel.transform.forward, out RaycastHit hit, checkRayLength, climbableLayer))
        {
            float realDistance = hit.distance - backwardOffset;
            if (realDistance >= minClimbDistance && realDistance <= maxClimbDistance)
            {
                float angle = Vector3.Angle(currentPlayerModel.transform.forward, -hit.normal);
                [cite_start]if (angle <= maxClimbAngle) return true; [cite: 69, 70, 71]
            }
        }
        [cite_start]return false; [cite: 71]
    }

    [cite_start]// 🗣️ 大白话：只有当你同时按住 W 和空格，且前面有墙，大脑才同意让你爬 [cite: 72, 73]
    public bool CheckClimbInput()
    {
        if (climbCooldownTimer > 0) return false;
        bool holdingForward = moveInput.y > 0.1f;
        bool holdingJump = isJumping;
        if (!holdingForward || !holdingJump) return false;
        [cite_start]return CheckWallInFront(); [cite: 72, 73]
    }

    public void StartClimbCooldown(float duration) { climbCooldownTimer = duration; } [cite: 74]

    [cite_start]// 🗣️ 大白话：控制肉体的下蹲，用 SmoothDamp 让碰撞体像海绵一样软绵绵地变矮，而不是瞬间压扁 [cite: 75, 76, 77]
    public void SetBodyHeight(bool isCrouching)
    {
        if (currentPlayerModel == null || currentPlayerModel.cc == null) return;
        float targetH = isCrouching ? crouchHeight : standHeight;
        [cite_start]Vector3 targetC = isCrouching ? crouchCenter : standCenter; [cite: 75, 76]
        currentPlayerModel.cc.height = Mathf.SmoothDamp(currentPlayerModel.cc.height, targetH, ref currentHeightVelocity, crouchSmoothTime);
        [cite_start]currentPlayerModel.cc.center = Vector3.SmoothDamp(currentPlayerModel.cc.center, targetC, ref currentCenterVelocity, crouchSmoothTime); [cite: 76, 77]
    }

    [cite_start]// 🗣️ 大白话：进出掩体的控制方法 [cite: 78, 79, 80, 81]
    public void StartCoverCooldown() { nextCoverTime = Time.time + 0.5f; } [cite: 78]
    public void EnterCoverZone(CoverSpot spot) { currentCoverSpot = spot; canEnterCover = true; } [cite: 79]
    public void ExitCoverZone()
    {
        if (currentPlayerModel.GetCurrentState() != PlayerState.Cover)
        {
            currentCoverSpot = null;
            [cite_start]canEnterCover = false; [cite: 80, 81]
        }
    }

    [cite_start]// 🗣️ 大白话：控制屏幕风线特效：冲刺时拉满，跑时变弱，开枪时停风 [cite: 81, 82, 83]
    private void HandleWindEffect()
    {
        if (windEffect == null) return;
        var emission = windEffect.emission;
        float targetRate = 0f;
        if (isAiming || isFire) targetRate = 0f;
        [cite_start]else if (moveInput.magnitude > 0.1f) targetRate = isSprint ? sprintWindRate : runWindRate; [cite: 81, 82, 83]
        [cite_start]emission.rateOverTime = targetRate; [cite: 83]
    }

    // ==========================================
    // 🗣️ 大白话：【顶级硬核】处理 IK（反向动力学）权重
    // 控制双手和腰部去扭动适应枪口的瞄准。用 Lerp (插值) 让举枪的过程有 0.几秒的平滑抬手动画。
    // ==========================================
    private void HandleAimIKWeight()
    {
        if (currentPlayerModel == null) return;
        [cite_start]// 🗣️ 大白话：手里没枪，或者躲在掩体里，所有的 IK 骨骼全部放松（权重设为 0） [cite: 84, 85, 86, 87, 88, 89, 90, 91]
        if (currentPlayerModel.weapon == null)
        {
            currentPlayerModel.rightHandAimConstraint.weight = Mathf.Lerp(currentPlayerModel.rightHandAimConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            if (currentPlayerModel.bodyAimConstraint != null) currentPlayerModel.bodyAimConstraint.weight = Mathf.Lerp(currentPlayerModel.bodyAimConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            if (currentPlayerModel.leftHandConstraint != null) currentPlayerModel.leftHandConstraint.weight = Mathf.Lerp(currentPlayerModel.leftHandConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            if (currentPlayerModel.rightHandConstraint != null) currentPlayerModel.rightHandConstraint.weight = Mathf.Lerp(currentPlayerModel.rightHandConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            [cite_start]return; [cite: 85, 86, 87, 88]
        }

        if (currentPlayerModel.GetCurrentState() == PlayerState.Cover)
        {
            currentPlayerModel.rightHandAimConstraint.weight = 0f;
            if (currentPlayerModel.bodyAimConstraint != null) currentPlayerModel.bodyAimConstraint.weight = 0f;
            if (currentPlayerModel.leftHandConstraint != null) currentPlayerModel.leftHandConstraint.weight = 0f;
            if (currentPlayerModel.rightHandConstraint != null) currentPlayerModel.rightHandConstraint.weight = 0f;
            [cite_start]return; [cite: 89, 90, 91]
        }

        [cite_start]// 🗣️ 大白话：平滑过渡瞄准的手部动作 [cite: 91, 92, 93, 94]
        float currentAimWeight = currentPlayerModel.rightHandAimConstraint.weight;
        [cite_start]float smoothedAimWeight = Mathf.Lerp(currentAimWeight, targetAimWeight, aimSmoothSpeed * Time.deltaTime); [cite: 91, 92]
        currentPlayerModel.rightHandAimConstraint.weight = smoothedAimWeight;
        [cite_start]if (currentPlayerModel.bodyAimConstraint != null) currentPlayerModel.bodyAimConstraint.weight = smoothedAimWeight; [cite: 92]
        
        float targetHandWeight = 1f;
        bool isSliding = currentPlayerModel.animator.GetCurrentAnimatorStateInfo(0).IsName("Slide");
        if (isSliding) targetHandWeight = 1f;
        else if (isSprint) targetHandWeight = 0f; // 🗣️ 大白话：冲刺时双手要随动画摆动，不能死死抓着枪 [cite: 93]
        [cite_start]else targetHandWeight = 1f - smoothedAimWeight; [cite: 93, 94]

        [cite_start]if (currentPlayerModel.rightHandConstraint != null) currentPlayerModel.rightHandConstraint.weight = Mathf.Lerp(currentPlayerModel.rightHandConstraint.weight, targetHandWeight, aimSmoothSpeed * Time.deltaTime); [cite: 94]
        
        float targetTPSWeight = (!isAiming && !isSprint && !isSlide) ? 1f : 0f;
        [cite_start]if (currentPlayerModel.leftHandConstraint != null) currentPlayerModel.leftHandConstraint.weight = Mathf.Lerp(currentPlayerModel.leftHandConstraint.weight, targetTPSWeight, aimSmoothSpeed * Time.deltaTime); [cite: 95, 96]
    }

    [cite_start]// 🗣️ 大白话：进瞄准。强制摄影师把开镜相机的权重调高，日常相机的权重归零 [cite: 97, 98]
    public void EnterAim()
    {
        targetAimWeight = 1f;
        freeLookCamera.Priority = 0;
        aimingCamera.Priority = 100;
        aimingCamera.m_XAxis.Value = freeLookCamera.m_XAxis.Value; // 🗣️ 大白话：保证切相机时镜头角度对齐 [cite: 97, 98]
        [cite_start]aimingCamera.m_YAxis.Value = freeLookCamera.m_YAxis.Value; [cite: 98]
    }

    [cite_start]// 🗣️ 大白话：退镜。把相机的权重切回来 [cite: 99, 100, 101]
    public void ExitAim(bool syncCamera = true)
    {
        targetAimWeight = 0f;
        if (syncCamera && freeLookCamera != null)
        {
            float currentYRotation = Camera.main.transform.eulerAngles.y;
            [cite_start]freeLookCamera.m_XAxis.Value = currentYRotation; [cite: 99, 100, 101]
        }
        freeLookCamera.Priority = 100;
        [cite_start]aimingCamera.Priority = 0; [cite: 101]
    }

    [cite_start]// 🗣️ 大白话：调用组件震动屏幕（开枪后坐力） [cite: 102, 103]
    public void ShakeCamera()
    {
        [cite_start]if (impulseSource != null) impulseSource.GenerateImpulse(); [cite: 102]
    }
    // =========================================================
    // 【高级音效模块】
    // =========================================================
    [Header("音效随机化参数 (防机械音)")]
    public float minPitch = 0.85f;
    public float maxPitch = 1.15f; // 🗣️ 大白话：每次播放声音，音调都有点细微差别 [cite: 103, 104]
    public float minVolume = 0.8f;
    public float maxVolume = 1.0f;

    [Header("脚步冷却防抽风")]
    public float stepCooldown = 0.3f; // 🗣️ 大白话：防止同一秒内踩出十步的鬼畜声音 [cite: 104, 105]
    private float lastStepTime = 0f;
    [cite_start]private int lastStepFrame = -1; [cite: 105]

    private void PlayRandomSoundFromArray(AudioClip[] clipArray)
    {
        if (Time.frameCount == lastStepFrame) return; // 🗣️ 大白话：同一帧绝不播两次 [cite: 106]
        if (Time.time - lastStepTime < stepCooldown) return; // 🗣️ 大白话：没冷却完不播 [cite: 106, 107]
        if (isJumping) return; // 🗣️ 大白话：人在天上不播脚步声 [cite: 107]

        if (clipArray != null && clipArray.Length > 0 && audioSource != null)
        {
            lastStepTime = Time.time;
            [cite_start]lastStepFrame = Time.frameCount; [cite: 108, 109]

            int randomIndex = Random.Range(0, clipArray.Length); // 🗣️ 大白话：随机抽盲盒 [cite: 109]
            AudioClip selectedClip = clipArray[randomIndex];

            audioSource.pitch = Random.Range(minPitch, maxPitch); // 🗣️ 大白话：稍微改变音高变魔术 [cite: 109]
            audioSource.volume = Random.Range(minVolume, maxVolume);
            [cite_start]audioSource.PlayOneShot(selectedClip); [cite: 109, 110]
        }
    }

    private void PlayActionSound(AudioClip[] clipArray)
    {
        if (clipArray != null && clipArray.Length > 0 && audioSource != null)
        {
            int randomIndex = Random.Range(0, clipArray.Length);
            audioSource.pitch = Random.Range(0.95f, 1.05f);
            audioSource.volume = Random.Range(0.9f, 1.0f);
            [cite_start]audioSource.PlayOneShot(clipArray[randomIndex]); [cite: 110, 111]
        }
    }

    [cite_start]// 👇 🗣️ 大白话：带兜底功能的交互音效。如果没指定放什么，就放默认的 interactSound [cite: 111, 112, 113, 114]
    public void PlayInteractSound(AudioClip customSound = null)
    {
        if (audioSource == null) return;
        [cite_start]AudioClip clipToPlay = (customSound != null) ? customSound : interactSound; [cite: 111, 112]
        if (clipToPlay != null)
        {
            audioSource.pitch = Random.Range(0.95f, 1.05f);
            audioSource.volume = 1.0f;
            [cite_start]audioSource.PlayOneShot(clipToPlay); [cite: 112, 113, 114]
        }
    }

    [cite_start]// 🗣️ 大白话：下面这几个方法，是为了让挂在模型上的动画事件（Animation Event）方便调用的 [cite: 114, 115, 116]
    public void PlayRunSound()
    {
        if (isSprint) PlayRandomSoundFromArray(sprintSounds);
        [cite_start]else PlayRandomSoundFromArray(runSounds); [cite: 114, 115]
    }
    public void PlaySprintSound() { PlayRandomSoundFromArray(sprintSounds); }
    public void PlaySlideSound() { PlayActionSound(slideSounds); }
    public void PlayClimbSound() { PlayActionSound(climbSounds); } [cite: 115, 116]

    private void OnEnable() { input.Enable(); } // 🗣️ 大白话：启用脚本时激活输入系统 [cite: 116, 117]
    private void OnDisable() { input.Disable(); }

    // 🗣️ 大白话：这是程序员自己用的调试工具。
    [cite_start]// 在 Unity 里的 Scene 视图里画出胸口那根探测能不能爬墙的激光，红线是不能爬，绿线是能爬！ [cite: 117, 118, 119, 120, 121, 122, 123]
    private void OnDrawGizmos()
    {
        if (currentPlayerModel == null) return;
        Vector3 origin = currentPlayerModel.transform.position + Vector3.up * 1.0f - (currentPlayerModel.transform.forward * backwardOffset);
        [cite_start]float checkRayLength = backwardOffset + maxClimbDistance + 0.2f; [cite: 117, 118]
        [cite_start]bool isHit = Physics.Raycast(origin, currentPlayerModel.transform.forward, out RaycastHit hit, checkRayLength, climbableLayer); [cite: 118, 119]
        if (isHit)
        {
            float realDistance = hit.distance - backwardOffset;
            [cite_start]float angle = Vector3.Angle(currentPlayerModel.transform.forward, -hit.normal); [cite: 120, 121]
            if (realDistance >= minClimbDistance && realDistance <= maxClimbDistance && angle <= maxClimbAngle) Gizmos.color = Color.green;
            [cite_start]else Gizmos.color = Color.yellow; [cite: 121, 122]
            [cite_start]Gizmos.DrawRay(origin, currentPlayerModel.transform.forward * hit.distance); [cite: 122]
        }
        else
        {
            Gizmos.color = Color.red;
            [cite_start]Gizmos.DrawRay(origin, currentPlayerModel.transform.forward * checkRayLength); [cite: 122, 123]
        }
        [cite_start]Gizmos.DrawSphere(origin, 0.05f); [cite: 123]
    }
}
    ```