using UnityEngine;

public class LootPulseLight : MonoBehaviour
{
    [Header("配置")]
    [Tooltip("需要闪烁的灯光组件")]
    public Light pulseLight;

    [Header("闪烁参数")]
    [Tooltip("最暗时的亮度")]
    public float minIntensity = 0.5f;
    [Tooltip("最亮时的亮度")]
    public float maxIntensity = 2.5f;
    [Tooltip("闪烁的速度 (越大越快)")]
    public float pulseSpeed = 4f;

    void Start()
    {
        // 如果没手动拖拽，自动获取身上的 Light 组件
        if (pulseLight == null)
        {
            pulseLight = GetComponent<Light>();
        }
    }

    void Update()
    {
        if (pulseLight != null)
        {
            // 【核心数学魔法】：Mathf.Sin 可以生成一个 -1 到 1 的波浪线
            // 我们把它转换成 0 到 1 的比例，就能做出完美的呼吸效果
            float pingPong = (Mathf.Sin(Time.time * pulseSpeed) + 1f) / 2f;

            pulseLight.intensity = Mathf.Lerp(minIntensity, maxIntensity, pingPong);
        }
    }
}