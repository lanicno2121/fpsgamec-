using UnityEngine;
using UnityEngine.UI;
using TMPro;
using Cinemachine; // 必须引入这个命名空间才能控制摄像机

public class InventoryUI : MonoBehaviour
{
    [Header("核心引用")]
    public InventoryManager inventoryManager;
    public PlayerController playerController;
    public PlayerHealth playerHealth;
    public WeaponManager weaponManager;

    [Header("UI 容器绑定")]
    public GameObject inventoryPanel;
    public Transform gridZone;
    public GameObject slotPrefab;

    [Header("详情区绑定")]
    public TMP_Text itemNameText;
    public TMP_Text itemDescText;

    [Header("状态区与武器区绑定")]
    public TMP_Text statusHealthText;
    public Image currentWeaponIcon;
    public TMP_Text currentWeaponText;

    // 👇 ==========================================
    // 【新增】背包音效设置
    // ==========================================
    [Header("背包音效 (Foley)")]
    public AudioClip openSound;   // 打开背包时的声音 (拉链声等)
    public AudioClip closeSound;  // 关闭背包时的声音 (布料摩擦/合上声)
    private AudioSource audioSource;

    private bool isOpen = false;

    // 用来保存相机本来的转动速度
    private float cachedFreeCamSpeedX = 300f;
    private float cachedFreeCamSpeedY = 2f;
    private float cachedAimCamSpeedX = 300f;
    private float cachedAimCamSpeedY = 2f;

    void Start()
    {
        // 【新增】初始化发声器
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null) audioSource = gameObject.AddComponent<AudioSource>();
        audioSource.playOnAwake = false;

        if (inventoryPanel != null) inventoryPanel.SetActive(false);
        ClearDescription();

        if (playerController == null) playerController = FindObjectOfType<PlayerController>();
        if (inventoryManager == null && playerController != null)
            inventoryManager = playerController.GetComponent<InventoryManager>();
        if (playerHealth == null && playerController != null)
            playerHealth = playerController.GetComponent<PlayerHealth>();
        if (weaponManager == null && playerController != null)
            weaponManager = playerController.GetComponent<WeaponManager>();

