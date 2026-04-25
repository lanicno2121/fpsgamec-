using UnityEngine;
using System.Collections;
using UnityEngine.Animations.Rigging;

public class WeaponRecoil : MonoBehaviour
{
    [Header("=== 核心引用 ===")]
    public Transform visualModel;
    public PlayerModel playerModel;
    public SimpleShellEjector shellEjector;

    [Header("=== 物理换弹设置 ===")]
    public GameObject realMagazine;
    public GameObject droppedMagPrefab;
    public Transform magSpawnPoint;

    [Header("=== FPS 专属设置 ===")]
    public Vector3 fpsReloadRot = new Vector3(10f, 0f, 45f);
    public Vector3 fpsBasePosOffset;
    public Vector3 fpsBaseRotOffset;

    [Header("=== TPS 动画与微调 ===")]
    public string reloadTriggerName = "ReloadTrigger";
    public float ikSmoothSpeed = 10f;
    public Vector3 tpsReloadPosOffset;
    public Vector3 tpsReloadRotOffset;

    [Header("=== 后坐力参数 (通用) ===")]
    public Vector3 fpsRecoilAxis = new Vector3(0, 0, 1);
    public float fpsPosStrength = 0.05f;
    public float fpsRotStrength = 2f;
    public Transform fpsCamera;
    public float fpsCamShake = 0.5f;

    [Header("=== TPS 全身震动参数 (有IK时) ===")]
    public Vector3 tpsRecoilAxis = new Vector3(0, 0, 1);
    public float tpsHandStrength = 0.15f;
    [Range(0f, 1f)] public float shoulderRatio = 0.4f;
    public float spineRotStrength = 3f;

    // =========================================================
    // 【核心新增】无 IK 时的震动参数 (手枪专用 - 左右分离)
    // =========================================================
    [Header("=== TPS 手枪震动 (无IK时) ===")]
    [Tooltip("手枪开火震动强度 (建议 30~50)")]
    public float pistolArmKickStrength = 50f;

    [Tooltip("【左手】跟随强度 (0=不动, 1=完全跟随)")]
    [Range(0f, 1f)] public float leftArmFollowStrength = 0.8f;

    [Header("=== 骨骼轴向调试 (核心) ===")]
    [Tooltip("勾选后，在Scene窗口画出旋转轴 (红色=右臂, 蓝色=左臂)")]
    public bool showDebugRays = true;

    [Tooltip("【右手】旋转轴：(1,0,0)=X, (0,1,0)=Y, (0,0,1)=Z")]
    public Vector3 rightRecoilAxis = new Vector3(0, 0, 1); // 右手默认 Z

    [Tooltip("【左手】旋转轴：左手通常需要和右手反过来！试着填 (0,0,-1) 或者 (-1,0,0)")]
    public Vector3 leftRecoilAxis = new Vector3(0, 0, -1); // 左手默认 -Z (反向)

    public float snappiness = 20f;
    public float returnSpeed = 10f;

    // 内部变量
    private float currentRecoilWeight = 0f, targetRecoilWeight = 0f;
    private Vector3 currentRot, targetRot;
    private Vector3 initialGunPos;
    private Quaternion initialGunRot;

    // 换弹插值
    private Vector3 currentReloadPos, targetReloadPos;
    private Vector3 currentReloadRot, targetReloadRot;
    private float reloadSmooth = 6f;

    // IK
    private float targetIKWeight = 1f;
    private bool isControllingIK = false;

    // === 骨骼缓存 ===
    private Transform rightHandTarget;
    private Transform leftHandTarget;
    private Transform rightShoulderBone;
    private Transform leftShoulderBone;
    private Transform spineBone;

    // 【新增】大臂骨骼
    private Transform rightUpperArmBone;
    private Transform leftUpperArmBone;

    // 缓存的震动量
    private Vector3 currentTpsHandOffset;
    private Vector3 currentTpsShoulderOffset;
    private float currentSpineRot;

    private bool isInitialized = false;

