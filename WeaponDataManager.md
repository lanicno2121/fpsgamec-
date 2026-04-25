using UnityEngine;
using System.IO;
using System.Collections.Generic;

[System.Serializable]
public class WeaponEntry
{
    public string weaponID;
    public float damage;
    public float fireRate;
    public int magCapacity;
    public float reloadTime;
}

[System.Serializable]
public class WeaponDataList
{
    public List<WeaponEntry> weapons;
}

public class WeaponDataManager : MonoBehaviour
{
    public static WeaponDataManager Instance;

    private Dictionary<string, WeaponEntry> weaponDatabase = new Dictionary<string, WeaponEntry>();
    private string filePath;

    void Awake()
    {
        Instance = this;
        // 缓存路径，方便后面多次读取
        filePath = Path.Combine(Application.streamingAssetsPath, "WeaponData.json");
        LoadData();
    }

    void Update()
    {
        // 👇 【热重载核心】在游戏里按下 F5 键，瞬间刷新数据！
        if (Input.GetKeyDown(KeyCode.F5))
        {
            LoadData();
            RefreshAllGunsInScene();
        }
    }

    void LoadData()
    {
        if (File.Exists(filePath))
        {
            string jsonContent = File.ReadAllText(filePath);
            WeaponDataList dataList = JsonUtility.FromJson<WeaponDataList>(jsonContent);

            weaponDatabase.Clear(); // 清空旧数据

            foreach (var entry in dataList.weapons)
            {
                weaponDatabase[entry.weaponID] = entry;
            }
            // 换个显眼的青色日志
            Debug.Log($"<color=#00FFFF>【数据中心】成功加载 {weaponDatabase.Count} 把武器数据！(可按 F5 键热重载)</color>");
        }
        else
        {
            Debug.LogError("【严重错误】找不到 WeaponData.json 文件！");
        }
    }

    public WeaponEntry GetWeaponData(string id)
    {
        if (weaponDatabase.ContainsKey(id)) return weaponDatabase[id];
        return null;
    }

    // 👇 【热重载核心】通知场景里所有的枪，数据更新啦！
    void RefreshAllGunsInScene()
    {
        GunAmmoSystem[] allGuns = FindObjectsOfType<GunAmmoSystem>();
        foreach (var gun in allGuns)
        {
            gun.SyncDataWithJSON();
        }
        Debug.Log("<color=#FFD700>【系统】已触发 F5 热重载！所有枪械已更新为最新 JSON 数据！</color>");
    }
}