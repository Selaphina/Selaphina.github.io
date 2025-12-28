UDP

```
using System.Collections;
using System.Collections.Generic;
using System.Diagnostics;
using UnityEngine;
using System.Text;
using System.Net.Sockets;
using System.Net;
using System;
using System.IO;
using UnityEngine.Networking;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using TMPro;

public class UDPSender : MonoBehaviour
{
    #region TTS模型设置
    [Header("聊天模型")]
    [SerializeField] public DeepSeekDialogueManager m_ChatModel;
    [Header("发音器官")]
    [SerializeField] private AudioSource m_AudioSource;
    #endregion

    #region STT Settings
    [Header("语音输入按钮")]
    [SerializeField] private Button m_VoiceInputButton;
    [Header("语音按钮文本")]
    [SerializeField] private TMP_Text m_VoiceButtonText;
    [Header("识别结果文本")]
    [SerializeField] private TMP_Text m_RecognizedText;
    [Header("语音输入框")]
    [SerializeField] public TMP_InputField m_InputWord;
    #endregion
    private string _lastSentMsg;
    private Process process;
    private UdpClient sendClient;
    private UdpClient receiveClient;
    private IPEndPoint pythonEP;
    private IPEndPoint unityEP;
    private int pythonProcessId = -1;
    private string uniqueId;

    void Start()
    {
        // 确保主线程调度器存在
        //UnityMainThreadDispatcher.Instance();
        _ = UnityMainThreadDispatcher.Instance;

        // 输入框连接一下
        m_InputWord = m_ChatModel.userInputField;
        
        // 生成唯一标识符
        uniqueId = $"TTStool_{Guid.NewGuid().ToString("N").Substring(0, 8)}";

        KillTargetPythonProcess();

        // 设置端点
        pythonEP = new IPEndPoint(IPAddress.Parse("127.0.0.1"), 31415);
        unityEP = new IPEndPoint(IPAddress.Any, 31416);

        // 创建UDP客户端
        sendClient = new UdpClient();
        receiveClient = new UdpClient(unityEP);
        receiveClient.BeginReceive(ReceiveCallback, null);

        StartPythonProcess();
        RegisterButtonEvents();

    }
    //20250614
    #region Voice Input // 语音输入区域
    [Header("语音识别脚本")]
    public STT m_SpeechToText;
    [SerializeField] private bool m_AutoSend = true; // 是否自动发送
    [SerializeField] private Button m_VoiceInputBotton; // 语音输入按钮
    [SerializeField] private Text m_VoiceBottonText; // 语音按钮文本
    //[SerializeField] private Text m_RecordTips; // 录音提示
    [SerializeField] private VoiceInputs m_VoiceInputs; // 语音输入组件


    // 注册按钮事件
    private void RegistButtonEvent()
    {
        if (m_VoiceInputBotton == null || m_VoiceInputBotton.GetComponent<EventTrigger>()) // 如果按钮为空或已经添加了事件触发器
            return; // 直接返回

        EventTrigger _trigger = m_VoiceInputBotton.gameObject.AddComponent<EventTrigger>(); // 为按钮添加事件触发器组件

        EventTrigger.Entry _pointDown_entry = new EventTrigger.Entry(); // 创建按下事件条目
        _pointDown_entry.eventID = EventTriggerType.PointerDown; // 设置事件类型为按下
        _pointDown_entry.callback = new EventTrigger.TriggerEvent(); // 初始化回调事件

        EventTrigger.Entry _pointUp_entry = new EventTrigger.Entry(); // 创建抬起事件条目
        _pointUp_entry.eventID = EventTriggerType.PointerUp; // 设置事件类型为抬起
        _pointUp_entry.callback = new EventTrigger.TriggerEvent(); // 初始化回调事件

        _pointDown_entry.callback.AddListener(delegate { StartRecord(); }); // 为按下事件添加开始录音回调
        _pointUp_entry.callback.AddListener(delegate { StopRecord(); }); // 为抬起事件添加停止录音回调

        _trigger.triggers.Add(_pointDown_entry); // 将按下事件条目添加到触发器中
        _trigger.triggers.Add(_pointUp_entry); // 将抬起事件条目添加到触发器中
    }

    // 开始录音
    public void StartRecord()
    {
        m_VoiceBottonText.text = "Recording..."; // 设置语音按钮文本为“正在录音”
        m_VoiceInputs.StartRecordAudio(); // 调用语音输入组件开始录音
    }

    // 停止录音
    public void StopRecord()
    {
        m_VoiceBottonText.text = "Hold to record"; // 设置语音按钮文本为“按住录音”
        //m_RecordTips.text = "Processing..."; // 设置录音提示为“正在处理”
        m_InputWord.text = "语音识别中..."; // 设置录音提示为“正在处理”
        m_VoiceInputs.StopRecordAudio(AcceptClip); // 调用语音输入组件停止录音并处理音频
    }

    // 发送消息
    public void SendData()
    {
        if (m_InputWord.text.Equals("")) // 如果输入框为空
            return; // 直接返回

        //if (m_CreateVoiceMode) // 如果是创建语音模式
        //{
        //    CallBack(m_InputWord.text); // 调用回调函数
        //    m_InputWord.text = ""; // 清空输入框
        //    return;
        //}

        //m_ChatHistory.Add(m_InputWord.text); // 将输入的消息添加到聊天历史中
        string _msg = m_InputWord.text; // 获取输入的消息

        m_ChatModel.OnSendMessage(_msg); // 调用聊天模型发送消息

        m_InputWord.text = ""; // 清空输入框
    }

    // 接收音频剪辑
    private void AcceptClip(AudioClip _audioClip)
    {
        if (m_SpeechToText == null) // 如果没有语音到文本组件
            return; // 直接返回

        m_SpeechToText.SpeechToText(_audioClip, DealingTextCallback); // 调用语音到文本组件将音频转换为文本并设置回调
    }

    // 处理文本回调
    private void DealingTextCallback(string _msg)
    {
        //m_RecordTips.text = _msg; // 设置录音提示为转换后的文本
        //m_RecordTips.text = _msg; // 设置录音提示为转换后的文本
        //StartCoroutine(SetTextVisible(m_RecordTips)); // 开始协程设置文本可见性

        if (m_AutoSend) // 如果自动发送
        {
            m_ChatModel.OnSendMessage(_msg); // 发送转换后的文本
            return;
        }

        m_InputWord.text = _msg; // 将转换后的文本设置为输入框内容
    }

    // 设置文本可见性协程
    private IEnumerator SetTextVisible(Text _textbox)
    {
        yield return new WaitForSeconds(3f); // 等待3秒
        _textbox.text = ""; // 清空文本
    }
    #endregion
//20250614
    void StartPythonProcess()
    {
        string pythonPath = Path.Combine(Application.dataPath, "PY", "TTStool.py");
        if (!File.Exists(pythonPath))
        {
            UnityEngine.Debug.LogError($"Python脚本不存在: {pythonPath}");
            return;
        }

        ProcessStartInfo startInfo = new ProcessStartInfo();
        startInfo.FileName = GetPythonExecutable();
        startInfo.Arguments = $"\"{pythonPath}\" --unique-id \"{uniqueId}\"";
        startInfo.CreateNoWindow = true;
        startInfo.UseShellExecute = false;
        startInfo.RedirectStandardOutput = true;
        startInfo.RedirectStandardError = true;
        startInfo.StandardOutputEncoding = Encoding.UTF8;
        startInfo.StandardErrorEncoding = Encoding.UTF8;

        process = new Process();
        process.StartInfo = startInfo;
        process.EnableRaisingEvents = true;

        // 处理输出
        process.OutputDataReceived += (s, e) => {
            if (!string.IsNullOrEmpty(e.Data))
            {
                UnityEngine.Debug.Log($"Python输出: {e.Data}");

                // 尝试从输出中提取进程ID
                if (e.Data.StartsWith("TTStool PID:"))
                {
                    if (int.TryParse(e.Data.Split(':')[1].Trim(), out int pid))
                    {
                        pythonProcessId = pid;
                        UnityEngine.Debug.Log($"记录目标Python进程ID: {pid}");
                    }
                }
            }
        };

        // 处理错误
        process.ErrorDataReceived += (s, e) => {
            if (!string.IsNullOrEmpty(e.Data))
                UnityEngine.Debug.LogError($"Python错误: {e.Data}");
        };

        // 进程退出时处理
        process.Exited += (s, e) => {
            UnityEngine.Debug.Log("Python进程已退出");
            pythonProcessId = -1;
        };

        // 启动进程
        try
        {
            process.Start();
            process.BeginOutputReadLine();
            process.BeginErrorReadLine();
            UnityEngine.Debug.Log($"启动Python进程: {process.Id}");

            // 记录进程ID作为备选
            pythonProcessId = process.Id;
        }
        catch (Exception e)
        {
            UnityEngine.Debug.LogError($"启动Python进程失败: {e.Message}");
        }
    }

    private string GetPythonExecutable()
    {
        // 根据平台选择合适的Python可执行文件
        if (Application.platform == RuntimePlatform.WindowsEditor ||
            Application.platform == RuntimePlatform.WindowsPlayer)
        {
            return "python.exe";
        }
        else
        {
            return "python3";
        }
    }

    private void KillTargetPythonProcess()
    {
        // 方法1: 使用记录的进程ID
        if (pythonProcessId > 0)
        {
            try
            {
                Process targetProc = Process.GetProcessById(pythonProcessId);
                if (!targetProc.HasExited)
                {
                    UnityEngine.Debug.Log($"结束目标Python进程: {pythonProcessId}");
                    targetProc.Kill();
                    targetProc.WaitForExit(1000);
                }
            }
            catch (ArgumentException)
            {
                // 进程不存在
                UnityEngine.Debug.Log($"目标进程 {pythonProcessId} 不存在");
            }
            catch (Exception ex)
            {
                UnityEngine.Debug.LogWarning($"结束目标进程失败: {ex.Message}");
            }
            finally
            {
                pythonProcessId = -1;
            }
        }

        // 方法2: 使用进程名称和命令行参数匹配
        Process[] allProcesses = Process.GetProcessesByName(GetPythonExecutable().Replace(".exe", ""));
        foreach (Process proc in allProcesses)
        {
            try
            {
                if (proc.Id == Process.GetCurrentProcess().Id) continue;

                // 使用更简单的方法获取命令行参数
                string commandLine = GetProcessCommandLineSimple(proc);
                if (!string.IsNullOrEmpty(commandLine) &&
                    commandLine.Contains($"--unique-id \"{uniqueId}\""))
                {
                    UnityEngine.Debug.Log($"结束关联Python进程: {proc.Id}");
                    proc.Kill();
                    proc.WaitForExit(1000);
                }
            }
            catch (Exception ex)
            {
                UnityEngine.Debug.LogWarning($"结束进程失败: {ex.Message}");
            }
        }
    }

    // 简单的进程命令行获取方法（不依赖WMI）
    private string GetProcessCommandLineSimple(Process process)
    {
        try
        {
            // 在Windows上，我们可以尝试获取进程启动信息
            if (Application.platform == RuntimePlatform.WindowsEditor ||
                Application.platform == RuntimePlatform.WindowsPlayer)
            {
                // 注意：这只能获取当前进程启动的进程信息
                if (process.StartInfo != null)
                {
                    return process.StartInfo.Arguments;
                }
            }

            // 其他平台或无法获取的情况，返回空
            return string.Empty;
        }
        catch
        {
            return string.Empty;
        }
    }

    private void ReceiveCallback(System.IAsyncResult result)
    {
        try
        {
            IPEndPoint remoteEP = new IPEndPoint(IPAddress.Any, 0);
            byte[] data = receiveClient.EndReceive(result, ref remoteEP);
            string audioPath = Encoding.UTF8.GetString(data);

            // // 使用单例调主线程调度器处理:原本是:Instance().xxx

            UnityMainThreadDispatcher.Instance.Enqueue(() =>
            {
                UnityEngine.Debug.Log($"收到音频路径: {audioPath}");
                StartCoroutine(LoadAndPlayAudio(audioPath));
            });

            // 继续监听
            receiveClient.BeginReceive(ReceiveCallback, null);
        }
        catch (ObjectDisposedException)
        {
            // 正常关闭
        }
        catch (Exception e)
        {
            UnityEngine.Debug.LogError($"接收回调错误: {e.Message}");
        }
    }

    //private void ReceiveCallback(System.IAsyncResult result)
    //{
    //    try
    //    {
    //        IPEndPoint remoteEP = new IPEndPoint(IPAddress.Any, 0);
    //        byte[] data = receiveClient.EndReceive(result, ref remoteEP);
    //        string message = Encoding.UTF8.GetString(data);

    //        UnityMainThreadDispatcher.Instance().Enqueue(() => {
    //            if (message.StartsWith("STT_RESULT:"))
    //            {
    //                string recognizedText = message.Substring("STT_RESULT:".Length);
    //                m_RecognizedText.text = recognizedText;
    //                UnityEngine.Debug.Log($"识别结果: {recognizedText}");
    //                StartCoroutine(ClearRecognizedText());
    //            }
    //            else
    //            {
    //                UnityEngine.Debug.Log($"收到音频路径: {message}");
    //                StartCoroutine(LoadAndPlayAudio(message));
    //            }
    //        });

    //        receiveClient.BeginReceive(ReceiveCallback, null);
    //    }
    //    catch (ObjectDisposedException)
    //    {
    //    }
    //    catch (Exception e)
    //    {
    //        UnityEngine.Debug.LogError($"接收回调错误: {e.Message}");
    //    }
    //}


    IEnumerator LoadAndPlayAudio(string filePath)
    {
        if (string.IsNullOrEmpty(filePath))
        {
            UnityEngine.Debug.LogError("音频路径为空");
            yield break;
        }

        // 处理Windows路径
        //string uri = filePath.Replace("\\", "/");
        //if (!uri.StartsWith("file://"))
        //{
        //    uri = "file:///" + uri;
        //}
        string uri = new Uri(filePath).AbsoluteUri;

        if (!File.Exists(filePath))
        {
            UnityEngine.Debug.LogError($"音频文件不存在: {filePath}");
            yield break;
        }

        UnityEngine.Debug.Log($"加载音频: {uri}");

        using (UnityWebRequest www = UnityWebRequestMultimedia.GetAudioClip(uri, AudioType.WAV))
        {
            yield return www.SendWebRequest();

            if (www.result != UnityWebRequest.Result.Success)
            {
                UnityEngine.Debug.LogError($"音频加载失败: {www.error}\nURL: {uri}");
                yield break;
            }

            AudioClip clip = DownloadHandlerAudioClip.GetContent(www);
            if (clip == null)
            {
                UnityEngine.Debug.LogError("无法创建音频剪辑");
                yield break;
            }

            m_AudioSource.clip = clip;
            m_AudioSource.Play();
            UnityEngine.Debug.Log("播放音频");
        }
    }

    void Update()
    {
        //if (Input.GetMouseButtonDown(1))
        //{
        //    SendTextToPython();
        //}
        //--------------------------------2025-1228
        // 自动发：只要检测到新内容就发
        if (m_ChatModel != null &&
            !string.IsNullOrEmpty(m_ChatModel.botMessage) &&
            m_ChatModel.botMessage != _lastSentMsg)
        {
            SendTextToPython();
            _lastSentMsg = m_ChatModel.botMessage;
        }

    }

    public void SendTextToPython()
    {
        if (m_ChatModel == null || string.IsNullOrEmpty(m_ChatModel.botMessage))
        {
            UnityEngine.Debug.LogWarning("没有有效的消息可发送");
            return;
        }

        string message = m_ChatModel.botMessage;
        _lastSentMsg = m_ChatModel.botMessage;
        byte[] data = Encoding.UTF8.GetBytes(message);

        try
        {
            sendClient.Send(data, data.Length, pythonEP);
            UnityEngine.Debug.Log($"发送到Python: {message}");
        }
        catch (Exception e)
        {
            UnityEngine.Debug.LogError($"发送失败: {e.Message}");
        }
    }

    //public void SendTextToPython()
    //{
    //    if (m_ChatModel == null || string.IsNullOrEmpty(m_ChatModel.botMessage))
    //    {
    //        UnityEngine.Debug.LogWarning("没有有效的消息可发送");
    //        return;
    //    }

    //    string message = m_ChatModel.botMessage;
    //    SendDataToPython("TTS_TEXT:" + message);
    //    //SendDataToPython(message);
    //}

    //private void SendDataToPython(string message)
    //{
    //    try
    //    {
    //        byte[] data = Encoding.UTF8.GetBytes(message);
    //        sendClient.Send(data, data.Length, pythonEP);
    //        UnityEngine.Debug.Log($"发送到Python: {message}");
    //    }
    //    catch (Exception e)
    //    {
    //        UnityEngine.Debug.LogError($"发送失败: {e.Message}");
    //    }
    //}

    void OnApplicationQuit()
    {
        UnityEngine.Debug.Log("清理资源...");

        if (sendClient != null)
        {
            sendClient.Close();
            sendClient = null;
        }

        if (receiveClient != null)
        {
            receiveClient.Close();
            receiveClient = null;
        }

        KillTargetPythonProcess();

        if (process != null && !process.HasExited)
        {
            try
            {
                UnityEngine.Debug.Log($"结束Python进程句柄: {process.Id}");
                process.Kill();
                process.WaitForExit(1000);
            }
            catch (Exception ex)
            {
                UnityEngine.Debug.LogWarning($"结束进程失败: {ex.Message}");
            }
            finally
            {
                process.Dispose();
            }
        }
    }

    private void RegisterButtonEvents()
    {
        if (m_VoiceInputButton == null) return;
        EventTrigger trigger = m_VoiceInputButton.gameObject.AddComponent<EventTrigger>();
        EventTrigger.Entry pointerDown = new EventTrigger.Entry { eventID = EventTriggerType.PointerDown };
        pointerDown.callback.AddListener((data) => { StartRecord(); });
        trigger.triggers.Add(pointerDown);
        EventTrigger.Entry pointerUp = new EventTrigger.Entry { eventID = EventTriggerType.PointerUp };
        pointerUp.callback.AddListener((data) => { StopRecord(); });
        trigger.triggers.Add(pointerUp);
    }


    private IEnumerator ClearRecognizedText()
    {
        yield return new WaitForSeconds(5f);
        m_RecognizedText.text = "";
    }
}

```



