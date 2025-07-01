

✅ 전제 조건: CSV 파일 구성

📁 회의록 요약 + Status CSV (meeting_df)

회의일 | EMD | 팀 | 과제명 | 회의 요약 | 액션 아이템 | LLM zero-shot status | 검토 status

📁 과제 발의서 CSV (proposal_df)

EMD | 팀 | 과제명 | 목적 | 배경 필요성 | 이전 지표 | 개선 목표 지표


---

✅ Python 자동화 코드 (pandas + 사내 LLM API 호출)
```
import pandas as pd
from tqdm import tqdm

# 예시용 데이터프레임 (실제 환경에서는 meeting_df, proposal_df를 merge한 merged_df 사용)
# merged_df = pd.merge(meeting_df, proposal_df, on=['EMD', '팀', '과제명'], how='left')

# 프롬프트 생성 함수 (출력 예시 포함)
def generate_prompt(row):
    prompt = f"[과제명] {row['과제명']}\n[소속 팀] {row['팀']}\n"
    
    if pd.notna(row.get('개선 목표 지표', '')):
        prompt += f"[과제 목적] {row.get('목적', '')}\n"
        prompt += f"[배경 및 필요성] {row.get('배경 필요성', '')}\n"
        prompt += f"[이전 지표] {row.get('이전 지표', '')}\n"
        prompt += f"[개선 목표 지표] {row.get('개선 목표 지표', '')}\n"

    if pd.notna(row.get('이전_요약', '')):
        prompt += f"\n[이전 회차 요약 및 상태]\n- {row['이전_요약']} → {row['이전_상태']}\n"

    prompt += f"\n[현재 회의 요약] ({row['회의일'].date()})\n{row['회의 요약']}\n"
    
    if pd.notna(row.get('액션 아이템', '')):
        prompt += f"\n[액션 아이템]\n{row['액션 아이템']}\n"

    # 출력 예시 포함
    prompt += """
질문: 이 과제의 현재 상태를 아래 중 하나로 판단하고, 간단히 이유를 설명해주세요.
→ ['on-track', 'at-risk', 'delayed', 'completed']

[출력 형식 예시]
Status: at-risk
Reason: 목표 수율에 도달하지 못했고, 개선 방향이 구체화되지 않음
"""
    return prompt

# tqdm 적용
tqdm.pandas(desc="LLM 프롬프트 생성 중")
merged_df['LLM_프롬프트'] = merged_df.progress_apply(generate_prompt, axis=1)

# 사내 LLM 호출 함수 예시 (모의 응답 형태)
def call_llm(prompt: str) -> str:
    # 실제로는 API 호출해야 함
    # 예시 출력:
    return "Status: on-track\nReason: 목표 수율 도달했고, 액션 아이템 진행 중"

# LLM 호출 및 응답 저장
tqdm.pandas(desc="LLM 호출 중")
merged_df['LLM_raw_response'] = merged_df['LLM_프롬프트'].progress_apply(call_llm)

# 결과 파싱 함수
def parse_status(response):
    lines = response.splitlines()
    status_line = next((l for l in lines if l.lower().startswith("status:")), "")
    reason_line = next((l for l in lines if l.lower().startswith("reason:")), "")
    status = status_line.replace("Status:", "").strip()
    reason = reason_line.replace("Reason:", "").strip()
    return pd.Series([status, reason], index=["LLM_status3", "LLM_reason"])

# 응답 파싱 결과 분리
parsed_df = merged_df['LLM_raw_response'].apply(parse_status)
merged_df = pd.concat([merged_df, parsed_df], axis=1)
```
---

✅ 출력 예시 프롬프트

[과제명] GAA 트랜지스터 특성 정합
[소속 팀] 소자개발팀
[과제 목적] 고속 동작을 위한 저전압 동작 특성 확보
[배경 및 필요성] 기존 Bulk 구조에서 FinFET으로 전환 중, 특성 재현이 어려움
[이전 지표] Vth 편차 ±60mV
[개선 목표 지표] Vth 편차 ±20mV 이내

