using UnityEngine;

public class CoverSpot : MonoBehaviour
{
    [Header("配置")]
    public Transform standPoint;
    public Transform faceDirection;

    void OnTriggerEnter(Collider other)
    {
        // 1. 打印调试信息，看看撞到了谁
        Debug.Log("检测到碰撞体进入: " + other.name);

        // 2. 【核心修改】尝试往上找，看看这个物体的老祖宗身上有没有 PlayerController
        // 这样即使撞到的是子物体 (如 lrtclear)，也能找到主角
        PlayerController pc = other.GetComponentInParent<PlayerController>();

        // 3. 判断条件：
        // A: 撞到的物体标签直接就是 "Player"
        // OR
        // B: 顺藤摸瓜找到了 PlayerController 组件
        if (other.CompareTag("Player") || pc != null)
        {
            Debug.Log(">> 身份确认成功！通知控制器...");

            // 如果找到了实例，直接调用它的方法
            if (pc != null)
            {
                pc.EnterCoverZone(this);
            }
            else
            {
                // 如果只匹配了 Tag 但没找到脚本 (极少见)，用单例兜底
                PlayerController.INSTANCE.EnterCoverZone(this);
            }
        }
    }

    void OnTriggerExit(Collider other)
    {
        // 离开时也要用同样的逻辑判断，否则可能会出现“子物体出去了，但系统没识别”的情况
        PlayerController pc = other.GetComponentInParent<PlayerController>();

        if (other.CompareTag("Player") || pc != null)
        {
            if (pc != null)
            {
                pc.ExitCoverZone();
            }
            else
            {
                PlayerController.INSTANCE.ExitCoverZone();
            }
        }
    }
}