using UnityEngine;

public class CameraDrift : MonoBehaviour
{
    [Header("镜头漂移设置 (每秒移动的距离/角度)")]
    [Tooltip("位移速度：X是左右，Y是上下，Z是前后 (推进/拉远)")]
    public Vector3 moveSpeed = new Vector3(0f, 0f, 0.5f);

    [Tooltip("旋转速度：缓慢转头")]
    public Vector3 rotateSpeed = new Vector3(0f, 0f, 0f);

    void Update()
    {
        // 按照设定的速度，基于相机自身的坐标系进行极其缓慢的移动和旋转
        transform.Translate(moveSpeed * Time.deltaTime, Space.Self);
        transform.Rotate(rotateSpeed * Time.deltaTime, Space.Self);
    }
}