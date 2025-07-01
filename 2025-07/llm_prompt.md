

✅ 전제 조건: CSV 파일 구성

📁 회의록 요약 + Status CSV (meeting_df)

회의일 | EMD | 팀 | 과제명 | 회의 요약 | 액션 아이템 | LLM zero-shot status | 검토 status

📁 과제 발의서 CSV (proposal_df)

EMD | 팀 | 과제명 | 목적 | 배경 필요성 | 이전 지표 | 개선 목표 지표


---

✅ Python 자동화 코드 (pandas + 사내 LLM API 호출)
```
import pandas as pd

# CSV 파일 로드 (로컬 파일에 맞게 경로 수정 필요)
meeting_df = pd.read_csv('회의록_요약_status.csv')
proposal_df = pd.read_csv('과제발의서.csv')

# 날짜 정렬 및 이전 회의 요약/상태 연결
meeting_df['회의일'] = pd.to_datetime(meeting_df['회의일'])
meeting_df = meeting_df.sort_values(by=["과제명", "회의일"])
meeting_df['이전_요약'] = meeting_df.groupby('과제명')['회의 요약'].shift(1)
meeting_df['이전_상태'] = meeting_df.groupby('과제명')['검토 status'].shift(1)

# 발의서 정보 병합
merged_df = pd.merge(meeting_df, proposal_df, on=['EMD', '팀', '과제명'], how='left')

# 프롬프트 생성 함수
def generate_prompt(row):
    prompt = f"[과제명] {row['과제명']}\n[소속 팀] {row['팀']}\n"

    # 발의서 정보가 있을 경우
    if pd.notna(row['개선 목표 지표']):
        prompt += f"[과제 목적] {row.get('목적', '')}\n"
        prompt += f"[배경 및 필요성] {row.get('배경 필요성', '')}\n"
        prompt += f"[이전 지표] {row.get('이전 지표', '')}\n"
        prompt += f"[개선 목표 지표] {row.get('개선 목표 지표', '')}\n"

    # 이전 회차 요약 포함
    if pd.notna(row['이전_요약']):
        prompt += f"\n[이전 회차 요약 및 상태]\n- {row['이전_요약']} → {row['이전_상태']}\n"

    prompt += f"\n[현재 회의 요약] ({row['회의일'].date()})\n{row['회의 요약']}\n"

    if pd.notna(row.get('액션 아이템', '')):
        prompt += f"\n[액션 아이템]\n{row['액션 아이템']}\n"

    prompt += (
        "\n질문: 이 과제의 현재 상태를 아래 중 하나로 판단하고, 간단히 이유를 설명해주세요.\n"
        "→ ['on-track', 'at-risk', 'delayed', 'completed']"
    )
    return prompt

# 프롬프트 생성
merged_df['LLM_프롬프트'] = merged_df.apply(generate_prompt, axis=1)

# ✅ 사내 LLM 호출 예시
def call_llm(prompt: str) -> str:
    # 이 부분은 사내 API 구조에 맞게 수정 필요
    import requests
    response = requests.post(
        url="https://internal-api.company.local/llm/call",
        json={"prompt": prompt}
    )
    return response.json()["status_prediction"]

# LLM status3 예측 결과 저장
merged_df['LLM_status3'] = merged_df['LLM_프롬프트'].apply(call_llm)

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



 
