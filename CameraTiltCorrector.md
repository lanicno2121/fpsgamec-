using UnityEngine;
using Cinemachine;

// 这是一个 Cinemachine 扩展插件
// 它的作用是在相机计算完成后，额外增加一个旋转偏移
[ExecuteInEditMode]
[SaveDuringPlay]
[AddComponentMenu("")] // 不在菜单显示，直接挂载
public class CameraTiltCorrector : CinemachineExtension
{
    [Tooltip("垂直倾斜角度修正（正数抬头，负数低头）")]
    public float pitchOffset = 0f;

    protected override void PostPipelineStageCallback(
        CinemachineVirtualCameraBase vcam,
        CinemachineCore.Stage stage, ref CameraState state, float deltaTime)
    {
        // 我们在 "Aim" (瞄准) 阶段结束后介入
        if (stage == CinemachineCore.Stage.Aim)
        {
            // 获取当前相机的旋转
            Quaternion currentRot = state.RawOrientation;

            // 创建一个基于自身坐标系的垂直旋转偏移 (绕 X 轴旋转)
            Quaternion extraRot = Quaternion.Euler(pitchOffset, 0, 0);

            // 叠加旋转：让相机在原有基础上再抬一点头
            state.RawOrientation = currentRot * extraRot;
        }
    }
}