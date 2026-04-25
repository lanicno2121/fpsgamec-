using UnityEngine;
using TMPro; // 因为你用的是 TextMeshPro

public class CorpseLoot : MonoBehaviour
{
    [Header("搜刮设置")]
    [Tooltip("要给玩家什么物品？(把做好的 门禁卡ItemData 拖进来)")]
    public ItemData lootItem;
    public int amount = 1;

    [Header("UI 提示设置")]
    [Tooltip("挂在尸体上的 '按F搜刮' 提示UI")]
    public GameObject hintUI;

    [Tooltip("屏幕左侧的通知容器 (例如一个包含背景图和文字的Panel)")]
    public GameObject notificationPanel;
    [Tooltip("用来显示 '获得门禁卡' 的文字组件")]
    public TMP_Text notificationText;

    // ==========================================
    // 视觉提示
    // ==========================================
    [Header("视觉提示")]
    [Tooltip("尸体上闪烁的提示特效 (搜刮后会自动隐藏)")]
    public GameObject interactableEffect;

    // 内部状态
    private bool isPlayerInRange = false;
    private bool hasLooted = false; // 是否已经被搜刮过了
    private InventoryManager cachedInventory;

    void Start()
    {
        // 游戏开始时隐藏所有相关UI
        if (hintUI != null) hintUI.SetActive(false);
        if (notificationPanel != null) notificationPanel.SetActive(false);

        // 游戏一开始，确保护城河的光点是亮着的
        if (interactableEffect != null) interactableEffect.SetActive(true);
    }

    void Update()
    {
        // 如果玩家在范围内、没被搜刮过、且按下了 F 键
        if (!hasLooted && isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            if (cachedInventory != null && lootItem != null)
            {
                // 尝试放入背包
                bool success = cachedInventory.AddItem(lootItem, amount);

                if (success)
                {
                    // 1. 标记为已搜刮
                    hasLooted = true;

                    // 2. 隐藏按键提示
                    if (hintUI != null) hintUI.SetActive(false);

                    // 3. 弹出左侧获取提示
                    ShowNotification($"获得 {lootItem.itemName} x{amount}");

                    // 4. 东西拿到了，把尸体上的光点熄灭
                    if (interactableEffect != null) interactableEffect.SetActive(false);

                    // 👇 ==========================================
                    // 【绝对暴力解法】无视单例，全图强行搜索主角的大喇叭！
                    // ==========================================
                    PlayerController player = FindObjectOfType<PlayerController>();
                    if (player != null)
                    {
                        player.PlayInteractSound();
                    }
                    else
                    {
                        Debug.LogWarning("🔴 见鬼了，全地图都找不到 PlayerController！");
                    }
                    // 👆 ==========================================
                }
            }
        }
    }

    private void ShowNotification(string message)
    {
        if (notificationPanel != null && notificationText != null)
        {
            notificationText.text = message;
            notificationPanel.SetActive(true);

            // 取消之前可能存在的延时关闭任务，重新计时 2.5 秒后关闭
            CancelInvoke(nameof(HideNotification));
            Invoke(nameof(HideNotification), 2.5f);
        }
    }

    private void HideNotification()
    {
        if (notificationPanel != null)
        {
            notificationPanel.SetActive(false);
        }
    }

    private void OnTriggerEnter(Collider other)
    {
        // 如果已经被搜刮过了，就不再显示提示
        if (hasLooted) return;

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