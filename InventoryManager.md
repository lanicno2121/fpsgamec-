using System.Collections.Generic;
using UnityEngine;
using System;

// 定义一个“背包格子”的数据结构
[System.Serializable]
public class InventorySlot
{
    public ItemData item;
    public int amount;

    public InventorySlot(ItemData item, int amount)
    {
        this.item = item;
        this.amount = amount;
    }
}

public class InventoryManager : MonoBehaviour
{
    [Header("生化危机背包设置")]
    [Tooltip("背包的最大格子数 (初始 12 格)")]
    public int maxSlots = 12;

    // 这就是你口袋里真正装的东西
    public List<InventorySlot> slots = new List<InventorySlot>();

    // 当背包里的东西发生变化时，呼叫 UI 刷新
    public event Action OnInventoryChanged;

    /// <summary>
    /// 核心功能 1：捡起物品放进包里
    /// </summary>
    public bool AddItem(ItemData itemToAdd, int amountToAdd = 1)
    {
        // 1. 如果这个物品可以堆叠 (比如子弹、药草)，先找找包里有没有没装满的
        if (itemToAdd.maxStack > 1)
        {
            foreach (var slot in slots)
            {
                if (slot.item == itemToAdd && slot.amount < itemToAdd.maxStack)
                {
                    // 计算这个格子还能塞多少
                    int spaceLeft = slot.item.maxStack - slot.amount;
                    if (amountToAdd <= spaceLeft)
                    {
                        slot.amount += amountToAdd;
                        OnInventoryChanged?.Invoke();
                        return true; // 完美塞入
                    }
                    else
                    {
                        slot.amount += spaceLeft;
                        amountToAdd -= spaceLeft; // 剩下的继续往下个格子塞
                    }
                }
            }
        }

        // 2. 如果不能堆叠 (钥匙)，或者上面的格子塞满了还有剩余，那就开辟一个新格子！
        if (slots.Count < maxSlots && amountToAdd > 0)
        {
            slots.Add(new InventorySlot(itemToAdd, amountToAdd));
            OnInventoryChanged?.Invoke();
            return true;
        }

        // 3. 背包满了！
        Debug.LogWarning("【系统提示】背包已满，无法拾取：" + itemToAdd.itemName);
        return false;
    }

    /// <summary>
    /// 核心功能 2：消耗/丢弃物品
    /// </summary>
    public void RemoveItem(ItemData itemToRemove, int amountToRemove = 1)
    {
        for (int i = slots.Count - 1; i >= 0; i--)
        {
            if (slots[i].item == itemToRemove)
            {
                if (slots[i].amount > amountToRemove)
                {
                    slots[i].amount -= amountToRemove;
                    OnInventoryChanged?.Invoke();
                    return;
                }
                else
                {
                    amountToRemove -= slots[i].amount;
                    slots.RemoveAt(i); // 这个格子彻底空了，删掉
                    if (amountToRemove <= 0)
                    {
                        OnInventoryChanged?.Invoke();
                        return;
                    }
                }
            }
        }
    }

    /// <summary>
    /// 核心功能 3：检查包里有没有某个东西 (关卡解谜专用！)
    /// </summary>
    public bool HasItem(ItemData itemToCheck, int amountRequired = 1)
    {
        int count = 0;
        foreach (var slot in slots)
        {
            if (slot.item == itemToCheck) count += slot.amount;
        }
        return count >= amountRequired;
    }
}