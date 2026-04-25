using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Animations.Rigging;

public class WeaponManager : MonoBehaviour
{
    [Header("配置")]
    public Transform weaponSocket;
    public List<WeaponEntry> allWeapons = new List<WeaponEntry>();

    [Header("组件引用")]
    public PlayerModel playerModel;
    public AmmoDisplay ammoDisplay;
    public TwoBoneIKConstraint leftHandIK;
    public TwoBoneIKConstraint rightHandIK;

    [Header("战术切枪设置 (秒)")]
    public float holsterDuration = 0.6f;
    public float drawDuration = 0.6f;

    // 👇 ==========================================
    // 【新增】切枪音效设置
    // ==========================================
    [Header("切枪音效 (Foley)")]
    public AudioClip drawSound;     // 拔出武器的声音 (按1/2)
    public AudioClip holsterSound;  // 收起武器的声音 (按3)
    private AudioSource audioSource;

    // 层级平滑过渡时间
    public float layerTransitionDuration = 0.3f;

    // 动画参数 Hash
    private int holsterHash;
    private int drawHash;

    // 动画层索引
    private Animator animator;
    private int pistolLayerIndex = -1;
    private int unarmedLayerIndex = -1;
    private int upperBodyLayerIndex = -1;

    // 内部记录
    private GameObject currentWeaponObj;
    private int currentIndex = -1;

    public int CurrentIndex => currentIndex;

    // 保存当前枪械的弹药系统引用
    private GunAmmoSystem currentAmmoSystem;

    // 状态标记
    public bool IsUnarmed => currentIndex == -1;
    private bool isUsingPistol = false;
    private bool isSwitching = false;
    private bool isLayerWeightChanging = false;

    [Header("武器解锁状态")]
    [Tooltip("分别记录每把武器是否已解锁 (0=步枪, 1=手枪)")]
    public bool[] unlockedWeapons;

    [System.Serializable]
    public struct WeaponEntry
    {
        public string name;
        public GameObject prefab;
        public GunData data;
        public bool usePistolLayer;
    }

    IEnumerator Start()
    {
        // 【新增】初始化音效播放器
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null) audioSource = gameObject.AddComponent<AudioSource>();
        audioSource.playOnAwake = false;

        if (playerModel != null)
            animator = playerModel.GetComponent<Animator>();

        if (animator != null)
        {
            pistolLayerIndex = animator.GetLayerIndex("Pistol Layer");
            unarmedLayerIndex = animator.GetLayerIndex("Unarmed Layer");
            upperBodyLayerIndex = animator.GetLayerIndex("UpperBody");
            holsterHash = Animator.StringToHash("TriggerHolster");
            drawHash = Animator.StringToHash("TriggerDraw");
        }

        if (allWeapons.Count > 0)
        {
            unlockedWeapons = new bool[allWeapons.Count];
        }

