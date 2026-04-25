using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerWeapon : MonoBehaviour
{
    [Header("配置引用 (新增)")]
    public GunData gunData; // 【重要】把 AK47_Data 拖到这里，子弹伤害从这里读！

    public GameObject armModel; // 用来存这把枪配套的断臂

    [Header("特效配置")]
    [Tooltip("子弹生成的位置")]
    public Transform bulletSpawnPoint;
    [Tooltip("子弹预制体")]
    public PlayerWeaponBullet bulletEffectPrefab;
    [Tooltip("枪管火花预制体")]
    public GameObject bulletSparkPrefab;
    [Tooltip("枪管热气/烟雾特效预制体")]
    public GameObject muzzleSmokePrefab;

    // 内部引用
    private WeaponRecoil recoil;
    private FPSWeaponLogic fpsLogic;

    void Start()
    {
        recoil = GetComponent<WeaponRecoil>();

        // 自动查找 FPS 逻辑脚本
        fpsLogic = FindObjectOfType<FPSWeaponLogic>();
        if (fpsLogic == null) Debug.LogWarning("注意：没找到 FPSWeaponLogic");
    }

    public void ExecuteShoot(Vector3 targetPos)
    {
        // 1. 计算射击方向 (目标点 - 枪口点)
        Vector3 direction = targetPos - bulletSpawnPoint.position;
        direction.Normalize();

        // 👇 ==========================================
        // 【核心优化】全面接入对象池
        // ==========================================

        // 2. 生成子弹
        if (bulletEffectPrefab != null)
        {
            GameObject bulletObj;
            if (ObjectPoolManager.Instance != null)
            {
                bulletObj = ObjectPoolManager.Instance.Spawn(bulletEffectPrefab.gameObject, bulletSpawnPoint.position, Quaternion.identity);
            }
            else
            {
                bulletObj = Instantiate(bulletEffectPrefab.gameObject, bulletSpawnPoint.position, Quaternion.identity);
                Destroy(bulletObj, 3f); // 兜底方案
            }

            PlayerWeaponBullet bullet = bulletObj.GetComponent<PlayerWeaponBullet>();
            if (bullet != null)
            {
                bullet.transform.forward = direction;
                if (gunData != null) bullet.damage = (int)gunData.damage;
            }
        }

        // 3. 生成枪口火花
        if (bulletSparkPrefab != null)
        {
            if (ObjectPoolManager.Instance != null)
            {
                GameObject spark = ObjectPoolManager.Instance.Spawn(bulletSparkPrefab, bulletSpawnPoint.position, Quaternion.identity);
                spark.transform.forward = direction;
            }
            else
            {
                GameObject spark = Instantiate(bulletSparkPrefab, bulletSpawnPoint.position, Quaternion.identity);
                spark.transform.forward = direction;
                Destroy(spark, 0.5f);
            }
        }

        // 4. 生成烟雾
        if (muzzleSmokePrefab != null)
        {
            if (ObjectPoolManager.Instance != null)
            {
                GameObject smoke = ObjectPoolManager.Instance.Spawn(muzzleSmokePrefab, bulletSpawnPoint.position, Quaternion.identity);
                smoke.transform.forward = direction;
            }
            else
            {
                GameObject smoke = Instantiate(muzzleSmokePrefab, bulletSpawnPoint.position, Quaternion.identity);
                smoke.transform.forward = direction;
                Destroy(smoke, 3f);
            }
        }

        // 👆 ==========================================

        // 5. 触发后坐力 (动感)
        if (recoil != null) recoil.FireRecoil();

        // 6. 触发稳定性 (呼吸停止)
        if (fpsLogic != null) fpsLogic.TriggerStability();
    }
}