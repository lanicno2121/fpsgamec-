using UnityEngine;
using TMPro; // 引入 TextMeshPro 命名空间

public class ItemPickup : MonoBehaviour
{
    [Header("物品设置")]
    [Tooltip("这个掉落物代表什么物品？(把做好的 ItemData 拖进来)")]
    public ItemData itemData;

    [Tooltip("捡起来能获得几个？(默认是 1)")]
    public int amount = 1;

    [Header("UI 提示设置")]
    [Tooltip("把做好的 '按 F 拾取' UI 拖到这里（可以直接用你之前做武器箱的那个 UI）")]
    public GameObject hintUI;

    [Tooltip("屏幕左侧的通知容器 (和尸体上用的是同一个 Panel)")]
    public GameObject notificationPanel;
    [Tooltip("用来显示 '获得物品' 的文字组件")]
    public TMP_Text notificationText;

    // 内部状态
    private bool isPlayerInRange = false;
    private bool hasPickedUp = false; // 新增：防止重复拾取
    private InventoryManager cachedInventory;

    void Start()
    {
        // 游戏开始时，先隐藏按键提示和左下角提示
        if (hintUI != null) hintUI.SetActive(false);
        if (notificationPanel != null) notificationPanel.SetActive(false);
    }

    void Update()
    {
        // 如果玩家在范围内、没有被捡过、并且按下了 F 键
        if (!hasPickedUp && isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            if (cachedInventory != null && itemData != null)
            {
                // 【核心逻辑】尝试把东西塞进玩家的背包
                bool success = cachedInventory.AddItem(itemData, amount);

                if (success)
                {
                    Debug.Log($"【拾取成功】获得 {itemData.itemName} x{amount}");

                    // 标记为已拾取，防止玩家狂按F键触发多次
                    hasPickedUp = true;

                    // 👇 ==========================================
                    // 【绝对暴力解法】无视单例，全图强行搜索主角的大喇叭并播放拾取声！
                    // ==========================================
                    PlayerController player = FindObjectOfType<PlayerController>();
                    if (player != null)
                    {
                        player.PlayInteractSound();
                    }
                    // 👆 ==========================================

                    // 1. 隐藏 3D 悬浮提示
                    if (hintUI != null) hintUI.SetActive(false);

                    // 2. 弹出左下角获取提示
                    ShowNotification($"获得 {itemData.itemName} x{amount}");

                    // 3. 【障眼法】隐藏地上的模型和碰撞体 (假装它消失了)
                    // 这样即使还没 Destroy，玩家也看不见、碰不到它了
                    foreach (var renderer in GetComponentsInChildren<Renderer>())
                        renderer.enabled = false;
                    foreach (var collider in GetComponentsInChildren<Collider>())
                        collider.enabled = false;

                    // 4. 延迟 3 秒后再彻底销毁物体 (留出时间让左下角的 Invoke 执行完毕)
                    Destroy(gameObject, 3f);
                }
                else
                {
                    // 背包塞满了
                    ShowNotification("背包已满！");
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
        if (hasPickedUp) return; // 如果已经捡了，就不触发了

        // 智能查找 InventoryManager (主角身上的背包管家)
        var inv = other.GetComponent<InventoryManager>();
        if (inv == null) inv = other.GetComponentInParent<InventoryManager>();

        if (inv != null)
        {
            // 玩家进入范围，记录下这位玩家，并显示提示
            isPlayerInRange = true;
            cachedInventory = inv;

            if (hintUI != null) hintUI.SetActive(true);
        }
    }

    private void OnTriggerExit(Collider other)
    {
        // 玩家离开时，清空记录并隐藏提示
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