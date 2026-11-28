# Core v0.1.0 오프라인 Behavioral Cloning 규격

현재 관측 벡터에서 단일 행동을 고르는 최소 경로다. 호출자는 관측 특성의 순서, 정규화와 라벨의 의미를 정의한다. 특성 순서와 정규화는 학습과 추론에서 일치해야 한다. 판단 주기와 요청 행동의 적용은 호출자에 속한다. [검증 대상 Core 소스](https://github.com/ex-anima/core/tree/a1f40ae007feb54e11c3c90fe5f276395e1dbaf0) · [v0.1.0 설계 승인](https://app.notion.com/p/4252b85101c881058e7ec46cad2aa0ef)

## 빌드와 테스트 기준

C++17과 CMake 3.20 이상을 기준으로 한다. v0.1.0의 검증 플랫폼은 Windows x64와 MSVC다. 확인한 환경은 Visual Studio 18 2026, MSVC 19.50.35722.0, CMake 4.1.1-msvc1, Release 구성이다. 다른 플랫폼과 CMake 최소 버전에서의 실행 지원은 해당 환경에서 검증한 뒤 확정한다.

행렬과 벡터는 표준 라이브러리와 자체 CPU 연산으로 처리한다. nlohmann/json 3.12.0은 JSON을 읽고 쓰는 구현에 사용한다. CMake의 `FetchContent`가 공식 소스 묶음의 SHA-256을 확인한다. 네트워크 없이 구성할 때는 같은 묶음을 `EXANIMA_JSON_ARCHIVE`에 지정한다. Core만 체크아웃해 빌드하고 CTest로 검증할 수 있다. [Core CMake 구성](https://github.com/ex-anima/core/blob/a1f40ae007feb54e11c3c90fe5f276395e1dbaf0/CMakeLists.txt)

```powershell
cmake -S . -B build/full -DEXANIMA_BUILD_TRAINING=ON -DBUILD_TESTING=ON
cmake --build build/full --config Release
ctest --test-dir build/full -C Release --output-on-failure
cmake -S . -B build/inference -DEXANIMA_BUILD_TRAINING=OFF -DBUILD_TESTING=OFF
cmake --build build/inference --config Release
```

전체 빌드의 `core_gradients_and_model_roundtrip`은 수치 미분과 역전파의 일치, 저장 전후 파라미터·로그잇 일치, 비유한 파라미터와 잘린 가중치 파일의 거부를 확인한다. 별도 추론 빌드에서는 훈련 실행 파일과 학습 라이브러리 대상이 생성되지 않는지 확인하고, 전체 빌드가 저장한 모델로 추론을 실행한다. [빌드와 테스트 기준 기록](https://app.notion.com/p/4252b85101c88159b8b3d7dba15e768f)

## C++ 공개 인터페이스

| CMake 대상 | 헤더 | 공개 인터페이스 |
| --- | --- | --- |
| `exanima::core` | `exanima/model.h` | `ModelSpec`, `Model(spec, parameters)`, `spec()`, `parameters()`, `logits(observation)`, `predict(observation)` |
| `exanima::training` | `exanima/training.h` | `Demonstration`, `TrainingOptions`, `TrainingResult`, `fit(spec, demonstrations, options)`, `save_model(model, directory)` |
| `exanima::inference` | `exanima/inference.h` | `load_model(directory)`와 공통 Core API |

세 대상은 정적 라이브러리다. 추론 대상은 공통 Core만 공개 링크한다. `Model::logits`는 라벨 순서의 출력 점수를 반환한다. `Model::predict`는 최고 점수의 0 기반 인덱스를 반환하며 동점이면 먼저 정의한 라벨을 고른다. `Demonstration`의 라벨은 0 기반 인덱스다. `fit`은 학습된 모델과 초기·최종 손실을 반환한다. 조밀 은닉층 하나, ReLU, 안정화한 Softmax 교차 엔트로피, 샘플 단위 SGD를 사용한다.

입력과 은닉층 크기는 각각 1~4,096, 출력은 2~4,096라벨이다. 파라미터 수는 `H*I + H + O*H + O`이며 상한은 1,000,000개다. 특성명과 라벨명은 각 목록 안에서 서로 달라야 하고 1~256바이트다. 관측은 특성 수와 같은 길이의 유한한 `float` 배열이다. C++ API는 잘못된 인자, 지원하지 않는 모델, 비유한 연산 결과, JSON 파싱 오류와 파일 실패를 예외로 전달한다. CLI는 실패 사유를 표준 오류에 출력하고 종료 코드 1을 반환한다.

v0.1.0의 공개 범위는 현재 관측, 조밀 은닉층 하나와 단일 선택 출력이다. 호출자는 정규화 및 행동 변환을 정의하고 Core는 라벨을 예측한다. Adam, 프레임 스태킹, 추가 출력 모드는 별도 설계와 검증이 필요하다. [공개 API와 JSON Schema 검토 기록](https://app.notion.com/p/4252b85101c881d9bbc5c0d48cb654c5)

## CLI 설정과 시연 데이터

`exanima-train CONFIG.json DEMONSTRATIONS.jsonl NEW_MODEL_DIRECTORY`는 시연 파일을 읽고 새 디렉터리에 모델을 저장한다. 필수 구성은 다음 예시의 첫 세 필드다. 학습 옵션을 생략하면 예시의 기본값을 사용한다.

```json
{
  "input_features": ["x", "y"],
  "hidden_units": 32,
  "labels": ["left", "right"],
  "seed": 42,
  "epochs": 200,
  "learning_rate": 0.03,
  "balance_classes": false
}
```

`epochs`는 1~10,000, 학습률은 `float`로 변환한 뒤 0 초과 1 이하다. `seed`는 uint32다. `balance_classes`가 참이면 클래스별 시연 건수의 제곱근 역수로 SGD 갱신을 가중하고 전체 시연에서 평균 가중치를 1로 맞춘다. 초기·최종 손실은 가중치 없는 평균 교차 엔트로피로 출력한다.

훈련 설정 파일은 UTF-8 JSON 객체이며 최대 4 MiB다. 시연 파일은 UTF-8 JSON Lines이며 한 행에 관측과 선택한 라벨을 담는다. 빈 행이나 빈 데이터셋은 오류다. 행의 관측 개수와 라벨은 훈련 설정에 맞아야 한다. `episode_id`, `step` 등 출처 추적용 추가 필드는 호출자가 관리할 수 있다. 데이터 준비 단계에서는 에피소드 또는 세션 경계에서 분할하고, 환경·시드와 전처리 버전을 별도로 남긴다. Sandbox는 원본 TSV에 요청·적용 행동을 보존하고 판단 데이터 변환에서 관측 시점과 라벨을 정의한다. [Sandbox 판단 데이터](https://github.com/ex-anima/docs/blob/9a069aca8ff250a4453f1467799f4277c8e0bdc7/SANDBOX_DECISIONS.md)

```jsonl
{"observation":[0.1,0.2],"decision":"left"}
{"observation":[0.2,0.4],"decision":"left"}
{"observation":[0.8,0.1],"decision":"right"}
{"observation":[0.9,0.25],"decision":"right"}
```

[훈련 설정 JSON Schema](schemas/core-training-config.schema.json)와 [시연 한 행의 JSON Schema](schemas/core-demonstration.schema.json)는 구조를 검사한다. Core는 정수 필드의 토큰, 이름의 UTF-8 바이트 길이, 특성 수와 관측 길이의 일치, 라벨 존재 여부, `float` 유한성, 파라미터 총수를 실행 시 검사한다. 그래서 `hidden_units: 2.0`, 중복 특성명, `[0.1,true]` 관측, 설정에 없는 라벨은 오류다. 필수 필드와 알려진 옵션 외의 키는 무시되며 동작을 바꾸지 않는다.

`exanima-infer MODEL_DIRECTORY`는 표준 입력의 JSON Lines 수치 배열을 순서대로 처리한다. 응답 행은 `index`, `label`, `logits`를 포함한 JSON 객체다. 형식 오류는 표준 오류로 알리고 종료 코드 1을 반환한다.

## 모델 파일

같은 디렉터리에 UTF-8 `model.json`과 `weights.bin`을 둔다. 메타데이터는 최대 4 MiB다. `format`은 `exanima-mlp-v1`, `dtype`은 `float32`, `byte_order`는 `little`, `parameter_order`는 `w1,b1,w2,b2`다. 나머지 필드는 `input_features`, `hidden_units`, `labels`, `parameter_count`다. [모델 메타데이터 JSON Schema](schemas/core-model-metadata.schema.json)

`weights.bin`에는 `W1[H,I]`, `b1[H]`, `W2[O,H]`, `b2[O]`가 행 우선 순서로 연속 저장된다. 헤더와 패딩 없이 각 값을 IEEE 754 binary32 리틀엔디언 4바이트로 기록한다. 예시 설정의 `I=2`, `H=32`, `O=2`에서 파라미터는 162개이고 파일은 648바이트다.

호환 모델은 지원 버전과 고정 형식을 사용하고, 파라미터 개수와 파일 길이가 정확히 맞으며 관측 순서 및 라벨이 호출자의 전처리·어댑터와 일치해야 한다. 로더는 알 수 없는 버전, 잘못된 이름과 형상, 개수 불일치, 잘린 파일, 추가 바이트, NaN과 무한대를 거부한다. 저장은 새 디렉터리에 바이너리를 기록한 뒤 `model.json.partial`을 `model.json`으로 바꿔 완료한다. 중단 후 남은 디렉터리는 별도로 보존하거나 정리하고 재시도에는 다른 출력 디렉터리를 사용한다.

```powershell
build/full/Release/exanima-train.exe config.json demonstrations.jsonl model
'[0.1,0.2]' | build/inference/Release/exanima-infer.exe model
```

동일 빌드에서 모델 저장·로드 후 파라미터와 출력 점수가 일치하는지 확인한다. 학습 재현에는 실행 파일과 소스 버전, 데이터 해시와 행 순서, 설정과 시드를 함께 기록한다. [데이터셋과 모델 파일 규격 검토 기록](https://app.notion.com/p/4252b85101c8819b8d53d1d6f7c78aa9) · [학습·독립 추론 검증 기록](https://app.notion.com/p/4252b85101c88131808fd8bffa100f48)
