```csharp
using System.Collections;
using System.Collections.Generic;
using Unity.IO.LowLevel.Unsafe;
using UnityEditor;
using UnityEngine;
using UnityEngine.AI;

public enum EnemyState
{
    Idle, Move, Attack, Dead, Patrol
}

public abstract class EnemyBase : MonoBehaviour, IStateMachineOwner
{
    [HideInInspector]
    public Animator animator;
    protected StateMachine stateMachine;

    #region 寻路相关
    [HideInInspector]
    public NavMeshAgent navMeshAgent;
    [Tooltip("转向速度")]
    public float rotationSpeed = 300f;
    [Tooltip("最小攻击距离")]
    public float minAttackDistance = 1f;

    [Header("警戒设置")]
    [Tooltip("警戒距离：玩家靠近这个距离才会触发追击")]
    public float detectionDistance = 15f;

    [HideInInspector]
    public PlayerModel attackTarget;
    #endregion

    #region 巡逻相关
    [Header("巡逻设置")]
    [Tooltip("存放巡逻点的位置数组")]
    public Transform[] waypoints;
    [HideInInspector]
    public int currentWaypointIndex = 0;
    [Tooltip("到达巡逻点后的发呆休息时间")]
    public float idleWaitTime = 2.0f;
    [HideInInspector]
    public float idleTimer = 0f;
    #endregion

    [Header("攻击设置")]
    [Tooltip("两次攻击之间的间隔时间 (秒)")]
    public float attackInterval = 2.0f;
    [HideInInspector]
    public float lastAttackTime;

    #region 流血相关预制体
    [Tooltip("喷血/受击溅射特效")]
    public GameObject bloodSmashPrefab;
    [Tooltip("滴血/漏油特效")]
    public GameObject bloodDrippingPrefab;
    #endregion

    #region 受击相关
    protected int hitHash;
    protected int moveSpeedHash;
    protected float normalMoveSpeed = 1;
    protected float slowMoveSpeed = 0.5f;
    protected Coroutine recoverSpeedCoroutine;
    #endregion

    #region 血条相关
    [Tooltip("生命值")]
    public int health = 100;
    private float currentHealth;
    private bool isDead = false;
    [Tooltip("血条预制体")]
    public GameObject healthBarPrefab;
    [Tooltip("血条的位置")]
    public Transform healthBarPos;
    [HideInInspector]
    public GameObject healthBar;

    [Tooltip("血条框显示时间")]
    public float healthBarShowTime = 6;
    private float healthBarShow_timer;
    #endregion

    #region 音效相关
    [Header("音效设置 (受击与死亡)")]
    [Tooltip("常规受击音效池 (打肉的噗噗声)")]
    public AudioClip[] hitSounds;
    [Range(0f, 1f)] public float hitVolume = 1.0f;

    [Tooltip("爆头/弱点专属音效 (建议放一个清脆的声音)")]
    public AudioClip headshotSound;
    [Range(0f, 1f)] public float headshotVolume = 1.0f;

    [Tooltip("怪物死亡时的惨叫声")]
    public AudioClip deathSound;
    [Range(0f, 1f)] public float deathVolume = 1.0f;

    protected AudioSource audioSource;
    #endregion

    protected virtual void Awake()
    {
        stateMachine = new StateMachine(this);
        animator = GetComponent<Animator>();
        navMeshAgent = GetComponent<NavMeshAgent>();
        navMeshAgent.stoppingDistance = minAttackDistance;
        navMeshAgent.angularSpeed = rotationSpeed;
        hitHash = Animator.StringToHash("Hit");
        moveSpeedHash = Animator.StringToHash("MoveSpeed");
        currentHealth = health;
        healthBarShow_timer = healthBarShowTime;

        lastAttackTime = -attackInterval;

        // 强制设置为纯 2D 贴耳播放
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null) audioSource = gameObject.AddComponent<AudioSource>();
        audioSource.playOnAwake = false;

        // 核心：0 = 纯 2D 声音。完全无视距离衰减，直接在玩家耳边按你设定的音量播放！
        audioSource.spatialBlend = 0f;
    }

    protected virtual void Start()
    {
        SwitchState(EnemyState.Idle);
        FindAttackTarget();
        if (healthBarPrefab != null && healthBarPos != null)
        {
            healthBar = Instantiate(healthBarPrefab, healthBarPos.position, Quaternion.identity);
            if (UIManager.INSTANCE != null)
            {
                healthBar.transform.SetParent(UIManager.INSTANCE.WorldSpaceCanvas.transform);
            }
        }
    }

    protected virtual void Update()
    {
        if (isDead) return;
        if (healthBar != null)
        {
            if (healthBarShow_timer < healthBarShowTime)
            {
                healthBar.SetActive(true);
                healthBar.transform.position = healthBarPos.transform.position;
                healthBarShow_timer += Time.deltaTime;
            }
            else healthBar.SetActive(false);
        }
    }

    public virtual void FindAttackTarget()
    {
        if (GameManager.INSTANCE == null) return;
        PlayerModel[] playerModels = GameManager.INSTANCE.playerModels;
        if (playerModels != null && playerModels.Length > 0)
        {
            PlayerModel closestPlayer = null;
            float minDistance = float.MaxValue;
            foreach (PlayerModel player in playerModels)
            {
                if (player != null)
                {
                    float distance = Vector3.Distance(transform.position, player.transform.position);
                    if (distance < minDistance) { minDistance = distance; closestPlayer = player; }
                }
            }
            attackTarget = closestPlayer;
        }
    }

    protected virtual void SlowMoveAnimation()
    {
        animator.SetFloat(moveSpeedHash, slowMoveSpeed);
        if (recoverSpeedCoroutine != null) StopCoroutine(recoverSpeedCoroutine);
        recoverSpeedCoroutine = StartCoroutine(RecoverMoveSpeed(0.5f));
    }

    protected IEnumerator RecoverMoveSpeed(float delay)
    {
        yield return new WaitForSeconds(delay);
        animator.SetFloat(moveSpeedHash, normalMoveSpeed);
        recoverSpeedCoroutine = null;
    }

    public virtual void Hurt(PlayerWeaponBullet bullet, float damageMultiplier = 1)
    {
        if (isDead) return;

        animator.SetTrigger(hitHash);
        SlowMoveAnimation();

        // 👇 ==========================================
        // 【核心优化】使用对象池替换掉高耗能的 Instantiate 和 Destroy
        // ==========================================
        if (bloodSmashPrefab != null)
        {
            Vector3 bulletDir = bullet.transform.forward;
            Quaternion rotation = Quaternion.LookRotation(-bulletDir);

            // 如果场景中存在对象池管理器，则使用对象池生成，否则作为安全降级使用原本的 Instantiate
            if (ObjectPoolManager.Instance != null)
            {
                ObjectPoolManager.Instance.Spawn(bloodSmashPrefab, bullet.transform.position, rotation);
            }
            else
            {
                // 防止你在某个测试场景里忘了放 ObjectPoolManager 报错，做个备用兜底
                Destroy(Instantiate(bloodSmashPrefab, bullet.transform.position, rotation), 3f);
            }
        }
        // 👆 ==========================================

        // 播放受击音效 (现在的声音是无视距离直接入耳的)
        if (audioSource != null)
        {
            audioSource.pitch = Random.Range(0.9f, 1.1f);

            if (damageMultiplier > 1.0f && headshotSound != null)
            {
                audioSource.PlayOneShot(headshotSound, headshotVolume);
            }
            else if (hitSounds != null && hitSounds.Length > 0)
            {
                AudioClip randomHit = hitSounds[Random.Range(0, hitSounds.Length)];
                audioSource.PlayOneShot(randomHit, hitVolume);
            }
        }

        currentHealth -= bullet.damage * damageMultiplier;

        if (healthBar != null)
        {
            if (currentHealth > 0)
            {
                healthBarShow_timer = 0;
                healthBar.GetComponent<EnemyHealthBarUI>().UpdateHealthBar(currentHealth / health);
            }
            else
            {
                PlayDeathLogic();
            }
        }
        else if (currentHealth <= 0 && !isDead)
        {
            PlayDeathLogic();
        }
    }

    private void PlayDeathLogic()
    {
        if (deathSound != null && audioSource != null)
        {
            audioSource.pitch = Random.Range(0.9f, 1.1f);
            audioSource.PlayOneShot(deathSound, deathVolume);
        }

        SwitchState(EnemyState.Dead);
        if (navMeshAgent != null) navMeshAgent.enabled = false;
        if (GetComponent<BoxCollider>() != null) GetComponent<BoxCollider>().enabled = false;
        currentHealth = 0;
        isDead = true;
        if (healthBar != null) Destroy(healthBar);

        // 👇 ==========================================
        // 【核心解耦】怪物死亡时，不需要管谁在统计击杀，直接广播！
        // ==========================================
        // 这里的 ? 的意思是“如果有人监听了这个频道，才广播，没人听就算了”
        GameEventManager.OnEnemyDied?.Invoke(this);
    }

    public virtual bool HasAttackTarget() => attackTarget != null;

    public virtual bool IsAttackTargetInAttackRange()
    {
        if (HasAttackTarget()) return Vector3.Distance(transform.position, attackTarget.transform.position) < minAttackDistance;
        return false;
    }

    public virtual bool IsTargetInDetectionRange(float multiplier = 1.0f)
    {
        if (HasAttackTarget()) return Vector3.Distance(transform.position, attackTarget.transform.position) <= (detectionDistance * multiplier);
        return false;
    }

    public virtual void chaseTarget()
    {
        if (HasAttackTarget() && navMeshAgent.enabled) navMeshAgent.SetDestination(attackTarget.transform.position);
    }

    public abstract void SwitchState(EnemyState state);

    public void PlayStateAnimation(string animationName, float transition = 0.25f, int layer = 0)
    {
        animator.CrossFadeInFixedTime(animationName, transition, layer);
    }

    public void Clear()
    {
        stateMachine.Stop();
        if (healthBar != null) Destroy(healthBar);
        Destroy(gameObject);
    }
}
```