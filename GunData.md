using UnityEngine;

// 1. [CreateAssetMenu] 是核心！
// 这行代码的作用是：让你可以在 Project 窗口右键菜单里直接创建这个数据文件
// menuName 决定了它在右键菜单里的路径
[CreateAssetMenu(fileName = "New Gun Data", menuName = "FPS System/Gun Data")]
public class GunData : ScriptableObject // 2. 注意：这里继承的是 ScriptableObject，而不是 MonoBehaviour
{


    [Tooltip("这把枪在背包里显示的图标")]
    public Sprite weaponIcon2;


    [Header("UI 显示 (新增)")]
    [Tooltip("这把枪在右下角显示的图标 Sprite")]
    public Sprite weaponIcon; // <--- 新增这一行


    [Header("身份信息")]
    public string gunName = "Rifle"; // 枪的名字

    [Header("弹药配置")]
    [Tooltip("一个弹匣能装多少发")]
    public int magCapacity = 30;

    [Tooltip("身上最大能带多少备弹")]
    public int maxReserveAmmo = 120;

    [Tooltip("一开始背包里给多少子弹")]
    public int initialReserveAmmo = 60;

    [Header("战斗参数")]
    [Tooltip("射击间隔 (秒) - 越小越快")]
    public float fireRate = 0.1f;

    [Tooltip("换弹需要几秒")]
    public float reloadTime = 1.5f;

    [Tooltip("伤害值")]
    public float damage = 20f;

    [Tooltip("是否全自动")]
    public bool isAutomatic = true;


    // ... 其他变量保持不变 ...

    [Header("视角设置")]
    [Tooltip("勾选=允许进入第一人称; 不勾选=保持第三人称肩射")]
    public bool useFirstPersonAim = true; // 默认为 true，保证步枪不受影响
    // 【新增这个开关】
    [Tooltip("勾选=开火时自动放大视角(步枪); 不勾选=开火视角不动(手枪)")]
    public bool aimOnFire = true;


}