using UnityEngine;
using System.Collections;

public class AutoElevator : MonoBehaviour
{
    [Header("电梯移动设置")]
    public Transform targetPoint;
    public Vector3 moveOffset = new Vector3(0, 5f, 0);
    public float moveDuration = 3.0f;
    public float startDelay = 2.0f;

    // 内部状态
    private Vector3 startPos;
    private Vector3 targetPos;
    private bool isMoving = false;
    private bool isAtTarget = false;

    // 用来获取刚体
    private Rigidbody rb;

    void Start()
    {
        startPos = transform.position;
        rb = GetComponent<Rigidbody>();

        if (targetPoint != null)
        {
            targetPos = new Vector3(startPos.x, targetPoint.position.y, startPos.z);
        }
        else
        {
            targetPos = startPos + moveOffset;
        }
    }

    private void OnTriggerEnter(Collider other)
    {
        // 向上顺藤摸瓜，找到主角的最顶级总控脚本
        PlayerController pc = other.GetComponentInParent<PlayerController>();

        if (pc != null)
        {
            // 【终极修复 1】必须绑定 pc.transform (根节点)，绝对不能绑定 other.transform (防止骨肉分离)
            pc.transform.SetParent(transform);

            if (!isMoving && !isAtTarget)
            {
                Invoke(nameof(StartMoving), startDelay);
            }
        }
    }

    private void OnTriggerExit(Collider other)
    {
        // 同样顺藤摸瓜找到主角根节点
        PlayerController pc = other.GetComponentInParent<PlayerController>();

        if (pc != null)
        {
            // 【终极修复 2】预防 CharacterController 休克假死：先关，解绑，再开！
            CharacterController cc = pc.GetComponent<CharacterController>();

            if (cc != null) cc.enabled = false; // 先关掉物理碰撞

            // 把整个主角根节点从电梯上解绑
            pc.transform.SetParent(null);

            if (cc != null) cc.enabled = true; // 解绑后再重新打开！

            if (!isMoving)
            {
                CancelInvoke(nameof(StartMoving));
            }
        }
    }

    private void StartMoving()
    {
        StartCoroutine(MoveRoutine());
    }

    private IEnumerator MoveRoutine()
    {
        isMoving = true;
        float timer = 0f;

        while (timer < moveDuration)
        {
            // 使用物理时间 Time.fixedDeltaTime
            timer += Time.fixedDeltaTime;
            Vector3 newPos = Vector3.Lerp(startPos, targetPos, timer / moveDuration);

            // 使用刚体的 MovePosition，告诉物理引擎“我要合法移动了”
            if (rb != null)
            {
                rb.MovePosition(newPos);
            }
            else
            {
                transform.position = newPos;
            }

            // 等待下一个物理帧，完美同步主角的物理运算！
            yield return new WaitForFixedUpdate();
        }

        // 最终位置对齐
        if (rb != null) rb.MovePosition(targetPos);
        else transform.position = targetPos;

        isMoving = false;
        isAtTarget = true;
    }
}