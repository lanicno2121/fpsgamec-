using UnityEngine;
using UnityEngine.UI;

public class DynamicCrosshair : MonoBehaviour
{
    [Header("核心引用")]
    public PlayerController playerController;
    public CanvasGroup crosshairGroup; // 拖入 CrossHair 自己

    [Header("准星的四个部分 (拖入对应的子物体)")]
    public RectTransform TopLine;
    public RectTransform BottomLine;
    public RectTransform LeftLine;
    public RectTransform RightLine;

    [Header("扩散参数 (Spread Settings)")]
    public float baseSpread = 10f;       // 静止时，线条离中心的距离
    public float walkSpreadFactor = 20f; // 走路时增加的距离
    public float runSpreadFactor = 40f;  // 跑步时增加的距离
    public float fireSpreadFactor = 30f; // 开火时增加的距离

    [Header("平滑设置")]
    public float smoothSpeed = 10f;      // 扩散/回缩的灵敏度
    public float fadeSpeed = 15f;        // 瞄准时消失的速度

    // 内部变量
    private float currentSpread;
    private float targetSpread;

    void Start()
    {
        if (playerController == null)
            playerController = FindObjectOfType<PlayerController>();

        // 如果没手动拖，尝试自动获取 CanvasGroup
        if (crosshairGroup == null)
            crosshairGroup = GetComponent<CanvasGroup>();

        currentSpread = baseSpread;
    }

    void Update()
    {
        if (playerController == null) return;

        // ============================
        // 1. 处理显隐 (机瞄时消失)
        // ============================
        bool isAiming = playerController.isAiming;

        // 【唯一修改处】：如果是手枪，强制打断隐藏指令
        if (isAiming && playerController.currentPlayerModel != null)
        {
            // 自动寻找 WeaponManager
            WeaponManager wm = playerController.currentPlayerModel.GetComponent<WeaponManager>();
            if (wm == null) wm = playerController.GetComponentInChildren<WeaponManager>();

            if (wm != null && !wm.IsUnarmed)
            {
                // 如果当前武器使用了手枪动画层，就不隐藏准星
                if (wm.allWeapons[wm.CurrentIndex].usePistolLayer)
                {
                    isAiming = false;
                }
            }
        }

        if (isAiming)
        {
            // 瞄准时：迅速变透明 (步枪专享)
            crosshairGroup.alpha = Mathf.Lerp(crosshairGroup.alpha, 0f, Time.deltaTime * fadeSpeed);
        }
        else
        {
            // 腰射或手枪瞄准时：迅速变回不透明，保持显示
            crosshairGroup.alpha = Mathf.Lerp(crosshairGroup.alpha, 1f, Time.deltaTime * fadeSpeed);
        }

        // 如果完全透明了，就不算位置了，省性能
        if (crosshairGroup.alpha < 0.01f) return;

        // ============================
        // 2. 计算目标扩散值
        // ============================
        targetSpread = baseSpread;

        // 移动带来的扩散
        float speed = playerController.moveInput.magnitude; // 0~1
        if (playerController.isSprint)
        {
            targetSpread += speed * runSpreadFactor;
        }
        else
        {
            targetSpread += speed * walkSpreadFactor;
        }

        // 开火带来的扩散
        if (playerController.isFire)
        {
            targetSpread += fireSpreadFactor;
        }

        // ============================
        // 3. 应用扩散 (Lerp 平滑移动)
        // ============================
        // 让当前的 spread 慢慢追上目标的 spread
        currentSpread = Mathf.Lerp(currentSpread, targetSpread, Time.deltaTime * smoothSpeed);

        UpdateCrosshairPosition();
    }

    void UpdateCrosshairPosition()
    {
        // 这里的逻辑是：修改 anchoredPosition 让它们往四个方向跑

        if (TopLine) TopLine.anchoredPosition = new Vector2(0, currentSpread);  // 往上
        if (BottomLine) BottomLine.anchoredPosition = new Vector2(0, -currentSpread); // 往下
        if (LeftLine) LeftLine.anchoredPosition = new Vector2(-currentSpread, 0); // 往左
        if (RightLine) RightLine.anchoredPosition = new Vector2(currentSpread, 0);  // 往右
    }
}