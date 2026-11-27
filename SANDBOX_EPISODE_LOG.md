# Ex Anima Sandbox 에피소드 기록

Combat Sandbox의 초기 시연과 규칙 기반 정책 실행을 재현하기 위한 TSV 형식이다. Core 데이터셋이나 모델 직렬화 규격은 Core 상세 설계에서 정한다.

## 파일 구성

`--record NEW_DIRECTORY`는 새 디렉터리를 만들고 에피소드별로 `episode-1.tsv`, `episode-2.tsv`를 기록한다. 같은 실행에서 플레이어가 재시작하면 번호가 증가하고 초기 시드는 유지된다. 저장 중에는 이름 끝에 `.partial`이 붙는다. 종료 레코드를 기록하고 파일을 닫은 뒤에만 `.tsv`로 이름을 바꾼다.

1. 메타데이터는 버전 표기, `seed=<uint32>`, `controller=<human|cautious|aggressive|external>`을 탭으로 구분한다. 새 로그에는 거리 판단 중심 전투 규칙을 뜻하는 `# sandbox-episode-v2`가 들어간다. 기존 `# sandbox-episode-v1`은 최초 전투 규칙으로 재생한다. 두 버전의 열과 종료 레코드는 동일하다. `external`은 [표준 입출력 정책](SANDBOX_DECISIONS.md#외부-정책-실행)의 기록이다.
2. 열 이름 행 뒤에 시뮬레이션 틱마다 한 행을 기록한다. UTF-8 호환 ASCII, 탭 구분, LF 줄바꿈을 사용한다.
3. 마지막 줄은 `# end`와 종료 사유를 탭으로 구분한다. 종료 사유는 `won`, `lost`, `limit`(틱 한도), `stopped`(정상 종료), `restart`다.

## 틱과 열의 의미

시뮬레이션은 초당 60틱을 처리한다. 첫 틱은 0이다. 각 행의 관측은 입력 적용 직전의 상태이며 좌표는 아레나 왼쪽 위가 원점이다. X는 오른쪽, Y는 아래쪽으로 증가한다. 접미사 `_cd`, `_ticks`와 상대의 준비·회복 시간은 남은 틱 수다.

| 순서 | 열 | 의미 |
| --- | --- | --- |
| 1–5 | `tick`, `move_x`, `move_y`, `requested`, `applied` | 이동 축은 각각 -1, 0, 1이다. 행동은 `none`, `melee`, `ranged`, `parry`, `dodge`다. 쿨다운이나 현재 동작 때문에 요청을 적용하지 못하면 `applied`에 `none`을 기록한다. 적용과 적중은 별개다. |
| 6–11 | `player_x`, `player_y`, `player_hp`, `enemy_x`, `enemy_y`, `enemy_hp` | 입력 적용 전 두 캐릭터의 좌표와 체력. |
| 12–19 | `melee_cd`, `ranged_cd`, `parry_cd`, `dodge_cd`, `parry_ticks`, `dodge_ticks`, `enemy_windup`, `enemy_recovery` | 캐릭터의 행동 재사용 대기 및 상태와 상대의 공격 준비·회복. |
| 20–26 | `projectile_ticks`, `projectile_x`, `projectile_y`, `projectile_vx`, `projectile_vy`, `dodge_x`, `dodge_y` | 투사체 상태, 틱당 이동 벡터와 회피 방향. 해당 상태가 비활성이면 관련 열은 0이다. |
| 27–31 | `damage_to_enemy`, `damage_to_player`, `parried`, `dodged`, `outcome` | 실제 체력 감소량, 패링 성공, 회피로 피격 무효화, 틱 처리 후 결과(`active`, `won`, `lost`). |
| 32–37 | `next_player_x`, `next_player_y`, `next_player_hp`, `next_enemy_x`, `next_enemy_y`, `next_enemy_hp` | 입력 적용 후 두 캐릭터의 좌표와 체력. |

`--replay EPISODE.tsv`는 헤더 버전에 맞는 전투 규칙을 선택하고, 시드와 각 행의 요청 입력으로 시뮬레이션을 재실행한다. 기록 행과 실제 결과가 다르거나 종료 레코드가 누락되면 오류를 반환한다. 빌드나 플랫폼을 바꾼 경우 부동소수점 결과가 달라질 수 있다. 실험에는 커밋, 형식 버전, 빌드 구성, 실행 명령과 시드를 함께 기록한다.
