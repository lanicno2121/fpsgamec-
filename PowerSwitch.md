using UnityEngine;
using UnityEngine.Events;
using TMPro;

public class PowerSwitch : MonoBehaviour
{
    [Header("交互设置")]
    public string switchName = "主电源";
    public GameObject hintUI; // 门上的 F 提示

    [Header("UI 通知")]
    public GameObject notificationPanel;
    public TMP_Text notificationText;

    [Header("视觉效果")]
    public GameObject powerOnVFX; // 电力开启时的火花或灯光
    public bool disableAfterUse = true; // 开启后是否就不能再按了

    // 👇 ==========================================
    // 【新增】专属交互音效槽位
    // ==========================================
    [Header("专属交互音效 (可选)")]
    [Tooltip("如果不填，会播放默认音效。建议放个：沉重的金属拉闸声")]
    public AudioClip specificInteractSound;

    [Header("【核心】开启后触发的事件")]
    [Tooltip("在这里拖入你要触发的：大门开启、运镜动画、灯光变亮等")]
    public UnityEvent onPowerOn;

    private bool isPlayerInRange = false;
    private bool isActivated = false;

    void Start()
    {
        if (hintUI != null) hintUI.SetActive(false);
    }

    void Update()
    {
        if (!isActivated && isPlayerInRange && Input.GetKeyDown(KeyCode.F))
        {
            PlayerController player = FindObjectOfType<PlayerController>();
            if (player != null)
            {
                // 👇 ==========================================
                // 【升级】把专属音效传给喇叭！
                // ==========================================
                player.PlayInteractSound(specificInteractSound);
            }

            ActivatePower();
        }
    }

    private void ActivatePower()
    {
        isActivated = true;
        Debug.Log($"【{switchName}】已开启！");

        // 1. 弹出左下角提示
        if (notificationPanel != null && notificationText != null)
        {
            notificationText.text = $"{switchName} 已启动";
            notificationPanel.SetActive(true);
            Invoke(nameof(HideNotification), 2.5f);
        }

        // 2. 播放视觉特效
        if (powerOnVFX != null) powerOnVFX.SetActive(true);

        // 3. 执行所有连线好的事件（运镜、开门等）
        onPowerOn?.Invoke();

        // 4. 隐藏交互提示并禁用脚本
        if (hintUI != null) hintUI.SetActive(false);
        if (disableAfterUse) this.enabled = false;
    }

    private void HideNotification() => notificationPanel.SetActive(false);

    private void OnTriggerEnter(Collider other)
    {
        if (isActivated) return;
        if (other.CompareTag("Player") || other.GetComponentInParent<PlayerController>() != null)
        {
            isPlayerInRange = true;
            if (hintUI != null) hintUI.SetActive(true);
        }
    }

    private void OnTriggerExit(Collider other)
    {
        isPlayerInRange = false;
        if (hintUI != null) hintUI.SetActive(false);
    }
}