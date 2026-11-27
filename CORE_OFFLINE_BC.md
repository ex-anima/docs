# Core 오프라인 Behavioral Cloning v1

현재 관측 벡터에서 단일 행동 범주를 예측하는 최소 경로다. Sandbox가 관측 특성과 행동의 의미를 정의한다. Core는 벡터의 순서와 이름, 라벨의 순서를 사용하며 게임 엔진에 의존하지 않는다. [학습·독립 추론 검증 카드](https://app.notion.com/p/4252b85101c88131808fd8bffa100f48)

## C++ 인터페이스와 빌드

- `exanima::core`는 모델 구조와 CPU 순전파를 제공한다. `ModelSpec`에 입력 특성명, 은닉층 크기, 라벨명을 넘긴다. `Model::logits`는 라벨 순서의 출력 점수, `Model::predict`는 최대 점수의 인덱스를 반환한다. 동점일 때 먼저 정의한 라벨을 선택한다.
- `exanima::training`은 `fit(spec, demonstrations, options)`과 `save_model(model, directory)`를 제공한다. 시연은 관측 벡터와 0부터 시작하는 라벨 인덱스로 구성한다. 학습은 조밀 은닉층 하나, ReLU, 최대 로그잇을 뺀 Softmax 교차 엔트로피, 샘플 단위 SGD를 사용한다.
- `exanima::inference`는 `load_model(directory)`를 제공한다. `-DEXANIMA_BUILD_TRAINING=OFF`로 구성하면 Core와 모델 로드·추론 대상만 생성한다. 학습 대상의 함수는 추론에 사용되지 않는다.

관측은 구성과 같은 길이의 유한한 `float` 배열이다. 입력·은닉·출력 크기는 각각 4,096 이하이며, 출력은 최소 두 라벨이다. 층과 편향을 합한 파라미터는 최대 1,000,000개다. 특성명 또는 라벨명이 비어 있거나 중복되면 오류를 반환한다. C++ 함수는 잘못된 입력과 I/O 실패에 예외를 발생시킨다.

## CLI와 시연 형식

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

`epochs`는 1–10,000, 학습률은 0 초과 1 이하다. `seed`는 uint32다. `balance_classes`가 참이면 클래스별 시연 건수의 제곱근 역수로 SGD 갱신을 가중하고 전체 시연에서 평균 가중치를 1로 맞춘다. 초기·최종 손실은 가중치 없는 평균 교차 엔트로피로 출력한다. 예시 시연 행은 `{"observation":[0.1,0.2],"decision":"left"}`다. Sandbox 추적 필드는 `sandbox-decisions-v1`에 정의되어 있다. [Sandbox 판단 데이터](SANDBOX_DECISIONS.md)

`exanima-infer MODEL_DIRECTORY`는 표준 입력의 JSON Lines 수치 배열을 순서대로 처리한다. 응답 행은 `index`, `label`, `logits`를 포함한 JSON 객체다. 형식 오류는 표준 오류로 알리고 종료 코드 1을 반환한다.

## 모델 파일

같은 디렉터리에 `model.json`과 `weights.bin`을 둔다. 메타데이터의 `format`은 `exanima-mlp-v1`, `dtype`은 `float32`, `byte_order`는 `little`, `parameter_order`는 `w1,b1,w2,b2`다. 나머지 필드는 `input_features`, `hidden_units`, `labels`, `parameter_count`다.

`weights.bin`은 IEEE 754 binary32로, `W1[hidden,input]`, `b1[hidden]`, `W2[classes,hidden]`, `b2[classes]`를 행 우선 순서로 연속 저장한다. 로드는 버전, 값의 형상, 각 이름의 유효성, 파라미터 개수, 정확한 파일 길이와 유한성을 확인한다. 쓰기는 새 디렉터리에서 시작하고 바이너리를 닫은 뒤 메타데이터의 최종 이름을 게시한다.

동일 빌드에서 모델 저장·로드 후 파라미터와 출력 점수가 일치하는지 확인한다. 학습 재현에는 실행 파일과 소스 버전, 데이터 해시와 행 순서, 설정과 시드를 함께 기록한다.
