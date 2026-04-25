using UnityEngine;

public class AutoDespawn : MonoBehaviour
{
    [Tooltip("特效存活几秒后自动回收？")]
    public float lifeTime = 3f;

    void OnEnable()
    {
        // 物体每次被激活(Spawn)时，开始倒计时
        Invoke(nameof(DespawnSelf), lifeTime);
    }

    void OnDisable()
    {
        // 物体被隐藏时，取消倒计时（防止Bug）
        CancelInvoke(nameof(DespawnSelf));
    }

    void DespawnSelf()
    {
        // 尝试放回池子；如果池子不存在（比如直接在测试场景），就直接隐藏
        if (ObjectPoolManager.Instance != null)
        {
            ObjectPoolManager.Instance.Despawn(gameObject);
        }
        else
        {
            gameObject.SetActive(false);
        }
    }
}