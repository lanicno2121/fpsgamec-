using UnityEngine;

// 保证在 PlayerController (Order 100) 之后执行，防止被覆盖
[DefaultExecutionOrder(200)]
public class ShootingPoseAdjuster : MonoBehaviour
{
    [Header("核心引用")]
    public PlayerController playerController;
    [Tooltip("你的枪械根物体 (Visual Model的父物体)")]
    public Transform weaponRoot;
    [Tooltip("你的左手 IK Target")]
    public Transform leftHandTarget;

    [Header("枪的偏移 (开火时枪怎么动?)")]
    [Tooltip("枪向前/向后移动的距离 (Z轴正数通常是前)")]
    public Vector3 gunRecoilOffset = new Vector3(0, 0, 0.15f);

    [Header("左手的额外微调 (开火时左手怎么动?)")]
    [Tooltip("左手向左/向右移动的距离 (X轴负数通常是左)")]
    public Vector3 leftHandFireOffset = new Vector3(-0.05f, 0, 0);

    [Header("参数")]
    public float smoothSpeed = 20f;

    // 内部变量：用于平滑过渡
    private Vector3 currentGunOffset = Vector3.zero;
    private Vector3 currentLeftHandOffset = Vector3.zero;

    // 【关键】记录原始位置
    private Vector3 defaultLeftHandPos;
    private bool initialized = false;

    void Start()
    {
        if (playerController == null) playerController = FindObjectOfType<PlayerController>();

        // 1. 游戏开始瞬间，死死记住左手原本完美的待机位置！
        if (leftHandTarget != null)
        {
            defaultLeftHandPos = leftHandTarget.localPosition;
            initialized = true;
        }
    }

    void LateUpdate()
    {
        if (playerController == null) return;

        bool isFiring = playerController.isFire;

        // ================= 1. 枪械处理 (叠加模式) =================
        // 枪比较特殊，因为 PlayerController 每一帧都会把它重置回原点
        // 所以我们必须用 += 在它的基础上再次叠加
        if (weaponRoot != null)
        {
            Vector3 targetGunOffset = isFiring ? gunRecoilOffset : Vector3.zero;
            currentGunOffset = Vector3.Lerp(currentGunOffset, targetGunOffset, Time.deltaTime * smoothSpeed);

            // 在控制器重置后的位置上，叠加偏移
            weaponRoot.localPosition += currentGunOffset;
        }

        // ================= 2. 左手处理 (绝对模式) =================
        // 左手没人重置它，所以绝对不能用 +=，必须用 "原始位置 + 偏移"
        if (leftHandTarget != null && initialized)
        {
            Vector3 targetLeftHandOffset = isFiring ? leftHandFireOffset : Vector3.zero;
            currentLeftHandOffset = Vector3.Lerp(currentLeftHandOffset, targetLeftHandOffset, Time.deltaTime * smoothSpeed);

            // 【核心修复】永远等于：原始待机位置 + 当前偏移量
            // 这样松开左键时，偏移量归零，它就会乖乖回到 defaultLeftHandPos
            leftHandTarget.localPosition = defaultLeftHandPos + currentLeftHandOffset;
        }
    }

    // 如果你在运行时觉得待机位置本身就不对，调整好后点这个按钮更新“原始点”
    [ContextMenu("更新左手原始位置")]
    public void UpdateLeftHandDefault()
    {
        if (leftHandTarget != null) defaultLeftHandPos = leftHandTarget.localPosition;
    }
}