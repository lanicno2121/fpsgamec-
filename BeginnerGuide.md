using UnityEngine;

public class BeginnerGuide : MonoBehaviour
{
    [Header("UI 设置")]
    [Tooltip("要把哪个提示面板隐藏掉？(拖入包含提示文字的 Panel)")]
    public GameObject tutorialPanel;

    // 内部状态：记录每个按键是否被按过
    private bool pressedW = false;
    private bool pressedS = false;
    private bool pressedA = false;
    private bool pressedD = false;
    private bool pressedShift = false;
    private bool pressedSpace = false;

    // 是否已经完成引导
    private bool isCompleted = false;

    void Start()
    {
        // 游戏开始时，确保提示 UI 是显示状态的
        if (tutorialPanel != null)
        {
            tutorialPanel.SetActive(true);
        }
    }

    void Update()
    {
        // 如果已经完成了，就不再浪费性能去检测了
        if (isCompleted) return;

        // 监听按键按下 (只要按过一次，就标记为 true)
        if (Input.GetKeyDown(KeyCode.W)) pressedW = true;
        if (Input.GetKeyDown(KeyCode.S)) pressedS = true;
        if (Input.GetKeyDown(KeyCode.A)) pressedA = true;
        if (Input.GetKeyDown(KeyCode.D)) pressedD = true;
        if (Input.GetKeyDown(KeyCode.Space)) pressedSpace = true;

        // 冲刺键通常是左 Shift
        if (Input.GetKeyDown(KeyCode.LeftShift) || Input.GetKeyDown(KeyCode.RightShift))
        {
            pressedShift = true;
        }

        // 判断：如果这 6 个条件全部达成了 (全都变成了 true)
        if (pressedW && pressedS && pressedA && pressedD && pressedShift && pressedSpace)
        {
            FinishTutorial();
        }
    }

    private void FinishTutorial()
    {
        isCompleted = true;

        // 隐藏提示 UI
        if (tutorialPanel != null)
        {
            tutorialPanel.SetActive(false);
        }

        // 任务完成，把这个脚本自己销毁掉，释放后台性能
        Destroy(this);
    }
}