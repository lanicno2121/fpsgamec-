	using UnityEngine;
using TMPro;

public class BrokenDoor : MonoBehaviour
{
    [Header("UI 提示设置")]
    [Tooltip("悬浮在门上的 '按F交互' 提示UI")]
    public GameObject hintUI;

    [Tooltip("屏幕正中间的通知容器 (例如一个Panel)")]
    public GameObject centerNotificationPanel;
    [Tooltip("用来显示 '已损坏无法打开' 的文字组件")]
    public TMP_Text centerNotificationText;

    [Header("交互设置")]
    [Tooltip("按下F后屏幕中间显示的文字")]
    public string brokenMessage = "已损坏，无法打开";
    [Tooltip("提示文字在屏幕上停留几秒？")]
    public float messageDuration = 2.0f;

    [Header("视觉提示 (可选)")]
    [Tooltip("门上的闪烁提示灯光 (把你的原神光点拖进来)")]
    public GameObject interactableEffect;
    [Tooltip("交互后是否熄灭提示光？(勾选后，摸过一次光就灭了，且不再提示按F)")]
    public bool turnOffEffectAfterInteract = true;

    // 👇 ==========================================
    // 【新增】专属交互音效槽位
    // ==========================================
    [Header("专属交互音效 (可选)")]
    [Tooltip("如果不填，就会播放主角身上的默认音效")]
    public AudioClip specificInteractSound;

    private bool isPlayerInRange = false;
    private bool hasChecked = false; // 是否已经检查过这扇门

    void Start()
    {
        // 游戏开始时隐藏UI
        if (hintUI != null) hintUI.SetActive(false);
        if (centerNotificationPanel != null) centerNotificationPanel.SetActive(false);

        if (interactableEffect != null) interactableEffect.SetActive(true);
    }

    void Update()
    {
        // 如果玩家在范围内，且按下 F 键
        if (isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            PlayerController player = FindObjectOfType<PlayerController>();
            if (player != null)
            {
                // 👇 ==========================================
                // 【升级】把专属音效传给主角的大喇叭！
                // 如果 specificInteractSound 是空的，喇叭会自动播默认声音
                // ==========================================
                player.PlayInteractSound(specificInteractSound);
            }

            ShowBrokenMessage();
        }
    }

    private void ShowBrokenMessage()
    {
        // 1. 显示屏幕中间的文字
        if (centerNotificationPanel != null && centerNotificationText != null)
        {
            centerNotificationText.text = brokenMessage;
            centerNotificationPanel.SetActive(true);

            // 取消之前的定时关闭任务，重新开始倒计时
            CancelInvoke(nameof(HideMessage));
            Invoke(nameof(HideMessage), messageDuration);
        }

        // 2. 处理交互后的状态
        if (turnOffEffectAfterInteract)
        {
            hasChecked = true;
            isPlayerInRange = false; // 强制脱离交互状态

            if (hintUI != null) hintUI.SetActive(false);
            if (interactableEffect != null) interactableEffect.SetActive(false);
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
        // 如果设置为摸一次就熄灭，且已经摸过了，就不再弹出 F 提示
        if (hasChecked && turnOffEffectAfterInteract) return;

        // 识别是否是玩家 (通过标签或找 PlayerController)
        if (other.CompareTag("Player") || other.GetComponentInParent<PlayerController>() != null)
        {
            isPlayerInRange = true;
            if (hintUI != null) hintUI.SetActive(true);
        }
    }

    private void OnTriggerExit(Collider other)
    {
        if (other.CompareTag("Player") || other.GetComponentInParent<PlayerController>() != null)
        {
            isPlayerInRange = false;
            if (hintUI != null) hintUI.SetActive(false);
        }
    }
}