using UnityEngine;
using TMPro;

public class KeycardDoor : MonoBehaviour
{
    [Header("门禁设置")]
    [Tooltip("需要什么物品才能开门？(拖入你之前做的 门禁卡 ItemData)")]
    public ItemData requiredKeycard;
    [Tooltip("开门后是否消耗掉门禁卡？")]
    public bool consumeKeycard = false;

    [Header("开门动画设置")]
    [Tooltip("把门轴物体 (DoorHinge) 拖到这里")]
    public Transform doorHinge;
    [Tooltip("门打开的最终角度 (通常是 Y轴 90度 或 -90度)")]
    public Vector3 openRotation = new Vector3(0, 90f, 0);
    [Tooltip("开门的平滑速度")]
    public float openSpeed = 3f;

    [Header("UI 提示设置")]
    [Tooltip("悬浮在门上的 '需要门禁卡(按F)' 提示UI")]
    public GameObject hintUI;
    [Tooltip("屏幕正中间的通知容器 (复用之前坏门的那个即可)")]
    public GameObject centerNotificationPanel;
    [Tooltip("用来显示 '门已解锁' 或 '缺少门禁卡' 的文字组件")]
    public TMP_Text centerNotificationText;

    // 👇 ==========================================
    // 【新增】专属交互音效槽位
    // ==========================================
    [Header("专属交互音效 (可选)")]
    [Tooltip("如果不填，就会播放默认音效。建议放个：电子刷卡声")]
    public AudioClip specificInteractSound;

    // 内部状态
    private bool isPlayerInRange = false;
    private bool isOpen = false;
    private InventoryManager cachedInventory;

    private Quaternion closedRotation;
    private Quaternion targetOpenRotation;

    void Start()
    {
        // 游戏开始时隐藏UI
        if (hintUI != null) hintUI.SetActive(false);
        if (centerNotificationPanel != null) centerNotificationPanel.SetActive(false);

        // 记录门轴的初始旋转，并计算出打开后的目标旋转
        if (doorHinge != null)
        {
            closedRotation = doorHinge.localRotation;
            targetOpenRotation = Quaternion.Euler(openRotation) * closedRotation;
        }
    }

    void Update()
    {
        // ==========================================
        // 1. 物理开门动画 (平滑旋转门轴)
        // ==========================================
        if (isOpen && doorHinge != null)
        {
            doorHinge.localRotation = Quaternion.Lerp(doorHinge.localRotation, targetOpenRotation, Time.deltaTime * openSpeed);
        }

        // ==========================================
        // 2. 交互逻辑
        // ==========================================
        if (!isOpen && isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            PlayerController player = FindObjectOfType<PlayerController>();
            if (player != null)
            {
                // 👇 ==========================================
                // 【升级】把专属音效传给喇叭！
                // ==========================================
                player.PlayInteractSound(specificInteractSound);
            }

            if (cachedInventory != null && requiredKeycard != null)
            {
                // 【核心】调用你背包系统里的 HasItem 方法！
                if (cachedInventory.HasItem(requiredKeycard, 1))
                {
                    UnlockDoor();
                }
                else
                {
                    ShowMessage("缺少门禁卡！");
                }
            }
        }
    }

    private void UnlockDoor()
    {
        isOpen = true;

        // 隐藏门上的 3D 提示
        if (hintUI != null) hintUI.SetActive(false);

        ShowMessage("权限确认，门已解锁");

        // 如果设置了开门消耗门禁卡，就从背包里扣除
        if (consumeKeycard && cachedInventory != null)
        {
            cachedInventory.RemoveItem(requiredKeycard, 1);
        }
    }

    private void ShowMessage(string message)
    {
        if (centerNotificationPanel != null && centerNotificationText != null)
        {
            centerNotificationText.text = message;
            centerNotificationPanel.SetActive(true);

            // 重新计时 2 秒后关闭提示
            CancelInvoke(nameof(HideMessage));
            Invoke(nameof(HideMessage), 2f);
        }
    }

    private void HideMessage()
    {
        if (centerNotificationPanel != null)
        {
            centerNotificationPanel.SetActive(false);
        }
    }

    private void OnTriggerEnter(Collider other)
    {
        if (isOpen) return; // 门开了就不再显示按键提示了

        var inv = other.GetComponent<InventoryManager>();
        if (inv == null) inv = other.GetComponentInParent<InventoryManager>();

        if (inv != null)
        {
            isPlayerInRange = true;
            cachedInventory = inv;
            if (hintUI != null) hintUI.SetActive(true);
        }
    }

    private void OnTriggerExit(Collider other)
    {
        var inv = other.GetComponent<InventoryManager>();
        if (inv == null) inv = other.GetComponentInParent<InventoryManager>();

        if (inv != null && inv == cachedInventory)
        {
            isPlayerInRange = false;
            cachedInventory = null;
            if (hintUI != null) hintUI.SetActive(false);
        }
    }
}