ttsTOOL.PY

```
import socket
import sys
import io
import traceback
import logging
import argparse
import os
import time
import psutil
from gradio_client import Client, handle_file

# 强制设置标准输出和标准错误的编码为 UTF-8
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
sys.stderr = io.TextIOWrapper(sys.stderr.buffer, encoding='utf-8')
# 确定ref的位置
script_dir = os.path.dirname(os.path.abspath(__file__))
ref_wav_path = os.path.join(script_dir, '此外，《沉秋拾剑录》的新刊会在容彩祭上发售，我们即使身为作者也没有收到样刊，你为什么能说出插图的细节？.wav')
# 配置命令行参数
parser = argparse.ArgumentParser(description='TTStool for Unity TTS')
parser.add_argument('--unity-app', type=str, help='Unity application name')
parser.add_argument('--unique-id', type=str, help='Unique identifier for this process')
args = parser.parse_args()

# 配置详细的日志记录
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout)
    ],
    encoding='utf-8'  # 明确指定编码为 UTF-8
)
logger = logging.getLogger(__name__)

# 打印进程信息
current_pid = os.getpid()
logger.info(f"TTStool PID: {current_pid}")  # Unity 将捕获此行获取PID
if args.unity_app:
    logger.info(f"关联Unity应用: {args.unity_app}")
if args.unique_id:
    logger.info(f"唯一标识符: {args.unique_id}")

# 设置端口
HOST = '127.0.0.1'
PYTHON_RECV_PORT = 31415
UNITY_RECV_PORT = 31416

# 创建发送到Unity的socket
unity_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)


def init_gradio_client():
    """初始化Gradio客户端，重试机制"""
    max_retries = 3  # 增加重试次数
    for attempt in range(max_retries):
        try:
            logger.info(f"尝试连接Gradio服务 (尝试 {attempt + 1}/{max_retries})...")
            client = Client("http://localhost:9872/")

            # 检查是否真正连接成功
            if client is not None:
                logger.info("成功连接到Gradio服务")
                return client
            else:
                logger.warning("客户端初始化成功，但未检测到有效连接，等待重试...")
        except Exception as err:
            logger.error(f"连接失败: {str(err)}")
            logger.error("完整堆栈跟踪:\n" + traceback.format_exc())

        # 增加重试间隔时间
        if attempt < max_retries - 1:
            logger.info("10秒后重试...")
            time.sleep(10)  # 增加重试间隔时间
    raise ConnectionError("无法连接到Gradio服务")


import shutil
import uuid

def process_text_message(message):
    try:
        logger.info(f"开始处理消息: '{message}'")

        client = init_gradio_client()

        result = client.predict(
            ref_wav_path=handle_file(ref_wav_path),
            prompt_text="此外，《沉秋拾剑录》的新刊会在容彩祭上发售，我们即使身为作者也没有收到样刊，你为什么能说出插图的细节？",
            prompt_language="中文",
            text=message,
            text_language="中文",
            how_to_cut="凑四句一切",
            top_k=5,
            top_p=1,
            temperature=1,
            ref_free=False,
            speed=1,
            if_freeze=False,
            inp_refs=None,
            sample_steps="32",
            if_sr=False,
            pause_second=0.3,
            api_name="/get_tts_wav"
        )

       

        gradio_audio = result[0]

        audio_path = result  # result 本身就是 string

        if not isinstance(audio_path, str):
            raise TypeError(f"predict 返回值不是字符串: {type(audio_path)}")

        if not os.path.exists(audio_path):
            raise FileNotFoundError(f"Gradio 返回路径不存在: {audio_path}")

        # 可选：拷贝到你自己的目录（推荐）
        output_dir = os.path.join(script_dir, "tts_output")
        os.makedirs(output_dir, exist_ok=True)

        out_wav = os.path.join(output_dir, os.path.basename(audio_path))
        shutil.copy(audio_path, out_wav)

        out_wav = os.path.abspath(out_wav).replace("\\", "/")
        logger.info(f"最终输出音频: {out_wav}")
        return out_wav


        # ===== 关键修复结束 =====

    except Exception as e:
        logger.error(f"处理过程中出错: {str(e)}")
        logger.error("完整堆栈跟踪:\n" + traceback.format_exc())
        return None



def main():
    """主监听循环"""
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.bind((HOST, PYTHON_RECV_PORT))
        logger.info(f"监听Unity消息 (端口 {PYTHON_RECV_PORT})")
        sys.stdout.flush()

        # 设置心跳检查
        last_activity = time.time()

        while True:
            try:
                # 设置超时以允许定期检查
                s.settimeout(5.0)

                # 接收Unity消息
                try:
                    data, addr = s.recvfrom(1024)
                    message = data.decode('utf-8', errors='ignore')
                    logger.info(f"收到来自Unity的消息: '{message}'")
                    last_activity = time.time()
                except socket.timeout:
                    # 检查Unity是否仍在运行
                    if args.unity_app and not is_unity_running(args.unity_app):
                        logger.info("Unity应用已退出，终止TTStool")
                        return
                    continue

                # 处理消息并生成音频
                audio_path = process_text_message(message)

                if audio_path:
                    # 发送音频路径回Unity
                    try:
                        unity_socket.sendto(audio_path.encode('utf-8'), (HOST, UNITY_RECV_PORT))
                        logger.info(f"发送音频路径到Unity: '{audio_path}'")
                    except Exception as e:
                        logger.error(f"发送回Unity失败: {str(e)}")

                sys.stdout.flush()
            except Exception as e:
                logger.error(f"主循环错误: {str(e)}")
                logger.error("完整堆栈跟踪:\n" + traceback.format_exc())


def is_unity_running(app_name):
    """检查Unity应用是否仍在运行"""
    try:
        for proc in psutil.process_iter(['name']):
            if proc.info['name'] and app_name.lower() in proc.info['name'].lower():
                return True
    except Exception:
        pass
    return False


if __name__ == "__main__":
    try:
        logger.info("TTStool 启动")
        main()
    except KeyboardInterrupt:
        logger.info("程序被用户中断")
    except Exception as e:
        logger.error(f"致命错误: {str(e)}")
        logger.error("完整堆栈跟踪:\n" + traceback.format_exc())
        sys.exit(1)
```