        yield return null;
        EnterUnarmedMode(true);
    }

    void Update()
    {
        if (playerModel != null && playerModel.GetCurrentState() == PlayerState.Cover)
        {
            return; // 掩体状态无视按键
        }

        if (isSwitching) return;

        if (!IsUnarmed)
        {
            if (Input.GetKeyDown(KeyCode.Alpha1) && currentIndex != 0 && CheckUnlocked(0))
                StartCoroutine(SwitchWeaponRoutine(0));

            if (Input.GetKeyDown(KeyCode.Alpha2) && currentIndex != 1 && CheckUnlocked(1))
                StartCoroutine(SwitchWeaponRoutine(1));

            if (Input.GetKeyDown(KeyCode.Alpha3))
                StartCoroutine(HolsterToUnarmedRoutine());
        }
        else
        {
            if (Input.GetKeyDown(KeyCode.Alpha1) && CheckUnlocked(0))
                StartCoroutine(DrawFromUnarmedRoutine(0));

            if (Input.GetKeyDown(KeyCode.Alpha2) && CheckUnlocked(1))
                StartCoroutine(DrawFromUnarmedRoutine(1));
        }

        // G键：丢弃并锁定所有武器 (测试用)
        if (Input.GetKeyDown(KeyCode.G))
        {
            EnterUnarmedMode(false);
            for (int i = 0; i < unlockedWeapons.Length; i++)
            {
                unlockedWeapons[i] = false;
            }
        }
    }

    private bool CheckUnlocked(int index)
    {
        if (unlockedWeapons != null && index >= 0 && index < unlockedWeapons.Length)
        {
            return unlockedWeapons[index];
        }
        return false;
    }

    IEnumerator SwitchWeaponRoutine(int targetIndex)
    {
        isSwitching = true;
        if (animator != null && upperBodyLayerIndex != -1)
            StartCoroutine(FadeSingleLayer(upperBodyLayerIndex, 1f, 0.2f));

        // 播报：开始收枪音效
        if (holsterSound != null) audioSource.PlayOneShot(holsterSound);
        if (animator != null) animator.SetTrigger(holsterHash);

        yield return new WaitForSeconds(holsterDuration);

        EquipWeapon(targetIndex);

        // 播报：开始拔枪音效
        if (drawSound != null) audioSource.PlayOneShot(drawSound);
        if (animator != null) animator.SetTrigger(drawHash);

        yield return new WaitForSeconds(drawDuration);
        isSwitching = false;
    }

    IEnumerator HolsterToUnarmedRoutine()
    {
        isSwitching = true;

        // 播报：开始收枪音效
        if (holsterSound != null) audioSource.PlayOneShot(holsterSound);
        if (animator != null) animator.SetTrigger(holsterHash);

        yield return new WaitForSeconds(holsterDuration);

        EnterUnarmedMode(false);

        isSwitching = false;
    }

    IEnumerator DrawFromUnarmedRoutine(int targetIndex)
    {
        isSwitching = true;
        EquipWeapon(targetIndex);

        // 播报：开始拔枪音效
        if (drawSound != null) audioSource.PlayOneShot(drawSound);
        if (animator != null) animator.SetTrigger(drawHash);

        yield return new WaitForSeconds(drawDuration);
        isSwitching = false;
    }

    void LateUpdate()
    {
        if (isSwitching || isLayerWeightChanging)
        {
            if (rightHandIK != null) rightHandIK.weight = 0f;
            if (leftHandIK != null) leftHandIK.weight = 0f;
            return;
        }

        if (playerModel != null)
        {
            PlayerState state = playerModel.GetCurrentState();

            if (state == PlayerState.Cover || state == PlayerState.Vault)
            {
                if (rightHandIK != null) rightHandIK.weight = 0f;
                if (leftHandIK != null) leftHandIK.weight = 0f;
                return;
            }

            if (state == PlayerState.Hover || state == PlayerState.Climb)
            {
                if (rightHandIK != null) rightHandIK.weight = 0f;
                if (leftHandIK != null) leftHandIK.weight = 0f;

                if (IsUnarmed && animator != null && unarmedLayerIndex != -1)
                {
                    float currentW = animator.GetLayerWeight(unarmedLayerIndex);
                    animator.SetLayerWeight(unarmedLayerIndex, Mathf.Lerp(currentW, 0f, Time.deltaTime * 15f));
                }
                return;
            }

            if (state == PlayerState.Jump)
            {
                if (rightHandIK != null) rightHandIK.weight = 0f;
                if (leftHandIK != null) leftHandIK.weight = 0f;

                if (IsUnarmed && animator != null && unarmedLayerIndex != -1)
                {
                    float currentW = animator.GetLayerWeight(unarmedLayerIndex);
                    animator.SetLayerWeight(unarmedLayerIndex, Mathf.Lerp(currentW, 1f, Time.deltaTime * 15f));
                }
                return;
            }
        }

        if (IsUnarmed)
        {
            if (rightHandIK != null) rightHandIK.weight = 0f;
            if (leftHandIK != null) leftHandIK.weight = 0f;

            if (animator != null && unarmedLayerIndex != -1)
            {
                float currentW = animator.GetLayerWeight(unarmedLayerIndex);
                animator.SetLayerWeight(unarmedLayerIndex, Mathf.Lerp(currentW, 1f, Time.deltaTime * 15f));

                if (pistolLayerIndex != -1) animator.SetLayerWeight(pistolLayerIndex, 0f);
                if (upperBodyLayerIndex != -1) animator.SetLayerWeight(upperBodyLayerIndex, 0f);
            }
        }
        else
        {
            bool isSprinting = false;
            if (PlayerController.INSTANCE != null) isSprinting = PlayerController.INSTANCE.isSprint;

            if (isSprinting)
            {
                if (rightHandIK != null) rightHandIK.weight = 0f;
                if (leftHandIK != null) leftHandIK.weight = 0f;
            }
            else
            {
                bool isReloading = (currentAmmoSystem != null && currentAmmoSystem.isReloading);

                if (isUsingPistol)
                {
                    if (rightHandIK != null) rightHandIK.weight = 0f;
                    if (leftHandIK != null) leftHandIK.weight = isReloading ? 0f : 1f;
                }
                else
                {
                    if (!isReloading)
                    {
                        if (rightHandIK != null) rightHandIK.weight = 1f;
                        if (leftHandIK != null) leftHandIK.weight = 1f;
                    }
                    else
                    {
                        if (rightHandIK != null) rightHandIK.weight = 0f;
                        if (leftHandIK != null) leftHandIK.weight = 0f;
                    }
                }
            }
        }
    }

    public void EnterUnarmedMode(bool instant = false)
    {
        currentIndex = -1;
        isUsingPistol = false;
        currentAmmoSystem = null;

        if (leftHandIK != null && weaponSocket != null)
        {
            var data = leftHandIK.data;
            data.target = weaponSocket;
            leftHandIK.data = data;
        }

        if (currentWeaponObj != null) Destroy(currentWeaponObj);
        if (playerModel != null) playerModel.weapon = null;

        if (playerModel != null && playerModel.fpsLeftArmModel != null)
            playerModel.fpsLeftArmModel.SetActive(false);
        if (ammoDisplay != null) ammoDisplay.BindNewWeapon(null);

        if (animator != null && unarmedLayerIndex != -1)
        {
            bool isMoving = false;
            if (PlayerController.INSTANCE != null)
            {
                float currentMoveInput = PlayerController.INSTANCE.moveInput.magnitude;
                isMoving = currentMoveInput > 0.1f;
                animator.SetFloat("MoveBlend", isMoving ? 1f : 0f);
            }

            if (isMoving) animator.CrossFadeInFixedTime("Move", 0.2f, unarmedLayerIndex);
            else animator.CrossFadeInFixedTime("Idle", 0.2f, unarmedLayerIndex);

            if (instant)
            {
                animator.SetLayerWeight(unarmedLayerIndex, 1f);
                if (upperBodyLayerIndex != -1) animator.SetLayerWeight(upperBodyLayerIndex, 0f);
                if (pistolLayerIndex != -1) animator.SetLayerWeight(pistolLayerIndex, 0f);
            }
            else
            {
                StartCoroutine(SmoothUnarmedTransition());
            }
        }
    }

    IEnumerator SmoothUnarmedTransition()
    {
        isLayerWeightChanging = true;
        float timer = 0f;

        float startUnarmed = animator.GetLayerWeight(unarmedLayerIndex);
        float startUpper = (upperBodyLayerIndex != -1) ? animator.GetLayerWeight(upperBodyLayerIndex) : 0f;
        float startPistol = (pistolLayerIndex != -1) ? animator.GetLayerWeight(pistolLayerIndex) : 0f;

        while (timer < layerTransitionDuration)
        {
            timer += Time.deltaTime;
            float t = timer / layerTransitionDuration;

            animator.SetLayerWeight(unarmedLayerIndex, Mathf.Lerp(startUnarmed, 1f, t));
            if (upperBodyLayerIndex != -1)
                animator.SetLayerWeight(upperBodyLayerIndex, Mathf.Lerp(startUpper, 0f, t));
            if (pistolLayerIndex != -1)
                animator.SetLayerWeight(pistolLayerIndex, Mathf.Lerp(startPistol, 0f, t));

            yield return null;
        }

        animator.SetLayerWeight(unarmedLayerIndex, 1f);
        if (upperBodyLayerIndex != -1) animator.SetLayerWeight(upperBodyLayerIndex, 0f);
        if (pistolLayerIndex != -1) animator.SetLayerWeight(pistolLayerIndex, 0f);

        isLayerWeightChanging = false;
    }

    public void EquipWeapon(int index)
    {
        if (index < 0 || index >= allWeapons.Count) return;

        if (unlockedWeapons != null && index < unlockedWeapons.Length)
        {
            unlockedWeapons[index] = true;
        }

        if (currentWeaponObj != null) Destroy(currentWeaponObj);

        WeaponEntry weaponInfo = allWeapons[index];
        currentWeaponObj = Instantiate(weaponInfo.prefab, weaponSocket);

        var newWeaponScript = currentWeaponObj.GetComponent<PlayerWeapon>();
        currentAmmoSystem = currentWeaponObj.GetComponent<GunAmmoSystem>();

        if (newWeaponScript != null) newWeaponScript.gunData = weaponInfo.data;
        if (currentAmmoSystem != null) currentAmmoSystem.gunData = weaponInfo.data;
        if (playerModel != null) playerModel.weapon = newWeaponScript;

        if (playerModel != null && newWeaponScript != null)
        {
            playerModel.fpsLeftArmModel = newWeaponScript.armModel;
            if (playerModel.fpsLeftArmModel != null) playerModel.fpsLeftArmModel.SetActive(false);
        }

        if (ammoDisplay != null) ammoDisplay.BindNewWeapon(currentAmmoSystem);
        UpdateLeftHandIK();

        isUsingPistol = weaponInfo.usePistolLayer;
        currentIndex = index;

        if (animator != null)
        {
            StartCoroutine(FadeSingleLayer(unarmedLayerIndex, 0f, 0.25f));
            if (pistolLayerIndex != -1)
            {
                float targetWeight = isUsingPistol ? 1f : 0f;
                StartCoroutine(FadeSingleLayer(pistolLayerIndex, targetWeight, 0.25f));
            }
            if (upperBodyLayerIndex != -1)
                StartCoroutine(FadeSingleLayer(upperBodyLayerIndex, 1f, 0.25f));
        }

        if (!isUsingPistol)
        {
            if (rightHandIK != null) rightHandIK.weight = 1f;
            if (leftHandIK != null) leftHandIK.weight = 1f;
        }
    }

    IEnumerator FadeSingleLayer(int layerIndex, float targetWeight, float duration)
    {
        isLayerWeightChanging = true;
        float startWeight = animator.GetLayerWeight(layerIndex);
        float timer = 0f;

        while (timer < duration)
        {
            timer += Time.deltaTime;
            float newWeight = Mathf.Lerp(startWeight, targetWeight, timer / duration);
            animator.SetLayerWeight(layerIndex, newWeight);
            yield return null;
        }
        animator.SetLayerWeight(layerIndex, targetWeight);
        isLayerWeightChanging = false;
    }

    void UpdateLeftHandIK()
    {
        if (leftHandIK == null || currentWeaponObj == null) return;
        Transform newTarget = FindDeepChild(currentWeaponObj.transform, "Left Hand Target");
        if (newTarget != null)
        {
            var data = leftHandIK.data;
            data.target = newTarget;
            leftHandIK.data = data;
            var rb = playerModel.GetComponent<RigBuilder>();
            if (rb == null) rb = playerModel.GetComponentInChildren<RigBuilder>();
            if (rb != null) rb.Build();
        }
    }

    Transform FindDeepChild(Transform parent, string name)
    {
        foreach (Transform child in parent)
        {
            if (child.name == name) return child;
            var result = FindDeepChild(child, name);
            if (result != null) return result;
        }
        return null;
    }
}