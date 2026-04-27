```csharp

using UnityEngine;

public class HitFeedbackManager : MonoBehaviour
{
    public static HitFeedbackManager Instance;

    [Header("UI 引用")]
    [Tooltip("把挂载了这个脚本的物体自己拖进来")]
    public CanvasGroup hitMarkerGroup;

    [Header("声音反馈")]
    [Tooltip("把 AudioSource 拖进来")]
    public AudioSource audioSource;
    [Tooltip("打中敌人时的音效")]
    public AudioClip hitSound;

    [Header("动画参数")]
    [Tooltip("X 图标消失的速度，数字越大消失越快")]
    public float fadeSpeed = 5f;

    void Awake()
    {
        Instance = this;

        // 游戏一开始，把透明度设为0，隐藏起来
        if (hitMarkerGroup != null)
        {
            hitMarkerGroup.alpha = 0f;
        }
    }

    void Update()
    {
        // 每一帧让透明度慢慢下降，直到变成0
        if (hitMarkerGroup != null && hitMarkerGroup.alpha > 0)
        {
            hitMarkerGroup.alpha -= Time.deltaTime * fadeSpeed;
        }
    }

    // 给子弹调用的公开接口
    public void TriggerHit()
    {
        // 1. 瞬间把透明度拉满 (显示出 4 根线)
        if (hitMarkerGroup != null)
        {
            hitMarkerGroup.alpha = 1f;
        }

        // 2. 播放命中音效
        if (hitSound != null && audioSource != null)
        {
            // 允许声音叠加播放，连射时更爽！
            audioSource.PlayOneShot(hitSound);
        }
    }
}
```