        if (inventoryManager != null) inventoryManager.OnInventoryChanged += RefreshUI;
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Tab)) ToggleInventory();
    }

    public void ToggleInventory()
    {
        if (inventoryPanel == null) return;
        isOpen = !isOpen;
        inventoryPanel.SetActive(isOpen);

        if (isOpen)
        {
            // 👇 ==========================================
            // 【新增】播放打开背包的音效
            // ==========================================
            if (openSound != null && audioSource != null)
            {
                audioSource.pitch = Random.Range(0.95f, 1.05f); // 微微随机化音调
                audioSource.PlayOneShot(openSound);
            }

            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;

            if (playerController != null)
            {
                // 优化一：在禁用控制器前，瞬间强制清空所有输入信号！
                playerController.moveInput = Vector2.zero;
                playerController.worldMovement = Vector3.zero;
                playerController.localMovement = Vector3.zero;
                playerController.isSprint = false;
                playerController.isAiming = false;
                playerController.isFire = false;
                playerController.isJumping = false;
                playerController.isSlide = false;

                // 优化二：冻结相机的鼠标转动
                if (playerController.freeLookCamera != null)
                {
                    cachedFreeCamSpeedX = playerController.freeLookCamera.m_XAxis.m_MaxSpeed;
                    cachedFreeCamSpeedY = playerController.freeLookCamera.m_YAxis.m_MaxSpeed;
                    playerController.freeLookCamera.m_XAxis.m_MaxSpeed = 0f;
                    playerController.freeLookCamera.m_YAxis.m_MaxSpeed = 0f;
                }
                if (playerController.aimingCamera != null)
                {
                    cachedAimCamSpeedX = playerController.aimingCamera.m_XAxis.m_MaxSpeed;
                    cachedAimCamSpeedY = playerController.aimingCamera.m_YAxis.m_MaxSpeed;
                    playerController.aimingCamera.m_XAxis.m_MaxSpeed = 0f;
                    playerController.aimingCamera.m_YAxis.m_MaxSpeed = 0f;
                }

                // 安全地关掉玩家控制
                playerController.enabled = false;
            }

            RefreshUI();
            ClearDescription();
        }
        else
        {
            // 👇 ==========================================
            // 【新增】播放关闭背包的音效
            // ==========================================
            if (closeSound != null && audioSource != null)
            {
                audioSource.pitch = Random.Range(0.95f, 1.05f);
                audioSource.PlayOneShot(closeSound);
            }

            Cursor.lockState = CursorLockMode.Locked;
            Cursor.visible = false;

            if (playerController != null)
            {
                // 恢复：把相机本来的速度还给它
                if (playerController.freeLookCamera != null)
                {
                    playerController.freeLookCamera.m_XAxis.m_MaxSpeed = cachedFreeCamSpeedX;
                    playerController.freeLookCamera.m_YAxis.m_MaxSpeed = cachedFreeCamSpeedY;
                }
                if (playerController.aimingCamera != null)
                {
                    playerController.aimingCamera.m_XAxis.m_MaxSpeed = cachedAimCamSpeedX;
                    playerController.aimingCamera.m_YAxis.m_MaxSpeed = cachedAimCamSpeedY;
                }

                // 重新接管角色
                playerController.enabled = true;
            }
        }
    }

    public void RefreshUI()
    {
        if (!isOpen) return;

        foreach (Transform child in gridZone) Destroy(child.gameObject);

        if (inventoryManager != null)
        {
            foreach (var slot in inventoryManager.slots)
            {
                GameObject newSlotObj = Instantiate(slotPrefab, gridZone);
                Image icon = newSlotObj.transform.Find("ItemIcon").GetComponent<Image>();
                TMP_Text amountText = newSlotObj.transform.Find("AmountText").GetComponent<TMP_Text>();

                if (slot.item.icon != null)
                {
                    icon.sprite = slot.item.icon;
                    icon.color = Color.white;
                }
                else icon.color = Color.clear;

                amountText.text = slot.amount > 1 ? "x" + slot.amount : "";

                Button btn = newSlotObj.GetComponent<Button>();
                if (btn == null) btn = newSlotObj.AddComponent<Button>();
                ItemData currentItem = slot.item;
                btn.onClick.AddListener(() => ShowDescription(currentItem));
            }
        }

        if (playerHealth != null && statusHealthText != null)
        {
            int currentHP = Mathf.CeilToInt(playerHealth.currentHealth);
            statusHealthText.text = $"生命值: {currentHP} / {playerHealth.maxHealth}";
        }

        if (weaponManager != null && currentWeaponText != null)
        {
            if (weaponManager.IsUnarmed)
            {
                currentWeaponText.text = "【当前武装】 空手";
                if (currentWeaponIcon != null) currentWeaponIcon.gameObject.SetActive(false);
            }
            else
            {
                int index = weaponManager.CurrentIndex;
                var weaponInfo = weaponManager.allWeapons[index];

                GunAmmoSystem activeAmmo = null;
                if (playerController != null) activeAmmo = playerController.GetComponentInChildren<GunAmmoSystem>();

                if (activeAmmo != null)
                {
                    currentWeaponText.text = $"【{weaponInfo.name}】\n弹药: {activeAmmo.currentMagAmmo} / {activeAmmo.currentReserveAmmo}";
                }

                if (currentWeaponIcon != null && weaponInfo.data != null && weaponInfo.data.weaponIcon != null)
                {
                    currentWeaponIcon.sprite = weaponInfo.data.weaponIcon;
                    currentWeaponIcon.gameObject.SetActive(true);
                }
                else if (currentWeaponIcon != null)
                {
                    currentWeaponIcon.gameObject.SetActive(false);
                }
            }
        }
    }

    public void ShowDescription(ItemData item)
    {
        if (itemNameText != null) itemNameText.text = item.itemName;
        if (itemDescText != null) itemDescText.text = item.description;
    }

    public void ClearDescription()
    {
        if (itemNameText != null) itemNameText.text = "";
        if (itemDescText != null) itemDescText.text = "";
    }
}