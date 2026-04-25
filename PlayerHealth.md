using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class PlayerHealth : MonoBehaviour
{
    [Header("=== 基础属性 ===")]
    public float maxHealth = 100f;
    public float currentHealth;

    [Header("=== 组件引用 ===")]
    // 【新增 1】我们需要引用 PlayerModel 来播放动画
    public PlayerModel playerModel;

    [Header("=== UI 引用 ===")]
    public Slider healthSlider;
    public TMP_Text healthText;

    [Header("=== 受伤特效 ===")]
    public Image damageImage;
    public float flashSpeed = 5f;
    public Color flashColor = new Color(1f, 0f, 0f, 0.5f);

    private bool isDead = false;
    private bool damaged = false;

    void Start()
    {
        currentHealth = maxHealth;

        // 【新增 2】自动查找 PlayerModel (如果没手动拖的话)
        if (playerModel == null)
        {
            // 尝试找子物体
            playerModel = GetComponentInChildren<PlayerModel>();
            // 尝试找兄弟物体
            if (playerModel == null) playerModel = GetComponent<PlayerController>()?.currentPlayerModel;
            // 尝试从单例找
            if (playerModel == null && PlayerController.INSTANCE != null) playerModel = PlayerController.INSTANCE.currentPlayerModel;
        }

        UpdateUI();
    }

    void Update()
    {
        // 测试按键
        if (Input.GetKeyDown(KeyCode.K)) TakeDamage(10);
        if (Input.GetKeyDown(KeyCode.H)) Heal(10);

        if (damageImage != null)
        {
            if (damaged) damageImage.color = flashColor;
            else damageImage.color = Color.Lerp(damageImage.color, Color.clear, flashSpeed * Time.deltaTime);
        }
        damaged = false;
    }

    public void TakeDamage(float amount)
    {
        if (isDead) return;

        damaged = true;
        currentHealth -= amount;

        // ==========================================
        // 【新增 3】核心：在这里调用受击动画！
        // ==========================================
        if (playerModel != null)
        {
            playerModel.TriggerHit();
        }

        if (currentHealth <= 0)
        {
            currentHealth = 0;
            Die();
        }

        UpdateUI();
    }

    public void Heal(float amount)
    {
        if (isDead) return;
        currentHealth += amount;
        if (currentHealth > maxHealth) currentHealth = maxHealth;
        UpdateUI();
    }

    void UpdateUI()
    {
        if (healthSlider != null) healthSlider.value = currentHealth / maxHealth;
        if (healthText != null) healthText.text = $"{currentHealth} / {maxHealth}";
    }

    void Die()
    {
        if (isDead) return; // 防止重复死亡
        isDead = true;
        Debug.Log("玩家已死亡！触发死亡状态...");

        // 1. 触发 UI 变色 (保留你原有的逻辑)
        if (damageImage != null)
        {
            var deadColor = flashColor;
            deadColor.a = 0.8f;
            damageImage.color = deadColor;
        }

        // 2. 【修改】不再粗暴地禁用组件，而是切换到 Dead 状态
        // 这样 PlayerController 还在运行，但 DeadState 屏蔽了所有输入
        if (playerModel != null)
        {
            playerModel.SwitchState(PlayerState.Dead);
        }

        // 3. 禁用武器脚本 (防止死后还能开枪/换弹)
        // 获取 PlayerWeapon 并禁用
        var weapon = GetComponentInChildren<PlayerWeapon>();
        if (weapon != null) weapon.enabled = false;

        // 禁用后坐力脚本
        var recoil = GetComponentInChildren<WeaponRecoil>();
        if (recoil != null) recoil.enabled = false;

        // 禁用 GunAmmoSystem (防止自动回弹等逻辑)
        var ammoSys = GetComponentInChildren<GunAmmoSystem>();
        if (ammoSys != null) ammoSys.enabled = false;
    }
}