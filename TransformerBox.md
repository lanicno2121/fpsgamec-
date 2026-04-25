using UnityEngine;
using UnityEngine.Events; // 必须引入这个命名空间才能用 UnityEvent

public class TransformerBox : MonoBehaviour
{
    [Header("属性设置")]
    public int health = 30; // 电压器的血量 (打几枪能打爆)
    private bool isBroken = false;

    [Header("视觉与音效")]
    [Tooltip("打爆时的火花或爆炸特效")]
    public GameObject explosionVFX;

    // 👇 ==========================================
    // 爆炸音效槽位
    // ==========================================
    [Tooltip("打爆时的爆炸音效")]
    public AudioClip explosionSound;

    [Tooltip("打爆后的破损模型/材质 (可选，用来替换完好的模型)")]
    public GameObject brokenModel;

    [Header("【核心】破坏后触发的事件")]
    [Tooltip("在这里添加打爆后要执行的事情（比如开门、弹UI）")]
    public UnityEvent onDestroyed;

    // 这个方法留给子弹来调用
    public void TakeDamage(int damage)
    {
        if (isBroken) return; // 已经坏了就不再触发了

        health -= damage;
        Debug.Log($"电压器受到攻击！剩余血量: {health}");

        if (health <= 0)
        {
            Break();
        }
    }

    private void Break()
    {
        isBroken = true;
        Debug.Log("【电压器】已摧毁！触发连锁反应！");

        // 1. 播放爆炸特效
        if (explosionVFX != null)
        {
            Instantiate(explosionVFX, transform.position, Quaternion.identity);
        }

        // 👇 ==========================================
        // 【终极修复】强行贴着主摄像机(玩家耳朵)播放，彻底解决距离太远听不见的问题！
        // ==========================================
        if (explosionSound != null)
        {
            AudioSource.PlayClipAtPoint(explosionSound, Camera.main.transform.position);
        }

        // 2. 切换为破损状态 (或者直接让当前完好的模型消失)
        // 简单粗暴的做法：直接隐藏自己身上的 MeshRenderer
        if (GetComponent<MeshRenderer>() != null)
        {
            GetComponent<MeshRenderer>().enabled = false;
        }

        if (GetComponent<Collider>() != null)
        {
            GetComponent<Collider>().enabled = false; // 关闭碰撞体，子弹穿透
        }

        // 3. 【最神奇的一步】执行所有在面板里配置好的事件 (比如开门)
        onDestroyed?.Invoke();
    }
}