using UnityEngine;
using TMPro;

public class SlideTutorialTrigger : MonoBehaviour
{
    [Header("UI 设置")]
    [Tooltip("拖入你之前做的顶部提示 Panel")]
    public GameObject tutorialPanel;
    [Tooltip("拖入 Panel 里面的文字组件 (用来修改文字)")]
    public TMP_Text tutorialText;

    [Header("提示内容")]
    [Tooltip("踩中触发器后要显示的文字")]
    public string tutorialMessage = "新手提示：在冲刺时按 Ctrl 滑铲";

    private bool isListening = false; // 是否正在监听玩家按键
    private bool isCompleted = false; // 是否已经完成

    void Update()
    {
        // 如果玩家踩到了触发器，就开始监听按键
        if (isListening && !isCompleted)
        {
            // 监听左 Ctrl 键 (如果你的滑铲是 C 键，可以在这里改成 KeyCode.C)
            if (Input.GetKeyDown(KeyCode.LeftControl))
            {
                FinishTutorial();
            }
        }
    }

    private void OnTriggerEnter(Collider other)
    {
        // 如果已经完成，或者正在监听，就不再重复触发
        if (isCompleted || isListening) return;

        // 检查踩上来的是不是玩家
        if (other.CompareTag("Player") || other.GetComponentInParent<PlayerController>() != null)
        {
            isListening = true;

            // 1. 修改文字内容
            if (tutorialText != null)
            {
                tutorialText.text = tutorialMessage;
            }

            // 2. 显示提示面板
            if (tutorialPanel != null)
            {
                tutorialPanel.SetActive(true);
            }
        }
    }

    private void FinishTutorial()
    {
        isCompleted = true;

        // 隐藏 UI
        if (tutorialPanel != null)
        {
            tutorialPanel.SetActive(false);
        }

        // 任务完成，把这个隐形的触发器物体直接销毁，保持场景干净
        Destroy(gameObject);
    }
}