using UnityEngine;

public class FinalDoor : MonoBehaviour
{
    [Header("大门设置")]
    [Tooltip("拖入那把能开这扇门的钥匙的 ItemData")]
    public ItemData requiredKey;
    public Vector3 moveOffset = new Vector3(0, 5f, 0);
    public float moveDuration = 2.0f;

    [Header("UI 提示")]
    public GameObject hintUI;

    // 👇 ==========================================
    // 【新增】专属交互音效槽位
    // ==========================================
    [Header("专属交互音效 (可选)")]
    [Tooltip("如果不填，就会播放默认音效。建议放个：沉重的开锁声或金属转动声")]
    public AudioClip specificInteractSound;

    private bool isPlayerNear = false;
    private bool isOpen = false;
    private PlayerController player;
    private Vector3 startPos;

    void Start()
    {
        startPos = transform.position;
        if (hintUI != null) hintUI.SetActive(false);
    }

    void Update()
    {
        if (isPlayerNear && !isOpen)
        {
            if (hintUI != null) hintUI.SetActive(true);

            if (Input.GetKeyDown(KeyCode.F))
            {
                if (player != null)
                {
                    // 👇 ==========================================
                    // 【升级】把专属音效传给喇叭！
                    // ==========================================
                    player.PlayInteractSound(specificInteractSound);
                }

                if (player != null && requiredKey != null)
                {
                    InventoryManager inv = player.GetComponent<InventoryManager>();
                    if (inv == null) inv = player.GetComponentInParent<InventoryManager>();

                    if (inv != null)
                    {
                        if (inv.HasItem(requiredKey))
                        {
                            Debug.Log("【系统】钥匙验证通过，大门已解锁！");
                            if (hintUI != null) hintUI.SetActive(false);

                            // 开启协程，播放开门动画
                            StartCoroutine(OpenDoorRoutine());
                        }
                        else
                        {
                            Debug.Log("【系统】提示：背包里没有对应的钥匙，无法开门！");
                        }
                    }
                }
            }
        }
        else
        {
            // 玩家走远了或者门已经开了，隐藏按F提示
            if (hintUI != null) hintUI.SetActive(false);
        }
    }

    private System.Collections.IEnumerator OpenDoorRoutine()
    {
        isOpen = true;
        float timer = 0f;
        Vector3 targetPos = startPos + moveOffset;

        while (timer < moveDuration)
        {
            transform.position = Vector3.Lerp(startPos, targetPos, timer / moveDuration);
            timer += Time.deltaTime;
            yield return null;
        }

        transform.position = targetPos;
    }

    private void OnTriggerEnter(Collider other)
    {
        PlayerController pc = other.GetComponentInParent<PlayerController>();
        if (pc != null) { isPlayerNear = true; player = pc; }
    }

    private void OnTriggerExit(Collider other)
    {
        PlayerController pc = other.GetComponentInParent<PlayerController>();
        if (pc != null) { isPlayerNear = false; player = null; }
    }
}