    void Start()
    {
        InitializeOrigin();

        // 1. 寻找 PlayerModel
        if (playerModel == null) playerModel = GetComponentInParent<PlayerModel>();
        if (playerModel == null) playerModel = FindObjectOfType<PlayerModel>();

        if (playerModel == null)
        {
            Debug.LogError($"⛔ 【严重阻断】{name} 找不到 PlayerModel！");
            return;
        }

        // 2. 获取骨骼
        if (playerModel.animator != null)
        {
            // IK Targets
            if (playerModel.rightHandConstraint != null) rightHandTarget = playerModel.rightHandConstraint.data.tip;
            if (playerModel.leftHandConstraint != null) leftHandTarget = playerModel.leftHandConstraint.data.tip;

            // 身体骨骼
            spineBone = playerModel.animator.GetBoneTransform(HumanBodyBones.UpperChest);
            if (spineBone == null) spineBone = playerModel.animator.GetBoneTransform(HumanBodyBones.Chest);
            if (spineBone == null) spineBone = playerModel.animator.GetBoneTransform(HumanBodyBones.Spine);

            rightShoulderBone = playerModel.animator.GetBoneTransform(HumanBodyBones.RightShoulder);
            leftShoulderBone = playerModel.animator.GetBoneTransform(HumanBodyBones.LeftShoulder);

            // 【关键】获取左右大臂
            rightUpperArmBone = playerModel.animator.GetBoneTransform(HumanBodyBones.RightUpperArm);
            leftUpperArmBone = playerModel.animator.GetBoneTransform(HumanBodyBones.LeftUpperArm);

            if (rightUpperArmBone == null) Debug.LogWarning("⚠️ 警告：找不到 RightUpperArm 骨骼");
            if (leftUpperArmBone == null) Debug.LogWarning("⚠️ 警告：找不到 LeftUpperArm 骨骼");
        }

        // 注册事件
        var ammoSys = GetComponent<GunAmmoSystem>();
        if (ammoSys != null) ammoSys.OnReloadStatusChanged += OnReloadEvent;

        isInitialized = true;
    }

    void OnDestroy()
    {
        var ammoSys = GetComponent<GunAmmoSystem>();
        if (ammoSys != null) ammoSys.OnReloadStatusChanged -= OnReloadEvent;
    }

    void InitializeOrigin()
    {
        if (visualModel != null)
        {
            initialGunPos = visualModel.localPosition;
            initialGunRot = visualModel.localRotation;
        }
    }

    void OnReloadEvent(bool isReloading)
    {
        if (isReloading) StartCoroutine(ReloadRoutine());
        else
        {
            StopAllCoroutines();
            ResetState();
        }
    }

    void ResetState()
    {
        targetReloadPos = Vector3.zero;
        targetReloadRot = Vector3.zero;
        targetIKWeight = 1f;
        isControllingIK = false;
        if (realMagazine != null) realMagazine.SetActive(true);
    }

    IEnumerator ReloadRoutine()
    {
        // ... (保持原有的换弹逻辑不变) ...
        bool isFPS = (playerModel != null && playerModel.fpsLeftArmModel != null && playerModel.fpsLeftArmModel.activeSelf);
        if (isFPS)
        {
            targetReloadRot = fpsReloadRot;
            targetReloadPos = Vector3.zero;
            yield return new WaitForSeconds(0.4f);
            if (realMagazine != null) realMagazine.SetActive(false);
            if (droppedMagPrefab != null && magSpawnPoint != null) Destroy(Instantiate(droppedMagPrefab, magSpawnPoint.position, magSpawnPoint.rotation), 3f);
            yield return new WaitForSeconds(0.6f);
            targetReloadRot = Vector3.zero;
            yield return new WaitForSeconds(0.4f);
            if (realMagazine != null) realMagazine.SetActive(true);
        }
        else
        {
            if (playerModel.animator != null) playerModel.animator.SetTrigger(reloadTriggerName);
            isControllingIK = true;
            targetIKWeight = 0f;
            targetReloadPos = tpsReloadPosOffset;
            targetReloadRot = tpsReloadRotOffset;
            yield return new WaitForSeconds(0.5f);
            if (realMagazine != null) realMagazine.SetActive(false);
            if (droppedMagPrefab != null && magSpawnPoint != null)
            {
                GameObject mag = Instantiate(droppedMagPrefab, magSpawnPoint.position, magSpawnPoint.rotation);
                Rigidbody magRb = mag.GetComponent<Rigidbody>();
                if (magRb != null) magRb.AddForce(transform.right * 2f + Vector3.up * 1f, ForceMode.Impulse);
                Destroy(mag, 3f);
            }
            yield return new WaitForSeconds(1.5f);
            if (realMagazine != null) realMagazine.SetActive(true);
            yield return new WaitForSeconds(0.5f);
            targetIKWeight = 1f;
            targetReloadPos = Vector3.zero;
            targetReloadRot = Vector3.zero;
            yield return new WaitForSeconds(0.5f);
            isControllingIK = false;
        }
    }

    public void FireRecoil()
    {
        targetRecoilWeight += 1f;
        targetRot += new Vector3(fpsRotStrength, UnityEngine.Random.Range(-fpsRotStrength / 2f, fpsRotStrength / 2f), 0);
        if (shellEjector != null) shellEjector.Eject();
    }

    void Update()
    {
        if (!isInitialized || visualModel == null) return;

        targetRecoilWeight = Mathf.Lerp(targetRecoilWeight, 0f, returnSpeed * Time.deltaTime);
        currentRecoilWeight = Mathf.Lerp(currentRecoilWeight, targetRecoilWeight, snappiness * Time.deltaTime);

        targetRot = Vector3.Lerp(targetRot, Vector3.zero, returnSpeed * Time.deltaTime);
        currentRot = Vector3.Lerp(currentRot, targetRot, snappiness * Time.deltaTime);

        currentReloadRot = Vector3.Lerp(currentReloadRot, targetReloadRot, reloadSmooth * Time.deltaTime);
        currentReloadPos = Vector3.Lerp(currentReloadPos, targetReloadPos, reloadSmooth * Time.deltaTime);

        HandleSmartIK();

        bool isFPS = (playerModel != null && playerModel.fpsLeftArmModel != null && playerModel.fpsLeftArmModel.activeSelf);
        ApplyTransform(isFPS);
    }

