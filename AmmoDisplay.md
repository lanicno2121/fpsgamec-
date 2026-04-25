using UnityEngine;
using UnityEngine.UI;

public class AmmoDisplay : MonoBehaviour
{
    [Header("UI 组件")]
    public Text ammoText;        // 对应层级里的 AmmoText

    // 【新增】对应层级里的 GunIcon
    public Image weaponIconImage;

    [Header("当前监听的武器 (自动变动)")]
    public GunAmmoSystem gunSystem;

    void Start()
    {
        // 自动初始化
        if (gunSystem == null)
            gunSystem = FindObjectOfType<GunAmmoSystem>();

        if (gunSystem != null)
        {
            BindNewWeapon(gunSystem);
        }
    }

    // 这个方法会被 WeaponManager 调用
    public void BindNewWeapon(GunAmmoSystem newGun)
    {
        // 1. 断开旧连接
        if (gunSystem != null)
        {
            gunSystem.OnAmmoChanged -= UpdateUI;
        }

        // 2. 建立新连接
        gunSystem = newGun;

        if (gunSystem != null)
        {
            // 监听弹药变化
            gunSystem.OnAmmoChanged += UpdateUI;
            // 立即刷新一次数据
            gunSystem.InitAmmo();

            // ========================================================
            // 【核心新增】根据 GunData 更新 GunIcon 图片
            // ========================================================
            if (weaponIconImage != null)
            {
                // 开启图片显示 (防止空手时隐藏了没开回来)
                weaponIconImage.enabled = true;

                // 读取数据
                if (gunSystem.gunData != null && gunSystem.gunData.weaponIcon != null)
                {
                    // 替换图片
                    weaponIconImage.sprite = gunSystem.gunData.weaponIcon;

                    // 可选：如果图片被拉伸了，取消下面这行的注释来保持比例
                    // weaponIconImage.preserveAspect = true; 
                }
                else
                {
                    Debug.LogWarning($"注意：武器 {gunSystem.name} 的 GunData 里没放图标！");
                }
            }
        }
        else
        {
            // 3. 如果是空手 (newGun == null)
            // 清空子弹文字
            if (ammoText != null) ammoText.text = "";

            // 【核心新增】隐藏图标
            if (weaponIconImage != null)
            {
                weaponIconImage.enabled = false; // 直接隐藏 GunIcon
            }
        }
    }

    void UpdateUI(int currentMag, int currentReserve)
    {
        if (ammoText != null)
        {
            ammoText.text = $"{currentMag} / {currentReserve}";
        }
    }

    void OnDestroy()
    {
        if (gunSystem != null)
        {
            gunSystem.OnAmmoChanged -= UpdateUI;
        }
    }
}