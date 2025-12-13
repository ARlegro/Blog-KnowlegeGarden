---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/Faster Whisper 사용/","noteIcon":"","created":"2025-12-03T14:52:50.335+09:00","updated":"2025-12-13T10:26:04.205+09:00"}
---



### 개념 및 특징 
음성 인식 모델인 Whisper를 그대로 사용하면서 성능을 최적화한 라이브러리 - `uv add faster-whisper`
- *속도*
	- 기본 Whisper보다 훨씬 빠름 ➡ **실시간 음성 인식, 긴 오디오 처리에서 속도 차이가 남** 
		  
- *메모리 효율*
	- 기본 Whisper보다 메모리 사용량이 적다(CPU, GPU 모두)		  
		  
- *FastAPI와 호환*




https://github.com/SYSTRAN/faster-whisper
- 이 사이트는 OpenAI의 위스퍼 모델을 `CTranslate2`와 함께 사용해 더 빠른 위스퍼로 변환하는 것 
- *모델 표 (CPU 기준)*
	- ![Pasted image 20250910140731.png](/img/user/supporter/image/Pasted%20image%2020250910140731.png)
	- CPU 모델에서는 2가지 정밀도 타입이 있다
		1. `FP16`
		2. `int 8`

## 사용법 

사전 : `faster-whisper` 패키지 다운로드

### Whisper 모델 속성 설정
```PYTHON 
# 깃허브 홈페이지에 있는 내용
from faster_whisper import WhisperModel

model_size = "large-v3"

# Run on GPU with FP16
model = WhisperModel(model_size, device="cuda", compute_type="float16")
```
1. *modle_size*
	- [깃허브 페이지](https://github.com/SYSTRAN/faster-whisper)에 들어간 뒤 여러 모델들 중 하나를 선택하면 된다.
	  
2. *device(장치 선택)* - 必
	- 모델을 실행할 H/W 장치를 지정하는 것 
	- 옵션 
		1. *cuda* or *gpu* : 가장 좋은 H/W장치
		2. *cpu* : CPU 장치
		3. *auto* : 사용 가능한 하드웨어를 자동으로 감지해 최적의 장치를 선택
	  
3. *compute_type(연산 정밀도)*
	- 모델 연산의 정밀도를 결정 ➡ 메모리 사용량과 속도에 영향을 미침
	- *옵션*
		1. *float16* : GPU 사용하는 경우 사용. 가장 일반적인 방법. float32에 비해 메모리 사용량을 절반으로 줄여 속도 빨라짐 
		2. *int8* : CPU 사용하는 경우 사용. 모델의 크기를 크게 줄여 메모리 사용량을 낮추고 속도를 빠르게 해줌 
		3. *auto* : 라이브러리가 자동으로 장치에 맞는 최적의 정밀도를 선택

![Pasted image 20250910150513.png](/img/user/supporter/image/Pasted%20image%2020250910150513.png)
(참고 : https://bcuts.tistory.com/286)

### 사용 설명서 전체 보기

```python
# 깃허브 홈페이지에 있는 내용
from faster_whisper import WhisperModel

model_size = "large-v3"

# Run on GPU with FP16
model = WhisperModel(model_size, device="cuda", compute_type="float16")

# or run on GPU with INT8
# model = WhisperModel(model_size, device="cuda", compute_type="int8_float16")
# or run on CPU with INT8
# model = WhisperModel(model_size, device="cpu", compute_type="int8")

segments, info = model.transcribe("audio.mp3", beam_size=5)

print("Detected language '%s' with probability %f" % (info.language, info.language_probability))

for segment in segments:
    print("[%.2fs -> %.2fs] %s" % (segment.start, segment.end, segment.text))
```


#### transcribe 메서드 
> 오디오 파일에서 텍스트를 추출하는 작업을 수행 

✔*주요 매개변수*
1. *audio ⭐*
	- 전사할 오디오 소스를 지정 
	- 지정 방법은 여러가지가 있다.
		- *파일 경로* ex. `"audio.mp3"`, `"audio.wav"`
		- *파일 객체* : 파이썬 io모듈의 파일객체를 직접 전달
		- *스트림* : 실시간 오디오 스트림 처리 by `sounddevice` 라이브러리
2. *language*
	- 명시하지 않아도 기본으로 자동 감지는 한다.
	- 하지만, 명시하면 자동 감지 과정이 생략되어 처리속도랑 정확도가 높아질 수 있다.
	- ex. "en", "ko"
	  
3. *beam_size*
	- 디코딩 과정에서 사용할 beam search의 크기 설정
	- beam search : 여러 개의 가능성 있는 텍스트 후보를 동시에 탐색하여 가장 정확한 결과를 찾아내는 알고리즘
	- *기본값* : 5 (가장 큰 값)
	- **값이 클수록** 탐색 범위가 넓어 **정확성은 높아지지만 속도는 느려짐**
4. *condition_on_previous_text*
	- 현재 오디오 조각을 전사할 때 **이전 조각에서 전사된 텍스트를 고려할지 여부를 설정**
	- *기본 값* : True

*✔반환값*
1. *segments* : 전사된 텍스트 세그먼트를 포함하는 제너레이터
2. *info* : 오디오 파일에 대한 메타데이터 담고 있는 객체 ex. 감지 언어, 언어 확률 


#### segments❓
> faster-whisper 모델이 Audio 파일을 전사한(transcribe) **결과를 담고 있는 generator** ➡ for문 사용 권장 

- 오디오는 여러 segment로 나뉜다.
- 각 segment는 아래과 같은 정보가 포함
	- 해당 seg 시작 시간(초)
	- 해당 seg 종료 시간(초)
	- 해당 seg 전사된 텍스트 


>[!QUESTION] 왜 generator타입인 이유 
>1. *메모리 절약 - Lazy*
>	- generator로 하면 한 번에 메모리에 올리는 것이 아니라 필요할 때마다 한 번에 하나의 segment를 생성하여 효율적 
>2. *실시간 처리*
>	- 세그먼트 즉시 생성하여 빠르게 처리 가능 

>[!QUESTION] for문 사용 권장이유
>list(segments)로 하면 모든 전사 작업을 한 번에 끝내고 결과를 리스트에 저장하기 때문에 메모리와 실시간성에서 딸린다. 따라서 for문 권장

### 더 효율적 모델 - Distil-Whisper

> **OpenAI Whisper 모델**의 크기와 복잡성을 줄인 경량화 모델

```python
from faster_whisper import WhisperModel

model_size = "distil-large-v3" # <-- 여기에서 모델을 변경
model = WhisperModel(model_size, device="cuda", compute_type="float16")
segments, info = model.transcribe(
    "audio.mp3", 
    beam_size=5, 
    language="en", 
    condition_on_previous_text=False # <-- 이전 텍스트에 의존하지 않도록 설정
)

for segment in segments:
    print("[%.2fs -> %.2fs] %s" % (segment.start, segment.end, segment.text))
```