[이전 회차 요약 및 상태]
- 공정 조건 개선 실험 중, 편차 ±40mV 수준 → at-risk

[현재 회의 요약] (2025-06-28)
측정기 업그레이드 이후 재실험 결과 평균 ±22mV 도달. 고객 validation 필요

[액션 아이템]
고객 시료 제출 및 피드백 대기

질문: 이 과제의 현재 상태를 아래 중 하나로 판단하고, 간단히 이유를 설명해주세요.
→ ['on-track', 'at-risk', 'delayed', 'completed']


---

✅ 요약 정리

구성 요소	활용 방식

회의 요약	상태 판단의 핵심 입력
이전 회차 요약	흐름 파악, 진척/지연 여부 판단
발의서의 목표 지표	정량적 판단 기준 제공
목적/배경	정성적 방향성과 맥락 제공
액션 아이템	후속 조치 여부 판단 보조

---

 
과제의 전체 흐름을 보고 판단

지속적인 위험 요인 또는 진전 패턴을 파악

상태 변화(예: at-risk → on-track → completed)의 맥락 이해



---

✅ 코드 구조: 전체 회의 이력 누적 요약 생성

```
def generate_full_history(df):
    df = df.sort_values(by=["과제명", "회의일"])
    all_histories = []

    for name, group in df.groupby('과제명'):
        cumulative = []
        history_list = []

        for i in range(len(group)):
            # 0 ~ i까지의 회차 누적
            past = group.iloc[:i]
            history = '\n'.join(
                f"- {row['회의일'].date()}: {row['회의 요약']} → {row['검토 status']}"
                for _, row in past.iterrows()
            )
            history_list.append(history)
        group['전체_이력_요약'] = history_list
        all_histories.append(group)

    return pd.concat(all_histories)

```
전체_이력_요약: 지금까지 해당 과제에서 진행된 모든 회의 요약 + 상태를 누적한 문자열

LLM 프롬프트에 이걸 넣으면 회고적 상태 판단이 가능



---

✅ 프롬프트 구성 (전체 이력 포함)

```
def generate_prompt(row):
    prompt = f"[과제명] {row['과제명']}\n[소속 팀] {row['팀']}\n"

    if pd.notna(row.get('개선 목표 지표', '')):
        prompt += f"[과제 목적] {row.get('목적', '')}\n"
        prompt += f"[배경 및 필요성] {row.get('배경 필요성', '')}\n"
        prompt += f"[이전 지표] {row.get('이전 지표', '')}\n"
        prompt += f"[개선 목표 지표] {row.get('개선 목표 지표', '')}\n"

    if pd.notna(row.get('전체_이력_요약', '')):
        prompt += f"\n[과거 회의 요약 및 상태 이력]\n{row['전체_이력_요약']}\n"

    prompt += f"\n[현재 회의 요약] ({row['회의일'].date()})\n{row['회의 요약']}\n"

    if pd.notna(row.get('액션 아이템', '')):
        prompt += f"\n[액션 아이템]\n{row['액션 아이템']}\n"

    prompt += """
질문: 이 과제의 현재 상태를 아래 중 하나로 판단하고, 간단히 이유를 설명해주세요.
→ ['on-track', 'at-risk', 'delayed', 'completed']

[출력 형식 예시]
Status: on-track
Reason: 목표 조건 달성 중이며, 남은 액션이 명확히 진행 중
"""
    return prompt

```
---

✅ 주의할 점

항목	설명

📏 길이 초과	LLM에 따라 입력 길이 제한이 있으므로, 20~30회차 이상이면 자르거나 요약 필요
🧠 LLM의 장점	긴 문맥에서의 흐름 파악, 점진적 변화 감지
🧪 성능	기존 1~2회 기반 모델보다 예측 신뢰도 크게 향상 가능



---

✅ 전체 흐름

1. 회의일 기준으로 과제별 정렬


2. 회차별로 이전까지의 전체 요약 + 상태 누적


3. 발의서 정보 + 현재 요약 + 액션 아이템 포함


4. call_llm(prompt)로 호출


5. 결과 분리 → LLM_status3, LLM_reason 저장



 

 
