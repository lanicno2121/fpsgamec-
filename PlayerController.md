```csharp
﻿using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Cinemachine;
using UnityEngine.Animations.Rigging;

[DefaultExecutionOrder(100)]
public class PlayerController : SingleMonoBase<PlayerController>
{
    public PlayerModel currentPlayerModel;
    private Transform cameraTransform;

    [Header("瞄准灵敏度")]
    public float aimXSensitivity = 15f;
    public float aimYSensitivity = 0.002f;

    [Header("相机设置")]
    public CinemachineFreeLook freeLookCamera;
    public CinemachineFreeLook aimingCamera;
    public Transform fpsCameraTransform;

    [Header("UI 引用")]
    public GameObject deathPanel;

    [Header("视觉特效")]
    public ParticleSystem windEffect;
    public float runWindRate = 5f;
    public float sprintWindRate = 30f;

    [Header("掩体系统")]
    public CinemachineFreeLook coverCamera;
    [HideInInspector] public CoverSpot currentCoverSpot;
    private bool canEnterCover = false;
    private float nextCoverTime = 0f;
    public GameObject coverHintUI;

    [Header("攀爬系统 UI")]
    public GameObject climbHintUI;

    // =========================================================
    // 【高级音效系统】声音池
    // =========================================================
    [Header("玩家音效 (支持多个声音随机播放)")]
    public AudioClip[] runSounds;
    public AudioClip[] sprintSounds;
    public AudioClip[] slideSounds;
    public AudioClip[] climbSounds;

    [Header("跳跃/落地音效 (物理驱动)")]
    public AudioClip[] jumpSounds;
    public AudioClip[] landSounds;

    [Header("通用交互音效 (兜底默认声音)")]
    public AudioClip interactSound;

    // ==========================================
    // 【新增】瞄准音效 (Foley)
    // ==========================================
    [Header("瞄准音效 (Foley)")]
    public AudioClip aimInSound;  // 举枪瞄准时的声音 (衣服摩擦)
    public AudioClip aimOutSound; // 放下枪时的声音
    private bool wasAimingLastFrame = false; // 用于探测瞬间动作

    private bool wasGrounded = true;
    private float airTime = 0f;

    private AudioSource audioSource;

    [Header("攀爬系统 (Apex Style)")]
    public float climbSpeed = 5f;
    public float maxClimbDuration = 100f;
    public LayerMask climbableLayer;
    public float climbCheckDistance = 1.0f;

    [Header("=== 🧗‍♂️ 攀爬触发限制 ===")]
    public float minClimbDistance = 0.25f;
    public float maxClimbDistance = 0.8f;
    public float maxClimbAngle = 45f;
    public float backwardOffset = 0.5f;
    private float climbCooldownTimer = 0f;

    [Header("物理高度设置")]
    public float standHeight = 1.8f;
    public float crouchHeight = 1.0f;
    public Vector3 standCenter = new Vector3(0, 0.9f, 0);
    public Vector3 crouchCenter = new Vector3(0, 0.5f, 0);
    public float crouchSmoothTime = 0.1f;

    private float currentHeightVelocity;
    private Vector3 currentCenterVelocity;

    #region 玩家输入相关
    private Myinputsystem input;
    [HideInInspector] public Vector2 moveInput;
    [HideInInspector] public bool isSprint;
    [HideInInspector] private bool isSprintToggle = false;
    [HideInInspector] public bool isAiming;
    [HideInInspector] public bool isJumping;
    [HideInInspector] public bool isFire;
    [HideInInspector] public bool isSlide;
    #endregion

    #region 瞄准相关
    public Transform AimTarget;
    public float maxRayDistance = 1000f;
    public LayerMask aimLayerMask = ~0;
    private float targetAimWeight = 0f;
    public float aimSmoothSpeed = 10f;
    #endregion

    private CinemachineImpulseSource impulseSource;
    public float runSpeed = 4f;
    public float sprintSpeed = 7f;
    public float rotationSpeed = 300;

    [HideInInspector] public Vector3 localMovement;
    [HideInInspector] public Vector3 worldMovement;

    private Transform cachedAdsPoint;
    private Transform cachedWeaponTrans;
    private Vector3 defaultWeaponPos;
    private Quaternion defaultWeaponRot;
    private bool hasCachedDefaults = false;

    protected override void Awake()
    {
        base.Awake();
        input = new Myinputsystem();
    }

    void Start()
    {
        cameraTransform = Camera.main.transform;
        Cursor.lockState = CursorLockMode.Locked;
        ExitAim();

        impulseSource = GetComponent<CinemachineImpulseSource>();

        Transform searchRoot = fpsCameraTransform != null ? fpsCameraTransform : (aimingCamera != null ? aimingCamera.transform : null);
        if (searchRoot != null)
        {
            cachedAdsPoint = searchRoot.Find("ADS_Point");
            if (cachedAdsPoint == null)
            {
                cachedAdsPoint = searchRoot.GetComponentInChildren<Transform>().Find("ADS_Point");
            }
        }

        if (currentPlayerModel != null && currentPlayerModel.cc != null)
        {
            standHeight = currentPlayerModel.cc.height;
            standCenter = currentPlayerModel.cc.center;
        }

        audioSource = GetComponent<AudioSource>();
        if (audioSource == null)
        {
            audioSource = gameObject.AddComponent<AudioSource>();
        }
        audioSource.playOnAwake = false;
        audioSource.spatialBlend = 0f;
    }

    void Update()
    {
        #region 更新玩家输入
        Vector2 lookInput = input.Player.Look.ReadValue<Vector2>();
        moveInput = input.Player.Move.ReadValue<Vector2>().normalized;
        isSprint = input.Player.IsSprint.IsPressed();
        isAiming = input.Player.IsAming.IsPressed();

        // ==========================================
        // 瞄准瞬间发声探测
        // ==========================================
        if (isAiming && !wasAimingLastFrame)
        {
            // 刚刚按下右键，举枪
            if (aimInSound != null && audioSource != null) audioSource.PlayOneShot(aimInSound);
        }
        else if (!isAiming && wasAimingLastFrame)
        {
            // 刚刚松开右键，放下枪
            if (aimOutSound != null && audioSource != null) audioSource.PlayOneShot(aimOutSound);
        }
        wasAimingLastFrame = isAiming;

        isJumping = input.Player.IsJuming.IsPressed();
        isFire = input.Player.Fire.IsPressed();
        isSlide = input.Player.IsSlide.WasPressedThisFrame();

        if (input.Player.IsSprint.WasPressedThisFrame()) isSprintToggle = !isSprintToggle;
        if (moveInput.magnitude < 0.1f) isSprintToggle = false;
        isSprint = isSprintToggle;

        if (currentPlayerModel != null && currentPlayerModel.GetCurrentState() == PlayerState.Cover)
        {
            isFire = false;
            isAiming = false;
            isSprint = false;
            isJumping = false;
            isSlide = false;
            moveInput = Vector2.zero;
        }
        #endregion

        if (input.Player.IsJuming.WasPressedThisFrame() && currentPlayerModel != null)
        {
            PlayerState currentState = currentPlayerModel.GetCurrentState();
            if (wasGrounded && currentState != PlayerState.Climb && currentState != PlayerState.Cover && currentState != PlayerState.Dead)
            {
                PlayActionSound(jumpSounds);
            }
        }

        if (climbCooldownTimer > 0) climbCooldownTimer -= Time.deltaTime;

        if (coverHintUI != null)
        {
            bool shouldShow = canEnterCover &&
                              currentPlayerModel.GetCurrentState() != PlayerState.Cover &&
                              Time.time >= nextCoverTime;
            if (coverHintUI.activeSelf != shouldShow) coverHintUI.SetActive(shouldShow);
        }

        if (climbHintUI != null)
        {
            bool showClimbHint = false;
            if (currentPlayerModel != null)
            {
                PlayerState currentState = currentPlayerModel.GetCurrentState();
                if (currentState != PlayerState.Climb && currentState != PlayerState.Vault &&
                    currentState != PlayerState.Cover && currentState != PlayerState.Dead)
                {
                    showClimbHint = CheckWallInFront();
                }
            }
            if (climbHintUI.activeSelf != showClimbHint) climbHintUI.SetActive(showClimbHint);
        }

        if (canEnterCover && Input.GetKeyDown(KeyCode.F) && Time.time >= nextCoverTime)
        {
            if (currentPlayerModel.GetCurrentState() != PlayerState.Cover)
            {
                currentPlayerModel.SwitchState(PlayerState.Cover);
                PlayInteractSound(); // 进入掩体时如果你也想算作交互，可以直接在这里播
            }
        }

        #region 移动方向计算
        Vector3 cameraForwardProjection = new Vector3(cameraTransform.forward.x, 0, cameraTransform.forward.z).normalized;
        worldMovement = cameraForwardProjection * moveInput.y + cameraTransform.right * moveInput.x;
        localMovement = currentPlayerModel.transform.InverseTransformVector(worldMovement);
        #endregion

        HandleAimIKWeight();

        if ((isAiming || isFire) && aimingCamera != null)
        {
            aimingCamera.m_XAxis.Value += lookInput.x * aimXSensitivity * Time.deltaTime;
            aimingCamera.m_YAxis.Value -= lookInput.y * aimYSensitivity * Time.deltaTime;
            aimingCamera.m_YAxis.Value = Mathf.Clamp(aimingCamera.m_YAxis.Value, 0.01f, 0.85f);
        }

        HandleWindEffect();
    }

    void LateUpdate()
    {
        if (currentPlayerModel == null || currentPlayerModel.weapon == null) return;
        if (cachedWeaponTrans != null && cachedWeaponTrans != currentPlayerModel.weapon.transform)
        {
            cachedWeaponTrans = null;
            hasCachedDefaults = false;
        }
        if (cachedWeaponTrans == null)
        {
            cachedWeaponTrans = currentPlayerModel.weapon.transform;
            defaultWeaponPos = cachedWeaponTrans.localPosition;
            defaultWeaponRot = cachedWeaponTrans.localRotation;
            hasCachedDefaults = true;
        }

        bool useFPS = false;
        if (currentPlayerModel.weapon != null && currentPlayerModel.weapon.gunData != null)
            useFPS = currentPlayerModel.weapon.gunData.useFirstPersonAim;

        if (isAiming && useFPS && cachedAdsPoint != null)
        {
            cachedWeaponTrans.position = cachedAdsPoint.position;
            cachedWeaponTrans.rotation = cachedAdsPoint.rotation;
        }
        else if (hasCachedDefaults)
        {
            cachedWeaponTrans.localPosition = defaultWeaponPos;
            cachedWeaponTrans.localRotation = defaultWeaponRot;
        }

        if (currentPlayerModel != null)
        {
            Vector3 rayStart = currentPlayerModel.transform.position + Vector3.up * 0.5f;
            bool isCurrentlyGrounded = Physics.Raycast(rayStart, Vector3.down, 0.6f, ~0, QueryTriggerInteraction.Ignore);
            PlayerState currentState = currentPlayerModel.GetCurrentState();

            if (currentState != PlayerState.Climb)
            {
                if (!isCurrentlyGrounded) airTime += Time.deltaTime;

                if (!wasGrounded && isCurrentlyGrounded)
                {
                    if (airTime > 0.05f) PlayActionSound(landSounds);
                    airTime = 0f;
                }
            }
            wasGrounded = isCurrentlyGrounded;
        }
    }

    public bool CheckWallInFront()
    {
        if (currentPlayerModel == null) return false;
        Vector3 origin = currentPlayerModel.transform.position + Vector3.up * 1.0f - (currentPlayerModel.transform.forward * backwardOffset);
        float checkRayLength = backwardOffset + maxClimbDistance + 0.2f;

        if (Physics.Raycast(origin, currentPlayerModel.transform.forward, out RaycastHit hit, checkRayLength, climbableLayer))
        {
            float realDistance = hit.distance - backwardOffset;
            if (realDistance >= minClimbDistance && realDistance <= maxClimbDistance)
            {
                float angle = Vector3.Angle(currentPlayerModel.transform.forward, -hit.normal);
                if (angle <= maxClimbAngle) return true;
            }
        }
        return false;
    }

    public bool CheckClimbInput()
    {
        if (climbCooldownTimer > 0) return false;
        bool holdingForward = moveInput.y > 0.1f;
        bool holdingJump = isJumping;
        if (!holdingForward || !holdingJump) return false;
        return CheckWallInFront();
    }

    public void StartClimbCooldown(float duration) { climbCooldownTimer = duration; }

    public void SetBodyHeight(bool isCrouching)
    {
        if (currentPlayerModel == null || currentPlayerModel.cc == null) return;
        float targetH = isCrouching ? crouchHeight : standHeight;
        Vector3 targetC = isCrouching ? crouchCenter : standCenter;
        currentPlayerModel.cc.height = Mathf.SmoothDamp(currentPlayerModel.cc.height, targetH, ref currentHeightVelocity, crouchSmoothTime);
        currentPlayerModel.cc.center = Vector3.SmoothDamp(currentPlayerModel.cc.center, targetC, ref currentCenterVelocity, crouchSmoothTime);
    }

    public void StartCoverCooldown() { nextCoverTime = Time.time + 0.5f; }
    public void EnterCoverZone(CoverSpot spot) { currentCoverSpot = spot; canEnterCover = true; }
    public void ExitCoverZone()
    {
        if (currentPlayerModel.GetCurrentState() != PlayerState.Cover)
        {
            currentCoverSpot = null;
            canEnterCover = false;
        }
    }

    private void HandleWindEffect()
    {
        if (windEffect == null) return;
        var emission = windEffect.emission;
        float targetRate = 0f;
        if (isAiming || isFire) targetRate = 0f;
        else if (moveInput.magnitude > 0.1f) targetRate = isSprint ? sprintWindRate : runWindRate;
        emission.rateOverTime = targetRate;
    }

    private void HandleAimIKWeight()
    {
        if (currentPlayerModel == null) return;
        if (currentPlayerModel.weapon == null)
        {
            currentPlayerModel.rightHandAimConstraint.weight = Mathf.Lerp(currentPlayerModel.rightHandAimConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            if (currentPlayerModel.bodyAimConstraint != null) currentPlayerModel.bodyAimConstraint.weight = Mathf.Lerp(currentPlayerModel.bodyAimConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            if (currentPlayerModel.leftHandConstraint != null) currentPlayerModel.leftHandConstraint.weight = Mathf.Lerp(currentPlayerModel.leftHandConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            if (currentPlayerModel.rightHandConstraint != null) currentPlayerModel.rightHandConstraint.weight = Mathf.Lerp(currentPlayerModel.rightHandConstraint.weight, 0f, aimSmoothSpeed * Time.deltaTime);
            return;
        }

        if (currentPlayerModel.GetCurrentState() == PlayerState.Cover)
        {
            currentPlayerModel.rightHandAimConstraint.weight = 0f;
            if (currentPlayerModel.bodyAimConstraint != null) currentPlayerModel.bodyAimConstraint.weight = 0f;
            if (currentPlayerModel.leftHandConstraint != null) currentPlayerModel.leftHandConstraint.weight = 0f;
            if (currentPlayerModel.rightHandConstraint != null) currentPlayerModel.rightHandConstraint.weight = 0f;
            return;
        }

        float currentAimWeight = currentPlayerModel.rightHandAimConstraint.weight;
        float smoothedAimWeight = Mathf.Lerp(currentAimWeight, targetAimWeight, aimSmoothSpeed * Time.deltaTime);
        currentPlayerModel.rightHandAimConstraint.weight = smoothedAimWeight;
        if (currentPlayerModel.bodyAimConstraint != null) currentPlayerModel.bodyAimConstraint.weight = smoothedAimWeight;

        float targetHandWeight = 1f;
        bool isSliding = currentPlayerModel.animator.GetCurrentAnimatorStateInfo(0).IsName("Slide");
        if (isSliding) targetHandWeight = 1f;
        else if (isSprint) targetHandWeight = 0f;
        else targetHandWeight = 1f - smoothedAimWeight;

        if (currentPlayerModel.rightHandConstraint != null) currentPlayerModel.rightHandConstraint.weight = Mathf.Lerp(currentPlayerModel.rightHandConstraint.weight, targetHandWeight, aimSmoothSpeed * Time.deltaTime);

        float targetTPSWeight = (!isAiming && !isSprint && !isSlide) ? 1f : 0f;
        if (currentPlayerModel.leftHandConstraint != null) currentPlayerModel.leftHandConstraint.weight = Mathf.Lerp(currentPlayerModel.leftHandConstraint.weight, targetTPSWeight, aimSmoothSpeed * Time.deltaTime);
    }

    public void EnterAim()
    {
        targetAimWeight = 1f;
        freeLookCamera.Priority = 0;
        aimingCamera.Priority = 100;
        aimingCamera.m_XAxis.Value = freeLookCamera.m_XAxis.Value;
        aimingCamera.m_YAxis.Value = freeLookCamera.m_YAxis.Value;
    }

    public void ExitAim(bool syncCamera = true)
    {
        targetAimWeight = 0f;
        if (syncCamera && freeLookCamera != null)
        {
            float currentYRotation = Camera.main.transform.eulerAngles.y;
            freeLookCamera.m_XAxis.Value = currentYRotation;
        }
        freeLookCamera.Priority = 100;
        aimingCamera.Priority = 0;
    }

    public void ShakeCamera()
    {
        if (impulseSource != null) impulseSource.GenerateImpulse();
    }

    // =========================================================
    // 【高级音效模块】
    // =========================================================
    [Header("音效随机化参数 (防机械音)")]
    public float minPitch = 0.85f;
    public float maxPitch = 1.15f;
    public float minVolume = 0.8f;
    public float maxVolume = 1.0f;

    [Header("脚步冷却防抽风")]
    public float stepCooldown = 0.3f;
    private float lastStepTime = 0f;
    private int lastStepFrame = -1;

    private void PlayRandomSoundFromArray(AudioClip[] clipArray)
    {
        if (Time.frameCount == lastStepFrame) return;
        if (Time.time - lastStepTime < stepCooldown) return;
        if (isJumping) return;

        if (clipArray != null && clipArray.Length > 0 && audioSource != null)
        {
            lastStepTime = Time.time;
            lastStepFrame = Time.frameCount;

            int randomIndex = Random.Range(0, clipArray.Length);
            AudioClip selectedClip = clipArray[randomIndex];

            audioSource.pitch = Random.Range(minPitch, maxPitch);
            audioSource.volume = Random.Range(minVolume, maxVolume);

            audioSource.PlayOneShot(selectedClip);
        }
    }

    private void PlayActionSound(AudioClip[] clipArray)
    {
        if (clipArray != null && clipArray.Length > 0 && audioSource != null)
        {
            int randomIndex = Random.Range(0, clipArray.Length);
            audioSource.pitch = Random.Range(0.95f, 1.05f);
            audioSource.volume = Random.Range(0.9f, 1.0f);
            audioSource.PlayOneShot(clipArray[randomIndex]);
        }
    }

    // 👇 =======================================================
    // 【终极升级】带有可选参数的交互发声器！
    // =======================================================
    public void PlayInteractSound(AudioClip customSound = null)
    {
        if (audioSource == null) return;

        // 如果你传了专属声音，就用专属的；如果没传，就用默认的兜底声音 interactSound
        AudioClip clipToPlay = (customSound != null) ? customSound : interactSound;

        if (clipToPlay != null)
        {
            // 微微调整一点点音调，防止每次声音完全一样显得廉价
            audioSource.pitch = Random.Range(0.95f, 1.05f);
            audioSource.volume = 1.0f;
            audioSource.PlayOneShot(clipToPlay);
        }
    }

    public void PlayRunSound()
    {
        if (isSprint) PlayRandomSoundFromArray(sprintSounds);
        else PlayRandomSoundFromArray(runSounds);
    }

    public void PlaySprintSound() { PlayRandomSoundFromArray(sprintSounds); }
    public void PlaySlideSound() { PlayActionSound(slideSounds); }
    public void PlayClimbSound() { PlayActionSound(climbSounds); }

    private void OnEnable() { input.Enable(); }
    private void OnDisable() { input.Disable(); }

    private void OnDrawGizmos()
    {
        if (currentPlayerModel == null) return;
        Vector3 origin = currentPlayerModel.transform.position + Vector3.up * 1.0f - (currentPlayerModel.transform.forward * backwardOffset);
        float checkRayLength = backwardOffset + maxClimbDistance + 0.2f;

        bool isHit = Physics.Raycast(origin, currentPlayerModel.transform.forward, out RaycastHit hit, checkRayLength, climbableLayer);

        if (isHit)
        {
            float realDistance = hit.distance - backwardOffset;
            float angle = Vector3.Angle(currentPlayerModel.transform.forward, -hit.normal);
            if (realDistance >= minClimbDistance && realDistance <= maxClimbDistance && angle <= maxClimbAngle) Gizmos.color = Color.green;
            else Gizmos.color = Color.yellow;
            Gizmos.DrawRay(origin, currentPlayerModel.transform.forward * hit.distance);
        }
        else
        {
            Gizmos.color = Color.red;
            Gizmos.DrawRay(origin, currentPlayerModel.transform.forward * checkRayLength);
        }
        Gizmos.DrawSphere(origin, 0.05f);
    }
}
    ```