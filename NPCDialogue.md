using UnityEngine;
using TMPro;
using UnityEngine.AI;

public class NPCDialogue : MonoBehaviour
{
    [Header("场景与角色关联")]
    public GameObject nearbyEnemy;
    public Animator npcAnimator;

    [Header("UI 关联 (对话系统)")]
    public GameObject hintUI; // 原本的按F提示
    public GameObject dialoguePanel;
    public TMP_Text dialogueText;

    [Tooltip("拖入NPC身上那个黄色的任务视觉标识")]
    public GameObject visualIndicator; // 【新增】用来存放黄色的视觉标识

    [Header("对话内容设置")]
    [TextArea(2, 5)]
    public string[] dialogueLines;

    [Header("模型朝向补偿")]
    public float yRotationOffset = 0f;

    [Header("任务物品发放 (对接背包系统)")]
    public ItemData finalKeyItem;
    public int keyAmount = 1;

    [Header("获得物品 UI 提示 (对接左下角弹窗)")]
    public GameObject notificationPanel;
    public TMP_Text notificationText;

    [Header("小弟跟随设置")]
    [Tooltip("NPC 跟着你跑的速度")]
    public float followSpeed = 4.0f;
    [Tooltip("离你多远时停下脚步 (防止贴脸)")]
    public float stopDistance = 2.5f;

    // 内部状态
    private bool isPlayerInRange = false;
    private bool isTalking = false;
    private int currentLineIndex = 0;
    private bool hasGivenKey = false;

    private PlayerController playerController;
    private Transform playerTransform;
    private Transform actualBodyTransform; // 【上一轮修复】用来锁定天上的真实肉身

    // 导航代理组件
    private NavMeshAgent agent;

    void Start()
    {
        if (hintUI != null) hintUI.SetActive(false);
        if (dialoguePanel != null) dialoguePanel.SetActive(false);
        if (notificationPanel != null) notificationPanel.SetActive(false);

        agent = GetComponent<NavMeshAgent>();
        if (agent != null)
        {
            agent.speed = followSpeed;
            agent.stoppingDistance = stopDistance;
            agent.updateRotation = false;
        }
    }

    void Update()
    {
        if (!hasGivenKey)
        {
            bool isEnemyAlive = nearbyEnemy != null && nearbyEnemy.activeInHierarchy;
            if (npcAnimator != null) npcAnimator.SetBool("IsFear", isEnemyAlive);

            if (isPlayerInRange && !isTalking)
            {
                if (hintUI != null) hintUI.SetActive(true);
                if (!isEnemyAlive && Input.GetKeyDown(KeyCode.F)) StartDialogue();
            }
            else
            {
                if (hintUI != null) hintUI.SetActive(false);
            }
        }
        else
        {
            if (hintUI != null) hintUI.SetActive(false);
            if (npcAnimator != null) npcAnimator.SetBool("IsFear", false);

            // 【已修复】现在追踪的是你的真实肉身 (actualBodyTransform)
            if (actualBodyTransform != null && agent != null)
            {
                float distance = Vector3.Distance(transform.position, actualBodyTransform.position);

                if (distance > stopDistance)
                {
                    agent.isStopped = false;
                    agent.SetDestination(actualBodyTransform.position);

                    bool isActuallyMoving = agent.velocity.sqrMagnitude > 0.1f;
                    if (npcAnimator != null) npcAnimator.SetBool("IsMoving", isActuallyMoving);

                    Vector3 moveDirection = agent.velocity;
                    moveDirection.y = 0;

                    if (moveDirection.sqrMagnitude > 0.1f)
                    {
                        Quaternion baseRotation = Quaternion.LookRotation(moveDirection);
                        Quaternion targetRotation = baseRotation * Quaternion.Euler(0, yRotationOffset, 0);
                        transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, Time.deltaTime * 8f);
                    }
                }
                else
                {
                    agent.isStopped = true;
                    if (npcAnimator != null) npcAnimator.SetBool("IsMoving", false);
                }
            }
        }

        if (isTalking && Input.GetMouseButtonDown(0)) NextLine();
    }

    void LateUpdate()
    {
        if (isTalking && actualBodyTransform != null)
        {
            Vector3 lookDirection = actualBodyTransform.position - transform.position;
            lookDirection.y = 0;

            if (lookDirection.sqrMagnitude > 0.01f)
            {
                Quaternion baseRotation = Quaternion.LookRotation(lookDirection);
                Quaternion targetRotation = baseRotation * Quaternion.Euler(0, yRotationOffset, 0);
                transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, Time.deltaTime * 8f);
            }
        }
    }

    private void StartDialogue()
    {
        isTalking = true;
        currentLineIndex = 0;

        // 👇 ==========================================
        // 【新增】开始对话时，让主角播报交互音效！
        // ==========================================
        if (playerController != null)
        {
            playerController.PlayInteractSound();
        }

        if (hintUI != null) hintUI.SetActive(false);
        if (dialoguePanel != null) dialoguePanel.SetActive(true);
        if (dialogueText != null && dialogueLines.Length > 0) dialogueText.text = dialogueLines[currentLineIndex];

        if (playerController != null)
        {
            playerController.moveInput = Vector2.zero;
            playerController.enabled = false;
            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;

            if (agent != null) agent.isStopped = true;
        }
    }

    private void NextLine()
    {
        currentLineIndex++;
        if (currentLineIndex < dialogueLines.Length)
        {
            dialogueText.text = dialogueLines[currentLineIndex];

            // 👇 ==========================================
            // 【新增】成功翻页时，再次播放交互音效！
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

            if (!hasGivenKey && finalKeyItem != null)
            {
                InventoryManager inv = playerController.GetComponent<InventoryManager>();
                if (inv == null) inv = playerController.GetComponentInParent<InventoryManager>();

                if (inv != null)
                {
                    inv.AddItem(finalKeyItem, keyAmount);

                    hasGivenKey = true;

                    // ==========================================
                    // 【新增功能】永远关掉那个黄色的视觉标识！
                    // ==========================================
                    if (visualIndicator != null)
                    {
                        visualIndicator.SetActive(false);
                    }

                    if (notificationPanel != null && notificationText != null)
                    {
                        notificationText.text = "已获得: " + finalKeyItem.itemName;
                        notificationPanel.SetActive(true);
                        Invoke(nameof(HideNotification), 3f);
                    }
                }
            }
        }
    }

    private void HideNotification()
    {
        if (notificationPanel != null) notificationPanel.SetActive(false);
    }

    private void OnTriggerEnter(Collider other)
    {
        PlayerController pc = other.GetComponentInParent<PlayerController>();
        if (pc != null)
        {
            isPlayerInRange = true;
            playerController = pc;
            // 【已修复】抓取真实的肉身坐标
            actualBodyTransform = other.transform;
        }
    }

    private void OnTriggerExit(Collider other)
    {
        PlayerController pc = other.GetComponentInParent<PlayerController>();
        if (pc != null) { isPlayerInRange = false; playerController = null; actualBodyTransform = null; }
    }
}