using UnityEngine;

public class FPSWeaponLogic : MonoBehaviour
{
    [Header("核心引用")]
    public PlayerController playerController;

    [Header("1. 走路起伏 (Bobbing)")]
    public float walkingBobbingSpeed = 14f;
    public float walkingBobbingAmount = 0.05f;
    public float sprintBobbingAmount = 0.1f;

    [Header("2. 静止呼吸 (Breathing)")]
    public float idleBobbingSpeed = 2f;
    public float idleBobbingAmount = 0.005f;

    [Header("3. 鼠标惯性 (Sway)")]
    public float swayAmount = 0.02f;
    public float maxSwayAmount = 0.06f;
    public float swaySmooth = 6f;

    [Header("4. 开火稳定性 (新功能)")]
    [Tooltip("开火后多久恢复呼吸晃动 (数值越大恢复越快)")]
    public float stabilityRecoverySpeed = 2f;

    // 内部变量
    private float defaultYPos = 0;
    private float defaultXPos = 0;
    private float timer = 0;
    private Vector3 initialLocalPos;

    // 【核心变量】稳定性系数：0=完全静止(开火时)，1=正常晃动
    private float currentStability = 1f;

    void Start()
    {
        if (playerController == null)
            playerController = FindObjectOfType<PlayerController>();

        initialLocalPos = transform.localPosition;
        defaultYPos = transform.localPosition.y;
        defaultXPos = transform.localPosition.x;
    }

    void Update()
    {
        if (playerController == null || !playerController.isAiming)
        {
            // 可选：非瞄准时让稳定性瞬间恢复，这样下次开镜就是满状态
            currentStability = 1f;
            return;
        }

        // ==========================================
        // 1. 恢复稳定性 (从 0 慢慢变回 1)
        // ==========================================
        if (currentStability < 1f)
        {
            currentStability += Time.deltaTime * stabilityRecoverySpeed;
            if (currentStability > 1f) currentStability = 1f;
        }

        // ==========================================
        // 2. 计算晃动 (计算结果 * 稳定性系数)
        // ==========================================

        // 如果 currentStability 是 0 (正在开火)，这里算出来的结果都会变成 0
        Vector3 swayOffset = CalculateSway() * currentStability;
        Vector3 bobbingOffset = CalculateBobbing() * currentStability;

        // 3. 应用最终位置
        // 最终位置 = 初始位置 + (晃动 * 系数)
        Vector3 targetPos = initialLocalPos + swayOffset + bobbingOffset;

        transform.localPosition = Vector3.Lerp(transform.localPosition, targetPos, Time.deltaTime * 10f);
    }

    // === 公共接口：给 PlayerWeapon 调用 ===
    public void TriggerStability()
    {
        // 开火瞬间，把晃动权重设为 0 (强制稳住)
        currentStability = 0f;
    }

    Vector3 CalculateSway()
    {
        float mouseX = -Input.GetAxis("Mouse X") * swayAmount;
        float mouseY = -Input.GetAxis("Mouse Y") * swayAmount;

        mouseX = Mathf.Clamp(mouseX, -maxSwayAmount, maxSwayAmount);
        mouseY = Mathf.Clamp(mouseY, -maxSwayAmount, maxSwayAmount);

        return new Vector3(mouseX, mouseY, 0);
    }

    Vector3 CalculateBobbing()
    {
        Vector3 offset = Vector3.zero;
        float horizontal = Input.GetAxis("Horizontal");
        float vertical = Input.GetAxis("Vertical");
        bool isMoving = Mathf.Abs(horizontal) > 0.1f || Mathf.Abs(vertical) > 0.1f;

        if (isMoving)
        {
            timer += Time.deltaTime * walkingBobbingSpeed;
            float currentAmount = playerController.isSprint ? sprintBobbingAmount : walkingBobbingAmount;
            offset.y = Mathf.Sin(timer) * currentAmount;
            offset.x = Mathf.Cos(timer / 2) * currentAmount * 2;
        }
        else
        {
            timer += Time.deltaTime * idleBobbingSpeed;
            offset.y = Mathf.Sin(timer) * idleBobbingAmount;
            offset.x = Mathf.Cos(timer) * idleBobbingAmount;
        }

        return offset;
    }
}