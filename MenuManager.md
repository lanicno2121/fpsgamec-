using UnityEngine;
using UnityEngine.SceneManagement; // 用于场景加载
using UnityEngine.Audio;         // 用于控制 Audio Mixer
using UnityEngine.UI;            // 用于引用 Slider UI 元素

public class MenuManager : MonoBehaviour
{
    // --- 场景引用 ---
    // 👇 ==========================================
    // 【修改】把默认加载的场景名改成了你的新场景！
    // ==========================================
    public string gameSceneName = "Level_Breakout";

    // --- UI 面板引用 ---
    // 从 Unity Editor 中拖拽 SettingsPanel 对象到这里
    [Header("UI 面板引用")]
    public GameObject settingsPanel;

    // --- 音量设置 ---
    [Header("音量设置")]
    // 从 Unity Editor 中拖拽 GameMixer 对象到这里
    public AudioMixer gameMixer;

    // 暴露的参数名称，必须与你在 Audio Mixer 中重命名的名字一致
    public string exposedVolumeParam = "MasterVolume";

    // 从 Unity Editor 中拖拽 VolumeSlider 对象到这里
    public Slider volumeSlider;

    void Start()
    {
        // 确保游戏开始时音量正确地根据滑块的初始值设置一次
        // 如果滑块存在，就调用设置音量的方法
        if (volumeSlider != null)
        {
            SetMasterVolume(volumeSlider.value);
        }
    }

    // ========================
    // 1. 核心菜单功能
    // ========================

    // Start Game 按钮点击时调用
    public void StartGame()
    {
        Debug.Log("加载主游戏场景：" + gameSceneName);
        // 通过场景名称加载场景，比使用索引号 1 更安全
        SceneManager.LoadScene(gameSceneName);
    }

    // Quit Game 按钮点击时调用
    public void QuitGame()
    {
        Debug.Log("退出游戏!");

        // 退出编辑器模式
#if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
        // 退出构建后的程序
#else
            Application.Quit();
#endif
    }

    // ========================
    // 2. 设置面板控制
    // ========================

    // Settings 按钮点击时调用 (开启面板)
    public void OpenSettings()
    {
        if (settingsPanel != null)
        {
            settingsPanel.SetActive(true); // 显示设置面板
        }
    }

    // Close/Back 按钮点击时调用 (关闭面板)
    public void CloseSettings()
    {
        if (settingsPanel != null)
        {
            settingsPanel.SetActive(false); // 隐藏设置面板
        }
    }

    // ========================
    // 3. 音量控制
    // ========================

    // Volume Slider 的 On Value Changed 事件调用此方法
    public void SetMasterVolume(float value)
    {
        // 音量滑块值 (value) 是线性值 (0.0001 - 1)。
        // Audio Mixer 需要对数分贝值 (-80 dB - 0 dB)。
        // 公式：20 * log10(值)

        float volume = Mathf.Log10(value) * 20;

        // 使用 Exposed Parameter 的名字将音量应用到 Audio Mixer
        gameMixer.SetFloat(exposedVolumeParam, volume);
    }
}