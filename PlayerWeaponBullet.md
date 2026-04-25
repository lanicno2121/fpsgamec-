using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerWeaponBullet : MonoBehaviour
{
    [Tooltip("伤害")]
    public int damage = 10;
    [HideInInspector]
    public Rigidbody rb;
    [Tooltip("推力")]
    public float flyPower = 30f;
    [Tooltip("存活时间")]
    public float lifeTime = 10f;

    private Vector3 prevPosition;

    [Tooltip("子弹气流预制体")]
    public GameObject trailEffect;
    [Tooltip("气流生成间隔")]
    public float trailInterval = 0.1f;
    private float trailInterval_timer = 0f;

    private void Awake()
    {
        rb = GetComponent<Rigidbody>();
    }

    private void Start()
    {
        rb.velocity = transform.forward * flyPower;
        Destroy(gameObject, lifeTime);
        prevPosition = transform.position;
        CheckInitalOverlap();
    }

    private void Update()
    {
        CheckCollision();
        prevPosition = transform.position;

        trailInterval_timer += Time.deltaTime;
        if (trailInterval_timer >= trailInterval)
        {
            SpawnTrailEffect();
            trailInterval_timer = 0;
        }
    }

    private void SpawnTrailEffect()
    {
        if (trailEffect != null)
        {
            Quaternion reverseRotation = Quaternion.LookRotation(-transform.forward);
            Destroy(Instantiate(trailEffect, transform.position, reverseRotation), 2f);
        }
    }

    void CheckInitalOverlap()
    {
        Collider[] hitColliders = Physics.OverlapSphere(transform.position, 0.1f);
        foreach (var hitCollider in hitColliders)
        {
            // 1. 判断是否贴脸打中敌人
            EnemyBase enemy = hitCollider.GetComponent<EnemyBase>();
            if (enemy != null)
            {
                enemy.Hurt(this, 1);
                if (HitFeedbackManager.Instance != null) HitFeedbackManager.Instance.TriggerHit();
                Destroy(gameObject);
                return;
            }

            // 2. 【新增】判断是否贴脸打中可破坏机关（电压器）
            TransformerBox box = hitCollider.GetComponent<TransformerBox>();
            if (box != null)
            {
                box.TakeDamage(damage);
                if (HitFeedbackManager.Instance != null) HitFeedbackManager.Instance.TriggerHit(); // 同样触发准星反馈
                Destroy(gameObject);
                return;
            }
        }
    }

    void CheckCollision()
    {
        RaycastHit hit;
        Vector3 dir = transform.position - prevPosition;
        float distance = Vector3.Distance(transform.position, prevPosition);

        if (Physics.Raycast(prevPosition, dir.normalized, out hit, distance))
        {
            // 1. 判断是否打中敌人
            EnemyBase enemy = hit.collider.GetComponent<EnemyBase>();
            if (enemy != null)
            {
                enemy.Hurt(this, 1);
                if (HitFeedbackManager.Instance != null) HitFeedbackManager.Instance.TriggerHit();
                Destroy(gameObject);
                return;
            }

            // 2. 【新增】判断是否打中可破坏机关（电压器）
            TransformerBox box = hit.collider.GetComponent<TransformerBox>();
            if (box != null)
            {
                box.TakeDamage(damage); // 把子弹的伤害传给电压器
                if (HitFeedbackManager.Instance != null) HitFeedbackManager.Instance.TriggerHit(); // 同样触发准星反馈
                Destroy(gameObject);
                return;
            }

            // 3. 【优化】如果打到了墙壁、地板（既不是敌人也不是机关），直接销毁子弹，防止子弹穿墙
            Destroy(gameObject);
        }
    }
}