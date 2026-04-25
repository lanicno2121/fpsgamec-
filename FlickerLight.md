using UnityEngine;

public class FlickerLight : MonoBehaviour
{
    // 引用灯光组件。如果为空，则尝试获取当前游戏对象上的 Light 组件。
    private Light targetLight;

    [Header("闪烁参数")]
    // 闪烁时的最小光照强度
    public float minIntensity = 0.5f;
    // 闪烁时的最大光照强度
    public float maxIntensity = 2.0f;

    // 多久改变一次强度（越小闪烁越快）
    public float baseFlickerInterval = 0.05f;

    private float timer;

    void Start()
    {
        // 尝试自动获取当前物体上的 Light 组件
        targetLight = GetComponent<Light>();
        // 初始化计时器
        timer = baseFlickerInterval;
    }

    void Update()
    {
        // 计时器递减
        timer -= Time.deltaTime;

        // 如果计时器归零，则进行一次闪烁
        if (timer <= 0)
        {
            // 1. 设置一个新的随机强度
            targetLight.intensity = Random.Range(minIntensity, maxIntensity);

            // 2. 重新设置计时器，并加入随机性，实现“不规则”闪烁
            // 每次闪烁间隔在 0.05 到 0.1 秒之间随机变化
            timer = baseFlickerInterval + Random.Range(0f, 0.05f);
        }
    }
}