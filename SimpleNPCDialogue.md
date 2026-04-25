using UnityEngine;
using TMPro;

public class SimpleNPCDialogue : MonoBehaviour
{
    [Header("UI 关联")]
    public GameObject hintUI;
    public GameObject dialoguePanel;
    public TMP_Text dialogueText;

    [Header("对话内容设置")]
    [TextArea(2, 5)]
    public string[] dialogueLines;

    [Header("模型朝向补偿")]
    [Tooltip("如果NPC转身后没正对你，请修改这个角度（通常是 90, -90, 或 180）")]
    public float yRotationOffset = 0f;

    private bool isPlayerInRange = false;
    private bool isTalking = false;
    private int currentLineIndex = 0;
    private PlayerController playerController;

    void Start()
    {
        if (hintUI != null) hintUI.SetActive(false);
        if (dialoguePanel != null) dialoguePanel.SetActive(false);
    }

    void Update()
    {
        if (isPlayerInRange && !isTalking)
        {
            if (hintUI != null) hintUI.SetActive(true);
            if (Input.GetKeyDown(KeyCode.F))
            {
                StartDialogue();
            }
        }
        else
        {
            if (hintUI != null) hintUI.SetActive(false);
        }

        if (isTalking && Input.GetMouseButtonDown(0))
        {
            NextLine();
        }
    }

    // 【终极必杀技：LateUpdate】在动画计算完之后，强行覆盖他的旋转！
    void LateUpdate()
    {
        if (isTalking && playerController != null)
        {
            Vector3 lookDirection = playerController.transform.position - transform.position;
            lookDirection.y = 0;

            if (lookDirection.sqrMagnitude > 0.01f)
            {
                // 算出原本应该对准你的旋转角度
                Quaternion baseRotation = Quaternion.LookRotation(lookDirection);
                // 加上我们手动设置的补偿偏转量
                Quaternion targetRotation = baseRotation * Quaternion.Euler(0, yRotationOffset, 0);
                // 持续平滑转向玩家
                transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, Time.deltaTime * 8f);
            }
        }
    }

    private void StartDialogue()
    {
        isTalking = true;
        currentLineIndex = 0;

        // 👇 ==========================================
        // 【新增】开始对话时，让主角播报交互音效
        // ==========================================
        if (playerController != null)
        {
            playerController.PlayInteractSound();
        }

        if (hintUI != null) hintUI.SetActive(false);
        if (dialoguePanel != null) dialoguePanel.SetActive(true);

        if (dialogueText != null && dialogueLines.Length > 0)
        {
            dialogueText.text = dialogueLines[currentLineIndex];
        }

        if (playerController != null)
        {
            playerController.moveInput = Vector2.zero;
            playerController.enabled = false;
            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;
        }
    }

    private void NextLine()
    {
        currentLineIndex++;
        if (currentLineIndex < dialogueLines.Length)
        {
            dialogueText.text = dialogueLines[currentLineIndex];

            // 👇 ==========================================
            // 【新增】成功翻页时，再次播放交互音效
            // ==========================================
            if (playerController != null)
            {
                playerController.PlayInteractSound();
            }
        }
        else
        {
            EndDialogue();
        }
    }

    private void EndDialogue()
    {
        isTalking = false;
        if (dialoguePanel != null) dialoguePanel.SetActive(false);

        if (playerController != null)
        {
            playerController.enabled = true;
            Cursor.lockState = CursorLockMode.Locked;
            Cursor.visible = false;
        }
    }

    private void OnTriggerEnter(Collider other)
    {
        PlayerController pc = other.GetComponentInParent<PlayerController>();
        if (pc != null) { isPlayerInRange = true; playerController = pc; }
    }

    private void OnTriggerExit(Collider other)
    {
        PlayerController pc = other.GetComponentInParent<PlayerController>();
        if (pc != null) { isPlayerInRange = false; playerController = null; }
    }
}