    void LateUpdate()
    {
        if (!isInitialized) return;

        bool isTPS = (playerModel != null && (playerModel.fpsLeftArmModel == null || !playerModel.fpsLeftArmModel.activeSelf));

        if (isTPS)
        {
            float currentIKWeight = GetIKWeight();

            // ======================================================
            // 分流逻辑
            // ======================================================
            if (currentIKWeight > 0.5f)
            {
                // === A: 步枪模式 (IK 有效) ===
                if (rightHandTarget != null)
                {
                    if (isControllingIK)
                    {
                        rightHandTarget.localPosition += currentReloadPos;
                        rightHandTarget.localRotation *= Quaternion.Euler(currentReloadRot);
                    }
                    rightHandTarget.localPosition -= currentTpsHandOffset;
                }
                if (leftHandTarget != null) leftHandTarget.localPosition -= currentTpsHandOffset;
            }
            else
            {
                // === B: 手枪模式 (IK 无效，直接动骨头) ===
                float angle = -currentRecoilWeight * pistolArmKickStrength;

                // 1. 动右手
                if (rightUpperArmBone != null)
                {
                    rightUpperArmBone.Rotate(rightRecoilAxis, angle, Space.Self);

                    // 调试画线 (红色 = 右手轴)
                    if (showDebugRays) Debug.DrawRay(rightUpperArmBone.position, rightUpperArmBone.TransformDirection(rightRecoilAxis) * 0.5f, Color.red);
                }

                // 2. 动左手
                if (leftUpperArmBone != null && leftArmFollowStrength > 0.01f)
                {
                    // 【核心】使用左手专用轴，不受右手影响！
                    // 注意：这里我们用 angle (负数)，如果轴向对的话，手就会抬起来
                    leftUpperArmBone.Rotate(leftRecoilAxis, angle * leftArmFollowStrength, Space.Self);

                    // 调试画线 (蓝色 = 左手轴)
                    if (showDebugRays) Debug.DrawRay(leftUpperArmBone.position, leftUpperArmBone.TransformDirection(leftRecoilAxis) * 0.5f, Color.blue);
                }
            }

            // === C: 全身通用震动 ===
            if (rightShoulderBone != null) rightShoulderBone.position -= rightShoulderBone.TransformDirection(currentTpsShoulderOffset);
            if (leftShoulderBone != null) leftShoulderBone.position -= leftShoulderBone.TransformDirection(currentTpsShoulderOffset);
            if (spineBone != null) spineBone.Rotate(Vector3.right, -currentSpineRot, Space.Self);
        }
    }

    void HandleSmartIK()
    {
        if (playerModel == null) return;
        if (isControllingIK)
        {
            float currentIK = GetIKWeight();
            float newIK = Mathf.Lerp(currentIK, targetIKWeight, Time.deltaTime * ikSmoothSpeed);
            SetIKWeight(newIK);
        }
    }

    void SetIKWeight(float w)
    {
        if (playerModel.leftHandConstraint != null) playerModel.leftHandConstraint.weight = w;
        if (playerModel.rightHandConstraint != null) playerModel.rightHandConstraint.weight = w;
    }

    float GetIKWeight()
    {
        if (playerModel.rightHandConstraint != null) return playerModel.rightHandConstraint.weight;
        return 1f;
    }

    void ApplyTransform(bool isFPS)
    {
        Quaternion recoilRot = Quaternion.Euler(-currentRot.x, Random.Range(-currentRot.y, currentRot.y), 0);
        Quaternion reloadRotation = Quaternion.Euler(currentReloadRot);

        if (isFPS)
        {
            Vector3 recoilPosOffset = (fpsRecoilAxis.normalized * (currentRecoilWeight * fpsPosStrength));
            visualModel.localPosition = initialGunPos + fpsBasePosOffset - recoilPosOffset;
            visualModel.localRotation = initialGunRot * Quaternion.Euler(fpsBaseRotOffset) * reloadRotation * recoilRot;
            if (fpsCamera != null) fpsCamera.localRotation = Quaternion.Euler(-currentRot.x * fpsCamShake, 0, 0);

            currentTpsHandOffset = Vector3.zero;
            currentTpsShoulderOffset = Vector3.zero;
            currentSpineRot = 0f;
        }
        else
        {
            visualModel.localPosition = initialGunPos;
            visualModel.localRotation = initialGunRot;

            currentTpsHandOffset = (tpsRecoilAxis.normalized * (currentRecoilWeight * tpsHandStrength));
            currentTpsShoulderOffset = currentTpsHandOffset * shoulderRatio;
            currentSpineRot = currentRecoilWeight * spineRotStrength;
        }
    }
}