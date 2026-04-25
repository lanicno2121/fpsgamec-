using UnityEngine;
using UnityEngine.SceneManagement;

public class GameEndingTrigger : MonoBehaviour
{
    [Header("结局运镜设置")]
    [Tooltip("摄像机绕着谁转？通常拖入汽车模型")]
    public Transform spiralCenter;
    [Tooltip("上升速度")]
    public float riseSpeed = 2.0f;
    [Tooltip("旋转速度")]
    public float spiralSpeed = 20.0f;

    [Header("UI 设置")]
    [Tooltip("拖入包含'游戏结束'字样的UI面板")]
    public GameObject endingUIPanel;
    [Tooltip("你的主菜单场景的名字 (务必与Build Settings一致)")]
    public string mainMenuSceneName = "MainMenuScene";

    private bool isEnding = false;
    private Camera playerCamera;
    private PlayerController player;

    void Start()
    {
        if (endingUIPanel != null) endingUIPanel.SetActive(false);
        if (spiralCenter == null) spiralCenter = transform;
    }

    void OnTriggerEnter(Collider other)
    {
        if (isEnding) return;

        // 尝试从触发者或其父物体获取玩家脚本
        PlayerController pc = other.GetComponentInParent<PlayerController>();

        if (pc != null)
        {
            isEnding = true;
            player = pc;

            // ==========================================
            // 1. 让角色彻底停下 (核心修正)
            // ==========================================
            // 将输入矢量清零，防止禁用脚本后角色还带着惯性滑行
            player.moveInput = Vector2.zero;

            // 如果你的角色有刚体(Rigidbody)，强制将物理速度清零
            Rigidbody rb = player.GetComponent<Rigidbody>();
            if (rb != null)
            {
                rb.velocity = Vector3.zero;
                rb.angularVelocity = Vector3.zero;
                rb.isKinematic = true; // 开启学运动学，防止被物理推走
            }

            // 最后禁用控制脚本
            player.enabled = false;

            // 释放鼠标
            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;

            // ==========================================
            // 2. 剥离主摄像机，执行螺旋升天
            // ==========================================
            playerCamera = Camera.main;
            if (playerCamera != null)
            {
                playerCamera.transform.SetParent(null);

                // 强制关停相机上一切抢方向盘的脚本（如相机跟随脚本或Cinemachine）
                MonoBehaviour[] cameraScripts = playerCamera.GetComponents<MonoBehaviour>();
                foreach (MonoBehaviour script in cameraScripts)
                {
                    if (script.GetType() != typeof(AudioListener) && script != this)
                    {
                        script.enabled = false;
                    }
                }
            }

            // ==========================================
            // 3. 弹出游戏结束 UI
            // ==========================================
            if (endingUIPanel != null) endingUIPanel.SetActive(true);
        }
    }

    void Update()
    {
        // 螺旋上升逻辑
        if (isEnding && playerCamera != null)
        {
            // 1. 向上平移
            playerCamera.transform.position += Vector3.up * riseSpeed * Time.deltaTime;
            // 2. 绕着中心点旋转
            playerCamera.transform.RotateAround(spiralCenter.position, Vector3.up, spiralSpeed * Time.deltaTime);
            // 3. 永远盯着中心点
            playerCamera.transform.LookAt(spiralCenter);
        }
    }

    // UI 按钮点击事件
    public void ReturnToMainMenu()
    {
        SceneManager.LoadScene(mainMenuSceneName);
    }
}