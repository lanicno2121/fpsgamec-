using UnityEngine;
using System; // 引用这个才能使用 Action

public class GunAmmoSystem : MonoBehaviour
{
    [Header("核心数据源")]
    public GunData gunData;

    [Header("组件连接")]
    public PlayerWeapon weaponAction;

    [Header("运行时状态")]
    [SerializeField] public int currentMagAmmo;
    [SerializeField] public int currentReserveAmmo;

    [Header("枪械音效")]
    public AudioClip fireSound;       // 开火声
    [Range(0f, 1f)]
    public float fireVolume = 0.5f;   // 开火音量滑块

    public AudioClip emptyClickSound; // 没子弹时的空仓咔哒声
    [Range(0f, 1f)]
    public float emptyClickVolume = 1.0f; // 空仓音量滑块

    public AudioClip reloadSound;     // 换弹声
    [Range(0f, 1f)]
    public float reloadVolume = 1.0f;     // 换弹音量滑块

    private AudioSource audioSource;

    // === 事件定义区 ===
    public event Action<int, int> OnAmmoChanged;
    public event Action<bool> OnReloadStatusChanged;

    public bool isReloading = false;
    private float nextFireTime = 0f;
    private bool wasFiringLastFrame = false;

    private Camera mainCam;
    private PlayerController playerController;

    void Start()
    {
        mainCam = Camera.main;

        audioSource = GetComponent<AudioSource>();
        if (audioSource == null) audioSource = gameObject.AddComponent<AudioSource>();
        audioSource.playOnAwake = false;

        playerController = FindObjectOfType<PlayerController>();
        if (playerController == null)
        {
            Debug.LogError("严重错误：GunAmmoSystem 找不到 PlayerController！请确保场景里有主角。");
        }

        if (weaponAction == null) weaponAction = GetComponent<PlayerWeapon>();

        // 放在 Start 的最后初始化
        InitAmmo();
    }

    void Update()
    {
        if (isReloading) return;
        HandleInput();
        if (Input.GetKeyDown(KeyCode.R)) Reload();
    }

    void HandleInput()
    {
        if (playerController == null) return;
        bool isFirePressed = playerController.isFire;
        bool shouldFire = false;

        if (gunData.isAutomatic) shouldFire = isFirePressed;
        else shouldFire = isFirePressed && !wasFiringLastFrame;

        if (shouldFire) TryFire();
        wasFiringLastFrame = isFirePressed;
    }

    // 👇 ==========================================
    // 【核心升级 1】在初始化时，先去读取 JSON！
    // ==========================================
    public void InitAmmo()
    {
        if (gunData == null) return;

        // 【防坑细节】复印一份数据，防止在 Unity 编辑器中把原始配置改乱了
        gunData = Instantiate(gunData);

        // 如果有开火脚本，让它的数据也指向这个安全的复印件
        if (weaponAction != null) weaponAction.gunData = gunData;

        // 去 JSON 查表并覆盖数据！
        SyncDataWithJSON();

        currentMagAmmo = gunData.magCapacity;
        currentReserveAmmo = gunData.initialReserveAmmo;
        isReloading = false;

        // 初始化时广播 UI
        OnAmmoChanged?.Invoke(currentMagAmmo, currentReserveAmmo);
    }

    // 👇 ==========================================
    // 【核心升级 2】执行查表并覆盖的逻辑
    // ==========================================
    public void SyncDataWithJSON()
    {
        // 如果场景里没有放数据管理器，就直接跳过（保证游戏不崩溃）
        if (WeaponDataManager.Instance == null) return;

        // 去掉克隆后缀，获取枪械纯净名字 (比如从 "AK47(Clone)" 变成 "AK47")
        string id = gunData.name.Replace("(Clone)", "").Trim();

        WeaponEntry jsonEntry = WeaponDataManager.Instance.GetWeaponData(id);

        if (jsonEntry != null)
        {
            // 🎉 用 JSON 里的数据，强行覆盖本地数据！
            gunData.magCapacity = jsonEntry.magCapacity;
            gunData.fireRate = jsonEntry.fireRate;
            gunData.reloadTime = jsonEntry.reloadTime;

            // 注意：伤害(damage) 也是在这里覆盖到 gunData 里的
            // weaponAction 脚本在射击时，会去读这个被覆盖过的新 damage
            gunData.damage = jsonEntry.damage;

            Debug.Log($"<color=#00FF00>【数据驱动成功】</color> {id} 正在使用 JSON 外部数据！当前伤害:{gunData.damage} | 弹匣:{gunData.magCapacity}");
        }
        else
        {
            Debug.LogWarning($"【数据驱动提示】JSON 里没找到名为 '{id}' 的武器，将使用原来的默认数据。");
        }
    }

    public void TryFire()
    {   
        if (Time.time < nextFireTime) return;

        if (currentMagAmmo <= 0)
        {
            if (!wasFiringLastFrame)
            {
                if (emptyClickSound != null && audioSource != null)
                {
                    audioSource.pitch = 1f;
                    audioSource.PlayOneShot(emptyClickSound, emptyClickVolume);
                }
            }
            return;
        }

        nextFireTime = Time.time + gunData.fireRate;
        currentMagAmmo--;

        if (fireSound != null && audioSource != null)
        {
            audioSource.pitch = UnityEngine.Random.Range(0.95f, 1.05f);
            audioSource.PlayOneShot(fireSound, fireVolume);
        }

        OnAmmoChanged?.Invoke(currentMagAmmo, currentReserveAmmo);

        if (playerController != null) playerController.ShakeCamera();

        Vector3 targetPoint = GetAimPoint();

        if (weaponAction != null) weaponAction.ExecuteShoot(targetPoint);
    }

    Vector3 GetAimPoint()
    {
        Ray ray = mainCam.ViewportPointToRay(new Vector3(0.5f, 0.5f, 0));
        RaycastHit hit;
        if (Physics.Raycast(ray, out hit, 1000f)) return hit.point;
        else return ray.GetPoint(1000f);
    }

    public void Reload()
    {
        if (isReloading || currentMagAmmo >= gunData.magCapacity || currentReserveAmmo <= 0) return;

        isReloading = true;

        if (reloadSound != null && audioSource != null)
        {
            audioSource.pitch = 1f;
            audioSource.PlayOneShot(reloadSound, reloadVolume);
        }

        OnReloadStatusChanged?.Invoke(true);
        Invoke(nameof(FinishReload), gunData.reloadTime);
    }

    void FinishReload()
    {
        int ammoNeeded = gunData.magCapacity - currentMagAmmo;
        int ammoToReload = Mathf.Min(ammoNeeded, currentReserveAmmo);
        currentMagAmmo += ammoToReload;
        currentReserveAmmo -= ammoToReload;

        isReloading = false;
        OnReloadStatusChanged?.Invoke(false);
        OnAmmoChanged?.Invoke(currentMagAmmo, currentReserveAmmo);
    }

    public void AddReserveAmmo(int amount)
    {
        currentReserveAmmo += amount;
        if (gunData != null && currentReserveAmmo > gunData.maxReserveAmmo) currentReserveAmmo = gunData.maxReserveAmmo;
        OnAmmoChanged?.Invoke(currentMagAmmo, currentReserveAmmo);
    }
}