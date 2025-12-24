```
from gradio_client import Client, file

client = Client("http://localhost:9872/")
result = client.predict(
		ref_wav_path=handle_file(ref_wav_path),
		prompt_text=message,
		prompt_language="英文",
		text="Good afternoon. I heard your footsteps. My, it certainly is lively outside of the workshop",
		text_language="英文",
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
print(result)
```

元

```
def process_text_message(message):
    """处理文本消息并生成语音"""
    try:
        logger.info(f"开始处理消息: '{message}'")

        # 初始化客户端
        client = init_gradio_client()

        # 语音合成
        result = client.predict(
            ref_wav_path=handle_file(ref_wav_path),
            prompt_text=message,
            prompt_language="英文",
            text="Good afternoon. I heard your footsteps. My, it certainly is lively outside of the workshop",
            text_language="英文",
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

        audio_path = result[0]
        logger.info(f"生成音频路径: {audio_path}")
        return audio_path
    except Exception as e:
        logger.error(f"处理过程中出错: {str(e)}")
        logger.error("完整堆栈跟踪:\n" + traceback.format_exc())
        return None
```

