using UnityEngine;

public class SimpleShellEjector : MonoBehaviour
{
    [Header("抛壳设置")]
    public GameObject casingPrefab;
    public Transform spawnPoint;
    public float ejectPower = 8f;
    public float torquePower = 5f;

    public void Eject()
    {
        if (casingPrefab == null || spawnPoint == null) return;

        // 👇 ==========================================
        // 使用对象池生成弹壳
        // ==========================================
        GameObject shell;
        if (ObjectPoolManager.Instance != null)
        {
            shell = ObjectPoolManager.Instance.Spawn(casingPrefab, spawnPoint.position, spawnPoint.rotation);
        }
        else
        {
            shell = Instantiate(casingPrefab, spawnPoint.position, spawnPoint.rotation);
            Destroy(shell, 2f); // 兜底方案
        }

        Rigidbody rb = shell.GetComponent<Rigidbody>();
        if (rb != null)
        {
            // 👇 【面试必杀技】清空刚体残留的历史动能，防止弹壳因为叠加受力而乱飞！
            rb.velocity = Vector3.zero;
            rb.angularVelocity = Vector3.zero;

            // 重新施加抛壳的力度
            Vector3 forceDir = spawnPoint.right * Random.Range(0.8f, 1.2f) + spawnPoint.up * Random.Range(0.2f, 0.5f);
            rb.AddForce(forceDir * ejectPower, ForceMode.Impulse);
            rb.AddTorque(Random.insideUnitSphere * torquePower, ForceMode.Impulse);
        }
    }
}