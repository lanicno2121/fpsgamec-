using UnityEngine;
using TMPro; // 引入 UI 命名空间

public class WeaponPickup : MonoBehaviour
{
    [Header("武器箱配置")]
    [Tooltip("装的是几号武器？(0=步枪, 1=手枪)")]
    public int weaponIndexToUnlock = 0;
    public string weaponName = "突击步枪";

    [Header("UI 提示设置")]
    [Tooltip("悬浮的 '按F拾取' 提示UI")]
    public GameObject hintUI;

    [Tooltip("屏幕左侧的通知容器 (和普通物品用同一个)")]
    public GameObject notificationPanel;
    [Tooltip("用来显示 '获得武器' 的文字组件")]
    public TMP_Text notificationText;

    // 👇 ==========================================
    // 【新增】专属交互音效槽位
    // ==========================================
    [Header("专属交互音效 (可选)")]
    [Tooltip("如果不填，会播放默认音效。建议放个：清脆的上膛声或金属碰撞声")]
    public AudioClip specificInteractSound;

    // 内部状态
    private bool isPlayerInRange = false;
    private bool hasPickedUp = false;
    private WeaponManager cachedManager;

    void Start()
    {
        if (hintUI != null) hintUI.SetActive(false);
        if (notificationPanel != null) notificationPanel.SetActive(false);
    }

    void Update()
    {
        if (!hasPickedUp && isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            PlayerController player = FindObjectOfType<PlayerController>();
            if (player != null)
            {
                // 👇 ==========================================
                // 【升级】把专属音效传给喇叭！
                // ==========================================
                player.PlayInteractSound(specificInteractSound);
            }

            if (cachedManager != null)
            {
                // 1. 装备武器
                cachedManager.EquipWeapon(weaponIndexToUnlock);

                // 2. 标记已拾取
                hasPickedUp = true;

                // 3. 隐藏 F 提示
                if (hintUI != null) hintUI.SetActive(false);

                // 4. 弹出左下角提示框！
                ShowNotification($"获得武器：{weaponName}");

                // 5. 【障眼法】隐藏模型和碰撞体
                foreach (var renderer in GetComponentsInChildren<Renderer>()) renderer.enabled = false;
                foreach (var collider in GetComponentsInChildren<Collider>()) collider.enabled = false;

                // 6. 延迟销毁
                Destroy(gameObject, 3f);
            }
        }
    }

    private void ShowNotification(string message)
    {
        if (notificationPanel != null && notificationText != null)
        {
            notificationText.text = message;
            notificationPanel.SetActive(true);
            CancelInvoke(nameof(HideNotification));
            Invoke(nameof(HideNotification), 2.5f);
        }
    }

    private void HideNotification()
    {
        if (notificationPanel != null) notificationPanel.SetActive(false);
    }

    private void OnTriggerEnter(Collider other)
    {
        if (hasPickedUp) return;
        var manager = other.GetComponent<WeaponManager>();
        if (manager == null) manager = other.GetComponentInParent<WeaponManager>();

        if (manager != null)
        {
            isPlayerInRange = true;
            cachedManager = manager;
            if (hintUI != null) hintUI.SetActive(true);
        }
    }

    private void OnTriggerExit(Collider other)
    {
        var manager = other.GetComponent<WeaponManager>();
        if (manager == null) manager = other.GetComponentInParent<WeaponManager>();

        if (manager != null && manager == cachedManager)
        {
            isPlayerInRange = false;
            cachedManager = null;
            if (hintUI != null) hintUI.SetActive(false);
        }
    }
}