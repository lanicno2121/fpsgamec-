using UnityEngine;
using System.Collections;

public class PuzzleDoor : MonoBehaviour
{
    [Header("大门设置 (有动画时)")]
    [Tooltip("如果大门有动画，拖入 Animator")]
    public Animator doorAnimator;
    public string openTriggerName = "Open";

    [Header("代码开门设置 (无动画时)")]
    [Tooltip("门要往哪个方向滑动多少米？(例如 Y: -4 就是往地下沉 4 米)")]
    public Vector3 moveOffset = new Vector3(0, -4f, 0);
    [Tooltip("开门滑动过程持续几秒？")]
    public float moveDuration = 2.0f;

    [Header("过场运镜设置")]
    public GameObject doorCamera;
    [Tooltip("等镜头飞过去要多久？(秒)")]
    public float cameraFlyTime = 1.5f;
    [Tooltip("门完全打开后，让玩家看多久大门？(秒)")]
    public float lookDuration = 2.5f;

    // 👇 ==========================================
    // 【新增】大门自动开启时的音效
    // ==========================================
    [Header("音效设置")]
    [Tooltip("大门开启时播放的轰隆声或机械运转声")]
    public AudioClip openSound;

    public void OpenDoor()
    {
        Debug.Log("【解谜大门】接收到开启信号，准备播放运镜与开门！");
        StartCoroutine(DoorCutsceneRoutine());
    }

    private IEnumerator DoorCutsceneRoutine()
    {
        // 1. 镜头切过去
        if (doorCamera != null) doorCamera.SetActive(true);
        yield return new WaitForSeconds(cameraFlyTime);

        // 👇 ==========================================
        // 【新增】正式开门前，播放轰隆隆的开门音效！
        // ==========================================
        if (openSound != null)
        {
            // 贴着摄像机播放，防止因为距离太远听不到声音
            AudioSource.PlayClipAtPoint(openSound, Camera.main.transform.position);
        }

        // 2. 正式开门
        if (doorAnimator != null)
        {
            doorAnimator.SetTrigger(openTriggerName);
            yield return new WaitForSeconds(lookDuration);
        }
        else
        {
            // 【全新升级：代码平滑移动】
            Vector3 startPos = transform.position;
            Vector3 targetPos = startPos + moveOffset;
            float timer = 0f;

            // 在设定的时间内，每一帧平滑移动一点点
            while (timer < moveDuration)
            {
                transform.position = Vector3.Lerp(startPos, targetPos, timer / moveDuration);
                timer += Time.deltaTime;
                yield return null; // 等待下一帧继续
            }

            // 确保门准确停在目标位置
            transform.position = targetPos;

            // 等门开完后，再让玩家欣赏一会 (如果看的时间比开门时间长，就等差值)
            float waitTime = lookDuration > moveDuration ? lookDuration - moveDuration : 0.5f;
            yield return new WaitForSeconds(waitTime);
        }

        // 3. 镜头切回玩家
        if (doorCamera != null) doorCamera.SetActive(false);

        // 此时门已经降入地下（或滑到墙里）了，玩家完全看不出了，再真正隐藏节省性能
        gameObject.SetActive(false);
    }
}