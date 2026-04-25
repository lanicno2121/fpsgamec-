using UnityEngine;
using TMPro; // 引入 UI 命名空间

public class AmmoBox : MonoBehaviour
{
    [Header("弹药设置")]
    [Tooltip("捡起后给多少发备用子弹")]
    public int ammoAmount = 30;

    [Header("UI 提示设置")]
    [Tooltip("悬浮的 '按F拾取' 提示UI")]
    public GameObject hintUI;

    [Tooltip("屏幕左侧的通知容器 (和普通物品用同一个)")]
    public GameObject notificationPanel;
    [Tooltip("用来显示 '获得弹药' 的文字组件")]
    public TMP_Text notificationText;

    // 内部状态
    private bool isPlayerInRange = false;
    private bool hasPickedUp = false; // 防止狂按F重复拾取
    private Collider playerCollider;

    void Start()
    {
        if (hintUI != null) hintUI.SetActive(false);
        if (notificationPanel != null) notificationPanel.SetActive(false);
    }

    void Update()
    {
        if (!hasPickedUp && isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            // 👇 ==========================================
            // 【绝对暴力解法】玩家按F拾取弹药，立刻播放交互声音！
            // ==========================================
            PlayerController player = FindObjectOfType<PlayerController>();
            if (player != null)
            {
                player.PlayInteractSound();
            }
            // 👆 ==========================================

            if (playerCollider != null)
            {
                GunAmmoSystem activeGun = playerCollider.GetComponentInChildren<GunAmmoSystem>();

                if (activeGun != null)
                {
                    // 1. 给枪加子弹
                    activeGun.AddReserveAmmo(ammoAmount);

                    // 2. 标记为已拾取
                    hasPickedUp = true;

                    // 3. 隐藏悬浮 F 提示
                    if (hintUI != null) hintUI.SetActive(false);

                    // 4. 弹出左下角提示框！
                    ShowNotification($"获得 备用弹药 x{ammoAmount}");

                    // 5. 【障眼法】隐藏模型和碰撞体，假装它消失了
                    foreach (var renderer in GetComponentsInChildren<Renderer>()) renderer.enabled = false;
                    foreach (var collider in GetComponentsInChildren<Collider>()) collider.enabled = false;

                    // 6. 留出 3 秒钟让 UI 倒计时跑完，然后再真正销毁
                    Destroy(gameObject, 3f);
                }
                else
                {
                    Debug.LogWarning("当前没有手持武器，无法补充弹药！");
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
        if (other.CompareTag("Player") || other.GetComponentInParent<PlayerController>() != null)
        {
            isPlayerInRange = true;
            playerCollider = other;
            if (hintUI != null) hintUI.SetActive(true);
        }
    }

    private void OnTriggerExit(Collider other)
    {
        if (other.CompareTag("Player") || other.GetComponentInParent<PlayerController>() != null)
        {
            isPlayerInRange = false;
            playerCollider = null;
            if (hintUI != null) hintUI.SetActive(false);
        }
    }
}