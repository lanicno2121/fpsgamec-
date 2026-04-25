using UnityEngine;

public class GenshinLootSparkle : MonoBehaviour
{
    [Header("配置")]
    [Tooltip("挂载了发光贴图的 SpriteRenderer")]
    public SpriteRenderer spriteRenderer;

    [Header("闪烁与缩放设置")]
    [Tooltip("最暗/最小时的状态 (0~1)")]
    public float minIntensity = 0.4f;
    [Tooltip("最亮/最大时的状态")]
    public float maxIntensity = 1.2f;
    [Tooltip("闪烁速度 (越大越快)")]
    public float pulseSpeed = 5f;

    [Tooltip("光点的颜色 (可以调成原神那种金色或紫色)")]
    public Color glowColor = new Color(1f, 0.8f, 0.2f, 1f); // 默认偏金色

    void Start()
    {
        // 自动获取组件
        if (spriteRenderer == null)
            spriteRenderer = GetComponent<SpriteRenderer>();
    }

    void Update()
    {
        if (spriteRenderer != null)
        {
            // 核心数学：生成 0 到 1 的平滑呼吸波浪
            float pingPong = (Mathf.Sin(Time.time * pulseSpeed) + 1f) / 2f;

            // 1. 改变大小 (忽大忽小，产生“闪烁”的错觉)
            float currentScale = Mathf.Lerp(minIntensity, maxIntensity, pingPong);
            transform.localScale = new Vector3(currentScale, currentScale, currentScale);

            // 2. 改变透明度 (忽明忽暗)
            Color c = glowColor;
            c.a = Mathf.Lerp(0.1f, 1f, pingPong); // 从几乎透明到完全不透明
            spriteRenderer.color = c;

            // 3. 【画龙点睛】永远面向主摄像机 (Billboard 效果)
            // 这样无论玩家绕着尸体怎么走，这个光点永远用最亮的正面对着玩家的眼睛
            if (Camera.main != null)
            {
                transform.forward = Camera.main.transform.forward;
            }
        }
    }
}