using UnityEngine;
using Cinemachine; // 引入Cinemachine命名空间

// 这行代码会让它出现在Unity的菜单里
[AddComponentMenu("Cinemachine/Impulse/My Impulse Source")]
public class MyImpulseSource : CinemachineImpulseSource
{
    // 这里什么都不用写
    // 因为 CinemachineImpulseSource 已经包含了所有你需要的功能
    // 我们只是把它“显露”出来
}