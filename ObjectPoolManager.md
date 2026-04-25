using System.Collections.Generic;
using UnityEngine;

public class ObjectPoolManager : MonoBehaviour
{
    // 单例模式，方便全图随意调用
    public static ObjectPoolManager Instance { get; private set; }

    // 核心字典：根据预制体的名字，分类存放不同的“闲置物品队列”
    private Dictionary<string, Queue<GameObject>> poolDictionary = new Dictionary<string, Queue<GameObject>>();

    void Awake()
    {
        if (Instance == null) Instance = this;
        else Destroy(gameObject);
    }

    /// <summary>
    /// 从池中拿取对象（完美替代 Instantiate）
    /// </summary>
    public GameObject Spawn(GameObject prefab, Vector3 position, Quaternion rotation)
    {
        string key = prefab.name;

        // 如果字典里还没有这个预制体的分类，就新建一个
        if (!poolDictionary.ContainsKey(key))
        {
            poolDictionary[key] = new Queue<GameObject>();
        }

        GameObject objToSpawn;

        // 如果池子里有闲置的，直接拿出来用
        if (poolDictionary[key].Count > 0)
        {
            objToSpawn = poolDictionary[key].Dequeue();
            objToSpawn.transform.position = position;
            objToSpawn.transform.rotation = rotation;
            objToSpawn.SetActive(true);
        }
        else
        {
            // 如果池子空了（或者第一次生成），才真正实例化一个新的
            objToSpawn = Instantiate(prefab, position, rotation);
            objToSpawn.name = prefab.name; // 去掉 "(Clone)" 后缀，保持名字干净
        }

        return objToSpawn;
    }

    /// <summary>
    /// 将对象放回池中（完美替代 Destroy）
    /// </summary>
    public void Despawn(GameObject obj)
    {
        obj.SetActive(false); // 隐藏物体，当作“销毁”了
        string key = obj.name;

        if (!poolDictionary.ContainsKey(key))
        {
            poolDictionary[key] = new Queue<GameObject>();
        }

        // 重新排队，等待下次被征用
        poolDictionary[key].Enqueue(obj);
    }
}