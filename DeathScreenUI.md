using UnityEngine;
using UnityEngine.SceneManagement; // 必须有这一行，不然没法重置游戏

public class DeathScreenUI : MonoBehaviour
{
    // 点击“重新开始”按钮时执行
    public void OnRestartClick()
    {
        // 获取当前场景的名字，然后重新加载它
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }

    // 点击“退出游戏”按钮时执行
    public void OnQuitClick()
    {
        Debug.Log("正在退出游戏...");

        // 这行代码在编辑器里没反应，打包后才有用
        Application.Quit();

        // 为了让你在编辑器里也能看到效果，加这一句：
#if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
#endif
    }
}