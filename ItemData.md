using UnityEngine;

// 物品类型的枚举，方便以后写逻辑（比如：消耗品能吃，钥匙能开门）
public enum ItemType
{
    Consumable, // 消耗品 (药包、食物)
    KeyItem,    // 关键道具 (门禁卡、钥匙)
    Ammo,       // 弹药 (备弹盒)
    Material    // 杂物/材料 (齿轮、火药)
}

[CreateAssetMenu(fileName = "New Item", menuName = "FPS System/Inventory Item")]
public class ItemData : ScriptableObject
{
    [Header("基础信息")]
    public string itemName = "新物品";

    [TextArea(3, 5)]
    public string description = "这是一个没有任何卵用的神秘物品。";

    [Tooltip("在背包里显示的图标")]
    public Sprite icon;

    [Header("属性设置")]
    public ItemType itemType;

    [Tooltip("一个格子最多能叠多少个？(钥匙通常是1，子弹可能是60)")]
    public int maxStack = 1;

    // 预留给“消耗品”的数值（比如药包回血量 50）
    public float actionValue = 0f;
}